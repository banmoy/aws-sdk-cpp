# StarRocks 中 AWS SDK 的使用方式与 IMDS 交互分析

本文档基于 StarRocks 源码（分支 `cursor/slack-claude-cf4a`），分析 StarRocks 如何使用 AWS C++ SDK 和 Java SDK，以及在哪些场景下会触发 IMDSv1 / IMDSv2 调用。

---

## 一、整体架构

StarRocks 中有 **两个组件** 与 AWS 交互，分别使用不同语言的 SDK：

| 组件 | 语言 | SDK | 版本 | 用途 |
|------|------|-----|------|------|
| **BE** (Backend) | C++ | AWS C++ SDK | 1.11.267 | S3 数据读写、凭证获取 |
| **FE** (Frontend) | Java | AWS Java SDK v2 | 2.29.52 | S3 文件列举（通过 Hadoop S3A） |
| **FE** | Java | Hadoop (hadoop-aws) | 3.4.1 | S3A 文件系统接口 |

---

## 二、BE 侧（C++）—— AWS SDK 使用详解

### 2.1 SDK 初始化

**文件**: `be/src/service/starrocks_main.cpp:204-219`

```cpp
Aws::SDKOptions aws_sdk_options;
aws_sdk_options.httpOptions.initAndCleanupCurl = false;
// 可选: 日志和 RFC3986 编码配置
Aws::InitAPI(aws_sdk_options);
// 可选: 使用 Poco HTTP 客户端替代 Curl
if (config::enable_poco_client_for_aws_sdk) {
    Aws::Http::SetHttpClientFactory(std::make_shared<PocoHttpClientFactory>());
}
```

**注意**：`Aws::SDKOptions` 中没有任何与 IMDS 相关的配置。

### 2.2 SDK 构建配置

**文件**: `thirdparty/build-thirdparty.sh:1015-1034`

```bash
$CMAKE_CMD -Bbuild -DBUILD_ONLY="core;s3;s3-crt;transfer;identity-management;sts;kms" \
           -DCMAKE_BUILD_TYPE=RelWithDebInfo \
           -DBUILD_SHARED_LIBS=OFF \
           -DENABLE_TESTING=OFF \
           -DENABLE_CURL_LOGGING=OFF \
           ...
```

**关键发现**：
- **未设置** `DISABLE_INTERNAL_IMDSV1_CALLS`（默认 `OFF`）
- 因此 `DISABLE_IMDSV1` 预处理宏 **未定义**，IMDSv1 回退代码完整保留在编译产物中
- 仅有两个补丁：静默 unzip 输出 和 禁用 chunked upload，均与 IMDS 无关

### 2.3 ClientConfiguration 静态实例

**文件**: `be/src/fs/fs_s3.h:67-75`

```cpp
static ClientConfiguration& getClientConfig() {
    static ClientConfiguration instance;  // ← 首次调用时触发默认构造函数
    return instance;
}
```

`ClientConfiguration` 默认构造函数会：
1. 检查环境变量 `AWS_DEFAULT_REGION`、`AWS_REGION`
2. 检查 AWS 配置文件中的 `region`
3. 若以上均为空，且 `AWS_EC2_METADATA_DISABLED != "true"`，调用 `EC2MetadataClient::GetCurrentRegion()` 查询 IMDS 获取区域
4. 若仍为空，默认使用 `us-east-1`

**这是 BE 中第一个 IMDS 触发点**（首次构建 S3 客户端时）。

**关键**：此静态 `ClientConfiguration` 的 `disableImdsV1` 字段保持默认值 `false`，**未被 StarRocks 修改**。

### 2.4 凭证提供者选择

**文件**: `be/src/fs/fs_s3.cpp:80-112`

```cpp
std::shared_ptr<Aws::Auth::AWSCredentialsProvider> S3ClientFactory::_get_aws_credentials_provider(
        const AWSCloudCredential& aws_cloud_credential) {
    if (aws_cloud_credential.use_aws_sdk_default_behavior) {
        // 路径 A: 使用 SDK 默认凭证链（包含 InstanceProfileCredentialsProvider）
        credential_provider = std::make_shared<Aws::Auth::DefaultAWSCredentialsProviderChain>();
    } else if (aws_cloud_credential.use_instance_profile) {
        // 路径 B: 直接使用实例配置文件凭证
        credential_provider = std::make_shared<Aws::Auth::InstanceProfileCredentialsProvider>();
    } else if (!aws_cloud_credential.access_key.empty() && ...) {
        // 路径 C: 静态 AK/SK，不涉及 IMDS
        credential_provider = std::make_shared<Aws::Auth::SimpleAWSCredentialsProvider>(...);
    }
    // 可选: Assume Role
    if (!aws_cloud_credential.iam_role_arn.empty()) {
        auto sts = std::make_shared<Aws::STS::STSClient>(credential_provider, clientConfiguration);
        credential_provider = std::make_shared<Aws::Auth::STSAssumeRoleCredentialsProvider>(...);
    }
    return credential_provider;
}
```

