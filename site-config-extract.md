# 站点配置驱动正文抽取流程分析

## 概述

Wallabag 的正文抽取系统基于 Graby 库构建，采用「站点配置驱动」的架构模式。整个流程分为三大核心模块：

1. **配置加载**：根据 URL 主机名匹配并加载对应的站点抽取规则
2. **解析器选择**：根据匹配到的配置选择合适的正文提取策略
3. **抽取后处理**：对抽取结果进行清洗、验证、丰富和持久化

## 一、配置加载模块

### 1.1 核心类与接口

| 类/接口 | 位置 | 职责 |
|---------|------|------|
| `SiteConfigBuilder` | `src/SiteConfig/SiteConfigBuilder.php:6` | 配置构建器接口，定义 `buildForHost($host)` 契约 |
| `GrabySiteConfigBuilder` | `src/SiteConfig/GrabySiteConfigBuilder.php:10` | 基于 Graby 的配置构建器实现 |
| `ArraySiteConfigBuilder` | `src/SiteConfig/ArraySiteConfigBuilder.php:6` | 基于数组的配置构建器（测试用） |
| `SiteConfig` | `src/SiteConfig/SiteConfig.php:8` | 站点配置值对象 |

### 1.2 配置源与优先级

#### 1.2.1 内置站点配置库
项目依赖 `j0k3r/graby-site-config` 包（见 `composer.json:87`），提供了数千个网站的预定义抽取规则。

#### 1.2.2 自定义配置目录
通过环境变量 `WALLABAG_SITE_CONFIG_FOLDERS` 可指定自定义配置目录，支持多目录以逗号分隔：
```yaml
# app/config/services.yml:8-9
parameters:
    env(WALLABAG_SITE_CONFIG_FOLDERS): ''
    wallabag.site_config_folders: '%env(csv:WALLABAG_SITE_CONFIG_FOLDERS)%'
```

Graby 配置构建器初始化时注入该配置：
```yaml
# app/config/services.yml:228-230
Graby\SiteConfig\ConfigBuilder:
    arguments:
        $config: "@=parameter('wallabag.site_config_folders') and parameter('wallabag.site_config_folders')[0] ? {'site_config': parameter('wallabag.site_config_folders')} : {}"
```

### 1.3 配置构建流程

`GrabySiteConfigBuilder::buildForHost()` 是配置加载的核心入口：

```php
// src/SiteConfig/GrabySiteConfigBuilder.php:23-79
public function buildForHost($host)
{
    $user = $this->getUser();
    
    // 1. 主机名规范化
    $host = strtolower($host);
    if (str_starts_with($host, 'www.')) {
        $host = substr($host, 4);
    }
    
    // 2. 生成主机名候选列表（支持通配符匹配）
    $hosts = [$host];
    $split = explode('.', $host);
    if (\count($split) > 1) {
        array_shift($split);
        $hosts[] = '.' . implode('.', $split);  // 如：.example.org
    }
    
    // 3. 查询用户站点凭证
    $credentials = $this->credentialRepository->findOneByHostsAndUser($hosts, $user->getId());
    
    // 4. 从 Graby 加载站点配置
    $config = $this->grabyConfigBuilder->buildForHost($host);
    
    // 5. 合并配置与凭证，构建 SiteConfig 对象
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
    
    return new SiteConfig($parameters);
}
```

### 1.4 配置结构说明

`SiteConfig` 对象包含以下核心配置项：
- **登录相关**：`requiresLogin`, `loginUri`, `usernameField`, `passwordField`, `extraFields`, `notLoggedInXpath`
- **凭证信息**：`username`, `password`（从 `SiteCredential` 实体加载）
- **HTTP 配置**：`httpHeaders`
- **主机标识**：`host`

## 二、解析器选择模块

### 2.1 HTTP 客户端与认证流程

#### 2.1.1 客户端包装链
```
WallabagClient → Authenticator → LoginFormAuthenticator → Graby
```

#### 2.1.2 WallabagClient 请求流程

