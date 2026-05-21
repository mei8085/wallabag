# 站点配置驱动正文抽取流程分析（R3 - 事实校准版）

> 本版本专注于事实校准与证据补全，所有结论均附带精确代码位置，避免推断性描述。
> 针对 R2 文档的关键错误修正已用 ⚠️ **R2 修正** 标记。

---

## 查证清单与逐条校对

### ✅ 断言 1：GrabySiteConfigBuilder 调用边界

| 断言 | 代码证据 | 反例说明 |
|------|----------|----------|
| `GrabySiteConfigBuilder::buildForHost()` 仅在受限访问登录链中被调用 | **唯一调用点**：`Authenticator::buildSiteConfig()` → `src/HttpClient/Authenticator.php:87` | 在正文抽取主链 `ContentProxy::updateEntry()` 中，`GrabySiteConfigBuilder` 不被直接调用，Graby 内部使用自己的 `Graby\SiteConfig\ConfigBuilder` → `src/Helper/ContentProxy.php:50-51` |

⚠️ **R2 修正**：R2 文档称 `buildSiteConfig()` 内部有 `restrictedAccess === 0` 检查，**实际不存在**。该检查位于 `WallabagClient::request()` 中。

---

### ✅ 断言 2：Authenticator::buildSiteConfig 实际分支条件

#### 2.1 WallabagClient 层的开关（上层）

```php
// src/HttpClient/WallabagClient.php:29-33
$this->logger->log('debug', 'Restricted access config enabled?', ['enabled' => (int) $this->restrictedAccess]);

if (0 === (int) $this->restrictedAccess) {
    return $this->httpClient->request($method, $url, $options);
}
```

**分支边界**：`restrictedAccess === 0` 时，直接绕过所有登录逻辑，`Authenticator` 的方法不会被调用。

#### 2.2 Authenticator::buildSiteConfig 实现（中间层）

```php
// src/HttpClient/Authenticator.php:85-88
private function buildSiteConfig(UriInterface $uri)
{
    return $this->configBuilder->buildForHost($uri->getHost());
}
```

**事实**：`buildSiteConfig()` 是一个**纯委托方法**，不包含任何条件判断，直接调用 `GrabySiteConfigBuilder::buildForHost()`。

#### 2.3 GrabySiteConfigBuilder::buildForHost 实际分支（底层）

```php
// src/SiteConfig/GrabySiteConfigBuilder.php:23-79
public function buildForHost($host)
{
    $user = $this->getUser();

    // 主机名规范化
    $host = strtolower($host);
    if (str_starts_with($host, 'www.')) {
        $host = substr($host, 4);
    }

    // 分支 1：无当前用户 → 返回 false
    if (!$user) {
        $this->logger->debug('Auth: no current user defined.');
        return false;  // src/SiteConfig/GrabySiteConfigBuilder.php:36
    }

    // 主机名候选列表（支持通配符 .example.org）
    $hosts = [$host];
    $split = explode('.', $host);
    if (\count($split) > 1) {
        array_shift($split);
        $hosts[] = '.' . implode('.', $split);
    }

    // 查询用户凭证
    $credentials = $this->credentialRepository->findOneByHostsAndUser($hosts, $user->getId());

    // 分支 2：无凭证 → 返回 false
    if (null === $credentials) {
        $this->logger->debug('Auth: no credentials available for host.', ['host' => $host]);
        return false;  // src/SiteConfig/GrabySiteConfigBuilder.php:54
    }

    // 从 Graby 加载站点配置并合并凭证
    $config = $this->grabyConfigBuilder->buildForHost($host);
    $parameters = [
        'host' => $host,
        'requiresLogin' => $config->requires_login ?: false,
        'loginUri' => $config->login_uri ?: null,
        'usernameField' => $config->login_username_field ?: null,
        'passwordField' => $config->login_password_field ?: null,
        'extraFields' => $this->processExtraFields($config->login_extra_fields),
        'notLoggedInXpath' => $config->not_logged_in_xpath ?: null,
        'username' => $credentials['username'],
        'password' => $credentials['password'],
        'httpHeaders' => $config->http_header,
    ];

    // 分支 3：有用户和凭证 → 返回 SiteConfig 对象
    return new SiteConfig($parameters);  // src/SiteConfig/GrabySiteConfigBuilder.php:79
}
```