### 2.5 S3 客户端创建

BE 中有 **两条** S3 客户端创建路径：

#### 路径 1：通过 `TCloudConfiguration`（新路径）

**文件**: `be/src/fs/fs_s3.cpp:127-191`（`new_client(const TCloudConfiguration&, ...)`）

- 从 `TCloudConfiguration` 解析出 `AWSCloudCredential`
- 调用 `_get_aws_credentials_provider()` 获取凭证提供者
- 构造 `Aws::S3::S3Client`

#### 路径 2：通过 `FSOptions` / `THdfsProperties`（旧路径/兼容路径）

**文件**: `be/src/fs/fs_s3.cpp:193-254`（`new_client(const ClientConfiguration&, const FSOptions&)`）和 `be/src/fs/fs_s3.cpp:269-356`（`new_s3client()`）

- 从 `THdfsProperties` 或 `FSOptions` 获取 AK/SK
- 若有 AK/SK → `SimpleAWSCredentialsProvider`（不涉及 IMDS）
- 若无 AK/SK → `S3Client` 默认构造（内部使用 `DefaultAWSCredentialsProviderChain`，可能涉及 IMDS）

### 2.6 环境变量

**文件**: `bin/common.sh:94`

```bash
export AWS_EC2_METADATA_DISABLED=${AWS_EC2_METADATA_DISABLED:-false}
```

默认值为 `false`，即 **不禁用** IMDS。此变量控制的是是否完全禁用 EC2 metadata 访问（v1 和 v2 均禁用）。

---

## 三、FE 侧（Java）—— AWS SDK 使用详解

### 3.1 凭证提供者选择

**文件**: `fe/fe-core/src/main/java/com/starrocks/credential/aws/AwsCloudCredential.java:191-210`

```java
private AwsCredentialsProvider getBaseAWSCredentialsProvider(...) {
    if (useAWSSDKDefaultBehavior) {
        return DefaultCredentialsProvider.builder().build();       // 路径 A
    } else if (useInstanceProfile) {
        return InstanceProfileCredentialsProvider.builder().build(); // 路径 B
    } else if (!accessKey.isEmpty() && !secretKey.isEmpty()) {
        return StaticCredentialsProvider.create(...);              // 路径 C（不涉及 IMDS）
    } else {
        return DefaultCredentialsProvider.builder().build();       // 路径 D（兜底）
    }
}
```

### 3.2 Hadoop S3A 集成

**文件**: `fe/fe-core/src/main/java/com/starrocks/credential/aws/AwsCloudCredential.java:233-268`

FE 通过设置 `fs.s3a.aws.credentials.provider` 属性来控制 Hadoop S3A 使用哪个凭证提供者：
- `useAWSSDKDefaultBehavior=true` → `OverwriteAwsDefaultCredentialsProvider`
- `useInstanceProfile=true` → `IAMInstanceCredentialsProvider`
- AK/SK → `SimpleAWSCredentialsProvider`

### 3.3 Iceberg AWS 客户端工厂

**文件**: `java-extensions/hadoop-ext/.../IcebergAwsClientFactory.java:259-275`

```java
if (s3UseAWSSDKDefaultBehavior || glueUseAWSSDKDefaultBehavior) {
    credentialsProvider = DefaultCredentialsProvider.builder().build();
} else if (s3UseInstanceProfile || glueUseInstanceProfile) {
    credentialsProvider = InstanceProfileCredentialsProvider.builder().build();
}
```

---

## 四、所有 IMDS 触发场景完整列表

### 4.1 BE 侧 IMDS 触发点

