# IMDSv1 与 IMDSv2 使用场景完整分析

> 本文档基于 AWS SDK for C++ 源码，全面分析 IMDSv1 和 IMDSv2 在所有情况下的使用逻辑。

---

## 1. 概述

AWS EC2 Instance Metadata Service (IMDS) 有两个版本：

| 版本 | 协议特征 | 安全性 |
|------|---------|--------|
| **IMDSv1** | 直接 HTTP GET 请求访问元数据端点，无需 token | 较低（易受 SSRF 攻击） |
| **IMDSv2** | 先通过 HTTP PUT 获取 session token，后续请求携带 token | 较高（基于 token 的会话认证） |

核心入口点为 `EC2MetadataClient` 类（定义于 `src/aws-cpp-sdk-core/include/aws/core/internal/AWSHttpResourceClient.h`），它提供了三个主要的公开方法来访问 IMDS：

- `GetDefaultCredentialsSecurely()` — 安全方式获取凭证（IMDSv2 优先）
- `GetDefaultCredentials()` — 不安全方式获取凭证（IMDSv1，仅在未禁用时存在）
- `GetCurrentRegion()` — 获取当前 EC2 实例所在 region

---

## 2. 控制 IMDSv1/v2 行为的所有配置维度

### 2.1 编译时配置（Build-time）

| 控制项 | 来源 | 效果 |
|--------|------|------|
| CMake 选项 `DISABLE_INTERNAL_IMDSV1_CALLS=ON` | `CMakeLists.txt:62` | 定义宏 `DISABLE_IMDSV1`，从编译层面彻底移除所有 IMDSv1 代码路径（`#if !defined(DISABLE_IMDSV1)` 守卫），`GetDefaultCredentials()` 方法将不存在 |

### 2.2 运行时配置（Runtime）

#### 2.2.1 完全禁用 IMDS（v1 + v2 都不使用）

| 控制项 | 来源 | 对应代码变量 |
|--------|------|-------------|
| `ClientConfiguration.disableIMDS = true` | 代码设置 | `EC2MetadataClient::m_disableIMDS` |
| `ClientConfigurationInitValues.shouldDisableIMDS = true` | 代码设置 | 传递到 `disableIMDS` |
| `CredentialProviderConfiguration.imdsConfig.disableImds = true` | 代码设置 | `EC2MetadataClient::m_disableIMDS` |
| 环境变量 `AWS_EC2_METADATA_DISABLED=true` | 环境变量 | 在 `DefaultAWSCredentialsProviderChain` 和 `ClientConfiguration` 构造函数中检查，跳过 EC2 metadata provider |

#### 2.2.2 仅禁用 IMDSv1（强制仅使用 IMDSv2）

| 控制项 | 来源 | 对应代码变量 |
|--------|------|-------------|
| `ClientConfiguration.disableImdsV1 = true` | 代码设置 | `EC2MetadataClient::m_disableIMDSV1` |
| `CredentialProviderConfiguration.imdsConfig.disableImdsV1 = true` | 代码设置 | `EC2MetadataClient::m_disableIMDSV1` |
| 环境变量 `ec2_metadata_v1_disabled=true` | 环境变量 | 通过 `setConfigFromEnvOrProfile()` 设置到 `disableImdsV1` |
| 配置文件 `~/.aws/config` 中 `AWS_EC2_METADATA_V1_DISABLED=true` | AWS 配置文件 | 通过 `LoadConfigFromEnvOrProfile()` 读取 |

---

## 3. 所有使用 IMDSv2 的情况（完整列表）

IMDSv2 的核心逻辑在 `GetDefaultCredentialsSecurely()` 方法中。以下是 **所有** 触发 IMDSv2 的情况：

### 情况 1：默认首选路径 — `m_tokenRequired == true`（初始状态）

- **触发条件**: `EC2MetadataClient` 构造时 `m_tokenRequired` 初始化为 `true`
- **行为**: `GetDefaultCredentialsSecurely()` 被调用时：
  1. 发送 `HTTP PUT` 到 `/latest/api/token`，携带 header `x-aws-ec2-metadata-token-ttl-seconds: 21600`
  2. 获取 token 成功后，后续对 `/latest/meta-data/iam/security-credentials` 和 `/latest/meta-data/iam/security-credentials/{profile}` 的 GET 请求都携带 header `x-aws-ec2-metadata-token: <token>`
- **代码位置**: `AWSHttpResourceClient.cpp:301-366`