**分支真值表**：

| 条件 | 返回值 | 代码位置 |
|------|--------|----------|
| `!$user`（无当前登录用户） | `false` | `src/SiteConfig/GrabySiteConfigBuilder.php:36` |
| `null === $credentials`（无该站点凭证） | `false` | `src/SiteConfig/GrabySiteConfigBuilder.php:54` |
| 有用户 + 有凭证 | `SiteConfig` 对象 | `src/SiteConfig/GrabySiteConfigBuilder.php:79` |

#### 2.4 loginIfRequired 完整分支链

```php
// src/HttpClient/Authenticator.php:32-50
public function loginIfRequired(string $url): bool
{
    $config = $this->buildSiteConfig(new Uri($url));

    // 分支 A：配置为 false 或不需要登录 → 不登录
    if (false === $config || !$config->requiresLogin()) {
        $this->logger->debug('loginIfRequired> will not require login');
        return false;  // src/HttpClient/Authenticator.php:38
    }

    // 分支 B：已有有效 Cookie → 不登录
    if ($this->authenticator->isLoggedIn($config)) {
        return false;  // src/HttpClient/Authenticator.php:42
    }

    // 分支 C：需要登录且无 Cookie → 执行登录
    $this->logger->debug('loginIfRequired> user is not logged in, attach authenticator');
    $this->authenticator->login($config);
    return true;  // src/HttpClient/Authenticator.php:49
}
```

#### 2.5 loginIfRequested 完整分支链（R2 遗漏 body 为空检查）

```php
// src/HttpClient/Authenticator.php:52-80
public function loginIfRequested(ResponseInterface $response): bool
{
    $config = $this->buildSiteConfig(new Uri($response->getInfo('url')));

    // 分支 A：配置为 false 或不需要登录 → 不登录
    if (false === $config || !$config->requiresLogin()) {
        $this->logger->debug('loginIfRequested> will not require login');
        return false;  // src/HttpClient/Authenticator.php:58
    }

    $body = $response->getContent();

    // 分支 B：响应体为空 → 不登录（R2 遗漏此分支）
    if ('' === $body) {
        $this->logger->debug('loginIfRequested> empty body, ignoring');
        return false;  // src/HttpClient/Authenticator.php:66
    }

    $isLoginRequired = $this->authenticator->isLoginRequired($config, $body);
    $this->logger->debug('loginIfRequested> retry with login ' . ($isLoginRequired ? '' : 'not ') . 'required');

    // 分支 C：XPath 未检测到登录页面 → 不登录
    if (!$isLoginRequired) {
        return false;  // src/HttpClient/Authenticator.php:74
    }

    // 分支 D：检测到登录页面 → 执行登录
    $this->authenticator->login($config);
    return true;  // src/HttpClient/Authenticator.php:79
}
```

#### 2.6 isLoggedIn 实现（Cookie 检查，不调用服务器）

```php
// src/SiteConfig/LoginFormAuthenticator.php:44-54
public function isLoggedIn(SiteConfig $siteConfig)
{
    foreach ($this->browser->getCookieJar()->all() as $cookie) {
        // 只要存在该域名的 Cookie 即认为已登录
        if ($cookie->getDomain() === $siteConfig->getHost()) {
            return true;  // src/SiteConfig/LoginFormAuthenticator.php:49
        }
    }
    return false;
}
```

**反例说明**：该实现存在局限性 - 只要有 Cookie 就认为已登录，不验证 Cookie 是否有效或过期。

---

### ✅ 断言 3：validateContent 实现

⚠️ **R2 修正**：R2 文档中注释与实现不一致问题需要明确。

```php
// src/Helper/ContentProxy.php:395-403
/**
 * Validate that the given content has at least a title, an html and a url.
 *
 * @return bool true if valid otherwise false
 */
private function validateContent(array $content)
{
    return !empty($content['title']) && !empty($content['html']) && !empty($content['url']);
}
```