```php
// src/HttpClient/WallabagClient.php:27-58
public function request(string $method, string $url, array $options = []): ResponseInterface
{
    // 受限访问模式开关
    if (0 === (int) $this->restrictedAccess) {
        return $this->httpClient->request($method, $url, $options);
    }
    
    // 1. 请求前预登录（基于 Cookie 检查）
    $login = $this->authenticator->loginIfRequired($url);
    if (!$login) {
        return $this->httpClient->request($method, $url, $options);
    }
    
    // 2. 附加 Cookie 并发起请求
    if (null !== $cookieHeader = $this->getCookieHeader($url)) {
        $options['headers']['cookie'] = $cookieHeader;
    }
    $response = $this->httpClient->request($method, $url, $options);
    
    // 3. 响应后检查是否需要登录（基于 XPath 检测）
    $login = $this->authenticator->loginIfRequested($response);
    if (!$login) {
        return $response;
    }
    
    // 4. 登录后重新请求
    if (null !== $cookieHeader = $this->getCookieHeader($url)) {
        $options['headers']['cookie'] = $cookieHeader;
    }
    return $this->httpClient->request($method, $url, $options);
}
```

#### 2.1.3 Authenticator 登录检查

```php
// src/HttpClient/Authenticator.php:32-88

// 登录时机1：请求前检查 Cookie
public function loginIfRequired(string $url): bool
{
    $config = $this->buildSiteConfig(new Uri($url));
    if (false === $config || !$config->requiresLogin()) {
        return false;
    }
    
    // 检查是否已有有效 Cookie
    if ($this->authenticator->isLoggedIn($config)) {
        return false;
    }
    
    $this->authenticator->login($config);
    return true;
}

// 登录时机2：响应后检测登录页面
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

#### 2.1.4 LoginFormAuthenticator 登录实现

```php
// src/SiteConfig/LoginFormAuthenticator.php:27-119
public function login(SiteConfig $siteConfig)
{
    $postFields = [
        $siteConfig->getUsernameField() => $siteConfig->getUsername(),
        $siteConfig->getPasswordField() => $siteConfig->getPassword(),
    ] + $this->getExtraFields($siteConfig);
    
    $this->browser->request(
        'POST', 
        $siteConfig->getLoginUri(), 
        $postFields, 
        [], 
        $this->getHttpHeaders($siteConfig)
    );
}

