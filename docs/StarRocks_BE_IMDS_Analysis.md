# StarRocks BE 使用 IMDS 的完整分析

> 基于 StarRocks branch-3.5.12（AWS SDK for C++ 1.11.267）源码分析

---

## 1. 概述

StarRocks BE 通过 AWS SDK for C++ 间接使用 IMDS（EC2 Instance Metadata Service）。BE 自身 **没有** 直接调用 IMDS 的代码——所有 IMDS 交互都发生在 AWS SDK 内部，由 `EC2MetadataClient` 自动完成。

StarRocks BE 使用 IMDS 的场景可分为两大类：
1. **Region 自动检测**：AWS SDK `ClientConfiguration` 构造函数自动从 IMDS 获取 EC2 实例所在的 region
2. **EC2 实例凭证获取**：通过 `InstanceProfileCredentialsProvider` 从 IMDS 获取 IAM Role 临时凭证

---

## 2. StarRocks BE 中触发 IMDS 调用的所有入口点

### 2.1 入口点 1：`S3ClientFactory::getClientConfig()` — Region 自动检测（间接触发 IMDS）

**代码位置**: `be/src/fs/fs_s3.h:67-75`

```cpp
static ClientConfiguration& getClientConfig() {
    // We cached config here and make a deep copy each time.Since aws sdk has changed the
    // Aws::Client::ClientConfiguration default constructor to search for the region
    // (where as before 1.8 it has been hard coded default of "us-east-1").
    // Part of that change is looking through the ec2 metadata, which can take a long time.
    // For more details, please refer https://github.com/aws/aws-sdk-cpp/issues/1440
    static ClientConfiguration instance;
    return instance;
}
```

**分析**：

- 这是一个 **静态单例** `ClientConfiguration`，整个进程生命周期中只在**首次调用时构造一次**
- `Aws::Client::ClientConfiguration` 的默认构造函数会尝试自动检测 region，调用链为：
  ```
  ClientConfiguration() 
    → calculateRegion()     // 先检查 AWS_DEFAULT_REGION / AWS_REGION 环境变量和 ~/.aws/config
    → 若 region 仍为空且 IMDS 未禁用且 AWS_EC2_METADATA_DISABLED != "true"
      → EC2MetadataClient::GetCurrentRegion()
        → GetDefaultCredentialsSecurely()  // 先获取 IMDSv2 token
        → GET /latest/meta-data/placement/availability-zone
  ```