| 验证字段 | 是否检查 | 代码位置 |
|---------|----------|----------|
| `title` | ✅ 是 | `src/Helper/ContentProxy.php:402` |
| `html` | ✅ 是 | `src/Helper/ContentProxy.php:402` |
| `url` | ✅ 是 | `src/Helper/ContentProxy.php:402` |
| `language` | ❌ 否 | - |
| `content_type` | ❌ 否 | - |

**注释与实现差异**：
- `AbstractImport::fetchContent()` 注释说需要 `title, html, url, language & content_type` 五个字段 → `src/Import/AbstractImport.php:123`
- 实际 `validateContent()` 只检查 `title, html, url` 三个字段 → `src/Helper/ContentProxy.php:402`

---

### ✅ 断言 4：disableContentUpdate 分支逻辑

```php
// src/Helper/ContentProxy.php:50
if ((empty($content) || false === $this->validateContent($content)) && false === $disableContentUpdate) {
    $fetchedContent = $this->graby->fetchContent($url);
    // ...
}
```

**完整条件真值表**：

| `$content` 状态 | `validateContent()` | `$disableContentUpdate` | 是否调用 `Graby::fetchContent()` | 代码依据 |
|----------------|---------------------|--------------------------|----------------------------------|----------|
| empty | - | `false` | ✅ 是 | `empty($content) = true` → 条件成立 |
| non-empty | `false` | `false` | ✅ 是 | `validateContent() = false` → 条件成立 |
| non-empty | `true` | `false` | ❌ 否 | `empty = false` + `validateContent = true` → 条件不成立 |
| - | - | `true` | ❌ 否 | `disableContentUpdate = true` → 条件不成立 |

**各调用场景参数组合**：

| 调用方 | `$content` | `$disableContentUpdate` | 行为 | 代码位置 |
|--------|------------|--------------------------|------|----------|
| Web 新建 | `[]` | `false` | 调用 Graby 抓取 | `src/Controller/EntryController.php:708` |
| Web 重载 | `[]` | `false` | 调用 Graby 重新抓取 | `src/Controller/EntryController.php:708` |
| API POST 新建 | `['title' => ..., 'html' => ..., 'url' => ...]` | `false` | 若三者都非空则跳过抓取 | `src/Controller/Api/EntryRestController.php:741` |
| API PATCH 更新 | `['html' => $newContent]` | `true` | 跳过抓取，仅清洗 HTML | `src/Controller/Api/EntryRestController.php:949` |
| API PATCH 重载 | `[]` | `false` | 调用 Graby 重新抓取 | `src/Controller/Api/EntryRestController.php:1057` |
| Import 同步（默认） | 导入的元数据 | `false` | 若导入内容不完整则抓取 | `src/Import/AbstractImport.php:128` |
| Import 同步（设置后） | 导入的元数据 | `true` | 跳过抓取，使用导入内容 | `src/Import/AbstractImport.php:84` |
| Consumer 异步 | 队列消息中的数据 | `false` | 同上 | `src/Consumer/AbstractConsumer.php:56` |
| CLI 单 URL 导入 | `[]` | `false` | 调用 Graby 抓取 | `src/Command/Import/UrlCommand.php:90` |
| CLI 批量重载 | `[]` | `false` | 调用 Graby 重新抓取 | `src/Command/ReloadEntryCommand.php:92` |

---

### ✅ 断言 5：抓取失败回退分支

```php
// src/Helper/ContentProxy.php:58-62
// when content is imported, we have information in $content
// in case fetching content goes bad, we'll keep the imported information instead of overriding them
if (empty($content) || $fetchedContent['html'] !== $this->fetchingErrorMessage) {
    $content = $fetchedContent;
}
```

**回退逻辑真值表**：

| `$content` 状态 | `$fetchedContent['html']` | 结果 | 说明 | 代码依据 |
|----------------|--------------------------|------|------|----------|
| empty | 错误消息 | `$content = $fetchedContent` | 无导入内容，即使失败也用错误消息 | `empty($content) = true` → 条件成立 |
| empty | 正常内容 | `$content = $fetchedContent` | 正常抓取结果 | `empty($content) = true` → 条件成立 |
| non-empty | 错误消息 | **保留 `$content`** | ⚠️ 抓取失败，回退到导入内容 | 两个条件都不成立 |
| non-empty | 正常内容 | `$content = $fetchedContent` | 抓取成功，覆盖导入内容 | `html !== 错误消息` → 条件成立 |