// XPath 检测是否需要登录
public function isLoginRequired(SiteConfig $siteConfig, $html)
{
    try {
        $crawler = new Crawler((string) $html);
        $loggedIn = $crawler->evaluate((string) $siteConfig->getNotLoggedInXpath());
    } catch (\Throwable) {
        return false;
    }
    return \count($loggedIn) > 0;
}
```

### 2.2 Graby 正文抽取流程

`ContentProxy::updateEntry()` 是抽取流程的入口：

```php
// src/Helper/ContentProxy.php:43-79
public function updateEntry(Entry $entry, $url, array $content = [], $disableContentUpdate = false): void
{
    // 1. 启用图片无引用策略
    $this->graby->toggleImgNoReferrer(true);
    
    // 2. 导入内容预处理
    if (!empty($content['html'])) {
        $content['html'] = $this->graby->cleanupHtml($content['html'], $url);
    }
    
    // 3. 调用 Graby 抽取正文
    if ((empty($content) || false === $this->validateContent($content)) && false === $disableContentUpdate) {
        $fetchedContent = $this->graby->fetchContent($url);
        
        // 4. 标题编码处理
        $fetchedContent['title'] = $this->sanitizeContentTitle(
            $fetchedContent['title'],
            $fetchedContent['headers']['content-type'] ?? ''
        );
        
        // 5. 结果融合（优先使用抽取结果，失败则回退到导入内容）
        if (empty($content) || $fetchedContent['html'] !== $this->fetchingErrorMessage) {
            $content = $fetchedContent;
        }
    }
    
    // 6. URL 规范化
    $content['url'] = !empty($content['url']) ? $content['url'] : $url;
    
    // 7. 持久化处理
    $this->stockEntry($entry, $content);
}
```

### 2.3 解析器选择策略

Graby 内部的解析器选择逻辑（基于 Graby 库设计）：

1. **配置匹配阶段**
   - 根据主机名查找 `*.txt` 站点配置文件
   - 支持精确匹配（`example.org.txt`）和通配符匹配（`.example.org.txt`）
   - 配置文件中定义了 `body`, `title`, `date`, `author` 等字段的 XPath 选择器

2. **解析策略选择**
   - **配置驱动解析**：如果找到匹配的站点配置，使用配置中定义的 XPath 规则精确提取
   - **通用解析回退**：如果没有匹配的配置，调用 `j0k3r/php-readability` 库进行基于文本密度的通用正文提取

3. **配置文件格式示例**
   ```
   title: //h1
   body: //div[@class="article-content"]
   date: //time/@datetime
   author: //span[@class="author"]
   requires_login: true
   login_uri: https://example.org/login
   login_username_field: username
   login_password_field: password
   not_logged_in_xpath: //div[@class="login-form"]
   ```

## 三、抽取后处理模块

### 3.1 核心后处理流程

`ContentProxy::stockEntry()` 负责对 Graby 返回的抽取结果进行全面后处理：

```php
// src/Helper/ContentProxy.php:243-320
private function stockEntry(Entry $entry, array $content): void
{
    // 1. URL 重定向处理
    $this->updateOriginUrl($entry, $content['url']);
    
    // 2. 设置域名
    $this->setEntryDomainName($entry);
    
    // 3. 标题处理
    if (!empty($content['title'])) {
        $entry->setTitle($content['title']);
    }
    
    // 4. 正文处理
    if (empty($content['html'])) {
        $content['html'] = $this->fetchingErrorMessage;
        $entry->setNotParsed(true);
        if (!empty($content['description'])) {
            $content['html'] .= '<p><i>But we found a short description: </i></p>';
            $content['html'] .= $content['description'];
        }
    }
    $entry->setContent($content['html']);
    
    // 5. 阅读时间计算
    $entry->setReadingTime(Utils::getReadingTime($content['html']));
    
    // 6. HTTP 状态码
    if (!empty($content['status'])) {
        $entry->setHttpStatus($content['status']);
    }
    
    // 7. 作者信息
    if (!empty($content['authors']) && \is_array($content['authors'])) {
        $entry->setPublishedBy($content['authors']);
    }
    
    // 8. HTTP 响应头存储
    if (!empty($content['headers'])) {
        $entry->setHeaders($content['headers']);
    }
    
    // 9. 发布日期解析
    if (!empty($content['date'])) {
        $this->updatePublishedAt($entry, $content['date']);
    }
    
    // 10. 语言识别与验证
    if (!empty($content['language'])) {
        $this->updateLanguage($entry, $content['language']);
    }
    
    // 11. 预览图片提取
    $previewPictureUrl = '';
    if (!empty($content['image'])) {
        $previewPictureUrl = $content['image'];
    }
    // 图片类型内容直接作为预览
    if (!empty($content['headers']['content-type']) && \in_array(current($this->mimeTypes->getExtensions($content['headers']['content-type'])), ['jpeg', 'jpg', 'gif', 'png'], true)) {
        $previewPictureUrl = $content['url'];
    } elseif (empty($previewPictureUrl)) {
        // 从正文中提取第一张图片作为预览
        $imagesUrls = DownloadImages::extractImagesUrlsFromHtml($content['html']);
        if (!empty($imagesUrls)) {
            $previewPictureUrl = $imagesUrls[0];
        }
    }
    
    // 12. MIME 类型
    if (!empty($content['headers']['content-type'])) {
        $entry->setMimetype($content['headers']['content-type']);
    }
    
    // 13. 预览图片验证
    if (!empty($previewPictureUrl)) {
        $this->updatePreviewPicture($entry, $previewPictureUrl);
    }
    
    // 14. 自动打标签
    try {
        $this->tagger->tag($entry);
    } catch (\Exception $e) {
        $this->logger->error('Error while trying to automatically tag an entry.', [...]);
    }
}
```

### 3.2 URL 重定向处理

```php
// src/Helper/ContentProxy.php:328-393
private function updateOriginUrl(Entry $entry, $url)
{
    // 1. 忽略规则检查（如 feedproxy.google.com 等跳转服务）
    if ($this->ignoreOriginProcessor->process($entry)) {
        $entry->setUrl($url);
        return false;
    }
    
    // 2. 计算 URL 差异部分
    $parsed_entry_url = parse_url($entry->getUrl());
    $parsed_content_url = parse_url($url);
    $diff_ec = array_diff_assoc($parsed_entry_url, $parsed_content_url);
    $diff_ce = array_diff_assoc($parsed_content_url, $parsed_entry_url);
    $diff = array_merge($diff_ec, $diff_ce);
    $diff_keys = array_keys($diff);
    sort($diff_keys);
    
    // 3. 根据差异部分执行不同策略
    switch ($diff_keys) {
        case ['path']:
            // 仅尾部斜杠差异或 URL 编码差异 → 直接更新
            if (($parsed_entry_url['path'] . '/' === $parsed_content_url['path'])
                || ($url === urldecode($entry->getUrl()))) {
                $entry->setUrl($url);
            }
            break;
        case ['scheme']:
            // 仅协议差异（http→https）→ 直接更新
            $entry->setUrl($url);
            break;
        case ['fragment']:
            // 仅锚点差异 → 不处理
            break;
        default:
            // 其他情况 → 保存原始 URL 到 origin_url，更新为新 URL
            if (empty($entry->getOriginUrl())) {
                $entry->setOriginUrl($entry->getUrl());
            }
            $entry->setUrl($url);
            break;
    }
}
```

### 3.3 字段验证处理器

#### 3.3.1 语言验证
```php
// src/Helper/ContentProxy.php:86-106
public function updateLanguage(Entry $entry, $value): void
{
    $value = str_replace('-', '_', $value);
    $errors = $this->validator->validate(
        $value,
        new LocaleConstraint(['canonicalize' => true])
    );
    if (0 === \count($errors)) {
        $entry->setLanguage($value);
    }
}
```

#### 3.3.2 日期解析
```php
// src/Helper/ContentProxy.php:136-153
public function updatePublishedAt(Entry $entry, $value): void
{
    $date = $value;
    if (false !== filter_var($date, \FILTER_VALIDATE_INT)) {
        $date = '@' . $date;  // 支持 Unix 时间戳
    }
    try {
        $date = new \DateTime($date);
        $entry->setPublishedAt($date);
    } catch (\Exception $e) {
        $this->logger->warning('Error while defining date', [...]);
    }
}
```

#### 3.3.3 标题编码清洗
```php
// src/Helper/ContentProxy.php:191-234
private function sanitizeContentTitle($title, $contentType)
{
    if ('application/pdf' === $contentType) {
        $title = $this->convertPdfEncodingToUTF8($title);  // 支持 UTF-8, UTF-16BE, WINDOWS-1252
    }
    return $this->sanitizeUTF8Text($title);  // 移除无效 UTF-8 字符
}
```

### 3.4 自动标签处理

`RuleBasedTagger` 基于 RulerZ 规则引擎实现自动标签：

```php
// src/Helper/RuleBasedTagger.php:30-52
public function tag(Entry $entry): void
{
    // 1. 获取用户定义的标签规则
    $rules = $this->getRulesForUser($entry->getUser());
    
    // 2. 修正阅读时间（适配用户阅读速度设置）
    $clonedEntry = $this->fixEntry($entry);
    
    // 3. 规则匹配
    foreach ($rules as $rule) {
        if (!$this->rulerz->satisfies($clonedEntry, $rule->getRule())) {
            continue;
        }
        
        // 4. 应用标签
        foreach ($rule->getTags() as $label) {
            $tag = $this->getTag($label);
            $entry->addTag($tag);
        }
    }
}
```

### 3.5 图片本地化处理（异步事件驱动）

`DownloadImagesSubscriber` 监听 `EntrySavedEvent` 事件，在条目保存后异步下载图片：

```php
// src/Event/Subscriber/DownloadImagesSubscriber.php:34-61
public function onEntrySaved(EntrySavedEvent $event): void
{
    if (!$this->enabled) {
        return;
    }
    
    $entry = $event->getEntry();
    
    // 1. 下载正文中的所有图片并替换链接
    $html = $this->downloadImages($entry);
    if (false !== $html) {
        $entry->setContent($html);
    }
    
    // 2. 下载预览图片
    $previewPicture = $this->downloadPreviewImage($entry);
    if (false !== $previewPicture) {
        $entry->setPreviewPicture($previewPicture);
    }
    
    $this->em->persist($entry);
    $this->em->flush();
}
```

图片处理核心逻辑：
```php
// src/Helper/DownloadImages.php:65-222
public function processHtml($entryId, $html, $url)
{
    // 1. 提取所有图片 URL（包括 srcset）
    $imagesUrls = self::extractImagesUrlsFromHtml($html);
    
    // 2. 逐个下载处理
    foreach ($imagesUrls as $image) {
        $newImage = $this->processSingleImage($entryId, $image, $url, $relativePath);
        if (false === $newImage) {
            continue;
        }
        // 3. 替换 HTML 中的图片链接
        $html = str_replace($image, $newImage, $html);
    }
    
    return $html;
}