| # | 触发点 | 代码位置 | 触发条件 | 使用的 SDK 类 | IMDS 版本 |
|---|--------|---------|---------|--------------|----------|
| 1 | `getClientConfig()` 静态初始化 | `fs_s3.h:67-75` | 首次创建 S3 客户端时（且未设置 `AWS_DEFAULT_REGION` / `AWS_REGION` 环境变量，且配置文件无 region，且 `AWS_EC2_METADATA_DISABLED` != `true`） | `ClientConfiguration` 默认构造 → `EC2MetadataClient::GetCurrentRegion()` | 先尝试 IMDSv2，可能回退 v1 |
| 2 | `DefaultAWSCredentialsProviderChain` | `fs_s3.cpp:86` | `use_aws_sdk_default_behavior=true` | `DefaultAWSCredentialsProviderChain` → `InstanceProfileCredentialsProvider` → `EC2MetadataClient` | 先尝试 IMDSv2，可能回退 v1 |
| 3 | `InstanceProfileCredentialsProvider` | `fs_s3.cpp:88` | `use_instance_profile=true` | `InstanceProfileCredentialsProvider` → `EC2InstanceProfileConfigLoader` → `EC2MetadataClient::GetDefaultCredentialsSecurely()` | 先尝试 IMDSv2，可能回退 v1 |
| 4 | `S3Client` 默认构造（旧路径无 AK/SK） | `fs_s3.cpp:237-241` | `FSOptions` 或 `THdfsProperties` 未提供 AK/SK 时 | `S3Client` 内部使用 `DefaultAWSCredentialsProviderChain` | 先尝试 IMDSv2，可能回退 v1 |
| 5 | Smart Defaults AUTO 模式区域发现 | `ClientConfigurationDefaults.cpp:152-159` | `AWS_DEFAULTS_MODE=auto` 时 | `EC2MetadataClient::GetCurrentRegion()` | 先尝试 IMDSv2，可能回退 v1 |

### 4.2 FE 侧 IMDS 触发点

| # | 触发点 | 代码位置 | 触发条件 | 使用的 SDK 类 | IMDS 版本 |
|---|--------|---------|---------|--------------|----------|
| 6 | Hadoop S3A `IAMInstanceCredentialsProvider` | `AwsCloudCredential.java:87-88` | `use_instance_profile=true` | AWS Java SDK v2 `InstanceProfileCredentialsProvider` | IMDSv2（Java SDK v2 默认行为） |
| 7 | Hadoop S3A `OverwriteAwsDefaultCredentialsProvider` | `AwsCloudCredential.java:86` | `use_aws_sdk_default_behavior=true` | AWS Java SDK v2 `DefaultCredentialsProvider` | IMDSv2（Java SDK v2 默认行为） |
| 8 | `generateAWSCredentialsProvider()` for AWS API 直接调用 | `AwsCloudCredential.java:154-161` | 直接使用 AWS SDK v2 API（非 Hadoop）时 | `InstanceProfileCredentialsProvider` 或 `DefaultCredentialsProvider` | IMDSv2（Java SDK v2 默认行为） |
| 9 | Iceberg AWS 客户端工厂 | `IcebergAwsClientFactory.java:259-275` | `s3UseInstanceProfile=true` 或 `glueUseInstanceProfile=true` 或 `use_aws_sdk_default_behavior=true` | AWS Java SDK v2 `InstanceProfileCredentialsProvider` / `DefaultCredentialsProvider` | IMDSv2 |

### 4.3 完全不触发 IMDS 的场景

| # | 场景 | 条件 |
|---|------|------|
| 1 | 使用静态 AK/SK | 配置了 `aws.s3.access_key` 和 `aws.s3.secret_key` |
| 2 | 使用静态临时凭证 | AK/SK + session_token |
| 3 | `AWS_EC2_METADATA_DISABLED=true` | 环境变量禁用所有 IMDS |
| 4 | region 已通过其他方式配置 | 环境变量或配置文件已指定 region，跳过 IMDS region 查询 |

---

## 五、IMDSv1 回退问题分析

### 5.1 问题根因

BE 侧使用的 AWS C++ SDK 1.11.267 中，`EC2MetadataClient` 有一个 `m_tokenRequired` 状态标志：

1. **初始值**: `true`（首次尝试 IMDSv2）
2. **IMDSv2 PUT 请求失败** → 设为 `false` → **永久回退到 IMDSv1**
3. `EC2MetadataClient` 的默认超时仅 **1 秒**，重试 **1 次**
4. 若 BE 启动时 IMDS 响应慢（限流等），PUT 超时 → 永久 IMDSv1