**反例验证**：当有导入内容且抓取失败时，`$content` 不会被覆盖，这是有意设计的保护机制。

---

### ✅ 断言 6：updateOriginUrl 完整分支

```php
// src/Helper/ContentProxy.php:328-393
private function updateOriginUrl(Entry $entry, $url)
{
    // 分支 1：URL 相同或为空 → 不处理
    if (empty($url) || $entry->getUrl() === $url) {
        return false;  // src/Helper/ContentProxy.php:331
    }

    // 计算 URL 差异部分
    $parsed_entry_url = parse_url($entry->getUrl());
    $parsed_content_url = parse_url($url);
    $diff_ec = array_diff_assoc($parsed_entry_url, $parsed_content_url);
    $diff_ce = array_diff_assoc($parsed_content_url, $parsed_entry_url);
    $diff = array_merge($diff_ec, $diff_ce);
    $diff_keys = array_keys($diff);
    sort($diff_keys);

    // 分支 2：忽略规则匹配 → 直接更新 URL，不保存 origin_url
    if ($this->ignoreOriginProcessor->process($entry)) {
        $entry->setUrl($url);
        return false;  // src/Helper/ContentProxy.php:359
    }

    // 分支 3：根据差异部分执行不同策略
    switch ($diff_keys) {
        case ['path']:
            // 3a: 仅尾部斜杠差异 或 URL 编码差异 → 直接更新
            if (($parsed_entry_url['path'] . '/' === $parsed_content_url['path'])
                || ($url === urldecode($entry->getUrl()))) {
                $entry->setUrl($url);
            }
            // 其他 path 差异 → 不处理
            break;
        case ['scheme']:
            // 3b: 仅协议差异（http→https）→ 直接更新
            $entry->setUrl($url);  // src/Helper/ContentProxy.php:381
            break;
        case ['fragment']:
            // 3c: 仅锚点差异 → 不处理（noop）
            break;
        default:
            // 3d: 其他差异（host 变化、多部分变化等）
            // → 保存原 URL 到 origin_url，更新为新 URL
            if (empty($entry->getOriginUrl())) {
                $entry->setOriginUrl($entry->getUrl());
            }
            $entry->setUrl($url);  // src/Helper/ContentProxy.php:390
            break;
    }
}
```

**分支执行路径图**：

```
entry.url vs content.url
  ├─ 相同 或 url 为空 → 退出 [L331]
  ├─ ignoreOriginProcessor 匹配 → 直接更新 url，不存 origin_url [L359]
  └─ 有差异：
     ├─ 仅 path 差异：
     │   ├─ 尾部斜杠 / URL 编码差异 → 更新 url [L377]
     │   └─ 其他 path 差异 → 不处理
     ├─ 仅 scheme 差异 → 更新 url [L381]
     ├─ 仅 fragment 差异 → 不处理
     └─ 其他 → 保存 origin_url + 更新 url [L388-390]
```

**忽略规则处理器证据**：
```php
// src/Helper/RuleBasedIgnoreOriginProcessor.php:24-45
public function process(Entry $entry)
{
    $url = $entry->getUrl();
    $userRules = $entry->getUser()->getConfig()->getIgnoreOriginRules()->toArray();
    $rules = array_merge($this->ignoreOriginInstanceRuleRepository->findAll(), $userRules);

    $parsed_url = parse_url($url);
    $parsed_url['_all'] = $url;

    foreach ($rules as $rule) {
        if ($this->rulerz->satisfies($parsed_url, $rule->getRule())) {
            $this->logger->info('Origin url matching ignore rule.', ['rule' => $rule->getRule()]);
            return true;  // src/Helper/RuleBasedIgnoreOriginProcessor.php:40
        }
    }
    return false;
}
```

---

### ✅ 断言 7：图片处理同步/异步

**事实结论**：图片处理是**同步执行**的，不是异步。

