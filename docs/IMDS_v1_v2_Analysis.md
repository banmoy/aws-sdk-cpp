# IMDSv1 与 IMDSv2 使用场景完整分析

本文档基于 AWS SDK for C++ 源码，完整列举了 IMDSv1 和 IMDSv2 在所有场景下的使用情况。

---

## 一、核心架构概览

IMDS（EC2 Instance Metadata Service）调用集中在 `EC2MetadataClient` 类中，位于：
- **头文件**: `src/aws-cpp-sdk-core/include/aws/core/internal/AWSHttpResourceClient.h`
- **实现文件**: `src/aws-cpp-sdk-core/source/internal/AWSHttpResourceClient.cpp`

`EC2MetadataClient` 有三个核心方法调用 IMDS：

| 方法 | 协议版本 | 说明 |
|------|---------|------|
| `GetDefaultCredentials()` | **纯 IMDSv1** | 直接无 token GET 请求 |
| `GetDefaultCredentialsSecurely()` | **IMDSv2（含 v1 回退）** | 先获取 token，失败时可能回退到 v1 |
| `GetCurrentRegion()` | **视状态而定** | 根据 `m_tokenRequired` 标志决定使用 v1 或 v2 |

---

## 二、控制 IMDS 版本的配置机制

### 2.1 编译时控制

| 配置项 | 文件位置 | 效果 |
|--------|---------|------|
| `DISABLE_INTERNAL_IMDSV1_CALLS` (CMake option) | `CMakeLists.txt:62` | 定义预处理宏 `DISABLE_IMDSV1` |
| `DISABLE_IMDSV1` (预处理宏) | `CMakeLists.txt:81-82` | 编译时移除所有 `#if !defined(DISABLE_IMDSV1)` 保护的代码块 |

**当 `DISABLE_IMDSV1` 被定义时：**
- `GetDefaultCredentials()` 方法整体被移除（`AWSHttpResourceClient.h:122-128`, `AWSHttpResourceClient.cpp:216-267`）
- `GetDefaultCredentialsSecurely()` 中的 v1 回退路径被移除（`AWSHttpResourceClient.cpp:276-280`, `297-303`）
- SDK **只能**使用 IMDSv2，永远不会回退到 v1

### 2.2 运行时控制

| 配置项 | 来源 | 对应成员变量 | 效果 |
|--------|------|-------------|------|
| `disableIMDS` | `ClientConfiguration` 构造参数 `shouldDisableIMDS` | `m_disableIMDS` | 禁用**所有** IMDS 调用（v1 和 v2 均不调用） |
| `disableImdsV1` | 环境变量 `ec2_metadata_v1_disabled` 或配置文件键 `AWS_EC2_METADATA_V1_DISABLED` | `m_disableIMDSV1` | 仅禁用 IMDSv1，保留 IMDSv2 |
| `AWS_EC2_METADATA_DISABLED` | 环境变量 | （外部检查） | 阻止 IMDS 相关的 Provider 添加到凭证链，阻止区域发现 |

**`disableImdsV1` 的加载逻辑** (`ClientConfiguration.cpp:210-220`)：
```
环境变量 ec2_metadata_v1_disabled → 配置文件 AWS_EC2_METADATA_V1_DISABLED → 默认 "false"
```

---

## 三、IMDSv2 使用的所有场景

### 场景 1：`GetDefaultCredentialsSecurely()` — 获取 EC2 实例凭证（主路径）

**文件**: `AWSHttpResourceClient.cpp:269-334`

**流程**:
1. 通过 HTTP PUT 请求 `{endpoint}/latest/api/token` 获取会话 token
   - 请求头: `x-aws-ec2-metadata-token-ttl-seconds: 21600`
2. 使用获取到的 token，通过 HTTP GET 请求 `{endpoint}/latest/meta-data/iam/security-credentials`
   - 请求头: `x-aws-ec2-metadata-token: {token}`
3. 使用 token，通过 HTTP GET 请求 `{endpoint}/latest/meta-data/iam/security-credentials/{profile_name}`
   - 请求头: `x-aws-ec2-metadata-token: {token}`

**IMDSv2 使用条件**（同时满足）：
- `m_disableIMDS == false`
- `m_tokenRequired == true`（初始默认值为 true）或 `m_disableIMDSV1 == true`
- PUT `/latest/api/token` 返回 HTTP 200 且 token 非空

**调用链（谁触发了这个方法）**：

