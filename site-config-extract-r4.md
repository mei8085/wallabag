# 站点配置驱动正文抽取流程分析（R4 - 路径拆分版）

> 本版本专注于 WallabagClient::request() 的路径拆分与 R3 总结的易误读句子修正。
> 所有路径均附带精确代码位置与可观察结果。

---

## 一、WallabagClient::request() 完整路径拆分

### 1.0 方法完整代码

```php
// src/HttpClient/WallabagClient.php:27-58
public function request(string $method, string $url, array $options = []): ResponseInterface
{
    $this->logger->log('debug', 'Restricted access config enabled?', ['enabled' => (int) $this->restrictedAccess]);

    if (0 === (int) $this->restrictedAccess) {
        return $this->httpClient->request($method, $url, $options);           // L32
    }

    $login = $this->authenticator->loginIfRequired($url);                      // L35

    if (!$login) {
        return $this->httpClient->request($method, $url, $options);           // L38
    }

    if (null !== $cookieHeader = $this->getCookieHeader($url)) {              // L41
        $options['headers']['cookie'] = $cookieHeader;                         // L42
    }

    $response = $this->httpClient->request($method, $url, $options);          // L45

    $login = $this->authenticator->loginIfRequested($response);               // L47

    if (!$login) {
        return $response;                                                     // L50
    }

    if (null !== $cookieHeader = $this->getCookieHeader($url)) {              // L53
        $options['headers']['cookie'] = $cookieHeader;                         // L54
    }

    return $this->httpClient->request($method, $url, $options);               // L57
}
```

### 1.1 路径 0：受限访问关闭

**触发条件**：`restrictedAccess === 0`

**执行路径**：
```
L29: 日志记录
L31: 检查 restrictedAccess === 0 → true
L32: 直接发起 HTTP 请求 → 返回响应
```

**证据点**：
| 行号 | 代码 | 作用 |
|------|------|------|
| L31 | `if (0 === (int) $this->restrictedAccess)` | 开关检查 |
| L32 | `return $this->httpClient->request(...)` | 直接请求并返回 |

**可观察结果**：
| 指标 | 值 |
|------|-----|
| HTTP 请求次数 | **1 次** |
| 是否带 Cookie | **否** |
| 是否调用 loginIfRequired | **否** |
| 是否调用 loginIfRequested | **否** |
| 是否可能执行登录 | **否** |

**典型场景**：wallabag 实例未开启受限访问功能（绝大多数情况）。

---

### 1.2 路径 A：受限访问开启 + loginIfRequired 返回 false

**触发条件**：`restrictedAccess !== 0` 且 `loginIfRequired($url) === false`

**执行路径**：
```
L29: 日志记录
L31: 检查 restrictedAccess === 0 → false
L35: 调用 loginIfRequired($url) → 返回 false
L37: 检查 $login → false
L38: 直接发起 HTTP 请求 → 返回响应
```

**证据点**：
| 行号 | 代码 | 作用 |
|------|------|------|
| L35 | `$login = $this->authenticator->loginIfRequired($url)` | 检查是否需要登录 |
| L37-38 | `if (!$login) { return ... }` | 不需要登录，直接请求 |

**loginIfRequired 返回 false 的 3 种子原因**：

| 子原因 | 代码位置 | 说明 |
|--------|----------|------|
| `buildSiteConfig()` 返回 false | `Authenticator.php:35` | 无当前用户或无对应站点凭证 |
| `$config->requiresLogin() === false` | `Authenticator.php:35` | 站点配置标记为不需要登录 |
| `$this->authenticator->isLoggedIn($config) === true` | `Authenticator.php:41` | Cookie jar 中已有该域名的 Cookie |

**可观察结果**：
| 指标 | 值 |
|------|-----|
| HTTP 请求次数 | **1 次** |
| 是否带 Cookie | **否** |
| 是否调用 loginIfRequired | **是**（返回 false） |
| 是否调用 loginIfRequested | **否** |
| 是否可能执行登录 | **否** |

**典型场景**：
- 用户未配置受限访问站点的凭证
- 站点配置标记为不需要登录
- 已有有效 Cookie（之前登录过）

---

### 1.3 路径 B1：受限访问开启 + loginIfRequired=true + loginIfRequested=false

**触发条件**：`restrictedAccess !== 0` 且 `loginIfRequired($url) === true` 且 `loginIfRequested($response) === false`

