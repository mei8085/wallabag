# 站点配置驱动正文抽取流程分析（R2 - 深度核对版）

本文档针对 R1 文档中的理解偏差进行深度核对，重点明确三个方面：
1. **GrabySiteConfigBuilder 在受限访问登录链与正文抽取主链中的边界**
2. **Web/API/Import/Consumer 触发到 ContentProxy 与 EntrySavedEvent 的完整调用路径**
3. **图片处理同步/异步属性、validateContent/disableContentUpdate、updateOriginUrl 与抓取失败回退分支**

---

## 一、GrabySiteConfigBuilder 在两条链路中的边界

### 1.1 调用链边界的关键发现

**R1 理解偏差修正**：`GrabySiteConfigBuilder::buildForHost()` **仅在受限访问登录链中被调用**，在正文抽取主链中**不直接调用**。

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     调用边界对比                                          │
├────────────────────────────────────────┬─────────────────────────────────┤
│ 受限访问登录链                          │ 正文抽取主链                     │
├────────────────────────────────────────┼─────────────────────────────────┤
│ WallabagClient::request()              │ ContentProxy::updateEntry()     │
│   → Authenticator::loginIfRequired()   │   → Graby::fetchContent()       │
│     → buildSiteConfig()                │     → WallabagClient::request() │
│       → GrabySiteConfigBuilder::buildForHost() │   │   │             │
│                                          │     → Graby 内部解析器选择     │
│                                          │       (内置 Graby\SiteConfig\  │
│                                          │        ConfigBuilder 直接调用)  │
└────────────────────────────────────────┴─────────────────────────────────┘
```

### 1.2 受限访问登录链中的调用点

**唯一调用点**：`Authenticator::buildSiteConfig()` → `src/HttpClient/Authenticator.php:87`

```php
// src/HttpClient/Authenticator.php:32-88
private function buildSiteConfig(UriInterface $uri): SiteConfig|false
{
    if (0 === (int) $this->restrictedAccess) {
        return false;
    }

    return $this->configBuilder->buildForHost($uri->getHost());
}

// 登录时机1：请求前检查
public function loginIfRequired(string $url): bool
{
    $config = $this->buildSiteConfig(new Uri($url));
    if (false === $config || !$config->requiresLogin()) {
        return false;
    }
    if ($this->authenticator->isLoggedIn($config)) {
        return false;
    }
    $this->authenticator->login($config);
    return true;
}