### 5.2 为何 StarRocks 受影响

| 检查项 | 状态 | 影响 |
|--------|------|------|
| 编译时 `DISABLE_INTERNAL_IMDSV1_CALLS` | **未设置** | IMDSv1 回退代码存在于二进制中 |
| 运行时 `ClientConfiguration.disableImdsV1` | **未设置**（默认 `false`） | 运行时允许 IMDSv1 回退 |
| 环境变量 `AWS_EC2_METADATA_V1_DISABLED` | **未设置** | 运行时允许 IMDSv1 回退 |
| `InstanceProfileCredentialsProvider` 构造 | **默认构造函数** | 未传入任何禁用 v1 的配置 |
| `getClientConfig()` 静态初始化 | **默认构造** | 未设置 `disableImdsV1` |

### 5.3 FE 侧不受影响

FE 使用 AWS Java SDK v2 (2.29.52)，其 `InstanceProfileCredentialsProvider` 实现与 C++ SDK 不同：
- Java SDK v2 默认使用 IMDSv2
- 即使 token 获取失败，Java SDK 的行为更健壮
- 无 C++ SDK 那样的"永久缓存回退"问题

---

## 六、关键代码路径 — 调用链总览

```
Broker Load SQL (use_instance_profile=true)
  │
  ├──── FE（Pending 阶段，列举文件）──────────────────────────────
  │     BrokerLoadPendingTask.executeTask()
  │       → HdfsFsManager.getFileSystem()
  │         → getFileSystemByCloudConfiguration()
  │           → AwsCloudCredential.applyToConfiguration()
  │             → "fs.s3a.aws.credentials.provider" = "IAMInstanceCredentialsProvider"
  │           → FileSystem.get() → S3AFileSystem
  │             → AWS Java SDK v2 InstanceProfileCredentialsProvider
  │               → IMDS (IMDSv2) ✅
  │
  ├──── BE（Loading 阶段，读取数据）──────────────────────────────
  │     file_scanner.cpp (use_broker=false)
  │       → new_fs_s3() → S3FileSystem → new_s3client()
  │         ├── S3ClientFactory::getClientConfig()
  │         │   └── static ClientConfiguration instance
  │         │       └── EC2MetadataClient::GetCurrentRegion()        ← 触发点 1
  │         │           → IMDSv2 PUT /latest/api/token
  │         │             ├── 成功 → IMDSv2 GET (带 token)
  │         │             └── 失败（1s 超时）→ m_tokenRequired=false
  │         │                 → IMDSv1 GET (无 token) ← ⚠ 永久回退
  │         │
  │         ├── S3ClientFactory::new_client(tCloudConfiguration)
  │         │   → _get_aws_credentials_provider()
  │         │     → InstanceProfileCredentialsProvider()             ← 触发点 2
  │         │       → EC2InstanceProfileConfigLoader::LoadInternal()
  │         │         → EC2MetadataClient::GetDefaultCredentialsSecurely()
  │         │           ├── m_tokenRequired==true → IMDSv2 ✅
  │         │           └── m_tokenRequired==false → IMDSv1 ⚠
  │         │
  │         └── S3Client 发起 GetObject 请求
```

---

## 七、`EC2MetadataClient` 实例共享分析

### 7.1 全局单例

`EC2MetadataClient` 通过 `InitEC2MetadataClient()` / `GetEC2MetadataClient()` 管理为全局单例（`s_ec2metadataClient`）。

### 7.2 `InstanceProfileCredentialsProvider` 内部实例

`InstanceProfileCredentialsProvider` 默认构造函数内部创建独立的 `EC2InstanceProfileConfigLoader`，后者在构造时如果传入 `nullptr`，会调用 `InitEC2MetadataClient()` / `GetEC2MetadataClient()` 获取全局单例。

### 7.3 影响

由于使用全局单例，**一旦首次 IMDSv2 PUT 失败导致 `m_tokenRequired=false`**：
- 整个 BE 进程内所有后续 IMDS 调用都走 IMDSv1
- 包括 `getClientConfig()` 的区域查询
- 包括所有 `InstanceProfileCredentialsProvider` 的凭证获取
- **直到 BE 进程重启**

---

## 八、所有涉及 AWS SDK 的源文件索引

### BE 侧（C++）