| 调用者 | 文件位置 | 触发条件 |
|--------|---------|---------|
| `EC2InstanceProfileConfigLoader::LoadInternal()` | `EC2InstanceProfileConfigLoader.cpp:55` | 凭证过期或首次加载时 |
| `GetCurrentRegion()` 内部 | `AWSHttpResourceClient.cpp:357` | 当 `m_tokenRequired == true` 时，为获取 token 而调用 |

### 场景 2：`GetCurrentRegion()` — 获取 EC2 实例区域（IMDSv2 路径）

**文件**: `AWSHttpResourceClient.cpp:336-396`

**流程**:
1. 若 `m_tokenRequired == true`，先调用 `GetDefaultCredentialsSecurely()` 以获取/刷新 token
2. 通过 HTTP GET 请求 `{endpoint}/latest/meta-data/placement/availability-zone`
   - 请求头: `x-aws-ec2-metadata-token: {m_token}`

**IMDSv2 使用条件**：
- `m_disableIMDS == false`
- `m_tokenRequired == true`

**调用链（谁触发了这个方法）**：

| 调用者 | 文件位置 | 触发条件 |
|--------|---------|---------|
| `ClientConfiguration()` 默认构造 | `ClientConfiguration.cpp:228-237` | `disableIMDS==false` 且 `region` 为空 且 `AWS_EC2_METADATA_DISABLED != "true"` |
| `ClientConfiguration(ClientConfigurationInitValues)` | `ClientConfiguration.cpp:252-261` | 同上 |
| `ClientConfiguration(const char* profile, ...)` | `ClientConfiguration.cpp:277-287` | 同上 |
| `ClientConfiguration(bool, const char*, ...)` | `ClientConfiguration.cpp:325-336` | 同上 |
| Smart Defaults AUTO 模式 | `ClientConfigurationDefaults.cpp:152-159` / `DefaultsSource.vm:152-159` | `AWS_DEFAULTS_MODE=="auto"` 且尚未获取 EC2 区域 且 `AWS_EC2_METADATA_DISABLED != "true"` |
| `EC2InstanceProfileConfigLoader::LoadInternal()` | `EC2InstanceProfileConfigLoader.cpp:93` | 获取凭证后附带获取区域 |

---

## 四、IMDSv1 使用的所有场景

> **注意**: 所有 IMDSv1 路径均受 `#if !defined(DISABLE_IMDSV1)` 编译时保护，且受 `m_disableIMDSV1` 运行时保护。

### 场景 3：`GetDefaultCredentials()` — 纯 IMDSv1 凭证获取

**文件**: `AWSHttpResourceClient.cpp:216-267`

**流程**:
1. 直接通过 HTTP GET 请求 `{endpoint}/latest/meta-data/iam/security-credentials`（**无 token 头**）
2. 通过 HTTP GET 请求 `{endpoint}/latest/meta-data/iam/security-credentials/{profile_name}`（**无 token 头**）

**IMDSv1 使用条件**（同时满足）：
- 编译时未定义 `DISABLE_IMDSV1`
- `m_disableIMDS == false`
- `m_disableIMDSV1 == false`
- 且处于以下之一的调用路径中

**IMDSv1 被触发的具体情形**：

#### 情形 3a：`m_tokenRequired == false` 且直接调用 `GetDefaultCredentials()`

当之前的 IMDSv2 token 获取失败导致 `m_tokenRequired` 被设为 `false` 后，后续调用 `GetDefaultCredentialsSecurely()` 时：

```
GetDefaultCredentialsSecurely() → m_tokenRequired == false → 调用 GetDefaultCredentials()
```

**代码位置**: `AWSHttpResourceClient.cpp:276-279`
```cpp
#if !defined(DISABLE_IMDSV1)
    if (!m_disableIMDSV1 && !m_tokenRequired) {
        return GetDefaultCredentials();
    }
#endif
```

#### 情形 3b：IMDSv2 token PUT 请求失败时的回退

当 `GetDefaultCredentialsSecurely()` 中 PUT `/latest/api/token` 返回非 200 或 token 为空时：

**代码位置**: `AWSHttpResourceClient.cpp:297-303`
```cpp
#if !defined(DISABLE_IMDSV1)
    if (!m_disableIMDSV1 && (result.GetResponseCode() != HttpResponseCode::OK || trimmedTokenString.empty()))
    {
        m_tokenRequired = false;
        AWS_LOGSTREAM_TRACE(m_logtag.c_str(), "Calling EC2MetadataService to get token failed, falling back to less secure way.");
        return GetDefaultCredentials();
    }
#endif
```

此处还会将 `m_tokenRequired` 设为 `false`，使后续调用直接走情形 3a。