### 情况 2：从 IMDSv1 回退到 IMDSv2 — 收到 HTTP 401 Unauthorized

- **触发条件**: 使用 IMDSv1 (`GetDefaultCredentials()`) 访问元数据时，服务器返回 `HTTP 401 UNAUTHORIZED`
- **行为**: 设置 `m_tokenRequired = true`，下次调用时切换到 IMDSv2 路径
- **代码位置**: `AWSHttpResourceClient.cpp:273-277`

### 情况 3：编译时强制 — `DISABLE_IMDSV1` 宏定义

- **触发条件**: CMake 编译选项 `DISABLE_INTERNAL_IMDSV1_CALLS=ON`
- **行为**: `m_disableIMDSV1` 被强制设为 `true`，所有 IMDSv1 回退代码被编译移除，**只能** 使用 IMDSv2
- **代码位置**: `AWSHttpResourceClient.cpp:209-213`, `AWSHttpResourceClient.cpp:232-235`

### 情况 4：运行时配置禁用 IMDSv1 — `disableImdsV1 = true`

- **触发条件**: 通过以下任一方式设置：
  - `ClientConfiguration.disableImdsV1 = true`
  - `CredentialProviderConfiguration.imdsConfig.disableImdsV1 = true`
  - 环境变量 `ec2_metadata_v1_disabled=true`
  - AWS 配置文件 `AWS_EC2_METADATA_V1_DISABLED=true`
- **行为**: `GetDefaultCredentialsSecurely()` 中的 IMDSv1 回退逻辑被跳过，token 获取失败时不会降级到 v1，而是继续重试 IMDSv2
- **代码位置**: `AWSHttpResourceClient.cpp:309-312`, `AWSHttpResourceClient.cpp:330-335`

### 情况 5：获取 Region 时使用 IMDSv2

- **触发条件**: 调用 `GetCurrentRegion()` 且 `m_tokenRequired == true`
- **行为**: 先通过 `GetDefaultCredentialsSecurely()` 获取 token，然后对 `/latest/meta-data/placement/availability-zone` 的 GET 请求携带 `x-aws-ec2-metadata-token` header
- **代码位置**: `AWSHttpResourceClient.cpp:368-428`

### 情况 6：EC2InstanceProfileConfigLoader 加载凭证

- **触发条件**: `EC2InstanceProfileConfigLoader::LoadInternal()` 调用 `m_ec2metadataClient->GetDefaultCredentialsSecurely()`
- **行为**: 始终通过安全方式（IMDSv2 优先）获取凭证
- **代码位置**: `EC2InstanceProfileConfigLoader.cpp:62`

### 情况 7：ClientConfiguration 构造函数获取 Region

- **触发条件**: `ClientConfiguration` 的 4 个构造函数中，当 region 为空且 IMDS 未被禁用且 `AWS_EC2_METADATA_DISABLED != true` 时
- **行为**: 通过 `EC2MetadataClient::GetCurrentRegion()` 获取 region（走 IMDSv2 路径）
- **代码位置**: `ClientConfiguration.cpp:394-404`, `ClientConfiguration.cpp:421-431`, `ClientConfiguration.cpp:453-464`, `ClientConfiguration.cpp:501-513`

---

## 4. 所有使用 IMDSv1 的情况（完整列表）

IMDSv1 的核心逻辑在 `GetDefaultCredentials()` 方法中。以下是 **所有** 触发 IMDSv1 的情况：

### 前提条件

IMDSv1 要生效，必须同时满足：
- 编译时 **未** 定义 `DISABLE_IMDSV1` 宏（即 `DISABLE_INTERNAL_IMDSV1_CALLS=OFF`，默认值）
- 运行时 `m_disableIMDSV1 == false`（即未通过配置/环境变量/代码禁用 v1）
- `m_disableIMDS == false`（IMDS 未被完全禁用）

### 情况 1：IMDSv2 token 获取失败后回退

- **触发条件**: `GetDefaultCredentialsSecurely()` 中，PUT `/latest/api/token` 返回的 HTTP 状态码 **不是** OK 且 **不是** BAD_REQUEST（例如 404 Not Found、503 Service Unavailable 等），且 `m_disableIMDSV1 == false`
- **行为**: 设置 `m_tokenRequired = false`，回退到 `GetDefaultCredentials()`，直接用无 token 的 GET 请求访问元数据
- **代码位置**: `AWSHttpResourceClient.cpp:329-335`

### 情况 2：IMDSv2 token 获取返回空 token 后回退

