# StarRocks BE IMDSv1 回退问题：严格验证分析

> 本文档对 [analysis-broker-load-imdsv1-issue.md](https://github.com/banmoy/starrocks/blob/cursor%2Fslack-claude-cf4a/docs%2Fanalysis-broker-load-imdsv1-issue.md) 中的推论逐条进行源码级验证，区分"已确认事实"与"需修正的推论"。

---

## 一、原文根因推论的验证结论

| 原文推论 | 验证结论 | 详细说明 |
|---------|---------|---------|
| `m_tokenRequired` 初始为 `true`，PUT 失败后永久设为 `false` | **正确** | 代码确认：`AWSHttpResourceClient.cpp:332` |
| 问题 BE 启动时 PUT 失败导致永久回退 | **基本正确，但需精确化触发时机** | 第一次 PUT 不是在 credential fetch 时，而是在 `getClientConfig()` region 检测时 |
| 设置 `AWS_EC2_METADATA_V1_DISABLED=true` 可解决 | **不正确** | 全局单例 EC2MetadataClient 的创建不读取此配置，详见第三节 |
| 代码修复方案：给 `InstanceProfileCredentialsProvider` 传 `disableImdsV1` 配置 | **不正确** | 即使传入配置，全局单例已存在，不会被重建，详见第四节 |
| EC2 实例强制 IMDSv2（`--http-tokens required`） | **正确且有效** | 401 响应会触发 `m_tokenRequired = true` 自愈，详见第五节 |

---

## 二、EC2MetadataClient 全局单例的完整生命周期

这是理解整个问题的关键。通过源码追踪，确认只存在 **一个** `EC2MetadataClient` 全局单例，且其创建方式导致无法接受运行时配置。

### 2.1 单例创建：`Aws::InitAPI()` 阶段

```
Aws::InitAPI(options)                                   [Aws.cpp:170]
  └── Aws::Internal::InitEC2MetadataClient()            [AWSHttpResourceClient.cpp:496]
        └── s_ec2metadataClient = new EC2MetadataClient(endpoint)  [单参数构造]
```

单参数构造函数（`AWSHttpResourceClient.cpp:192-199`）：

```cpp
EC2MetadataClient::EC2MetadataClient(const char *endpoint) :
    AWSHttpResourceClient(EC2_METADATA_CLIENT_LOG_TAG),   // ← 默认 HTTP 配置
    m_endpoint(endpoint),
    m_disableIMDS(false),
    m_tokenRequired(true)                                  // ← 初始 true
{
    // m_disableIMDSV1 使用成员默认值 = false               // ← 不读取任何环境变量或配置
}
```

`AWSHttpResourceClient` 的默认配置（`AWSHttpResourceClient.cpp:51-81`）：

```cpp
res.connectTimeoutMs = 1000;     // 1 秒连接超时
res.requestTimeoutMs = 1000;     // 1 秒请求超时
res.retryStrategy = DefaultRetryStrategy(1, 1000);  // 仅 1 次重试
```

**关键结论**：全局单例在 `InitAPI()` 中创建时：
- `m_tokenRequired = true`
- `m_disableIMDSV1 = false`（硬编码默认值，**不读取环境变量或配置文件**）
- 使用默认 CURL HTTP 客户端
- 连接/请求超时各 1 秒，仅 1 次重试

### 2.2 HTTP Client Factory 替换：Poco 阶段

StarRocks 在 `InitAPI()` 之后替换 HTTP 客户端工厂（`starrocks_main.cpp:217-219`）：

```cpp
Aws::Http::SetHttpClientFactory(std::make_shared<starrocks::poco::PocoHttpClientFactory>());
```

`SetHttpClientFactory` 的实现（`HttpClientFactory.cpp:185-195`）：

```cpp
void SetHttpClientFactory(const std::shared_ptr<HttpClientFactory>& factory) {
    bool recreateEC2Client = Aws::Internal::GetEC2MetadataClient() ? true : false;
    CleanupHttp();
    GetHttpClientFactory() = factory;
    if (recreateEC2Client) {
        Aws::Internal::InitEC2MetadataClient();   // ← 试图重建
    }
}
```

但 `InitEC2MetadataClient()` 有守卫（`AWSHttpResourceClient.cpp:496-501`）：

```cpp
void InitEC2MetadataClient() {
    if (s_ec2metadataClient) { return; }   // ← 单例已存在，直接返回！
    // ...
}
```

**关键结论**：`SetHttpClientFactory()` **无法重建** EC2MetadataClient。单例继续使用在 `InitAPI()` 阶段创建的 **CURL HTTP 客户端**，而非 StarRocks 配置的 Poco 客户端。所有 IMDS 请求均通过 CURL 发出。

### 2.3 单例不可重建的保护机制

不论调用 `InitEC2MetadataClient()` 还是 `InitEC2MetadataClient(credentialConfig)`，只要单例已存在就直接返回：

```cpp
void InitEC2MetadataClient(const CredentialProviderConfiguration& credentialConfig) {
    if (s_ec2metadataClient) { return; }   // ← 同样的守卫
    // ...
}
```

唯一能重置单例的是 `CleanupEC2MetadataClient()`（将 `s_ec2metadataClient` 设为 nullptr），但这仅在 `Aws::ShutdownAPI()` 中被调用。

---

## 三、环境变量方案验证：为什么不生效

原文推荐设置 `export AWS_EC2_METADATA_V1_DISABLED=true`。以下从三个层面验证此方案无效。

### 3.1 环境变量名称不匹配

AWS C++ SDK 1.11.267 中的变量定义（`ClientConfiguration.cpp:40-41`）：

```cpp
static const char* DISABLE_IMDSV1_CONFIG_VAR = "AWS_EC2_METADATA_V1_DISABLED";  // ← 配置文件 key
static const char* DISABLE_IMDSV1_ENV_VAR = "ec2_metadata_v1_disabled";         // ← 环境变量名
```

加载逻辑（`ClientConfiguration.cpp:306-314`）：

```cpp
Aws::String disableIMDSv1 = LoadConfigFromEnvOrProfile(
    "ec2_metadata_v1_disabled",        // envKey: 检查此环境变量
    config.profileName,
    "AWS_EC2_METADATA_V1_DISABLED",   // profileProperty: 检查配置文件中此属性
    {"true", "false"}, "false");
```

`LoadConfigFromEnvOrProfile` 先检查环境变量 `ec2_metadata_v1_disabled`，再检查配置文件属性 `AWS_EC2_METADATA_V1_DISABLED`。

因此 `export AWS_EC2_METADATA_V1_DISABLED=true` 是错误的环境变量名，正确的应为 `export ec2_metadata_v1_disabled=true`。

### 3.2 即使环境变量名称正确，也不影响全局单例

设置 `export ec2_metadata_v1_disabled=true` 后的效果链：

```
setConfigFromEnvOrProfile()
  → 读取 ec2_metadata_v1_disabled=true
  → 设置 ClientConfiguration.disableImdsV1 = true
  → 设置 ClientConfiguration.credentialProviderConfig.imdsConfig.disableImdsV1 = true
```

但这仅影响 **`ClientConfiguration` 对象**，不影响全局单例 `EC2MetadataClient`。因为：

1. 全局单例在 `InitAPI()` 中用**单参数构造函数**创建，该构造函数 **不接受 `ClientConfiguration`** 参数
2. 全局单例的 `m_disableIMDSV1` 被硬编码为 `false`（成员默认值）
3. `ClientConfiguration.disableImdsV1` 仅在 `EC2MetadataClient(ClientConfiguration&)` 构造函数中被读取，但全局单例不使用此构造函数

### 3.3 `InstanceProfileCredentialsProvider` 也不创建新的 EC2MetadataClient

StarRocks 代码（`fs_s3.cpp:88`）：

```cpp
credential_provider = std::make_shared<Aws::Auth::InstanceProfileCredentialsProvider>();
```

调用链：

```
InstanceProfileCredentialsProvider()                    [默认构造]
  → EC2InstanceProfileConfigLoader(nullptr)
    → InitEC2MetadataClient()                           [无参版本]
      → if (s_ec2metadataClient) return;                [单例已存在，直接返回]
    → m_ec2metadataClient = GetEC2MetadataClient();     [获取已有单例]
```

即使用带 `CredentialProviderConfiguration` 参数的版本：

```
InstanceProfileCredentialsProvider(credentialConfig)
  → EC2InstanceProfileConfigLoader(credentialConfig)
    → InitEC2MetadataClient(credentialConfig)
      → if (s_ec2metadataClient) return;                [单例已存在，照样直接返回！]
```

**结论**：无论如何构造 `InstanceProfileCredentialsProvider`，它都使用 `InitAPI()` 创建的全局单例，该单例的 `m_disableIMDSV1` 始终为 `false`。环境变量方案 **无法生效**。

---

## 四、第一次 IMDS 调用的精确触发时机

原文推论"启动时或首次凭证获取时"需要精确化。

### 4.1 实际触发点：`getClientConfig()` 中的 region 检测

StarRocks `S3ClientFactory::getClientConfig()`（`fs_s3.h:67-75`）：

```cpp
static ClientConfiguration& getClientConfig() {
    static ClientConfiguration instance;   // ← 首次调用时构造
    return instance;
}
```

`ClientConfiguration` 默认构造函数（`ClientConfiguration.cpp:386-411`）：

```cpp
ClientConfiguration::ClientConfiguration() {
    // ...
    setLegacyClientConfigurationParameters(*this);
    setConfigFromEnvOrProfile(*this);
    // ...
    if (!this->disableIMDS &&
        region.empty() &&
        ToLower(GetEnv("AWS_EC2_METADATA_DISABLED").c_str()) != "true")
    {
        auto client = Aws::Internal::GetEC2MetadataClient();  // ← 获取全局单例
        if (client) {
            region = client->GetCurrentRegion();               // ← 第一次 IMDS 调用！
        }
    }
}
```

**前提条件**：当以下 **全部满足** 时触发 IMDS region 检测：
1. `disableIMDS = false`（默认值）
2. `region` 为空（未设置 `AWS_DEFAULT_REGION`、`AWS_REGION` 环境变量，且 `~/.aws/config` 无 region）
3. `AWS_EC2_METADATA_DISABLED` 不为 `true`

用户通过 SQL 设置 `"aws.s3.region" = "us-east-1"` 不影响此处——SQL 参数在 `getClientConfig()` 之后才应用。

### 4.2 `GetCurrentRegion()` 内部的 IMDS 调用

```cpp
Aws::String EC2MetadataClient::GetCurrentRegion() const {
    // ...
    std::lock_guard<std::recursive_mutex> locker(m_tokenMutex);
    if (m_tokenRequired) {                                  // ← true（初始值）
        GetDefaultCredentialsSecurely();                    // ← 这里触发 PUT /latest/api/token
        regionRequest->SetHeaderValue(EC2_IMDS_TOKEN_HEADER, m_token);
    }
    // GET /latest/meta-data/placement/availability-zone
}
```

`GetDefaultCredentialsSecurely()` 尝试 IMDSv2 PUT：

```cpp
auto result = GetResourceWithAWSWebServiceResult(tokenRequest);  // PUT /latest/api/token
// ...
if (!m_disableIMDSV1 && (result.GetResponseCode() != OK || trimmedTokenString.empty())) {
    m_tokenRequired = false;                                // ← 永久回退！
    return GetDefaultCredentials();                         // ← IMDSv1
}
```

### 4.3 调用时序

```
main()
  │
  ├── Aws::InitAPI()
  │     └── InitEC2MetadataClient() → 创建全局单例 (CURL, 1s timeout, m_tokenRequired=true)
  │
  ├── SetHttpClientFactory(Poco) → 无法重建单例，IMDS 仍用 CURL
  │
  └── 首次 S3 操作
        │
        ├── getClientConfig() → static ClientConfiguration instance
        │     │
        │     └── [region 为空] → GetCurrentRegion()
        │           │
        │           └── GetDefaultCredentialsSecurely()
        │                 │
        │                 └── PUT /latest/api/token          ← ★ 第一次 IMDS 调用
        │                       │
        │                       ├── [成功] → m_tokenRequired 保持 true → IMDSv2 ✅
        │                       │
        │                       └── [失败] → m_tokenRequired = false  ← ★ 永久回退到 v1
        │
        ├── _get_aws_credentials_provider()
        │     └── InstanceProfileCredentialsProvider() → 使用同一全局单例
        │
        └── 凭证获取 → GetDefaultCredentialsSecurely()
              │
              ├── [m_tokenRequired=true]  → PUT token → IMDSv2 ✅
              │
              └── [m_tokenRequired=false] → 直接 GetDefaultCredentials() → IMDSv1 ❌
                    （不再尝试 PUT，永久 v1）
```

**结论**：第一次 IMDS PUT 发生在 `getClientConfig()` 的 region 检测中，而非 credential fetch。如果此 PUT 失败，后续所有 IMDS 调用（包括 credential fetch）都用 IMDSv1。

---

## 五、方案 2（`--http-tokens required`）的验证：为什么有效

当 EC2 实例设置 `--http-tokens required` 后，IMDS 拒绝所有无 token 的 v1 请求，返回 HTTP 401。

`GetDefaultCredentials()`（v1 路径）中的 401 处理（`AWSHttpResourceClient.cpp:273-277`）：

```cpp
if (httpResponseCode == Http::HttpResponseCode::UNAUTHORIZED) {
    m_tokenRequired = true;    // ← 恢复为 true！
    return {};
}
```

**自愈流程**：

```
1. region 检测时 PUT 失败 → m_tokenRequired = false → 回退到 v1
2. v1 GET 请求 → IMDS 返回 401 (因为 --http-tokens required)
3. m_tokenRequired = true (自愈)
4. 当前调用返回空（region 检测失败，使用默认 region）
5. 后续 credential fetch → m_tokenRequired = true → 重新尝试 PUT → 成功 → IMDSv2 ✅
```

**此方案的代价**：第一次 region 检测会失败（返回空），导致 `ClientConfiguration` 使用默认 region `us-east-1`。但由于用户在 SQL 中显式设置了 `"aws.s3.region" = "us-east-1"`，实际不受影响。

---

## 六、正确的解决方案

### 方案 A：EC2 实例强制 IMDSv2（推荐，立即可用）

```bash
aws ec2 modify-instance-metadata-options \
    --instance-id <instance-id> \
    --http-tokens required \
    --http-put-response-hop-limit 2
```

**原理**：通过 401 自愈机制，即使首次 PUT 失败回退到 v1，v1 请求也会被 401 拒绝并恢复 `m_tokenRequired = true`。

### 方案 B：设置 `AWS_DEFAULT_REGION` 环境变量（缓解，减少但不消除 v1 风险）

```bash
export AWS_DEFAULT_REGION=us-east-1
```

**原理**：避免 `ClientConfiguration()` 构造时的 IMDS region 检测，消除第一个触发 IMDS 的入口点。但 `InstanceProfileCredentialsProvider` 的 credential fetch 仍然通过 IMDS，首次 PUT 仍可能失败并永久回退。

**此方案可降低风险但不能根除问题**，因为减少了启动时 IMDS 的并发负载。

### 方案 C：StarRocks 代码修复（长期，彻底）

需要在 `Aws::InitAPI()` 之后、首次使用前，重建全局单例以注入正确配置：

```cpp
// starrocks_main.cpp, after InitAPI and SetHttpClientFactory:
Aws::Internal::CleanupEC2MetadataClient();

Aws::Client::ClientConfiguration tempConfig;
// tempConfig.disableImdsV1 已通过 setConfigFromEnvOrProfile 从环境变量读取
Aws::Internal::InitEC2MetadataClient(tempConfig.credentialProviderConfig);
```

这样新的全局单例会：
1. 使用 Poco HTTP 客户端（因为此时 factory 已替换）
2. 从环境变量 `ec2_metadata_v1_disabled=true` 读取 `disableImdsV1` 设置
3. 使用 `CredentialProviderConfiguration` 中配置的超时和重试策略

### 方案 D：重新编译 AWS SDK（长期，彻底）

在 `thirdparty/build-thirdparty.sh` 的 AWS SDK 编译选项中添加：

```bash
-DDISABLE_INTERNAL_IMDSV1_CALLS=ON
```

这会在编译层面通过 `#if !defined(DISABLE_IMDSV1)` 移除所有 v1 回退代码。PUT 失败时 `m_tokenRequired` 不会被设为 `false`，下次调用会重新尝试 PUT。

---

## 七、原文推论修正汇总

| 编号 | 原文内容 | 修正 |
|------|---------|------|
| 1 | "启动时（或首次凭证获取时）IMDSv2 PUT 请求失败" | 精确化为：第一次 PUT 发生在 `getClientConfig()` 的 region 检测中，早于 credential fetch |
| 2 | "方案 1：`export AWS_EC2_METADATA_V1_DISABLED=true`" | 此环境变量名错误，正确的环境变量名为 `ec2_metadata_v1_disabled`；且即使名称正确也不影响全局单例 |
| 3 | "效果：SDK 内部 `m_disableIMDSV1 = true`" | 全局单例的 `m_disableIMDSV1` 始终为 `false`，不受环境变量影响 |
| 4 | "方案 3：传入禁用 IMDSv1 回退的配置" | 即使传入配置，因全局单例已存在（`InitAPI` 创建），`InitEC2MetadataClient(config)` 会被 `if (s_ec2metadataClient) return;` 守卫拦截，单例不会重建 |
| 5 | 未提及 | EC2MetadataClient 使用 CURL HTTP 客户端（非 Poco），因为 `SetHttpClientFactory()` 无法重建单例 |
| 6 | 未提及 | `getClientConfig()` 触发 region 检测的前提是 `AWS_DEFAULT_REGION`/`AWS_REGION` 环境变量未设置 |

---

## 八、涉及的关键源码位置

| 文件 | 行号 | 内容 |
|------|------|------|
| `Aws.cpp` | 170 | `InitAPI()` 调用 `InitEC2MetadataClient()` |
| `AWSHttpResourceClient.cpp` | 192-199 | `EC2MetadataClient` 单参数构造函数（`m_disableIMDSV1` 默认 `false`） |
| `AWSHttpResourceClient.cpp` | 496-501 | `InitEC2MetadataClient()` 守卫 `if (s_ec2metadataClient) return;` |
| `AWSHttpResourceClient.cpp` | 329-335 | PUT 失败时 `m_tokenRequired = false` 并回退到 v1 |
| `AWSHttpResourceClient.cpp` | 273-277 | v1 GET 收到 401 时 `m_tokenRequired = true`（自愈） |
| `AWSHttpResourceClient.cpp` | 308-311 | `m_tokenRequired=false` 时直接走 v1，不再尝试 PUT |
| `HttpClientFactory.cpp` | 185-195 | `SetHttpClientFactory()` 尝试重建但被守卫拦截 |
| `ClientConfiguration.cpp` | 40-41 | 环境变量名 `ec2_metadata_v1_disabled`（非 `AWS_EC2_METADATA_V1_DISABLED`） |
| `ClientConfiguration.cpp` | 394-404 | `ClientConfiguration()` region 检测逻辑 |
| `fs_s3.h` | 67-75 | StarRocks `getClientConfig()` 静态单例 |
| `fs_s3.cpp` | 87-88 | StarRocks `InstanceProfileCredentialsProvider()` 默认构造 |
| `starrocks_main.cpp` | 216-219 | `InitAPI()` 和 `SetHttpClientFactory(Poco)` 调用序列 |