#### 情形 3c：IMDSv1 收到 401 Unauthorized 后触发重新使用 IMDSv2

在 `GetDefaultCredentials()` 中，如果 IMDS 返回 HTTP 401:

**代码位置**: `AWSHttpResourceClient.cpp:241-244`
```cpp
if (httpResponseCode == Http::HttpResponseCode::UNAUTHORIZED)
{
    m_tokenRequired = true;
    return {};
}
```

这会将 `m_tokenRequired` 重置为 `true`，使下次调用回到 IMDSv2 路径。

### 场景 4：`GetCurrentRegion()` — IMDSv1 区域获取

**文件**: `AWSHttpResourceClient.cpp:353-360`

**流程**: 直接通过 HTTP GET 请求 `{endpoint}/latest/meta-data/placement/availability-zone`（**无 token 头**）

**IMDSv1 使用条件**：
- `m_disableIMDS == false`
- `m_tokenRequired == false`（意味着之前的 IMDSv2 token 获取已经失败过）

**代码位置**: `AWSHttpResourceClient.cpp:353-360`
```cpp
{
    std::lock_guard<std::recursive_mutex> locker(m_tokenMutex);
    if (m_tokenRequired)
    {
        GetDefaultCredentialsSecurely();
        regionRequest->SetHeaderValue(EC2_IMDS_TOKEN_HEADER, m_token);
    }
}
```

当 `m_tokenRequired == false` 时，不进入 if 分支，regionRequest **不携带** token 头 → IMDSv1。

---

## 五、IMDS 完全禁用的场景

### 场景 5：`disableIMDS == true` — 禁用所有 IMDS 调用

检查点如下：

| 方法 | 代码位置 |
|------|---------|
| `GetDefaultCredentials()` | `AWSHttpResourceClient.cpp:219-222` |
| `GetDefaultCredentialsSecurely()` | `AWSHttpResourceClient.cpp:271-274` |
| `GetCurrentRegion()` | `AWSHttpResourceClient.cpp:338-341` |

### 场景 6：`AWS_EC2_METADATA_DISABLED == "true"` — 环境变量禁用

检查点如下（**注意**：此变量在 `EC2MetadataClient` 外部检查，控制是否创建/使用 IMDS 客户端）：

| 调用者 | 代码位置 | 效果 |
|--------|---------|------|
| `DefaultAWSCredentialsProviderChain` 构造 | `AWSCredentialsProviderChain.cpp:60-85` | 不将 `InstanceProfileCredentialsProvider` 加入凭证链 |
| `ClientConfiguration()` 默认构造 | `ClientConfiguration.cpp:230` | 不通过 IMDS 获取区域 |
| `ClientConfiguration(ClientConfigurationInitValues)` | `ClientConfiguration.cpp:254` | 同上 |
| `ClientConfiguration(const char* profile, ...)` | `ClientConfiguration.cpp:279` | 同上 |
| `ClientConfiguration(bool, const char*, ...)` | `ClientConfiguration.cpp:327` | 同上 |
| Smart Defaults AUTO 模式 | `ClientConfigurationDefaults.cpp:164` / `DefaultsSource.vm:153` | 不通过 IMDS 获取区域 |

---

## 六、完整调用链汇总

### 6.1 凭证获取调用链

```
DefaultAWSCredentialsProviderChain
  └─→ InstanceProfileCredentialsProvider::GetAWSCredentials()         [AWSCredentialsProvider.cpp:237]
       └─→ RefreshIfExpired()                                          [AWSCredentialsProvider.cpp:282]
            └─→ Reload()                                               [AWSCredentialsProvider.cpp:271]
                 └─→ EC2InstanceProfileConfigLoader::LoadInternal()    [EC2InstanceProfileConfigLoader.cpp:41]
                      ├─→ EC2MetadataClient::GetDefaultCredentialsSecurely()   ← 主入口
                      │    ├─→ [IMDSv2] PUT /latest/api/token → 获取 token
                      │    │    ├─→ 成功 → [IMDSv2] GET /latest/meta-data/iam/security-credentials (带 token)
                      │    │    │         → [IMDSv2] GET /latest/meta-data/iam/security-credentials/{profile} (带 token)
                      │    │    ├─→ 返回 400 Bad Request → 直接失败，返回空
                      │    │    └─→ 失败（非 200 或空 token）
                      │    │         ├─→ [若 DISABLE_IMDSV1 未定义 且 !m_disableIMDSV1]
                      │    │         │    → 设置 m_tokenRequired = false
                      │    │         │    → [IMDSv1] 回退到 GetDefaultCredentials()
                      │    │         └─→ [若 DISABLE_IMDSV1 已定义 或 m_disableIMDSV1]
                      │    │              → 直接失败，返回空
                      │    └─→ [若 !m_tokenRequired 且 !m_disableIMDSV1 且 DISABLE_IMDSV1 未定义]
                      │         → [IMDSv1] 委托给 GetDefaultCredentials()
                      └─→ EC2MetadataClient::GetCurrentRegion()
                           ├─→ [若 m_tokenRequired] → 先调用 GetDefaultCredentialsSecurely() 获取 token
                           │    → [IMDSv2] GET /latest/meta-data/placement/availability-zone (带 token)
                           └─→ [若 !m_tokenRequired]
                                → [IMDSv1] GET /latest/meta-data/placement/availability-zone (无 token)
```