// 登录时机2：响应后检测
public function loginIfRequested(ResponseInterface $response): bool
{
    $config = $this->buildSiteConfig(new Uri($response->getInfo('url')));
    if (false === $config || !$config->requiresLogin()) {
        return false;
    }
    $body = $response->getContent();
    $isLoginRequired = $this->authenticator->isLoginRequired($config, $body);
    if (!$isLoginRequired) {
        return false;
    }
    $this->authenticator->login($config);
    return true;
}
```

**边界条件**：
- 当 `restrictedAccess === 0` 时，直接返回 `false`，不构建配置
- 当站点配置 `requiresLogin === false` 时，跳过登录流程
- 当已有有效 Cookie 时，跳过重复登录

### 1.3 正文抽取主链中的配置使用

在正文抽取主链中，`GrabySiteConfigBuilder` **不直接参与**。Graby 内部使用自己的 `Graby\SiteConfig\ConfigBuilder` 进行解析器选择：

```php
// src/Helper/ContentProxy.php:50-62
public function updateEntry(Entry $entry, $url, array $content = [], $disableContentUpdate = false): void
{
    // GrabySiteConfigBuilder 不在此处调用！
    // Graby 内部完成站点配置匹配和解析器选择
    if ((empty($content) || false === $this->validateContent($content)) && false === $disableContentUpdate) {
        $fetchedContent = $this->graby->fetchContent($url);
        // ...
    }
}
```

### 1.4 WallabagClient 中的登录请求流程

```php
// src/HttpClient/WallabagClient.php:27-58
public function request(string $method, string $url, array $options = []): ResponseInterface
{
    // 开关检查：restricted_access 为 0 时跳过所有登录逻辑
    if (0 === (int) $this->restrictedAccess) {
        return $this->httpClient->request($method, $url, $options);
    }

    // 登录时机1：请求前预登录
    $login = $this->authenticator->loginIfRequired($url);
    if (!$login) {
        return $this->httpClient->request($method, $url, $options);
    }

    // 附加 Cookie 后请求
    if (null !== $cookieHeader = $this->getCookieHeader($url)) {
        $options['headers']['cookie'] = $cookieHeader;
    }
    $response = $this->httpClient->request($method, $url, $options);

    // 登录时机2：响应后检测登录页面
    $login = $this->authenticator->loginIfRequested($response);
    if (!$login) {
        return $response;
    }

    // 登录后重新请求
    if (null !== $cookieHeader = $this->getCookieHeader($url)) {
        $options['headers']['cookie'] = $cookieHeader;
    }
    return $this->httpClient->request($method, $url, $options);
}
```

---

## 二、Web/API/Import/Consumer 到 ContentProxy 与 EntrySavedEvent 的完整调用路径

### 2.1 六大触发入口汇总

| 入口类型 | 触发点 | 调用路径 | EntrySavedEvent 触发位置 |
|---------|--------|----------|--------------------------|
| **Web 表单** | `EntryController::addEntryFormAction()` | `src/Controller/EntryController.php:192` | `EntryController.php:198` |
| **Web Bookmarklet** | `EntryController::addEntryViaBookmarkletAction()` | `src/Controller/EntryController.php:221` | `EntryController.php:227` |
| **Web 重载** | `EntryController::reloadAction()` | `src/Controller/EntryController.php:412` | `EntryController.php:425` |
| **API POST** | `EntryRestController::postEntriesAction()` | `src/Controller/Api/EntryRestController.php:741` | `EntryRestController.php:808` |
| **API PATCH 批量** | `EntryRestController::postEntriesListAction()` | `src/Controller/Api/EntryRestController.php:563` | `EntryRestController.php:572` |
| **API PATCH 重载** | `EntryRestController::patchEntriesReloadAction()` | `src/Controller/Api/EntryRestController.php:1057` | `EntryRestController.php:1076` |
| **Import 同步** | `AbstractImport::parseEntries()` | `src/Import/AbstractImport.php:128` | `AbstractImport.php:170` |
| **Import 异步** | `AbstractConsumer::handleMessage()` | `src/Consumer/AbstractConsumer.php:56` | `AbstractConsumer.php:69` |
| **CLI 单 URL** | `UrlCommand::execute()` | `src/Command/Import/UrlCommand.php:90` | **无**（未触发事件） |
| **CLI 批量重载** | `ReloadEntryCommand::execute()` | `src/Command/ReloadEntryCommand.php:92` | `ReloadEntryCommand.php:96` |
| **Fixtures** | `EntryFixtures::load()` | `fixtures/EntryFixtures.php` | **无** |

### 2.2 各入口详细调用链

#### 2.2.1 Web 入口（EntryController）

**新建条目**：
```
POST /new-entry
  → EntryController::addEntryFormAction()
    → checkIfEntryAlreadyExists()  # 查重
    → updateEntry() [私有方法]      # EntryController.php:703
      → ContentProxy::updateEntry($entry, $entry->getUrl())
    → EntityManager::persist()
    → EntityManager::flush()
    → EventDispatcher::dispatch(EntrySavedEvent)  # EntryController.php:198
```

**Bookmarklet 快捷保存**：
```
GET /bookmarklet?url=xxx
  → EntryController::addEntryViaBookmarkletAction()
    → checkIfEntryAlreadyExists()
    → updateEntry() [私有方法]
      → ContentProxy::updateEntry()
    → persist + flush
    → dispatch(EntrySavedEvent)  # EntryController.php:227
```

**重载条目**：
```
POST /reload/{id}
  → EntryController::reloadAction()
    → updateEntry($entry, 'entry_reloaded') [私有方法]
      → ContentProxy::updateEntry($entry, $entry->getUrl())
    → 检查内容是否为错误消息
      → 若是：不保存，直接返回
      → 若否：persist + flush + dispatch(EntrySavedEvent)
```

#### 2.2.2 API 入口（EntryRestController）

**POST 单条创建**：
```
POST /api/entries.json
  → EntryRestController::postEntriesAction()
    → 查重 findByUrlAndUserId()
    → ContentProxy::updateEntry($entry, $entry->getUrl(), $content)
    → 附加属性（archive/starred/tags/origin_url/public）
    → 验证域名为空则设置
    → 验证标题为空则设置默认
    → Validator::validate()
    → persist + flush
    → dispatch(EntrySavedEvent)  # EntryRestController.php:808