- **触发条件**: PUT `/latest/api/token` 返回 HTTP 200 OK 但响应体为空（`trimmedTokenString.empty()`），且 `m_disableIMDSV1 == false`
- **行为**: 设置 `m_tokenRequired = false`，回退到 `GetDefaultCredentials()`
- **代码位置**: `AWSHttpResourceClient.cpp:330-335`

### 情况 3：已经处于 IMDSv1 模式（`m_tokenRequired == false`）

- **触发条件**: 前一次 IMDSv2 token 获取失败后已将 `m_tokenRequired` 设为 `false`，后续调用时检测到该状态
- **行为**:
  - `GetDefaultCredentialsSecurely()` → 检测 `!m_tokenRequired` → 调用 `GetDefaultCredentials()`
  - `GetDefaultCredentials()` → 检测 `!m_tokenRequired` → 直接 GET `/latest/meta-data/iam/security-credentials`（不带 token）→ 获取 profile → GET `/latest/meta-data/iam/security-credentials/{profile}`（不带 token）
- **代码位置**: `AWSHttpResourceClient.cpp:308-312`（从 Securely 跳转到 v1）和 `AWSHttpResourceClient.cpp:260-298`（v1 完整流程）

### 情况 4：获取 Region 时使用 IMDSv1

- **触发条件**: `GetCurrentRegion()` 中 `m_tokenRequired == false`
- **行为**: 对 `/latest/meta-data/placement/availability-zone` 的 GET 请求 **不** 携带 `x-aws-ec2-metadata-token` header
- **代码位置**: `AWSHttpResourceClient.cpp:386-392`（条件判断：只有 `m_tokenRequired` 为 true 时才设置 token header）

---

## 5. IMDS 完全不使用的情况（完整列表）

### 情况 1：`disableIMDS = true` 或 `disableImds = true`

- 三个核心方法（`GetDefaultCredentials()`、`GetDefaultCredentialsSecurely()`、`GetCurrentRegion()`）在入口处检查 `m_disableIMDS`，若为 `true` 则立即返回空字符串
- **代码位置**: `AWSHttpResourceClient.cpp:251-254`、`AWSHttpResourceClient.cpp:302-306`、`AWSHttpResourceClient.cpp:370-373`

### 情况 2：环境变量 `AWS_EC2_METADATA_DISABLED=true`

- `DefaultAWSCredentialsProviderChain` 不会添加 `InstanceProfileCredentialsProvider`
- `ClientConfiguration` 构造函数不会调用 `GetCurrentRegion()`
- **代码位置**: `AWSCredentialsProviderChain.cpp:83-87`、`ClientConfiguration.cpp:396`

### 情况 3：IMDSv2 token 请求返回 HTTP 400 Bad Request

- `GetDefaultCredentialsSecurely()` 直接返回空字符串，**不会** 回退到 IMDSv1
- 这是 IMDS 明确拒绝的信号（例如 token TTL 无效）
- **代码位置**: `AWSHttpResourceClient.cpp:325-328`

### 情况 4：非 EC2 环境（无 IMDS 端点可达）

- 所有请求超时或无响应，最终 retry 耗尽后返回空字符串
- 凭证链会继续尝试下一个 provider

---

## 6. 状态转换完整流程图

```
EC2MetadataClient 初始化
  │
  ├── m_tokenRequired = true （默认）
  ├── m_disableIMDS = false （默认） 或由配置设置
  └── m_disableIMDSV1 = false （默认） 或由配置/编译设置
  │
  ▼
调用 GetDefaultCredentialsSecurely()
  │
  ├── [检查 m_disableIMDS == true] ──→ 返回空（不调用 IMDS）
  │
  ├── [m_tokenRequired == false 且 m_disableIMDSV1 == false 且未编译禁用]
  │     └──→ 调用 GetDefaultCredentials() ──→ 使用 IMDSv1
  │
  └── [m_tokenRequired == true 或 m_disableIMDSV1 == true]
        │
        ▼
      PUT /latest/api/token
        │
        ├── [HTTP 400 Bad Request] ──→ 返回空（终止）
        │
        ├── [HTTP 200 OK + 非空 token] ──→ 使用 IMDSv2 继续
        │     │
        │     ▼
        │   GET /latest/meta-data/iam/security-credentials  (带 token)
        │     │
        │     ▼
        │   GET /latest/meta-data/iam/security-credentials/{profile}  (带 token)
        │     │
        │     └──→ 返回凭证 (IMDSv2)
        │
        └── [非 OK 或空 token]
              │
              ├── [m_disableIMDSV1 == true 或编译禁用] ──→ 返回空（不回退）
              │
              └── [m_disableIMDSV1 == false]
                    │
                    ▼
                  设置 m_tokenRequired = false
                  调用 GetDefaultCredentials() ──→ 使用 IMDSv1
                    │
                    ▼
                  GET /latest/meta-data/iam/security-credentials  (不带 token)
                    │
                    ├── [HTTP 401 Unauthorized] ──→ 设置 m_tokenRequired = true, 返回空
                    │     （下次调用将重新尝试 IMDSv2）
                    │
                    └── [HTTP 200 OK]
                          │
                          ▼
                        GET /latest/meta-data/iam/security-credentials/{profile}  (不带 token)
                          │
                          └──→ 返回凭证 (IMDSv1)
```