### 6.2 区域发现调用链

```
ClientConfiguration 构造函数（4 个重载版本）
  └─→ EC2MetadataClient::GetCurrentRegion()
       ├─→ [IMDSv2 路径] m_tokenRequired == true
       └─→ [IMDSv1 路径] m_tokenRequired == false

Smart Defaults AUTO 模式
  └─→ EC2MetadataClient::GetCurrentRegion()
       ├─→ [IMDSv2 路径] m_tokenRequired == true
       └─→ [IMDSv1 路径] m_tokenRequired == false
```

---

## 七、所有 IMDS 请求端点和路径

| 请求 | HTTP 方法 | 路径 | 协议版本 | 额外请求头 |
|------|----------|------|---------|-----------|
| 获取 IMDSv2 会话 token | PUT | `/latest/api/token` | v2 | `x-aws-ec2-metadata-token-ttl-seconds: 21600` |
| 列出安全凭证（带 token） | GET | `/latest/meta-data/iam/security-credentials` | v2 | `x-aws-ec2-metadata-token: {token}` |
| 获取特定凭证（带 token） | GET | `/latest/meta-data/iam/security-credentials/{profile}` | v2 | `x-aws-ec2-metadata-token: {token}` |
| 列出安全凭证（无 token） | GET | `/latest/meta-data/iam/security-credentials` | v1 | 无 |
| 获取特定凭证（无 token） | GET | `/latest/meta-data/iam/security-credentials/{profile}` | v1 | 无 |
| 获取可用区（带 token） | GET | `/latest/meta-data/placement/availability-zone` | v2 | `x-aws-ec2-metadata-token: {token}` |
| 获取可用区（无 token） | GET | `/latest/meta-data/placement/availability-zone` | v1 | 无 |

**IMDS 端点地址**：
- 默认 IPv4: `http://169.254.169.254`
- IPv6: `http://[fd00:ec2::254]`（通过 `AWS_EC2_METADATA_SERVICE_ENDPOINT_MODE=ipv6` 启用）
- 自定义: 通过 `AWS_EC2_METADATA_SERVICE_ENDPOINT` 环境变量指定

---

## 八、状态转换与版本切换逻辑

`m_tokenRequired` 是控制 v1/v2 切换的核心状态变量：

```
初始状态: m_tokenRequired = true (默认尝试 IMDSv2)

┌────────────────────────────────────────────────────────────────┐
│                    m_tokenRequired = true                       │
│                    → 使用 IMDSv2 路径                            │
│                                                                  │
│  PUT /latest/api/token 失败（非 200 或空 token）                  │
│  且 DISABLE_IMDSV1 未定义 且 !m_disableIMDSV1                    │
│  → 设置 m_tokenRequired = false                                  │
│  → 回退到 IMDSv1                                                 │
└────────────────────────────┬───────────────────────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────────┐
│                    m_tokenRequired = false                      │
│                    → 使用 IMDSv1 路径                            │
│                                                                  │
│  GET /latest/meta-data/iam/security-credentials 返回 401        │
│  → 设置 m_tokenRequired = true                                   │
│  → 回到 IMDSv2 路径                                              │
└────────────────────────────────────────────────────────────────┘
```

---

## 九、代码生成模板中的 IMDS 引用

生成的服务客户端配置类继承自 `ClientConfiguration`，传递 `shouldDisableIMDS` 参数：

| 模板文件 | 涉及位置 |
|---------|---------|
| `ServiceClientConfigurationHeader.vm:46-56` | 声明 `shouldDisableIMDS` 构造参数 |
| `ServiceClientConfigurationSource.vm:85-95` | 传递参数到基类构造函数 |
| `DefaultsSource.vm:152-159` / `DefaultsHeader.vm:76` | Smart Defaults AUTO 模式的区域发现 |

