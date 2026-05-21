# 站点配置驱动正文抽取流程分析（R5 - 边界风险校准版）

> 本版本专注于边界风险校准：loginIfRequired=false 的 cookie 注入边界、请求次数口径统一、
> response->getInfo('url') 重定向影响、withOptions 可靠性评注，以及完整的可复核清单。

---

## 一、loginIfRequired=false 路径下 cookie 注入边界分析

### 1.0 代码证据

```php
// src/HttpClient/WallabagClient.php:35-38
$login = $this->authenticator->loginIfRequired($url);

if (!$login) {
    return $this->httpClient->request($method, $url, $options);  // L38 - ⚠️ 无 cookie 注入
}
```

### 1.1 关键发现：路径 A 完全跳过 cookie 注入

当 `loginIfRequired()` 返回 false 时（路径 A），代码在 L38 直接发起业务请求，**完全绕过了 L41-42 的 cookie 注入逻辑**。

**对比路径 B1/B2 的 cookie 注入**：
```php
// 路径 B1/B2 才会执行（loginIfRequired=true 时）
if (null !== $cookieHeader = $this->getCookieHeader($url)) {  // L41
    $options['headers']['cookie'] = $cookieHeader;            // L42
}
$response = $this->httpClient->request($method, $url, $options);  // L45
```

### 1.2 loginIfRequired=false 的 3 种子场景与风险

| 子场景 | 触发条件 | 代码位置 | cookie 注入行为 | 边界风险 |
|--------|----------|----------|-----------------|----------|
| **A1** | `buildSiteConfig()` 返回 false | `Authenticator.php:35` | ❌ 不注入 | 无用户或无凭证，预期行为 |
| **A2** | `$config->requiresLogin() === false` | `Authenticator.php:35` | ❌ 不注入 | 站点不需要登录，预期行为 |
| **A3** | `isLoggedIn() === true`（已有 Cookie） | `Authenticator.php:41` | ❌ 不注入 | ⚠️ **风险边界**：已有 Cookie 但不注入 |

### 1.3 A3 场景的边界风险详解

**触发链**：
```
首次请求（路径 B1）
  └─ loginIfRequired() → true
      └─ login() 执行 → Cookie jar 中写入 Cookie
      └─ 业务请求（L45）带 Cookie 成功

第二次请求（同 host）
  └─ loginIfRequired()
      ├─ buildSiteConfig() → 返回 SiteConfig
      ├─ requiresLogin() → true
      └─ isLoggedIn() → true（Cookie jar 中有该域名 Cookie）
          └─ loginIfRequired() → false
              └─ 路径 A：L38 直接请求，**不带 Cookie**
```

**代码证据**：
```php
// src/SiteConfig/LoginFormAuthenticator.php:44-54
public function isLoggedIn(SiteConfig $siteConfig)
{
    foreach ($this->browser->getCookieJar()->all() as $cookie) {
        // 只要存在该域名的 Cookie 即认为已登录
        if ($cookie->getDomain() === $siteConfig->getHost()) {
            return true;  // L49
        }
    }
    return false;
}
```

**风险总结**：
- 第二次请求时，`isLoggedIn()` 检测到已有 Cookie，返回 true
- `loginIfRequired()` 因此返回 false，进入路径 A
- 路径 A 在 L38 直接请求，**不注入 Cookie**
- 结果：业务请求不带 Cookie 发送到需要登录的站点

---

## 二、请求次数口径统一

### 2.1 两组独立的 HTTP 请求指标

**重要声明**：业务 HTTP 请求与登录内部 POST 请求使用**两个完全独立的 HTTP 客户端实例**，需要分别统计。

| 指标组 | 客户端实例 | 调用位置 | 说明 |
|--------|------------|----------|------|
| **组 A：业务 HTTP 请求** | `$this->httpClient`（WallabagClient 内部） | `WallabagClient.php` | 通过 `HttpClient::create()` 创建 |
| **组 B：登录内部 POST** | `$this->browser`（LoginFormAuthenticator 内部） | `LoginFormAuthenticator.php` | 注入的 `HttpBrowser` 实例 |