```

**POST 批量创建**：
```
POST /api/entries/lists.json
  → EntryRestController::postEntriesListAction()
    → 循环处理每个 URL
      → 查重
      → ContentProxy::updateEntry()
      → persist + flush
      → dispatch(EntrySavedEvent)  # EntryRestController.php:572
```

**PATCH 更新内容**（特殊调用模式）：
```
PATCH /api/entries/{id}.json
  → EntryRestController::patchEntriesAction()
    → if !empty($data['content']):
        → ContentProxy::updateEntry($entry, $entry->getUrl(), ['html' => $content], true)
          # disableContentUpdate = true，跳过 Graby 抓取
    → 其他字段单独更新
    → persist + flush
    → dispatch(EntrySavedEvent)  # EntryRestController.php:1022
```

**PATCH 重载**：
```
PATCH /api/entries/{id}/reload.json
  → EntryRestController::patchEntriesReloadAction()
    → ContentProxy::updateEntry($entry, $entry->getUrl())
    → 检查是否为错误消息
      → 若是：返回 304
      → 若否：persist + flush + dispatch(EntrySavedEvent)
```

#### 2.2.3 Import 同步入口

**AbstractImport 同步流程**：
```
AbstractImport::parseEntries()
  → 循环 foreach ($entries as $importedEntry):
    → validateEntry()
    → parseEntry() [具体导入类实现]
      → 准备 Entry 对象
      → fetchContent() [protected 方法]
        → ContentProxy::updateEntry($entry, $url, $content, $this->disableContentUpdate)
    → assignTagsToEntry()
    → persist
    → 每 20 条 flush 一次 + 批量 dispatch(EntrySavedEvent)
          # AbstractImport.php:169-171
          foreach ($entryToBeFlushed as $entry) {
              $this->eventDispatcher->dispatch(new EntrySavedEvent($entry), EntrySavedEvent::NAME);
          }
  → 最后 flush + 剩余条目 dispatch
```

**关键参数传递**：
- `$disableContentUpdate` 从导入类属性传递到 `ContentProxy::updateEntry()`
- `$content` 数组从 `prepareEntry()` 传递，包含导入时携带的元数据

#### 2.2.4 Import 异步 Consumer 入口

**RabbitMQ/Redis 消费流程**：
```
AbstractConsumer::handleMessage($body)
  → json_decode 消息体
  → UserRepository::find($storedEntry['userId'])
  → AbstractImport::setUser($user)
  → AbstractImport::validateEntry()
  → AbstractImport::parseEntry()
    → fetchContent()
      → ContentProxy::updateEntry()
  → flush
  → dispatch(EntrySavedEvent)  # AbstractConsumer.php:69
  → em->clear()
```

**注意**：Consumer 流程中，`parseEntry()` 内部完成 persist，Consumer 调用 flush 后触发事件。

#### 2.2.5 CLI 命令入口

**单 URL 导入**（注意：**不触发 EntrySavedEvent**）：
```
bin/console wallabag:import:url username url
  → UrlCommand::execute()
    → 构建用户 Token（用于受限访问登录链的用户识别）
    → 查重
    → ContentProxy::updateEntry($entry, $url)
    → persist + flush
    → **无 dispatch(EntrySavedEvent)**  → 图片不会被下载
```

**批量重载**：
```
bin/console wallabag:entry:reload [username] [--only-not-parsed]
  → ReloadEntryCommand::execute()
    → 批量查询条目 ID
    → 循环处理每个条目
      → ContentProxy::updateEntry($entry, $entry->getUrl())
      → persist + flush
      → dispatch(EntrySavedEvent)  # ReloadEntryCommand.php:96
    → detach 释放内存
```

---

## 三、图片处理同步/异步及相关分支逻辑

### 3.1 图片处理：同步执行，非异步

**R1 理解偏差修正**：图片处理是**同步执行**的，不是异步。代码中 `@todo` 注释明确指出「未来可以改为异步」。

```php
// src/Event/Subscriber/DownloadImagesSubscriber.php:80-91
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