```php
// src/Event/Subscriber/DownloadImagesSubscriber.php:77-91
/**
 * @todo If we want to add async download, it should be done in that method
 *
 * @return string
 */
private function downloadImages(Entry $entry)
{
    return $this->downloadImages->processHtml(
        $entry->getId(),
        $entry->getContent(),
        $entry->getUrl()
    );
}

// src/Event/Subscriber/DownloadImagesSubscriber.php:93-107
/**
 * @todo If we want to add async download, it should be done in that method
 *
 * @return string|false False in case of async
 */
private function downloadPreviewImage(Entry $entry)
{
    return $this->downloadImages->processSingleImage(
        $entry->getId(),
        $entry->getPreviewPicture(),
        $entry->getUrl()
    );
}
```

**执行时序证据**：
```php
// src/Event/Subscriber/DownloadImagesSubscriber.php:34-61
public function onEntrySaved(EntrySavedEvent $event): void
{
    if (!$this->enabled) {  // 来自 craue_config 的 download_images_enabled 配置
        return;  // src/Event/Subscriber/DownloadImagesSubscriber.php:39
    }

    $entry = $event->getEntry();

    // 同步执行：下载正文图片
    $html = $this->downloadImages($entry);  // 同步调用
    if (false !== $html) {
        $entry->setContent($html);
    }

    // 同步执行：下载预览图片
    $previewPicture = $this->downloadPreviewImage($entry);  // 同步调用
    if (false !== $previewPicture) {
        $entry->setPreviewPicture($previewPicture);
    }

    $this->em->persist($entry);
    $this->em->flush();  // 同步持久化
}
```

**配置来源证据**：
```yaml
# app/config/services.yml:267-269
Wallabag\Event\Subscriber\DownloadImagesSubscriber:
    arguments:
        $enabled: '@=service(''craue_config'').get(''download_images_enabled'')'
```

**反例说明**：翻译文件中的「异步导入」指的是将整个条目导入任务加入 RabbitMQ/Redis 队列，而非图片下载异步。

---

### ✅ 断言 8：EntrySavedEvent 触发点汇总

**实际代码触发点（共 15 处，不含文档）**：

| 文件 | 行号 | 场景 | 代码依据 |
|------|------|------|----------|
| `src/Controller/EntryController.php` | 198 | Web 表单新建 | ✅ 确认 |
| `src/Controller/EntryController.php` | 227 | Web Bookmarklet 新建 | ✅ 确认 |
| `src/Controller/EntryController.php` | 425 | Web 重载 | ✅ 确认 |
| `src/Controller/Api/EntryRestController.php` | 572 | API 批量新建 | ✅ 确认 |
| `src/Controller/Api/EntryRestController.php` | 808 | API 单条新建 | ✅ 确认 |
| `src/Controller/Api/EntryRestController.php` | 1022 | API PATCH 更新 | ✅ 确认 |
| `src/Controller/Api/EntryRestController.php` | 1076 | API 重载 | ✅ 确认 |
| `src/Import/AbstractImport.php` | 170 | Import 同步（每 20 条批量） | ✅ 确认 |
| `src/Import/AbstractImport.php` | 184 | Import 同步（剩余条目） | ✅ 确认 |
| `src/Import/HtmlImport.php` | 134 | HtmlImport 同步（每 20 条） | ✅ 确认 |
| `src/Import/HtmlImport.php` | 146 | HtmlImport 同步（剩余） | ✅ 确认 |
| `src/Import/BrowserImport.php` | 161 | BrowserImport 同步（每 20 条） | ✅ 确认 |
| `src/Import/BrowserImport.php` | 173 | BrowserImport 同步（剩余） | ✅ 确认 |
| `src/Consumer/AbstractConsumer.php` | 69 | Consumer 异步消费 | ✅ 确认 |
| `src/Command/ReloadEntryCommand.php` | 96 | CLI 批量重载 | ✅ 确认 |
| `src/Command/Import/UrlCommand.php` | - | CLI 单 URL 导入 | ❌ **未触发** |

**UrlCommand 未触发证据**：
```php
// src/Command/Import/UrlCommand.php:101-112
$this->entityManager->persist($entry);
// ... tags 处理 ...
$this->entityManager->flush();
// ⚠️ 此处无 dispatch(EntrySavedEvent) 调用 → 图片不会被下载

$output->writeln(\sprintf('URL %s successfully imported.', $url));
return 0;
```