- **关键影响**：在非 EC2 环境中（无 IMDS 端点可达），首次构造 `ClientConfiguration` 时会阻塞等待 IMDS 超时，这正是 StarRocks 注释中提到的 [aws/aws-sdk-cpp#1440](https://github.com/aws/aws-sdk-cpp/issues/1440) 问题
- 使用静态单例缓存是为了确保这个可能很慢的 IMDS 调用**只发生一次**

**此入口被以下所有路径调用**：

| 调用路径 | 说明 |
|---------|------|
| `S3ClientFactory::new_client(TCloudConfiguration&, ...)` | 通过 `CloudConfiguration` 创建 S3 客户端 |
| `S3ClientFactory::new_client(ClientConfiguration&, FSOptions&)` | 直接使用配置创建 S3 客户端 |
| `new_s3client(S3URI&, FSOptions&, ...)` | 内部辅助函数，创建 S3 客户端 |
| `S3ClientFactory::_get_aws_credentials_provider()` 中的 STS 调用 | 当需要 AssumeRole 时获取 STS client config |

所有 S3 文件操作（读/写/列目录/删除/重命名等）最终都通过上述路径获取 S3 客户端。

### 2.2 入口点 2：`DefaultAWSCredentialsProviderChain` — 使用默认凭证链（间接触发 IMDS）

**代码位置**: `be/src/fs/fs_s3.cpp:86`

```cpp
if (aws_cloud_credential.use_aws_sdk_default_behavior) {
    credential_provider = std::make_shared<Aws::Auth::DefaultAWSCredentialsProviderChain>();
}
```

**触发条件**：用户设置 `aws.s3.use_aws_sdk_default_behavior = true`

**分析**：

- `DefaultAWSCredentialsProviderChain` 的构造函数会按顺序初始化凭证提供者链：
  1. `EnvironmentAWSCredentialsProvider` — 检查 `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` 环境变量
  2. `ProfileCredentialsProvider` — 检查 `~/.aws/credentials`
  3. `STSAssumeRoleWebIdentityCredentialsProvider` — Web Identity Token
  4. `SSOCredentialsProvider` — SSO 凭证
  5. `GeneralHTTPCredentialsProvider`（若 `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` 或 `AWS_CONTAINER_CREDENTIALS_FULL_URI` 已设置） — ECS/通用 HTTP 凭证
  6. **`InstanceProfileCredentialsProvider`**（若环境变量 `AWS_EC2_METADATA_DISABLED` **不** 为 `true`，且无 ECS 环境变量） — **这会触发 IMDS 调用**

- 当凭证链走到第 6 步 `InstanceProfileCredentialsProvider` 时，实际获取凭证的调用链为：
  ```
  InstanceProfileCredentialsProvider::GetAWSCredentials()
    → EC2InstanceProfileConfigLoader::LoadInternal()
      → EC2MetadataClient::GetDefaultCredentialsSecurely()
        → PUT /latest/api/token (IMDSv2 token 请求)
        → GET /latest/meta-data/iam/security-credentials (带 token)
        → GET /latest/meta-data/iam/security-credentials/{role} (带 token)
  ```
- **注意**：只有当前面的提供者都未能返回有效凭证时，才会触发 IMDS 调用

### 2.3 入口点 3：`InstanceProfileCredentialsProvider` — 显式使用 EC2 实例 Profile（直接触发 IMDS）

**代码位置**: `be/src/fs/fs_s3.cpp:87-88`

```cpp
} else if (aws_cloud_credential.use_instance_profile) {
    credential_provider = std::make_shared<Aws::Auth::InstanceProfileCredentialsProvider>();
}
```

**触发条件**：用户设置 `aws.s3.use_instance_profile = true`

**分析**：

- **这是最直接触发 IMDS 的路径**
- 直接创建 `InstanceProfileCredentialsProvider`，每次获取凭证时都会通过 IMDS 获取 EC2 IAM Role 的临时凭证
- 调用链同入口点 2 中的第 6 步
- **这里会使用 IMDSv2 优先，若 v2 token 获取失败则可能回退到 IMDSv1**

### 2.4 入口点 4：无凭证时 S3Client 内部默认行为（间接触发 IMDS）

**代码位置**: `be/src/fs/fs_s3.cpp:236-241`

```cpp
} else {
    // if not cred provided, we can use default cred in aws profile.
    client = std::make_shared<Aws::S3::S3Client>(config,
                                                 Aws::Client::AWSAuthV4Signer::PayloadSigningPolicy::Never,
                                                 !path_style_access);
}
```

**触发条件**：`new_client(ClientConfiguration&, FSOptions&)` 路径中，`access_key_id` 和 `secret_access_key` 都为空

**分析**：

- 当不提供 `CredentialsProvider` 给 `S3Client` 时，AWS SDK 内部会使用 `DefaultAWSCredentialsProviderChain`
- 效果同入口点 2，但这发生在旧的代码路径中（非 `TCloudConfiguration` 路径）

---

## 3. IMDS 使用方式总结：IMDSv1 vs IMDSv2

### 3.1 StarRocks BE 编译 AWS SDK 时 **未** 开启 `DISABLE_INTERNAL_IMDSV1_CALLS`

从 `thirdparty/build-thirdparty.sh` 中的 AWS SDK 编译选项可以确认：

```bash
$CMAKE_CMD -Bbuild -DBUILD_ONLY="core;s3;s3-crt;transfer;identity-management;sts;kms" \
           -DCMAKE_BUILD_TYPE=RelWithDebInfo \
           -DBUILD_SHARED_LIBS=OFF \
           ...
```

**没有** `-DDISABLE_INTERNAL_IMDSV1_CALLS=ON`，因此 IMDSv1 代码路径 **完整保留**。

### 3.2 StarRocks BE **未** 设置任何运行时 IMDS 禁用配置

在 StarRocks BE 代码中：
- **没有** 设置 `ClientConfiguration.disableIMDS`
- **没有** 设置 `ClientConfiguration.disableImdsV1`
- **没有** 设置 `CredentialProviderConfiguration.imdsConfig.disableImdsV1`

因此 **全部依赖 AWS SDK 的默认行为**。

### 3.3 使用 IMDSv2 的所有情况

| # | 场景 | 触发路径 | 说明 |
|---|------|---------|------|
| 1 | `getClientConfig()` 首次调用时自动检测 region | `ClientConfiguration()` → `GetCurrentRegion()` → `GetDefaultCredentialsSecurely()` | 获取 IMDSv2 token 后带 token 请求 `/latest/meta-data/placement/availability-zone` |
| 2 | `use_instance_profile = true` 时获取凭证 | `InstanceProfileCredentialsProvider` → `EC2InstanceProfileConfigLoader` → `GetDefaultCredentialsSecurely()` | PUT token → 带 token GET security-credentials |
| 3 | `use_aws_sdk_default_behavior = true` 且前面所有凭证提供者都失败 | `DefaultAWSCredentialsProviderChain` → `InstanceProfileCredentialsProvider` | 同上 |
| 4 | 无显式凭证时 S3Client 内部默认行为 | S3Client 内部 `DefaultAWSCredentialsProviderChain` | 同上 |
| 5 | `iam_role_arn` 非空时，STS AssumeRole 的基础凭证获取 | `_get_aws_credentials_provider()` → 创建 base credential → STS AssumeRole | 当基础凭证来自 instance_profile 或 default_behavior 时 |

### 3.4 回退使用 IMDSv1 的所有情况

由于 StarRocks **未禁用 IMDSv1**，在以下情况下 AWS SDK 会自动从 IMDSv2 回退到 IMDSv1：

| # | 场景 | 条件 | 说明 |
|---|------|------|------|
| 1 | IMDSv2 token PUT 请求失败（非 400） | token 请求返回 404/503 等或超时 | SDK 设置 `m_tokenRequired = false`，回退到直接 GET（不带 token） |
| 2 | IMDSv2 token 为空 | token 请求返回 200 但 body 为空 | 同上 |
| 3 | 后续请求复用 v1 模式 | 前一次已回退到 v1，`m_tokenRequired = false` | 直接使用不带 token 的 GET |
| 4 | region 检测请求 | `GetCurrentRegion()` 中 `m_tokenRequired == false` | GET `/latest/meta-data/placement/availability-zone` 不带 token |

**重要**：这意味着 StarRocks BE 在 EC2 实例上运行时，如果 EC2 实例的 IMDS 配置为 "optional"（同时支持 v1 和 v2），SDK 会先尝试 v2，失败后回退到 v1。如果 IMDS 被配置为 "required"（仅 v2），则 v2 token 获取不会失败，不会有回退。

### 3.5 完全不使用 IMDS 的情况

| # | 场景 | 说明 |
|---|------|------|
| 1 | 提供了显式 AK/SK | `!access_key.empty() && !secret_key.empty()`，使用 `SimpleAWSCredentialsProvider` |
| 2 | 环境变量 `AWS_EC2_METADATA_DISABLED=true` | `DefaultAWSCredentialsProviderChain` 不添加 `InstanceProfileCredentialsProvider` |
| 3 | Region 检测：环境变量已设定 region | `AWS_DEFAULT_REGION` 或 `AWS_REGION` 或 `~/.aws/config` 中有 region，不走 IMDS |
| 4 | 非 EC2 环境（IMDS 不可达） | 请求超时后返回空，凭证链继续尝试下一个提供者或失败 |

---

## 4. StarRocks BE 特有的 IMDS 交互特征

### 4.1 Poco HTTP Client 替代默认 CURL

StarRocks BE 默认使用 Poco HTTP Client 替代 AWS SDK 默认的 CURL HTTP Client（`enable_poco_client_for_aws_sdk = true`）。

**代码位置**: `starrocks_main.cpp:217-219`

```cpp
if (starrocks::config::enable_poco_client_for_aws_sdk) {
    Aws::Http::SetHttpClientFactory(std::make_shared<starrocks::poco::PocoHttpClientFactory>());
}
```

**影响**：IMDS 的所有 HTTP 请求（包括 IMDSv2 PUT token 和 GET metadata）都通过 Poco HTTP Client 而非 CURL 发出。这影响：
- 连接池行为（Poco 有自己的连接池 `HTTPSessionPools`）
- 超时处理方式
- TLS 处理
- 代理设置

**注意**：`SetHttpClientFactory` 在 `Aws::InitAPI()` 之后调用，意味着 `InitAPI` 内部可能使用的早期 HTTP 调用仍使用默认 client。但关键的 IMDS 调用发生在 `ClientConfiguration` 构造或 `CredentialsProvider::GetAWSCredentials()` 时，此时 Poco client 已经生效。

### 4.2 静态 ClientConfiguration 单例

如第 2.1 节所述，`getClientConfig()` 返回一个静态单例。首次调用时的 IMDS region 检测结果会被缓存，后续所有 S3 客户端创建都复用这个结果。

但是，StarRocks 通常通过 `TCloudConfiguration` 显式指定 region（`aws.s3.region`），此时：
```cpp
if (!aws_cloud_credential.region.empty()) {
    config.region = aws_cloud_credential.region;
}
```
即使 `getClientConfig()` 的默认 region 是 IMDS 检测来的，也会被覆盖。

### 4.3 AWS SDK 版本

StarRocks 3.5.12 使用 AWS SDK 1.11.267，这是一个相对较新的版本，默认行为：
- IMDSv2 优先
- `m_tokenRequired` 初始为 `true`
- 支持 IMDSv1 回退（除非编译或运行时禁用）

---

## 5. IMDS 调用时序图

```
StarRocks BE 启动
  │
  ├── Aws::InitAPI(options)
  │     └── 初始化 AWS SDK，此时不触发 IMDS
  │
  ├── SetHttpClientFactory(PocoHttpClientFactory)  // 若 enable_poco_client_for_aws_sdk=true
  │
  └── 首次 S3 操作请求到达
        │
        ├── S3ClientFactory::getClientConfig()
        │     └── static ClientConfiguration instance;  // 首次构造
        │           │
        │           ├── [检查 AWS_DEFAULT_REGION / AWS_REGION / ~/.aws/config]
        │           │     └── 若有 region → 不触发 IMDS
        │           │
        │           └── [region 为空且 IMDS 未禁用]
        │                 │
        │                 ├── InitEC2MetadataClient()
        │                 │     └── 创建 EC2MetadataClient(endpoint="http://169.254.169.254")
        │                 │
        │                 └── EC2MetadataClient::GetCurrentRegion()
        │                       │
        │                       └── GetDefaultCredentialsSecurely()  // 获取 token
        │                             │
        │                             ├── PUT http://169.254.169.254/latest/api/token
        │                             │   Header: x-aws-ec2-metadata-token-ttl-seconds: 21600
        │                             │
        │                             ├── [成功] → 保存 token (IMDSv2)
        │                             │     └── GET /latest/meta-data/placement/availability-zone
        │                             │         Header: x-aws-ec2-metadata-token: <token>
        │                             │
        │                             └── [失败且 v1 未禁用] → 回退 (IMDSv1)
        │                                   └── GET /latest/meta-data/placement/availability-zone
        │                                       (无 token header)
        │
        ├── _get_aws_credentials_provider(credential)
        │     │
        │     ├── [use_aws_sdk_default_behavior=true]
        │     │     └── DefaultAWSCredentialsProviderChain()
        │     │           └── (可能包含 InstanceProfileCredentialsProvider)
        │     │
        │     ├── [use_instance_profile=true]
        │     │     └── InstanceProfileCredentialsProvider()
        │     │           └── 首次 GetAWSCredentials() 时触发 IMDS
        │     │                 │
        │     │                 └── GetDefaultCredentialsSecurely()
        │     │                       ├── PUT /latest/api/token (IMDSv2)
        │     │                       ├── GET /latest/meta-data/iam/security-credentials (带 token)
        │     │                       └── GET /latest/meta-data/iam/security-credentials/{role} (带 token)
        │     │
        │     └── [access_key + secret_key 非空]
        │           └── SimpleAWSCredentialsProvider (不触发 IMDS)
        │
        └── 创建 S3Client 并执行 S3 操作
```

---

## 6. 配置建议

### 6.1 在非 EC2 环境中避免 IMDS 延迟

在非 EC2 环境中运行 StarRocks BE 时，首次 `ClientConfiguration` 构造会因等待 IMDS 超时而产生延迟（默认 ~1 秒连接超时 × 重试）。建议：

1. **设置环境变量** `AWS_DEFAULT_REGION` 或 `AWS_REGION`，避免 IMDS region 检测
2. **设置环境变量** `AWS_EC2_METADATA_DISABLED=true`，完全禁用 IMDS
3. **显式提供 AK/SK**，避免凭证链走到 InstanceProfileCredentialsProvider

### 6.2 在 EC2 环境中的安全建议

如果 StarRocks BE 运行在 EC2 上并使用 IMDS 获取凭证：

1. **将 EC2 实例的 IMDS 配置为 "IMDSv2 required"**，避免 IMDSv1 的安全风险
2. 由于 StarRocks **未** 在编译时或运行时禁用 IMDSv1，SDK 会在 v2 失败时回退到 v1

### 6.3 StarRocks BE 无法控制的 IMDS 行为

StarRocks BE **没有** 暴露以下 AWS SDK 配置给用户：
- `ClientConfiguration.disableIMDS`
- `ClientConfiguration.disableImdsV1`
- `CredentialProviderConfiguration.imdsConfig.*`

因此用户只能通过以下方式间接控制 IMDS 行为：
- 环境变量 `AWS_EC2_METADATA_DISABLED=true`（禁用 IMDS）
- 环境变量 `ec2_metadata_v1_disabled=true`（禁用 IMDSv1）
- 配置文件 `~/.aws/config` 中设置 `AWS_EC2_METADATA_V1_DISABLED=true`
- 环境变量 `AWS_DEFAULT_REGION` / `AWS_REGION`（避免 region 检测触发 IMDS）

---

## 7. 涉及的所有源文件

### StarRocks BE 源码

| 文件 | 角色 |
|------|------|
| `be/src/fs/fs_s3.cpp` | S3 客户端工厂、凭证提供者选择、S3 文件系统操作 |
| `be/src/fs/fs_s3.h` | `S3ClientFactory` 声明，**`getClientConfig()` 静态单例** |
| `be/src/fs/credential/cloud_configuration.h` | `AWSCloudCredential` 定义（`use_instance_profile` 等字段） |
| `be/src/fs/credential/cloud_configuration_factory.cpp` | 从 `TCloudConfiguration` 解析 AWS 配置 |
| `be/src/fs/credential/cloud_configuration_factory.h` | 配置 key 常量定义 |
| `be/src/service/starrocks_main.cpp` | `Aws::InitAPI()`、Poco HTTP Client 设置 |
| `be/src/fs/s3/poco_http_client.cpp` | Poco HTTP Client 实现（IMDS 请求通过此发出） |
| `be/src/fs/s3/poco_http_client_factory.cpp` | Poco HTTP Client 工厂 |
| `be/src/common/config.h` | BE 配置项定义 |
| `thirdparty/build-thirdparty.sh` | AWS SDK 编译选项（**未启用 DISABLE_INTERNAL_IMDSV1_CALLS**） |

### AWS SDK 源码（被 StarRocks 链接）

| 文件 | 角色 |
|------|------|
| `source/internal/AWSHttpResourceClient.cpp` | IMDS v1/v2 核心实现 |
| `source/client/ClientConfiguration.cpp` | Region 自动检测、IMDS 配置加载 |
| `source/auth/AWSCredentialsProvider.cpp` | `InstanceProfileCredentialsProvider` |
| `source/auth/AWSCredentialsProviderChain.cpp` | 默认凭证链 |
| `source/config/EC2InstanceProfileConfigLoader.cpp` | 从 IMDS 加载凭证 |