**执行时序**：
```
Entry 持久化完成
  ↓
dispatch(EntrySavedEvent)  [同步事件，立即执行]
  ↓
DownloadImagesSubscriber::onEntrySaved()
  ├─ downloadImages()  → processHtml() → 下载所有图片 + 替换链接
  ├─ downloadPreviewImage()  → processSingleImage() → 下载预览图
  ├─ persist($entry)
  └─ flush()
  ↓
请求响应返回
```

**性能影响说明**：翻译文件中的警告提示了当图片下载+传统导入同时使用时可能耗时极长，建议启用异步导入（指将整个条目导入任务加入队列，而非图片下载异步）。

### 3.2 validateContent 与 disableContentUpdate 分支逻辑

#### 3.2.1 核心判断逻辑

```php
// src/Helper/ContentProxy.php:50-62
if ((empty($content) || false === $this->validateContent($content)) && false === $disableContentUpdate) {
    $fetchedContent = $this->graby->fetchContent($url);
    // ...

    // 抓取失败回退逻辑
    if (empty($content) || $fetchedContent['html'] !== $this->fetchingErrorMessage) {
        $content = $fetchedContent;
    }
}
```

**条件拆解**：

| $content 状态 | validateContent() | disableContentUpdate | 是否调用 Graby | 说明 |
|---------------|-------------------|----------------------|---------------|------|
| empty | - | false | 是 | 无导入内容，必须抓取 |
| non-empty | false | false | 是 | 导入内容不完整，需要抓取 |
| non-empty | true | false | 否 | 导入内容完整，跳过抓取 |
| - | - | true | 否 | 强制禁用抓取 |

#### 3.2.2 validateContent 实现

```php
// src/Helper/ContentProxy.php:400-403
private function validateContent(array $content)
{
    return !empty($content['title']) && !empty($content['html']) && !empty($content['url']);
}
```

**验证规则**：三个字段（title、html、url）必须同时非空。

#### 3.2.3 各调用场景的参数组合

| 调用方 | $content | $disableContentUpdate | 行为 |
|--------|----------|------------------------|------|
| Web 新建 | `[]` | false | 调用 Graby 抓取 |
| Web 重载 | `[]` | false | 调用 Graby 重新抓取 |
| API POST 新建 | `['title' => ..., 'html' => ..., 'url' => ...]` | false | 若三者都非空则跳过抓取 |
| API PATCH 更新 | `['html' => $newContent]` | true | 跳过抓取，仅清洗 HTML |
| API PATCH 重载 | `[]` | false | 调用 Graby 重新抓取 |
| Import 同步（默认） | 导入的元数据 | false | 若导入内容不完整则抓取 |
| Import 同步（disableContentUpdate=true） | 导入的元数据 | true | 跳过抓取，使用导入内容 |
| Consumer 异步 | 队列消息中的数据 | false | 同上 |

#### 3.2.4 抓取失败回退分支

```php
// src/Helper/ContentProxy.php:58-62
// when content is imported, we have information in $content
// in case fetching content goes bad, we'll keep the imported information instead of overriding them
if (empty($content) || $fetchedContent['html'] !== $this->fetchingErrorMessage) {
    $content = $fetchedContent;
}
```

**回退逻辑真值表**：

| $content 状态 | $fetchedContent['html'] | 结果 | 说明 |
|---------------|--------------------------|------|------|
| empty | 错误消息 | $content = $fetchedContent | 无导入内容，即使失败也用错误消息 |
| empty | 正常内容 | $content = $fetchedContent | 正常抓取结果 |
| non-empty | 错误消息 | 保留 $content | **抓取失败，回退到导入内容** |
| non-empty | 正常内容 | $content = $fetchedContent | 抓取成功，覆盖导入内容 |

### 3.3 updateOriginUrl 完整分支逻辑