// 单张图片处理
public function processSingleImage($entryId, $imagePath, $url, $relativePath = null)
{
    // 1. 构建绝对路径
    $absolutePath = $this->getAbsoluteLink($url, $imagePath);
    
    // 2. 下载图片
    $res = $this->client->request(Request::METHOD_GET, $absolutePath);
    
    // 3. 扩展名验证（MIME 头 + 文件头特征检测）
    $ext = $this->getExtensionFromResponse($res, $imagePath);
    
    // 4. 安全重生成图片（GD 库重编码，防范图片木马）
    switch ($ext) {
        case 'gif':
            imagegif($im, $localPath);
            break;
        case 'jpeg':
        case 'jpg':
            imagejpeg($im, $localPath, self::REGENERATE_PICTURES_QUALITY);
            break;
        case 'png':
            imagepng($im, $localPath, (int) ceil(self::REGENERATE_PICTURES_QUALITY / 100 * 9));
            break;
        case 'svg':
            // SVG 专用 sanitizer
            $sanitizer = new Sanitizer();
            $sanitizer->minify(true);
            $sanitizer->removeRemoteReferences(true);
            $cleanSVG = $sanitizer->sanitize($res->getContent());
            file_put_contents($localPath, $cleanSVG);
            break;
    }
    
    return $urlPath;  // 返回本地化后的 URL
}
```

## 四、整体协作时序

```
用户保存 URL
    ↓