**反例说明**：CLI 单 URL 导入不触发 `EntrySavedEvent`，因此即使启用了图片下载，图片也不会被本地化存储。

---

### ✅ 断言 9：stockEntry 抓取失败处理

```php
// src/Helper/ContentProxy.php:243-261
private function stockEntry(Entry $entry, array $content): void
{
    $this->updateOriginUrl($entry, $content['url']);
    $this->setEntryDomainName($entry);

    if (!empty($content['title'])) {
        $entry->setTitle($content['title']);
    }

    // 抓取失败处理：设置错误消息 + not_parsed 标记
    if (empty($content['html'])) {
        $content['html'] = $this->fetchingErrorMessage;
        $entry->setNotParsed(true);  // src/Helper/ContentProxy.php:255

        // 若有 description 则附加显示
        if (!empty($content['description'])) {
            $content['html'] .= '<p><i>But we found a short description: </i></p>';
            $content['html'] .= $content['description'];  // src/Helper/ContentProxy.php:259
        }
    }

    $entry->setContent($content['html']);
    // ... 其他字段处理
}
```

**重载场景额外保护**：
```php
// src/Controller/EntryController.php:412-428
public function reloadAction(Request $request, Entry $entry)
{
    $this->updateEntry($entry, 'entry_reloaded');

    // 若重载后内容仍是错误消息 → 不保存，直接返回
    if ($this->fetchingErrorMessage === $entry->getContent()) {
        $this->addFlash('notice', 'flashes.entry.notice.entry_reloaded_failed');
        return $this->redirect($this->generateUrl('view', ['id' => $entry->getId()]));
        // ⚠️ 此处 return，不执行 persist/flush/dispatch
    }

    // 正常保存并触发事件
    $this->entityManager->persist($entry);
    $this->entityManager->flush();
    $this->eventDispatcher->dispatch(new EntrySavedEvent($entry), EntrySavedEvent::NAME);
}
```

---

## 事实校对后的简版总结

### 主链时序（Web 新建为例）

```
用户提交 URL
  │
  ▼
EntryController::addEntryFormAction()
  ├─ 查重 findByUrlAndUserId()
  └─ updateEntry() [私有方法]
      └─ ContentProxy::updateEntry($entry, $url, [], false)
          ├─ 条件判断：empty($content) && !$disableContentUpdate
          │   └─ Graby::fetchContent($url)
          │       └─ WallabagClient::request()
          │           ├─ [开关] restrictedAccess === 0 → 跳过登录
          │           ├─ Authenticator::loginIfRequired()
          │           │   └─ buildSiteConfig()
          │           │       └─ GrabySiteConfigBuilder::buildForHost()
          │           │           ├─ 无用户 → return false
          │           │           ├─ 无凭证 → return false
          │           │           └─ 有用户+凭证 → return SiteConfig
          │           ├─ HTTP 请求
          │           └─ Authenticator::loginIfRequested()
          │               ├─ buildSiteConfig()
          │               ├─ body 为空 → return false
          │               └─ XPath 检测登录页面
          ├─ 标题编码清洗
          ├─ 失败回退：有导入内容且抓取失败 → 保留导入内容
          ├─ 设置 given_url
          └─ stockEntry()
              ├─ updateOriginUrl() [5 分支]
              ├─ 正文处理（失败时设置错误消息+not_parsed）
              ├─ 阅读时间/状态码/作者/日期/语言
              ├─ 预览图片提取
              └─ RuleBasedTagger 自动打标签
  │
  ▼
persist + flush
  │
  ▼
dispatch(EntrySavedEvent) [同步执行]
  └─ DownloadImagesSubscriber::onEntrySaved()
      ├─ 检查 download_images_enabled 配置
      ├─ processHtml() 下载正文图片 [同步]
      ├─ processSingleImage() 下载预览图 [同步]
      └─ persist + flush 更新内容
  │
  ▼
返回响应
```