**客户端隔离证据**：
```php
// WallabagClient 内部创建独立的 HttpClient
// src/HttpClient/WallabagClient.php:22-24
$this->httpClient = HttpClient::create([
    'timeout' => 10,
]);

// LoginFormAuthenticator 使用独立的 HttpBrowser
// src/SiteConfig/LoginFormAuthenticator.php:14-16
public function __construct(
    private readonly HttpBrowser $browser,  // 独立实例
    AuthenticatorProvider $authenticatorProvider,
    private readonly string $defaultUserAgent,
) { ... }
```

**Cookie 共享机制**：
- Cookie jar 是共享的（通过 `$this->browser->getCookieJar()` 访问）
- 登录产生的 Cookie 写入 `HttpBrowser` 的 Cookie jar
- 业务请求通过 `getCookieHeader()` 从同一个 Cookie jar 读取 Cookie

### 2.2 各路径的请求次数统计

| 路径 | 组 A 业务请求次数 | 组 B 登录 POST 次数 | 总请求次数 |
|------|------------------|---------------------|------------|
| **路径 0** | 1（L32） | 0 | 1 |
| **路径 A** | 1（L38） | 0 | 1 |
| **路径 B1** | 1（L45） | 1（Authenticator.php:47） | 2 |
| **路径 B2** | 2（L45 + L57） | 2（Authenticator.php:47 + 77） | 4 |

**组 A 业务请求详细位置**：
| 行号 | 路径 | 说明 |
|------|------|------|
| L32 | 路径 0 | `restrictedAccess === 0` 直接请求 |
| L38 | 路径 A | `loginIfRequired=false` 直接请求 |
| L45 | 路径 B1/B2 | `loginIfRequired=true` 后第一次请求（带 Cookie） |
| L57 | 路径 B2 | `loginIfRequested=true` 后第二次请求（带新 Cookie） |

**组 B 登录 POST 详细位置**：
| 行号 | 触发点 | 说明 |
|------|--------|------|
| `Authenticator.php:47` | `loginIfRequired()` 内部 | 返回 true 前执行 |
| `Authenticator.php:77` | `loginIfRequested()` 内部 | 返回 true 前执行 |

**login() 内部 POST 证据**：
```php
// src/SiteConfig/LoginFormAuthenticator.php:27-37
public function login(SiteConfig $siteConfig)
{
    $postFields = [
        $siteConfig->getUsernameField() => $siteConfig->getUsername(),
        $siteConfig->getPasswordField() => $siteConfig->getPassword(),
    ] + $this->getExtraFields($siteConfig);

    // ⚠️ 组 B 请求：使用独立的 HttpBrowser 实例
    $this->browser->request('POST', $siteConfig->getLoginUri(), $postFields, [], $this->getHttpHeaders($siteConfig));  // L34

    return $this;
}
```

### 2.3 路径 B2 的完整请求时序

```
WallabagClient::request() 调用
  │
  ├─ loginIfRequired() → true
  │   └─ login() → 组 B 请求 #1（POST 到 loginUri）
  │
  ├─ 注入 Cookie
  ├─ 组 A 请求 #1（L45，带 Cookie）
  │
  ├─ loginIfRequested() → true
  │   └─ login() → 组 B 请求 #2（POST 到 loginUri）
  │
  ├─ 注入新 Cookie
  └─ 组 A 请求 #2（L57，带新 Cookie）
```

---

## 三、response->getInfo('url') 重定向边界影响

### 3.0 代码证据

```php
// src/HttpClient/Authenticator.php:52-54
public function loginIfRequested(ResponseInterface $response): bool
{
    // ⚠️ 使用响应的最终 URL，而非原始请求 URL
    $config = $this->buildSiteConfig(new Uri($response->getInfo('url')));
    // ...
}
```

### 3.1 Symfony HttpClient getInfo('url') 行为

根据 Symfony HttpClient 契约：
- `$response->getInfo('url')` 返回**最终重定向后的 URL**
- 如果请求经历了 3xx 重定向，返回的是最后一个 URL
- 而非调用 `request()` 时传入的原始 URL

### 3.2 host 变更的边界场景

**场景：跨主机重定向**
```
原始请求 URL: https://example.org/article/123
  ↓ 302 重定向（host 变化）
最终 URL: https://login.example.org/login?redirect=/article/123
```