ContentProxy::updateEntry()
    ├─ Graby::fetchContent(url)
    │   ├─ WallabagClient::request()
    │   │   ├─ Authenticator::loginIfRequired()
    │   │   │   └─ GrabySiteConfigBuilder::buildForHost(host)
    │   │   │       ├─ 主机名规范化
    │   │   │       ├─ 查询 SiteCredential
    │   │   │       └─ 加载 Graby 站点配置
    │   │   ├─ HTTP 请求（可能附加 Cookie）
    │   │   └─ Authenticator::loginIfRequested()
    │   │       └─ XPath 检测登录页面
    │   ├─ 解析器选择（配置驱动 / 通用解析）
    │   └─ 正文提取
    ├─ 标题编码清洗
    ├─ ContentProxy::stockEntry()
    │   ├─ URL 重定向处理
    │   ├─ 字段提取与验证（标题、正文、日期、语言、作者等）
    │   ├─ 阅读时间计算
    │   ├─ 预览图片提取
    │   └─ RuleBasedTagger::tag()
    │       └─ 规则匹配与标签应用
    └─ Entry 实体持久化
        ↓
EntrySavedEvent 事件触发
    ↓
DownloadImagesSubscriber::onEntrySaved()
    ├─ 正文图片批量下载与本地化
    └─ 预览图片下载与更新
```

## 五、关键设计模式与扩展点

### 5.1 设计模式应用

| 模式 | 应用位置 | 说明 |
|------|----------|------|
| 策略模式 | 解析器选择 | 配置驱动解析 vs 通用解析 |
| 模板方法 | `updateEntry()` | 定义抽取流程骨架，具体步骤可扩展 |
| 观察者模式 | `EntrySavedEvent` | 图片下载等后处理通过事件解耦 |
| 建造者模式 | `SiteConfigBuilder` | 复杂配置对象的构建 |
| 装饰器模式 | `WallabagClient` | 在 HTTP 客户端外层包装认证逻辑 |

### 5.2 扩展点

1. **自定义站点配置**：通过 `WALLABAG_SITE_CONFIG_FOLDERS` 添加自定义规则目录
2. **自定义解析器**：扩展 Graby 库添加新的正文提取策略
3. **后处理扩展**：监听 `EntrySavedEvent` 添加自定义处理逻辑
4. **认证方式扩展**：实现新的 `Authenticator` 支持 OAuth、2FA 等认证方式
5. **标签规则扩展**：通过 RulerZ 自定义操作符扩展标签规则表达式能力