| 文件 | 职责 |
|------|------|
| `be/src/service/starrocks_main.cpp:204-219` | `Aws::InitAPI()` / `Aws::ShutdownAPI()` |
| `be/src/fs/fs_s3.h` | `S3ClientFactory` 声明、`getClientConfig()` 静态实例 |
| `be/src/fs/fs_s3.cpp` | `_get_aws_credentials_provider()`、`new_client()`、`new_s3client()`、`S3FileSystem` |
| `be/src/fs/credential/cloud_configuration.h` | `AWSCloudCredential` 结构体定义 |
| `be/src/fs/credential/cloud_configuration_factory.cpp` | `TCloudConfiguration` → `AWSCloudCredential` 转换 |
| `be/src/io/s3_input_stream.cpp` | S3 读取流 |
| `be/src/io/s3_output_stream.cpp` | S3 写入流（multipart upload） |
| `be/src/io/direct_s3_output_stream.cpp` | S3 直接写入流 |
| `be/src/fs/s3/poco_http_client.cpp` | 自定义 Poco HTTP 客户端 |
| `be/src/fs/s3/poco_http_client_factory.cpp` | Poco HTTP 客户端工厂 |
| `be/src/starrocks_format/starrocks_lib.cpp` | StarRocks format 库初始化（也调用 `Aws::InitAPI`） |
| `be/src/common/s3_uri.cpp` | S3 URI 解析 |
| `bin/common.sh:94` | `AWS_EC2_METADATA_DISABLED` 环境变量默认值 |

### FE 侧（Java）

| 文件 | 职责 |
|------|------|
| `fe/.../credential/aws/AwsCloudCredential.java` | AWS 凭证提供者选择、Hadoop 配置 |
| `fe/.../credential/aws/AwsCloudConfigurationProvider.java` | 解析 `aws.s3.*` 属性 |
| `fe/.../credential/provider/OverwriteAwsDefaultCredentialsProvider.java` | 自定义 DefaultCredentialsProvider 包装 |
| `fe/.../credential/provider/AssumedRoleCredentialProvider.java` | STS AssumeRole 凭证提供者 |
| `fe/.../connector/hive/glue/metastore/AWSGlueClientFactory.java` | Glue 客户端创建 |
| `java-extensions/.../IcebergAwsClientFactory.java` | Iceberg S3/Glue 客户端创建 |

### 构建配置

| 文件 | 职责 |
|------|------|
| `thirdparty/vars.sh:322-326` | AWS C++ SDK 版本（1.11.267） |
| `thirdparty/build-thirdparty.sh:1015-1034` | AWS C++ SDK 构建参数 |
| `thirdparty/download-thirdparty.sh:423-438` | AWS SDK 补丁应用 |
| `be/CMakeLists.txt:249-256` | `find_package(AWSSDK)` 链接配置 |

---

## 九、总结

### StarRocks 使用 AWS SDK 的模式

1. **BE 使用 AWS C++ SDK 1.11.267**，通过 `S3ClientFactory` 管理 S3 客户端生命周期
2. **FE 使用 AWS Java SDK v2 2.29.52**，通过 Hadoop S3A 或直接 SDK API 访问 S3
3. 凭证获取通过 3 种策略：`DefaultCredentialsProviderChain`、`InstanceProfileCredentialsProvider`、`SimpleAWSCredentialsProvider`
4. 前两种策略在 EC2 环境中会触发 IMDS 调用

### IMDS 版本使用现状

| 组件 | IMDS 使用 | 问题 |
|------|----------|------|
| **BE** | IMDSv2 优先，可能永久回退到 IMDSv1 | `m_tokenRequired=false` 永久缓存；`disableImdsV1` 未设置；`DISABLE_IMDSV1` 未编译 |
| **FE** | IMDSv2 | Java SDK v2 默认行为，无回退问题 |

### 防止 IMDSv1 回退的措施（StarRocks 当前均未采取）

| 措施 | 级别 | StarRocks 是否采取 |
|------|------|-------------------|
| 编译时 `DISABLE_INTERNAL_IMDSV1_CALLS=ON` | 编译 | ❌ 未设置 |
| `ClientConfiguration.disableImdsV1 = true` | 运行时代码 | ❌ 未设置 |
| 环境变量 `AWS_EC2_METADATA_V1_DISABLED=true` | 运行时环境 | ❌ 未设置 |
| EC2 实例 `--http-tokens required` | 基础设施 | 不在 StarRocks 控制范围 |