### 登录链边界图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         WallabagClient::request()                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─ restrictedAccess === 0 ── 直接返回，绕过所有登录逻辑 ─┐               │
│  │                                                         │               │
│  ▼                                                         ▼               │
│  受限访问关闭                                               受限访问开启     │
│                            ┌──────────────────────────────────────┐        │
│                            │       Authenticator 层                │        │
│                            ├──────────────────────────────────────┤        │
│                            │                                      │        │
│                            │  loginIfRequired()                   │        │
│                            │    ├─ buildSiteConfig()              │        │
│                            │    │   └─ GrabySiteConfigBuilder      │        │
│                            │    │       ├─ 无用户 → false          │        │
│                            │    │       ├─ 无凭证 → false          │        │
│                            │    │       └─ 返回 SiteConfig         │        │
│                            │    ├─ requiresLogin() === false → 跳过 │        │
│                            │    ├─ isLoggedIn() → 已有 Cookie → 跳过 │        │
│                            │    └─ 执行登录 → return true          │        │
│                            │                                      │        │
│                            │  loginIfRequested()                   │        │
│                            │    ├─ buildSiteConfig()              │        │
│                            │    ├─ body 为空 → return false        │        │
│                            │    ├─ XPath 未匹配 → return false      │        │
│                            │    └─ 检测到登录页 → 执行登录          │        │
│                            └──────────────────────────────────────┘        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

  正文抽取主链（Graby 内部）
  └─ Graby::fetchContent()
      └─ Graby 内部使用自己的 Graby\SiteConfig\ConfigBuilder
         进行站点配置匹配和解析器选择
         ⚠️  此处不调用 Wallabag\SiteConfig\GrabySiteConfigBuilder
```

---

## R2 关键错误修正汇总

| 序号 | R2 错误表述 | R3 事实 | 证据 |
|------|------------|---------|------|
| 1 | `buildSiteConfig()` 内部有 `restrictedAccess === 0` 检查 | `restrictedAccess` 检查在 `WallabagClient::request()` 中，不在 `buildSiteConfig()` | `src/HttpClient/WallabagClient.php:31-33` vs `src/HttpClient/Authenticator.php:85-88` |
| 2 | `loginIfRequested()` 分支描述遗漏 body 为空检查 | `'' === $body` 时直接返回 false，不检测登录 | `src/HttpClient/Authenticator.php:63-67` |
| 3 | `validateContent()` 注释与实现一致 | 注释说需要 5 个字段，实际只检查 3 个 | `src/Import/AbstractImport.php:123` vs `src/Helper/ContentProxy.php:402` |
| 4 | 未提及 `UrlCommand` 不触发 `EntrySavedEvent` 的影响 | 单 URL CLI 导入不触发事件，图片不会被下载 | `src/Command/Import/UrlCommand.php:112` |
| 5 | `GrabySiteConfigBuilder::buildForHost()` 分支描述不完整 | 有 2 个返回 false 的分支（无用户、无凭证） | `src/SiteConfig/GrabySiteConfigBuilder.php:36, 54` |

---

## 不可反驳的事实结论

1. **调用边界**：`GrabySiteConfigBuilder` 仅在 `Authenticator` 登录链中调用，正文抽取主链由 Graby 内部的 `Graby\SiteConfig\ConfigBuilder` 处理。
2. **开关位置**：`restrictedAccess` 开关在 `WallabagClient` 层，关闭时完全绕过 `Authenticator`。
3. **配置构建失败条件**：无当前用户 **或** 无对应站点凭证时，`buildForHost()` 返回 `false`。
4. **登录决策分支**：`loginIfRequired()` 有 3 个分支，`loginIfRequested()` 有 4 个分支（含 body 为空检查）。
5. **内容验证**：`validateContent()` 实际只检查 `title`、`html`、`url` 三个字段。
6. **抓取回退**：有导入内容时，若 Graby 抓取失败，保留导入内容而非错误消息。
7. **URL 处理**：`updateOriginUrl()` 有 5 个明确分支，忽略规则匹配时不保存 `origin_url`。
8. **图片处理**：同步执行，`@todo` 注释标记未来可改为异步。
9. **事件触发**：15 个代码触发点，CLI 单 URL 导入是唯一例外（不触发）。