```php
// src/Helper/ContentProxy.php:328-393
private function updateOriginUrl(Entry $entry, $url)
{
    // 分支1：URL 相同或为空 → 不处理
    if (empty($url) || $entry->getUrl() === $url) {
        return false;
    }

    // 计算 URL 差异部分
    $parsed_entry_url = parse_url($entry->getUrl());
    $parsed_content_url = parse_url($url);
    $diff_ec = array_diff_assoc($parsed_entry_url, $parsed_content_url);
    $diff_ce = array_diff_assoc($parsed_content_url, $parsed_entry_url);
    $diff = array_merge($diff_ec, $diff_ce);
    $diff_keys = array_keys($diff);
    sort($diff_keys);

    // 分支2：忽略规则匹配 → 直接更新 URL，不保存 origin_url
    if ($this->ignoreOriginProcessor->process($entry)) {
        $entry->setUrl($url);
        return false;
    }

    // 分支3：根据差异部分执行不同策略
    switch ($diff_keys) {
        case ['path']:
            // 3a: 仅尾部斜杠差异 → 直接更新
            if (($parsed_entry_url['path'] . '/' === $parsed_content_url['path'])
                || ($url === urldecode($entry->getUrl()))) {
                $entry->setUrl($url);
            }
            // 其他 path 差异 → 不处理
            break;
        case ['scheme']:
            // 3b: 仅协议差异（http→https）→ 直接更新
            $entry->setUrl($url);
            break;
        case ['fragment']:
            // 3c: 仅锚点差异 → 不处理
            break;
        default:
            // 3d: 其他差异（host 变化、多部分变化等）
            // → 保存原 URL 到 origin_url，更新为新 URL
            if (empty($entry->getOriginUrl())) {
                $entry->setOriginUrl($entry->getUrl());
            }
            $entry->setUrl($url);
            break;
    }
}
```

**分支执行路径总结**：

```
entry.url vs content.url
  ├─ 相同 → 退出
  ├─ 忽略规则匹配 → 直接更新 url（不存 origin_url）
  └─ 有差异：
     ├─ 仅 path 差异：
     │   ├─ 尾部斜杠或 URL 编码差异 → 更新 url
     │   └─ 其他 → 不处理
     ├─ 仅 scheme 差异 → 更新 url
     ├─ 仅 fragment 差异 → 不处理
     └─ 其他 → 保存 origin_url + 更新 url
```

**忽略规则处理器**（`RuleBasedIgnoreOriginProcessor`）：
- 用于处理 feedproxy.google.com 等跳转服务域名
- 匹配时直接更新 URL，不保留原始 URL

### 3.4 stockEntry 中抓取失败后的处理分支

```php
// src/Helper/ContentProxy.php:243-320
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
        $entry->setNotParsed(true);

        // 若有 description 则附加显示
        if (!empty($content['description'])) {
            $content['html'] .= '<p><i>But we found a short description: </i></p>';
            $content['html'] .= $content['description'];
        }
    }

    $entry->setContent($content['html']);
    // ... 其他字段处理
}
```

**重载场景的额外保护**：
```php
// src/Controller/EntryController.php:412-428
public function reloadAction(Request $request, Entry $entry)
{
    $this->updateEntry($entry, 'entry_reloaded');

    // 若重载后内容仍是错误消息 → 不保存，直接返回
    if ($this->fetchingErrorMessage === $entry->getContent()) {
        $this->addFlash('notice', 'flashes.entry.notice.entry_reloaded_failed');
        return $this->redirect($this->generateUrl('view', ['id' => $entry->getId()]));
    }

    // 正常保存并触发事件
    $this->entityManager->persist($entry);
    $this->entityManager->flush();
    $this->eventDispatcher->dispatch(new EntrySavedEvent($entry), EntrySavedEvent::NAME);
}
```

---

## 四、关键时序与数据流总图

### 4.1 完整调用时序（以 Web 新建为例）