**对 buildSiteConfig 的影响**：
```php
// 原始 host: example.org
// 实际传入 buildSiteConfig 的 host: login.example.org
$config = $this->buildSiteConfig(new Uri($response->getInfo('url')));
```

### 3.3 边界风险清单

| 风险类型 | 场景 | 影响 | 代码依据 |
|----------|------|------|----------|
| **配置不匹配** | 重定向到子域名（`example.org` → `login.example.org`） | 找不到匹配的站点配置，返回 false | `GrabySiteConfigBuilder.php:54` |
| **配置错误** | 重定向到完全不同的域名（`a.com` → `auth.b.com`） | 使用 b.com 的配置检测 a.com 的登录状态 | `Authenticator.php:54` |
| **XPath 失效** | 重定向后的页面结构与配置不匹配 | `notLoggedInXpath` 无法匹配，误判为已登录 | `LoginFormAuthenticator.php:69` |
| **host 不匹配** | `isLoggedIn()` 检查 `$siteConfig->getHost()` 与 Cookie domain | Cookie 是针对原始 host 的，但 `$siteConfig->getHost()` 是重定向后的 host | `LoginFormAuthenticator.php:48` |

### 3.4 风险链详解（host 不匹配场景）

```
原始请求 URL: https://paywall.example.org/article
  └─ loginIfRequired()
      └─ buildSiteConfig('paywall.example.org') → 返回配置 A
          └─ login() → Cookie domain 设为 .paywall.example.org
          └─ 组 A 请求 #1: https://paywall.example.org/article（带 Cookie）
              └─ 302 重定向到 https://auth.example.org/login
                  └─ 最终 URL: https://auth.example.org/login
  └─ loginIfRequested($response)
      └─ $response->getInfo('url') → https://auth.example.org/login
          └─ buildSiteConfig('auth.example.org') → 返回配置 B（不同配置！）
              └─ isLoggedIn($configB)
                  └─ 检查 Cookie domain === 'auth.example.org'
                      └─ ❌ 实际 Cookie domain 是 .paywall.example.org
                      └─ 返回 false
                          └─ isLoginRequired() 检测
                              └─ 使用配置 B 的 notLoggedInXpath 检测 auth.example.org 的页面
```

---

## 四、withOptions 未应用 options 的可靠性评注

### 4.0 代码证据

```php
// src/HttpClient/WallabagClient.php:65-68
public function withOptions(array $options): HttpClientInterface
{
    // ⚠️ 完全忽略传入的 $options 参数！
    return new self($this->restrictedAccess, $this->browser, $this->authenticator, $this->logger);
}
```

### 4.1 违反接口契约

`HttpClientInterface::withOptions()` 的契约要求：
> Returns a new instance with the specified options replacing the previous ones.

**实际行为**：创建新实例时，完全忽略传入的 `$options`，所有选项保持不变。

### 4.2 对链路分析的可靠性影响

| 影响类型 | 具体场景 | 分析偏差 |
|----------|----------|----------|
| **超时配置丢失** | Graby 通过 `withOptions(['timeout' => 30])` 设置超时 | 实际仍使用构造函数中的 `timeout: 10` |
| **Headers 丢失** | 调用方通过 `withOptions(['headers' => ['X-Custom' => 'value']])` | 自定义 headers 不会被发送 |
| **代理配置丢失** | 通过 `withOptions(['proxy' => '...'])` 设置代理 | 代理不生效 |
| **认证配置丢失** | 通过 `withOptions(['auth_basic' => '...'])` | HTTP Basic 认证不生效 |

### 4.3 构造函数的硬编码配置

```php
// src/HttpClient/WallabagClient.php:22-24
$this->httpClient = HttpClient::create([
    'timeout' => 10,  // ⚠️ 硬编码，无法通过 withOptions 修改
]);
```

### 4.4 可靠性评注

> **链路分析风险等级：中高**
>
> 由于 `withOptions()` 不生效，所有基于该方法的配置调整都会被静默丢弃。在进行链路分析时：
> 1. 不能假设调用方通过 `withOptions()` 设置的配置会生效
> 2. 必须检查构造函数中的硬编码配置
> 3. 如果 Graby 或其他调用方依赖 `withOptions()` 调整行为，实际行为会与预期不符
> 4. 这降低了链路行为的可预测性和可调试性