**执行路径**：
```
L29: 日志记录
L31: 检查 restrictedAccess === 0 → false
L35: 调用 loginIfRequired($url) → 返回 true
      │
      └─ 执行 login()：POST 到 loginUri（这是 loginIfRequired 内部的副作用）
L41-42: 附加 Cookie 到请求头
L45: 发起第一次 HTTP 请求（带 Cookie）
L47: 调用 loginIfRequested($response) → 返回 false
L49-50: 返回第一次请求的响应
```

**证据点**：
| 行号 | 代码 | 作用 |
|------|------|------|
| L35 | `loginIfRequired($url)` | 返回 true，且内部已执行 login() |
| L41-42 | 附加 Cookie | 将 login() 产生的 Cookie 附加到请求 |
| L45 | 第一次 HTTP 请求 | 带 Cookie 的请求 |
| L47 | `loginIfRequested($response)` | 检查响应是否为登录页 |
| L49-50 | `return $response` | 返回第一次请求的响应 |

**loginIfRequired 返回 true 的必要条件**（全部满足）：
1. `buildSiteConfig()` 返回 SiteConfig 对象（有用户+有凭证）
2. `$config->requiresLogin() === true`
3. `$this->authenticator->isLoggedIn($config) === false`（无 Cookie）

**loginIfRequested 返回 false 的 4 种子原因**：

| 子原因 | 代码位置 | 说明 |
|--------|----------|------|
| `buildSiteConfig()` 返回 false | `Authenticator.php:54` | 无用户或无凭证（理论上不应发生，因为 L35 已通过） |
| `$config->requiresLogin() === false` | `Authenticator.php:55` | 站点配置标记为不需要登录（理论上不应发生） |
| `$body === ''` | `Authenticator.php:63` | 响应体为空 |
| `$isLoginRequired === false` | `Authenticator.php:73` | XPath 未检测到登录页面元素（即已成功登录） |

**可观察结果**：
| 指标 | 值 |
|------|-----|
| HTTP 请求次数 | **1 次**（L45） |
| login() 调用次数 | **1 次**（L35 内部） |
| 是否带 Cookie | **是**（L41-42 附加） |
| 是否调用 loginIfRequired | **是**（返回 true） |
| 是否调用 loginIfRequested | **是**（返回 false） |
| 是否可能重试登录 | **否** |

**典型场景**：
- 用户配置了受限访问站点的凭证
- 首次访问：loginIfRequired 检测到无 Cookie，执行登录
- 登录成功后：带 Cookie 请求目标页面
- 响应正常（不是登录页）：loginIfRequested 返回 false
- 直接返回响应

---

### 1.4 路径 B2：受限访问开启 + loginIfRequired=true + loginIfRequested=true

**触发条件**：`restrictedAccess !== 0` 且 `loginIfRequired($url) === true` 且 `loginIfRequested($response) === true`

**执行路径**：
```
L29: 日志记录
L31: 检查 restrictedAccess === 0 → false
L35: 调用 loginIfRequired($url) → 返回 true
      │
      └─ 执行 login()：POST 到 loginUri（第 1 次登录）
L41-42: 附加 Cookie 到请求头
L45: 发起第一次 HTTP 请求（带 Cookie）
L47: 调用 loginIfRequested($response) → 返回 true
      │
      └─ 执行 login()：POST 到 loginUri（第 2 次登录）
L53-54: 附加新 Cookie 到请求头（登录后 Cookie 可能更新）
L57: 发起第二次 HTTP 请求（带新 Cookie）
      │
      └─ 返回第二次请求的响应
```

**证据点**：
| 行号 | 代码 | 作用 |
|------|------|------|
| L35 | `loginIfRequired($url)` | 返回 true，且内部已执行第 1 次 login() |
| L41-42 | 附加 Cookie | 将第 1 次 login() 产生的 Cookie 附加到请求 |
| L45 | 第一次 HTTP 请求 | 带 Cookie 的请求 |
| L47 | `loginIfRequested($response)` | 返回 true，且内部已执行第 2 次 login() |
| L53-54 | 附加新 Cookie | 将第 2 次 login() 产生的新 Cookie 附加到请求 |
| L57 | 第二次 HTTP 请求 | 带新 Cookie 的请求 |

**loginIfRequested 返回 true 的必要条件**（全部满足）：
1. `buildSiteConfig()` 返回 SiteConfig 对象
2. `$config->requiresLogin() === true`
3. `$body !== ''`
4. `$isLoginRequired === true`（XPath 检测到登录页面元素）