```
用户提交 URL 表单
    │
    ▼
EntryController::addEntryFormAction()
    ├─ 查重 findByUrlAndUserId()
    │   └─ 已存在 → 重定向到已有条目
    └─ 不存在 → 继续
        │
        ▼
EntryController::updateEntry() [私有方法]
    └─ ContentProxy::updateEntry($entry, $url, [], false)
        ├─ 条件判断：empty($content)=true && disableContentUpdate=false
        │   └─ → 调用 Graby::fetchContent($url)
        │       └─ Graby 内部 HTTP 请求
        │           └─ WallabagClient::request()
        │               ├─ restricted_access 检查
        │               ├─ Authenticator::loginIfRequired()
        │               │   └─ GrabySiteConfigBuilder::buildForHost()
        │               │       └─ [仅登录链使用]
        │               ├─ HTTP 请求
        │               └─ Authenticator::loginIfRequested()
        ├─ 标题编码清洗
        ├─ 结果融合（检查是否为错误消息）
        ├─ 设置 given_url
        └─ stockEntry($entry, $content)
            ├─ updateOriginUrl()
            ├─ 标题设置
            ├─ 正文处理（失败时设置错误消息+not_parsed）
            ├─ 阅读时间计算
            ├─ HTTP 状态码、作者、响应头
            ├─ 日期、语言解析
            ├─ 预览图片提取
            ├─ MIME 类型设置
            └─ RuleBasedTagger::tag() 自动打标签
    │
    ▼
EntityManager::persist($entry)
EntityManager::flush()
    │
    ▼
EventDispatcher::dispatch(EntrySavedEvent)
    └─ DownloadImagesSubscriber::onEntrySaved()  [同步执行]
        ├─ 检查 download_images_enabled 配置
        ├─ DownloadImages::processHtml() 下载正文中的所有图片
        ├─ DownloadImages::processSingleImage() 下载预览图
        ├─ persist + flush 更新条目内容
    │
    ▼
重定向到首页
```

### 4.2 EntrySavedEvent 触发位置汇总

| 触发文件 | 行号 | 场景 |
|---------|------|------|
| `src/Controller/EntryController.php` | 198 | Web 表单新建 |
| `src/Controller/EntryController.php` | 227 | Web Bookmarklet 新建 |
| `src/Controller/EntryController.php` | 425 | Web 重载 |
| `src/Controller/Api/EntryRestController.php` | 572 | API 批量新建 |
| `src/Controller/Api/EntryRestController.php` | 808 | API 单条新建 |
| `src/Controller/Api/EntryRestController.php` | 1022 | API PATCH 更新 |
| `src/Controller/Api/EntryRestController.php` | 1076 | API 重载 |
| `src/Import/AbstractImport.php` | 170 | Import 同步（每20条批量） |
| `src/Import/AbstractImport.php` | 184 | Import 同步（剩余条目） |
| `src/Import/HtmlImport.php` | 134 | HtmlImport 同步（每20条） |
| `src/Import/HtmlImport.php` | 146 | HtmlImport 同步（剩余） |
| `src/Import/BrowserImport.php` | 161 | BrowserImport 同步（每20条） |
| `src/Import/BrowserImport.php` | 173 | BrowserImport 同步（剩余） |
| `src/Consumer/AbstractConsumer.php` | 69 | Consumer 异步消费 |
| `src/Command/ReloadEntryCommand.php` | 96 | CLI 批量重载 |
| `src/Command/Import/UrlCommand.php` | - | **未触发** |

---

## 五、修正与补充总结

### 5.1 R1 理解偏差修正清单

| 序号 | R1 表述 | R2 修正 | 依据 |
|------|---------|---------|------|
| 1 | GrabySiteConfigBuilder 在正文抽取主链中调用 | 仅在受限访问登录链（Authenticator）中调用，正文抽取主链由 Graby 内部处理 | `src/HttpClient/Authenticator.php:87` + `ContentProxy.php:50-62` |
| 2 | 图片下载是异步的 | 图片下载是**同步执行**的，@todo 注释说明未来可以改为异步 | `DownloadImagesSubscriber.php:80,96` |
| 3 | EntrySavedEvent 是简单触发 | 有 14+ 个触发点，CLI 单 URL 导入**不触发** | 各调用方代码 |
| 4 | updateOriginUrl 逻辑简化 | 5 个分支，包含忽略规则、scheme/path/fragment 差异处理 | `ContentProxy.php:328-393` |

### 5.2 关键设计决策

1. **登录链与抽取链分离**：站点配置仅用于登录决策，正文抽取的解析器选择由 Graby 内部完成，职责清晰
2. **导入内容优先的回退策略**：抓取失败时保留导入内容，确保用户数据不丢失
3. **同步事件设计**：EntrySavedEvent 同步执行，确保图片下载等后处理在请求响应前完成（但可能影响性能）
4. **disableContentUpdate 开关**：支持纯导入模式，跳过网络抓取，适用于 API PATCH 更新场景
5. **notParsed 标记**：抓取失败的条目可通过 `--only-not-parsed` 选项批量重试