---

## 五、完整路径 + 边界风险可复核清单

### 5.1 路径决策树（含边界风险标注）

```
WallabagClient::request($method, $url, $options)
│
├─ [L31] restrictedAccess === 0?
│   ├─ YES → 路径 0
│   │   └─ [L32] 组 A 请求 #1 → 返回响应
│   │
│   └─ NO ↓
│
├─ [L35] loginIfRequired($url)
│   │
│   ├─ buildSiteConfig($url_host)
│   │   ├─ 无用户 → false [L36]
│   │   ├─ 无凭证 → false [L54]
│   │   └─ 有用户+凭证 → SiteConfig
│   │
│   ├─ false === $config || !requiresLogin() → false [L38]
│   ├─ isLoggedIn($config) → true → false [L42]
│   │   ⚠️  边界风险 A3：已有 Cookie 但后续不注入
│   │
│   └─ 返回 false → 路径 A
│   │   └─ [L38] 组 A 请求 #1（无 Cookie）→ 返回响应
│   │       ⚠️  边界风险：即使 Cookie jar 中有 Cookie 也不注入
│   │
│   └─ 返回 true → 继续
│       └─ [L47] 组 B 请求 #1（login() POST）
│
├─ [L41-42] 注入 Cookie 到 options
│
├─ [L45] 组 A 请求 #1（带 Cookie）
│   ⚠️  边界风险：响应可能被重定向到不同 host
│
├─ [L47] loginIfRequested($response)
│   │
│   ├─ buildSiteConfig($response->getInfo('url'))
│   │   ⚠️  边界风险：重定向后 host 变更导致配置不匹配
│   │
│   ├─ false === $config || !requiresLogin() → false [L58]
│   ├─ '' === $body → false [L66]
│   ├─ !isLoginRequired() → false [L74]
│   │   ⚠️  边界风险：XPath 可能不匹配重定向后的页面
│   │
│   └─ 返回 false → 路径 B1
│   │   └─ [L50] 返回组 A 请求 #1 的响应
│   │
│   └─ 返回 true → 路径 B2
│       └─ [L77] 组 B 请求 #2（login() POST 重试）
│
├─ [L53-54] 注入新 Cookie 到 options
│
└─ [L57] 组 A 请求 #2（带新 Cookie）→ 返回响应
```

### 5.2 四条路径的边界风险对比表

| 路径 | 组 A 请求 | 组 B 请求 | 主要风险 | 风险等级 | 可复核代码位置 |
|------|----------|-----------|----------|----------|----------------|
| **0** | 1 | 0 | 无（直接绕过所有登录逻辑） | 低 | L31-32 |
| **A** | 1 | 0 | ⚠️ A3 场景：已有 Cookie 但不注入 | 中高 | L35, L37-38 |
| **B1** | 1 | 1 | 重定向 host 变更导致配置不匹配 | 中 | L45, `Authenticator.php:54` |
| **B2** | 2 | 2 | 重定向 + 两次登录 + Cookie 域不匹配 | 高 | L45, L47, L57, `Authenticator.php:54` |

### 5.3 边界风险核对清单（可逐项验证）

#### ✅ 风险 1：loginIfRequired=false 不注入 Cookie
- [ ] 确认 L38 之前没有调用 `getCookieHeader()`
- [ ] 确认 `isLoggedIn()` 返回 true 时 `loginIfRequired()` 返回 false
- [ ] 确认路径 A 与路径 B1 的 cookie 注入逻辑差异

#### ✅ 风险 2：请求次数口径混淆
- [ ] 确认 WallabagClient 使用 `HttpClient::create()` 创建独立客户端（L22-24）
- [ ] 确认 LoginFormAuthenticator 使用注入的 `HttpBrowser`（`LoginFormAuthenticator.php:15`）
- [ ] 确认 `login()` 内部使用 `$this->browser->request()`（`LoginFormAuthenticator.php:34`）
- [ ] 确认组 A 与组 B 的 Cookie jar 是共享的（通过 `$this->browser->getCookieJar()`）