---

## 十、总结：所有 IMDSv1 和 IMDSv2 使用情况一览

### IMDSv2 使用（共 4 种 HTTP 请求）

| # | 场景 | 方法 | HTTP 请求 | 条件 |
|---|------|------|----------|------|
| 1 | 获取会话 token | `GetDefaultCredentialsSecurely()` | PUT `/latest/api/token` | `!m_disableIMDS` 且 (`m_tokenRequired` 或 `m_disableIMDSV1`) |
| 2 | 列出 IAM 角色（带 token） | `GetDefaultCredentialsSecurely()` | GET `/latest/meta-data/iam/security-credentials` + token header | token 获取成功后 |
| 3 | 获取凭证详情（带 token） | `GetDefaultCredentialsSecurely()` | GET `/latest/meta-data/iam/security-credentials/{profile}` + token header | profile 列出成功后 |
| 4 | 获取可用区（带 token） | `GetCurrentRegion()` | GET `/latest/meta-data/placement/availability-zone` + token header | `m_tokenRequired == true` |

### IMDSv1 使用（共 3 种 HTTP 请求，均需 `!defined(DISABLE_IMDSV1)` 且 `!m_disableIMDSV1`）

| # | 场景 | 方法 | HTTP 请求 | 触发条件 |
|---|------|------|----------|---------|
| 5 | 列出 IAM 角色（无 token） | `GetDefaultCredentials()` | GET `/latest/meta-data/iam/security-credentials` | `m_tokenRequired == false`（IMDSv2 token 获取曾失败） |
| 6 | 获取凭证详情（无 token） | `GetDefaultCredentials()` | GET `/latest/meta-data/iam/security-credentials/{profile}` 通过 `GetResource()` | profile 列出成功后 |
| 7 | 获取可用区（无 token） | `GetCurrentRegion()` | GET `/latest/meta-data/placement/availability-zone` | `m_tokenRequired == false` |

### 完全禁用 IMDS（不发出任何请求）

| # | 场景 | 控制方式 |
|---|------|---------|
| 8 | `m_disableIMDS == true` | `ClientConfiguration.disableIMDS` / 构造参数 `shouldDisableIMDS` |
| 9 | `AWS_EC2_METADATA_DISABLED == "true"` | 环境变量，在凭证链构造和 `ClientConfiguration` 构造中检查 |
| 10 | 编译时 `DISABLE_IMDSV1` + IMDSv2 token 获取失败 | IMDSv1 被移除，IMDSv2 失败则无法获取凭证 |
| 11 | 运行时 `m_disableIMDSV1 == true` + IMDSv2 token 获取失败 | IMDSv1 被禁用，IMDSv2 失败则无法获取凭证 |

---

## 十一、相关源文件索引

| 文件 | 角色 |
|------|------|
| `src/aws-cpp-sdk-core/include/aws/core/internal/AWSHttpResourceClient.h` | EC2MetadataClient 声明 |
| `src/aws-cpp-sdk-core/source/internal/AWSHttpResourceClient.cpp` | EC2MetadataClient 实现（核心 IMDS 调用逻辑） |
| `src/aws-cpp-sdk-core/include/aws/core/client/ClientConfiguration.h` | `disableIMDS`, `disableImdsV1` 字段声明 |
| `src/aws-cpp-sdk-core/source/client/ClientConfiguration.cpp` | 配置加载及区域发现调用 IMDS |
| `src/aws-cpp-sdk-core/source/auth/AWSCredentialsProvider.cpp` | `InstanceProfileCredentialsProvider` |
| `src/aws-cpp-sdk-core/source/auth/AWSCredentialsProviderChain.cpp` | 凭证链中 IMDS 条件判断 |
| `src/aws-cpp-sdk-core/source/config/EC2InstanceProfileConfigLoader.cpp` | 调用 `GetDefaultCredentialsSecurely()` |
| `src/aws-cpp-sdk-core/source/config/defaults/ClientConfigurationDefaults.cpp` | Smart Defaults AUTO 模式区域发现 |
| `CMakeLists.txt` | `DISABLE_INTERNAL_IMDSV1_CALLS` 编译选项 |
| `tools/code-generation/.../ServiceClientConfigurationSource.vm` | 服务客户端配置生成模板 |
| `tools/code-generation/.../ServiceClientConfigurationHeader.vm` | 服务客户端配置头文件生成模板 |
| `tools/code-generation/.../DefaultsSource.vm` | Smart Defaults 生成模板 |