---

## 7. 调用链中涉及 IMDS 的所有场景汇总

| 场景 | 触发位置 | 使用版本 | 说明 |
|------|---------|---------|------|
| 默认凭证链获取 EC2 实例凭证 | `DefaultAWSCredentialsProviderChain` → `InstanceProfileCredentialsProvider` → `EC2InstanceProfileConfigLoader` → `GetDefaultCredentialsSecurely()` | IMDSv2 优先，可回退到 IMDSv1 | 这是最常见的 IMDS 使用场景 |
| ClientConfiguration 构造时自动检测 region | `ClientConfiguration()` 4 个构造函数 → `EC2MetadataClient::GetCurrentRegion()` | 取决于 `m_tokenRequired` 状态 | 仅在 region 未通过环境变量/配置文件指定时触发 |
| EC2InstanceProfileConfigLoader 获取 region | `EC2InstanceProfileConfigLoader::LoadInternal()` → `m_ec2metadataClient->GetCurrentRegion()` | 取决于 `m_tokenRequired` 状态 | 用于填充 profile 的 region 信息 |
| 直接调用 `GetResource(resourcePath)` | 用户代码或内部代码直接调用 | 无 token（原始 HTTP GET） | 这是基类 `AWSHttpResourceClient::GetResource()` 的行为，不走 v1/v2 逻辑 |

---

## 8. 配置优先级汇总

```
编译时配置 (DISABLE_INTERNAL_IMDSV1_CALLS)
  ↓ 覆盖
代码配置 (ClientConfiguration.disableImdsV1)
  ↓ 若未代码设置
环境变量 (ec2_metadata_v1_disabled)
  ↓ 若环境变量未设置
AWS 配置文件 (~/.aws/config 中 AWS_EC2_METADATA_V1_DISABLED)
  ↓ 若配置文件未设置
默认值 (false，即 IMDSv1 允许作为回退)
```

**注意**: 编译时配置 `DISABLE_IMDSV1` 的优先级最高，会在构造函数中将 `m_disableIMDSV1` 强制设为 `true`，即使运行时配置试图启用 v1 也无效。

---

## 9. 涉及的所有源文件

| 文件路径 | 角色 |
|---------|------|
| `src/aws-cpp-sdk-core/source/internal/AWSHttpResourceClient.cpp` | IMDS v1/v2 核心实现 |
| `src/aws-cpp-sdk-core/include/aws/core/internal/AWSHttpResourceClient.h` | EC2MetadataClient 声明 |
| `src/aws-cpp-sdk-core/source/client/ClientConfiguration.cpp` | 从环境变量/配置文件加载 `disableImdsV1`、IMDS timeout/attempts 等 |
| `src/aws-cpp-sdk-core/include/aws/core/client/ClientConfiguration.h` | `disableIMDS`、`disableImdsV1` 字段声明 |
| `src/aws-cpp-sdk-core/source/config/EC2InstanceProfileConfigLoader.cpp` | 通过 `GetDefaultCredentialsSecurely()` 加载 EC2 凭证 |
| `src/aws-cpp-sdk-core/source/auth/AWSCredentialsProvider.cpp` | `InstanceProfileCredentialsProvider` 实现 |
| `src/aws-cpp-sdk-core/source/auth/AWSCredentialsProviderChain.cpp` | 默认凭证链，决定是否添加 EC2 metadata provider |
| `CMakeLists.txt` | `DISABLE_INTERNAL_IMDSV1_CALLS` 编译选项 |
| `tests/aws-cpp-sdk-core-tests/aws/auth/AWSHttpResourceClientTest.cpp` | IMDSv1/v2 行为的单元测试 |