#### ✅ 风险 3：response->getInfo('url') 重定向影响
- [ ] 确认 `loginIfRequested` 使用 `$response->getInfo('url')` 而非原始 URL（`Authenticator.php:54`）
- [ ] 确认 Symfony HttpClient `getInfo('url')` 返回最终重定向 URL
- [ ] 确认 `buildSiteConfig` 使用响应 URL 的 host 而非原始 host
- [ ] 确认 `isLoggedIn()` 检查 `$siteConfig->getHost()` 与 Cookie domain（`LoginFormAuthenticator.php:48`）

#### ✅ 风险 4：withOptions 不生效
- [ ] 确认 `withOptions()` 方法签名包含 `$options` 参数（L65）
- [ ] 确认构造新实例时未使用 `$options`（L67）
- [ ] 确认 `$this->httpClient` 在构造函数中硬编码 `timeout: 10`（L23）
- [ ] 确认 `HttpClientInterface::withOptions()` 契约要求应用 options

### 5.4 可观测指标验证清单

| 指标 | 路径 0 | 路径 A | 路径 B1 | 路径 B2 | 验证方法 |
|------|--------|--------|---------|---------|----------|
| 组 A 请求次数 | 1 | 1 | 1 | 2 | 日志或网络抓包 |
| 组 B 请求次数 | 0 | 0 | 1 | 2 | 日志或网络抓包 |
| 业务请求带 Cookie | ❌ 否 | ❌ 否 | ✅ 是 | ✅ 是 | 检查请求 headers |
| loginIfRequired 调用 | ❌ 否 | ✅ 是 | ✅ 是 | ✅ 是 | 日志 |
| loginIfRequested 调用 | ❌ 否 | ❌ 否 | ✅ 是 | ✅ 是 | 日志 |
| login() 执行次数 | 0 | 0 | 1 | 2 | 日志 |
| 可能重试登录 | ❌ 否 | ❌ 否 | ❌ 否 | ✅ 是 | 路径 B2 特征 |
| Cookie 注入风险 | 无 | ⚠️ A3 | 无 | ⚠️ 重定向 | 风险分析 |

---

## 六、不可反驳的事实结论（最终校准版）

### 6.1 路径与边界

1. **4 条互斥路径**：路径 0（受限访问关闭）、路径 A（不需登录）、路径 B1（登录成功）、路径 B2（登录重试）。
2. **路径 0 是默认路径**：`restrictedAccess === 0` 时直接绕过所有登录逻辑，是绝大多数场景。
3. **路径 A 不注入 Cookie**：`loginIfRequired=false` 时，即使 Cookie jar 中有 Cookie 也不注入业务请求。
4. **A3 边界风险**：`isLoggedIn()` 返回 true 导致 `loginIfRequired=false`，后续请求不带 Cookie。
5. **loginIfRequested 仅条件性执行**：只有 `loginIfRequired=true` 时才会被调用。

### 6.2 请求统计

6. **两组独立请求指标**：组 A（业务 HTTP 请求，WallabagClient 内部）与组 B（登录 POST 请求，LoginFormAuthenticator 内部）是完全独立的 HTTP 客户端。
7. **组 A 请求位置**：L32（路径 0）、L38（路径 A）、L45（B1/B2 第一次）、L57（B2 第二次）。
8. **组 B 请求位置**：`Authenticator.php:47`（loginIfRequired 内部）、`Authenticator.php:77`（loginIfRequested 内部）。
9. **Cookie jar 共享**：两组请求共享同一个 Cookie jar，但客户端实例独立。

### 6.3 边界风险

10. **重定向 host 变更风险**：`loginIfRequested` 使用 `$response->getInfo('url')`，重定向后 host 变更可能导致站点配置不匹配。
11. **XPath 不匹配风险**：重定向后的页面结构可能与站点配置中的 `notLoggedInXpath` 不匹配。
12. **Cookie domain 不匹配风险**：`isLoggedIn()` 检查使用重定向后的 host，可能与实际 Cookie domain 不一致。

### 6.4 可靠性

13. **withOptions 不生效**：`withOptions()` 方法完全忽略传入的 `$options` 参数，违反 `HttpClientInterface` 契约。
14. **硬编码配置**：`timeout: 10` 在构造函数中硬编码，无法通过 `withOptions()` 修改。
15. **分析可靠性影响**：所有通过 `withOptions()` 设置的配置都会被静默丢弃，降低链路行为的可预测性。