**可观察结果**：
| 指标 | 值 |
|------|-----|
| HTTP 请求次数 | **2 次**（L45 + L57） |
| login() 调用次数 | **2 次**（L35 内部 + L47 内部） |
| 第一次请求是否带 Cookie | **是** |
| 第二次请求是否带 Cookie | **是**（可能是新 Cookie） |
| 是否调用 loginIfRequired | **是**（返回 true） |
| 是否调用 loginIfRequested | **是**（返回 true） |
| 是否可能重试登录 | **是**（第 2 次登录在 L47 内部执行） |

**典型场景**：
- 某些站点的登录流程需要两次请求才能完成（如先获取 CSRF token 再提交表单）
- 或者第一次登录失败，响应仍然是登录页
- loginIfRequested 检测到仍是登录页，执行第二次登录
- 第二次登录成功后，带新 Cookie 请求目标页面

---

## 二、四条路径的对比与决策树

### 2.1 路径决策树

```
WallabagClient::request()
│
├─ restrictedAccess === 0?
│   ├─ YES → 路径 0：直接请求（1 次，无 Cookie）
│   └─ NO ↓
│
├─ loginIfRequired() 返回?
│   ├─ false → 路径 A：直接请求（1 次，无 Cookie）
│   └─ true ↓（已执行第 1 次 login）
│
├─ 执行第一次请求（带 Cookie）
│
├─ loginIfRequested() 返回?
│   ├─ false → 路径 B1：返回第一次响应（1 次，带 Cookie）
│   └─ true ↓（已执行第 2 次 login）
│
└─ 路径 B2：执行第二次请求（带新 Cookie）→ 返回第二次响应
```

### 2.2 路径特征对比表

| 特征 | 路径 0 | 路径 A | 路径 B1 | 路径 B2 |
|------|--------|--------|---------|---------|
| restrictedAccess | === 0 | !== 0 | !== 0 | !== 0 |
| loginIfRequired | 未调用 | false | true | true |
| 第 1 次 login() | 未执行 | 未执行 | ✅ 执行 | ✅ 执行 |
| 第一次 HTTP 请求 | ✅（无 Cookie） | ✅（无 Cookie） | ✅（带 Cookie） | ✅（带 Cookie） |
| loginIfRequested | 未调用 | 未调用 | false | true |
| 第 2 次 login() | 未执行 | 未执行 | 未执行 | ✅ 执行 |
| 第二次 HTTP 请求 | 无 | 无 | 无 | ✅（带新 Cookie） |
| **总请求次数** | **1** | **1** | **1** | **2** |
| **总 login 次数** | **0** | **0** | **1** | **2** |
| 代码返回位置 | L32 | L38 | L50 | L57 |

### 2.3 loginIfRequired 内部执行逻辑（路径 B1/B2 的前提）

```php
// src/HttpClient/Authenticator.php:32-50
public function loginIfRequired(string $url): bool
{
    $config = $this->buildSiteConfig(new Uri($url));     // L34

    // 子原因 1+2：无配置 或 不需要登录 → 返回 false
    if (false === $config || !$config->requiresLogin()) { // L35
        return false;                                      // L38
    }

    // 子原因 3：已有 Cookie → 返回 false
    if ($this->authenticator->isLoggedIn($config)) {      // L41
        return false;                                      // L42
    }

    // 执行登录（副作用）
    $this->authenticator->login($config);                 // L47

    return true;                                           // L49
}
```

**关键**：`login()` 在 L47 执行，是 `loginIfRequired` 返回 true 的副作用。即使 `loginIfRequested` 后续返回 true 并再次调用 `login()`，L47 的登录已经执行过。

### 2.4 loginIfRequested 内部执行逻辑

```php
// src/HttpClient/Authenticator.php:52-80
public function loginIfRequested(ResponseInterface $response): bool
{
    $config = $this->buildSiteConfig(new Uri($response->getInfo('url')));  // L54

    // 子原因 1+2：无配置 或 不需要登录 → 返回 false
    if (false === $config || !$config->requiresLogin()) {                   // L55
        return false;                                                        // L58
    }

    $body = $response->getContent();                                        // L61

    // 子原因 3：响应体为空 → 返回 false
    if ('' === $body) {                                                     // L63
        return false;                                                        // L66
    }

    $isLoginRequired = $this->authenticator->isLoginRequired($config, $body); // L69

    // 子原因 4：XPath 未匹配 → 返回 false
    if (!$isLoginRequired) {                                                // L73
        return false;                                                        // L74
    }

    // 执行第二次登录（副作用）
    $this->authenticator->login($config);                                   // L77

    return true;                                                             // L79
}
```

**关键**：`login()` 在 L77 执行，是 `loginIfRequested` 返回 true 的副作用。

---

## 三、R3 总结中容易误读句子的修正

### 3.1 容易误读句子清单与修正

#### ❌ R3 原文（易误读）：
> 主链时序图中 "Authenticator::loginIfRequired()" 和 "Authenticator::loginIfRequested()" 被列为必经步骤

✅ **修正后表述**：

```
Graby::fetchContent($url)
    └─ WallabagClient::request()
        │
        ├─ [条件分支] restrictedAccess === 0
        │   └─ 直接请求，不经过任何登录检查
        │
        └─ [条件分支] restrictedAccess !== 0
            │
            ├─ loginIfRequired()
            │   ├─ [条件分支] 返回 false → 直接请求
            │   └─ [条件分支] 返回 true → 执行登录 + 带 Cookie 请求
            │
            └─ [仅当 loginIfRequired=true 时才执行]
                loginIfRequested()
                ├─ [条件分支] 返回 false → 返回第一次响应
                └─ [条件分支] 返回 true → 再次登录 + 第二次请求
```

**修正理由**：原时序图将 `loginIfRequired` 和 `loginIfRequested` 画成必经步骤，容易让读者误以为所有请求都会经过这些检查。实际上它们是条件性执行的。

---

#### ❌ R3 原文（易误读）：
> "Authenticator::loginIfRequired() → buildSiteConfig() → GrabySiteConfigBuilder::buildForHost()" 被画成线性必经路径

✅ **修正后表述**：

```
WallabagClient::request()
│
├─ [开关] restrictedAccess === 0 → 完全绕过 Authenticator
│
└─ [开关打开时才可能进入] Authenticator 层
    │
    ├─ loginIfRequired()
    │   └─ buildSiteConfig()
    │       └─ GrabySiteConfigBuilder::buildForHost()
    │           ├─ [条件] 无用户 → false
    │           ├─ [条件] 无凭证 → false
    │           └─ [条件] 有用户+凭证 → SiteConfig
    │
    └─ [仅当 loginIfRequired=true 时才进入] loginIfRequested()
        └─ buildSiteConfig()  [再次调用，相同逻辑]
```

**修正理由**：原表述未明确 `restrictedAccess` 开关的前置作用，也未说明 `loginIfRequested` 是条件性执行的。

---

#### ❌ R3 原文（易误读）：
> 主链时序图中将登录链画成 Graby::fetchContent → WallabagClient::request → Authenticator → ... 的线性路径

✅ **修正后表述**：

```
ContentProxy::updateEntry()
│
└─ Graby::fetchContent($url)
    │
    └─ WallabagClient::request()
        │
        ├─ [路径 0] restrictedAccess=0 → 直接请求（最常见）
        │
        ├─ [路径 A] 受限访问 + 不需登录 → 直接请求
        │
        ├─ [路径 B1] 受限访问 + 需登录 + 登录成功 → 1 次请求
        │
        └─ [路径 B2] 受限访问 + 需登录 + 需重试 → 2 次请求
```

**修正理由**：原图将登录链画成必经路径，实际上路径 0（受限访问关闭）是绝大多数场景的实际执行路径。

---

#### ❌ R3 原文（易误读）：
> "无用户 → return false" 和 "无凭证 → return false" 被列在主链时序图中

✅ **修正后表述**：

这些分支仅在 `restrictedAccess !== 0` **且** `loginIfRequired()` 被调用时才会被触发。在路径 0 和路径 A 中，这些分支不会被执行。

**修正理由**：原图将这些条件分支画成必经步骤，容易让读者误以为每次请求都会检查用户和凭证。

---

### 3.2 R3 主链时序图的修正版

```
用户提交 URL
  │
  ▼
EntryController::addEntryFormAction()
  ├─ 查重 findByUrlAndUserId()
  └─ updateEntry() [私有方法]
      └─ ContentProxy::updateEntry($entry, $url, [], false)
          ├─ [条件] empty($content) && !$disableContentUpdate
          │   └─ Graby::fetchContent($url)
          │       └─ WallabagClient::request()
          │           │
          │           ├─ [路径 0] restrictedAccess=0 → 直接请求（最常见）
          │           │
          │           ├─ [路径 A] 受限访问 + 不需登录 → 直接请求
          │           │
          │           ├─ [路径 B1] 受限访问 + 需登录 + 登录成功 → 1 次请求
          │           │   └─ loginIfRequired() → login() → 带 Cookie 请求
          │           │       └─ loginIfRequested() → false → 返回响应
          │           │
          │           └─ [路径 B2] 受限访问 + 需登录 + 需重试 → 2 次请求
          │               └─ loginIfRequired() → login() → 带 Cookie 请求
          │                   └─ loginIfRequested() → true → 再次 login() → 新 Cookie 请求
          │
          ├─ 标题编码清洗
          ├─ [条件] 失败回退：有导入内容且抓取失败 → 保留导入内容
          ├─ 设置 given_url
          └─ stockEntry()
              ├─ updateOriginUrl() [5 分支，条件性执行]
              ├─ [条件] 正文为空 → 设置错误消息 + not_parsed 标记
              ├─ [条件] 有 description → 附加到错误消息
              ├─ 阅读时间计算
              ├─ [条件] 有 status → 设置 HTTP 状态码
              ├─ [条件] 有 authors → 设置作者
              ├─ [条件] 有 headers → 存储响应头
              ├─ [条件] 有 date → 解析发布日期
              ├─ [条件] 有 language → 验证并设置语言
              ├─ [条件] 有 image → 提取预览图
              ├─ [条件] 内容是图片类型 → 直接作为预览图
              ├─ [条件] 预览图为空 → 从正文中提取第一张
              ├─ [条件] 有 content-type → 设置 MIME 类型
              ├─ [条件] 有预览图 URL → 验证并设置
              └─ [条件] 自动打标签（try-catch 保护）
  │
  ▼
persist + flush
  │
  ▼
[条件] dispatch(EntrySavedEvent)
  │
  ├─ [路径 0] UrlCommand 单 URL 导入 → 不触发
  │
  └─ [路径 1] 其他 15 个触发点 → 同步执行
      └─ DownloadImagesSubscriber::onEntrySaved()
          ├─ [条件] download_images_enabled 关闭 → 跳过
          ├─ [条件] 启用 → processHtml() 下载正文图片
          ├─ [条件] 启用 → processSingleImage() 下载预览图
          └─ persist + flush 更新内容
  │
  ▼
返回响应
```

**修正说明**：
- 所有条件性分支均标注 `[条件]` 前缀
- 登录链的 4 条路径明确区分，路径 0 标注为"最常见"
- stockEntry 中的条件性字段处理明确标注
- EntrySavedEvent 的触发条件明确标注

---

## 四、事实校准后的不可反驳结论

### 4.1 WallabagClient::request() 路径

1. **4 条独立路径**：根据 `restrictedAccess`、`loginIfRequired`、`loginIfRequested` 的返回值组合，共有 4 条互斥执行路径。
2. **路径 0 是默认路径**：`restrictedAccess === 0` 时，直接绕过所有登录逻辑，是绝大多数场景的实际执行路径。
3. **路径 B2 是最复杂路径**：需要两次 HTTP 请求和两次 login() 调用，仅在登录流程需要重试时触发。
4. **loginIfRequested 是条件性的**：只有 `loginIfRequired` 返回 true 时才会被调用，不是必经步骤。

### 4.2 登录链边界

5. **GrabySiteConfigBuilder 仅在登录链中调用**：正文抽取主链由 Graby 内部处理，不经过 Wallabag 的站点配置构建器。
6. **开关在 WallabagClient 层**：`restrictedAccess` 检查在最外层，关闭时完全绕过 Authenticator。
7. **配置构建失败条件**：无当前用户 **或** 无对应站点凭证时，`buildForHost()` 返回 false。

### 4.3 内容处理

8. **validateContent 只检查 3 个字段**：title、html、url，与 AbstractImport 注释中的 5 个字段不一致。
9. **抓取回退是保护性设计**：有导入内容时，若 Graby 抓取失败，保留导入内容。
10. **updateOriginUrl 有 5 个分支**：忽略规则匹配时不保存 origin_url。

### 4.4 后处理

11. **图片处理同步执行**：@todo 注释标记未来可改为异步。
12. **EntrySavedEvent 有 15 个触发点**：CLI 单 URL 导入是唯一例外。
