# 忽略来源规则（Ignore Origin Rule）代码分析

## 概述

忽略来源规则用于在网页抓取过程中，当 URL 发生重定向变化时，控制是否记录原始来源 URL。如果 URL 匹配了忽略规则，则直接更新为最终 URL，不保留 `origin_url` 字段。

---

## 一、规则存储层

### 1.1 实体类结构

#### 接口定义
`src/Entity/IgnoreOriginRuleInterface.php` 定义了规则的基本契约：

```php
interface IgnoreOriginRuleInterface
{
    public function getId();
    public function setRule(string $rule);
    public function getRule();
}
```

#### 两种规则类型

| 类型 | 实体类 | 作用范围 | 数据表 |
|------|--------|----------|--------|
| 用户级规则 | `IgnoreOriginUserRule` | 单个用户 | `ignore_origin_user_rule` |
| 实例级规则 | `IgnoreOriginInstanceRule` | 全局所有用户 | `ignore_origin_instance_rule` |

#### 规则约束
两种规则实体都使用 `RulerZAssert\ValidRule` 注解进行验证：

```php
/**
 * @RulerZAssert\ValidRule(
 *  allowed_variables={"host","_all"},
 *  allowed_operators={"=","~"}
 * )
 */
private $rule;
```

- **允许的变量**：
  - `host` - URL 的主机部分（如 `example.com`）
  - `_all` - 完整的 URL 字符串

- **允许的操作符**：
  - `=` - 精确匹配
  - `~` - 正则表达式匹配

#### 存储关联
用户级规则通过 `Config` 实体与用户关联：

```php
// src/Entity/Config.php:144-148
#[ORM\OneToMany(targetEntity: IgnoreOriginUserRule::class, mappedBy: 'config', cascade: ['remove'])]
private $ignoreOriginRules;
```

### 1.2 Repository 层

- `IgnoreOriginUserRuleRepository` - 用户规则仓储
- `IgnoreOriginInstanceRuleRepository` - 实例规则仓储

均继承自 Doctrine 的 `ServiceEntityRepository`，提供基础 CRUD 操作。

---

## 二、规则匹配引擎

### 2.1 核心处理器

`src/Helper/RuleBasedIgnoreOriginProcessor.php` 是规则匹配的核心处理类：

```php
class RuleBasedIgnoreOriginProcessor
{
    public function process(Entry $entry): bool
    {
        $url = $entry->getUrl();
        $userRules = $entry->getUser()->getConfig()->getIgnoreOriginRules()->toArray();
        $rules = array_merge($this->ignoreOriginInstanceRuleRepository->findAll(), $userRules);

        $parsed_url = parse_url($url);
        $parsed_url['_all'] = $url;

        foreach ($rules as $rule) {
            if ($this->rulerz->satisfies($parsed_url, $rule->getRule())) {
                $this->logger->info('Origin url matching ignore rule.', ['rule' => $rule->getRule()]);
                return true;
            }
        }

        return false;
    }
}
```

**处理流程**：
1. 从 Entry 中获取当前 URL
2. 合并**实例级规则**（全局）和**用户级规则**（个人）
3. 解析 URL，添加 `_all` 字段保存完整 URL
4. 使用 RulerZ 引擎逐条匹配规则
5. 只要有一条规则匹配，立即返回 `true`

### 2.2 规则匹配的 URL 基准

**关键点**：规则匹配使用的是 `$entry->getUrl()`，即 **Entry 实体当前保存的 URL**。

在 `updateOriginUrl()` 的调用场景中：
- `$entry->getUrl()` = **用户提交的原始 URL**（未经过重定向的）
- `$url` 参数 = **Graby 抓取后的最终 URL**（经过重定向的）

```php
// src/Helper/ContentProxy.php:356
if ($this->ignoreOriginProcessor->process($entry)) {
    // 匹配时，$entry->getUrl() 是原始 URL，$url 是最终 URL
    $entry->setUrl($url);
    return false;
}
```

**匹配时机的 URL 状态**：
| 变量 | 含义 | 值 |
|------|------|-----|
| `$entry->getUrl()` | Entry 当前 URL | `http://feedproxy.google.com/~r/example/~3/xxx/article` |
| `$url`（参数） | 抓取后的最终 URL | `https://example.com/article.html` |
| 规则匹配基准 | 用于规则判断的 URL | `$entry->getUrl()`（原始 URL） |

> **重要结论**：规则匹配的是**重定向之前的原始 URL**，而不是重定向之后的最终 URL。这意味着你可以针对"来源域名"（如 RSS 订阅源、短链接服务）配置规则。

### 2.3 RulerZ 规则引擎

项目使用 [RulerZ](https://github.com/K-Phoen/rulerz) 作为规则引擎，在 `services.yml` 中配置：

```yaml
RulerZ\RulerZ:
    alias: rulerz
```

### 2.4 自定义操作符

#### `~` 正则匹配操作符
`src/Operator/PHP/PatternMatches.php`：

```php
class PatternMatches
{
    public function __invoke($subject, $pattern)
    {
        $count = preg_match("`$pattern`i", (string) $subject);
        return \is_int($count) && $count > 0;
    }
}
```

- 使用 `preg_match` 进行正则匹配
- 分隔符为反引号 `` ` ``
- 不区分大小写（`i` 修饰符）

#### 其他操作符
- `Matches` (`matches`) - 不区分大小写的字符串包含匹配（`stripos`）
- `NotMatches` (`notmatches`) - 不区分大小写的字符串不包含匹配

操作符在 `services.yml` 中注册：

```yaml
Wallabag\Operator\PHP\PatternMatches:
    tags:
        - { name: rulerz.operator, target: native, operator: "~" }
```

### 2.5 规则示例

| 规则 | 匹配逻辑 | 适用场景 |
|------|----------|----------|
| `host = "feedproxy.google.com"` | 精确匹配主机名为 feedproxy.google.com | Google FeedBurner 跳转 |
| `host = "feeds.reuters.com"` | 精确匹配主机名为 feeds.reuters.com | 路透社 RSS 跳转 |
| `host ~ "example\.(com|org)"` | 正则匹配主机名为 example.com 或 example.org | 多域名匹配 |
| `_all ~ "^https?://www\.lemonde\.fr/tiny.*"` | 正则匹配 URL 路径前缀 | lemonde.fr 短链接 |

---

## 三、三个 URL 字段的职责分工

Entry 实体中有三个相关的 URL 字段，各自职责明确：

| 字段 | 数据库列 | 设置时机 | 含义 |
|------|----------|----------|------|
| `url` | `url` | 多次更新 | **最终 URL** - 经过所有重定向后的实际访问地址 |
| `given_url` | `given_url` | `ContentProxy::updateEntry()`:76 | **本次调用的 URL** - 每次 updateEntry 调用时传入的 URL 参数 |
| `origin_url` | `origin_url` | 条件性设置 | **来源 URL** - 记录重定向前的地址（可被规则抑制） |

### 3.1 字段定义与注释

```php
// src/Entity/Entry.php:65-73
/**
 * Define the url fetched by wallabag (the final url after potential redirections).
 */
private $url;

// src/Entity/Entry.php:81-88
/**
 * From where user retrieved/found the url (an other article, a twitter, or the given_url if non are provided).
 */
private $originUrl;

// src/Entity/Entry.php:90-97
/**
 * Define the url entered by the user (without redirections).
 */
private $givenUrl;
```

### 3.2 given_url 的写入时机与覆盖行为

**⚠️ 重要修正**：`given_url` 并非永不修改！它会在**每次 `updateEntry` 调用时被无条件覆盖**。

```php
// src/Helper/ContentProxy.php:43-79
public function updateEntry(Entry $entry, $url, array $content = [], $disableContentUpdate = false): void
{
    // ... Graby 抓取或跳过抓取的判断 ...

    // 第76行：无条件设置 given_url，每次调用都会覆盖！
    // 注意：无论是否执行了抓取，这里都会执行！
    $entry->setGivenUrl($url);

    $this->stockEntry($entry, $content);
}
```

**`setGivenUrl` 的内部实现**：
```php
// src/Entity/Entry.php:903-908
public function setGivenUrl($givenUrl)
{
    $this->givenUrl = $givenUrl;
    $this->hashedGivenUrl = UrlHasher::hashUrl($givenUrl);
    return $this;
}
```

**特性**：
- ✅ 每次 `updateEntry` 调用都会设置（第76行，无条件执行）
- ⚠️ **会被后续调用覆盖**，不是"永不修改"
- ✅ 不受忽略规则影响
- ✅ 可用于重复检查（`findByUrlAndUserId` 会同时检查 url 和 given_url）

### 3.3 各个入口中 given_url 的写入时机详析

所有入口最终都调用 `ContentProxy::updateEntry($entry, $url, ...)`，但传递的 `$url` 参数不同，导致 `given_url` 的最终值不同。

#### 入口1：Web 表单创建（新 Entry）
```php
// src/Controller/EntryController.php:192
$this->updateEntry($entry);
    ↓
// src/Controller/EntryController.php:703-708
private function updateEntry(Entry $entry, $prefixMessage = 'entry_saved'): void
{
    try {
        $this->contentProxy->updateEntry($entry, $entry->getUrl());
    } ...
}
```
- **传递的 `$url`**：`$entry->getUrl()` = 用户在表单中输入的原始 URL
- **given_url 结果**：✅ 设置为用户提交的原始 URL

#### 入口2：Bookmarklet 创建（新 Entry）
```php
// src/Controller/EntryController.php:217-221
$entry = new Entry($this->getUser());
$entry->setUrl($request->query->get('url'));
$this->updateEntry($entry);
```
- **传递的 `$url`**：`$entry->getUrl()` = Bookmarklet 传入的 URL
- **given_url 结果**：✅ 设置为 Bookmarklet 传入的 URL

#### 入口3：Web 重新抓取（已有 Entry）
```php
// src/Controller/EntryController.php:412
$this->updateEntry($entry, 'entry_reloaded');
    ↓
// src/Controller/EntryController.php:708
$this->contentProxy->updateEntry($entry, $entry->getUrl());
```
- **传递的 `$url`**：`$entry->getUrl()` = **上一次保存的最终 URL**（不是原始 URL！）
- **given_url 结果**：⚠️ **被覆盖为当前的最终 URL**，丢失原始的 given_url

#### 入口4：API POST 单条创建

**⚠️ 重要发现：无论条目是否已存在，都会调用 updateEntry！**

```php
// src/Controller/Api/EntryRestController.php:726-754
$url = $request->request->get('url');

$entry = $entryRepository->findByUrlAndUserId(
    $url,
    $this->getUser()->getId()
);

if (false === $entry) {
    $entry = new Entry($this->getUser());
    $entry->setUrl($url);
}

// ⚠️ 注意：这里在 if 块之外！无论条目是否已存在，都会执行！
$contentProxy->updateEntry(
    $entry,
    $entry->getUrl(),  // ⚠️ 如果条目已存在，这里是已保存的最终 URL，不是用户传入的原始 URL！
    [...]
);
```

**分支 A：条目不存在（新创建）**
- **传递的 `$url`**：`$entry->getUrl()` = API 请求参数中的 url（用户传入的原始 URL）
- **given_url 结果**：✅ 设置为 API 传入的 URL

**分支 B：条目已存在**
- **传递的 `$url`**：`$entry->getUrl()` = **数据库中已保存的最终 URL**
- **given_url 结果**：⚠️ **被覆盖为已保存的最终 URL**，如果之前 given_url 是原始值则会丢失

#### 入口5：API POST 批量创建

**⚠️ 与单条创建不同：只有条目不存在时才调用 updateEntry！**

```php
// src/Controller/Api/EntryRestController.php:552-564
foreach ($urls as $key => $url) {
    $entry = $entryRepository->findByUrlAndUserId(
        $url,
        $this->getUser()->getId()
    );

    // ⚠️ 注意：这里在 if 块之内！只有条目不存在时才执行！
    if (false === $entry) {
        $entry = new Entry($this->getUser());
        $contentProxy->updateEntry($entry, $url);
    }
    // 条目已存在时：不调用 updateEntry，given_url 保持不变
}
```

- **条目不存在时**：✅ `given_url` 设置为批量列表中的 URL
- **条目已存在时**：❌ 不调用 `updateEntry`，`given_url` 保持数据库中的原值

#### 入口6：API PATCH 更新内容（已有 Entry）
```php
// src/Controller/Api/EntryRestController.php:947-956
if (!empty($data['content'])) {
    $contentProxy->updateEntry(
        $entry,
        $entry->getUrl(),  // 当前的最终 URL
        ['html' => $data['content']],
        true  // disableContentUpdate = true
    );
}
```
- **触发条件**：仅当 PATCH 请求包含 `content` 参数时才调用 `updateEntry`
- **传递的 `$url`**：`$entry->getUrl()` = 当前的最终 URL
- **given_url 结果**：⚠️ 如果传入了 content，**被覆盖为当前的最终 URL**

#### 入口7：API PATCH 重新抓取（已有 Entry）
```php
// src/Controller/Api/EntryRestController.php:1057
$contentProxy->updateEntry($entry, $entry->getUrl());
```
- **传递的 `$url`**：`$entry->getUrl()` = 当前的最终 URL
- **given_url 结果**：⚠️ **被覆盖为当前的最终 URL**

### 3.4 given_url 覆盖行为汇总表

| 入口类型 | 调用 `updateEntry`？ | 传递的 `$url` 参数 | given_url 最终值 | 会被覆盖？ |
|---------|---------------------|-------------------|-----------------|------------|
| Web 创建（新） | ✅ 是 | 用户提交的原始 URL | 用户提交的原始 URL | ❌ 首次设置 |
| Bookmarklet 创建（新） | ✅ 是 | Bookmarklet 传入的 URL | Bookmarklet 传入的 URL | ❌ 首次设置 |
| Web 重抓（已有） | ✅ 是 | 当前的最终 URL | 当前的最终 URL | ✅ 是，丢失原始值 |
| **API POST 单条（新）** | ✅ 是 | API 传入的原始 URL | API 传入的 URL | ❌ 首次设置 |
| **API POST 单条（已存在）** | ✅ 是 | **已保存的最终 URL** | 已保存的最终 URL | ✅ 是，可能丢失原始值 |
| **API POST 批量（新）** | ✅ 是 | 批量列表中的 URL | 批量列表中的 URL | ❌ 首次设置 |
| **API POST 批量（已存在）** | ❌ 否 | - | 保持数据库原值 | ❌ 否 |
| API PATCH（无 content） | ❌ 否 | - | 保持原值 | ❌ 否 |
| API PATCH（有 content） | ✅ 是 | 当前的最终 URL | 当前的最终 URL | ✅ 是，丢失原始值 |
| API PATCH 重抓 | ✅ 是 | 当前的最终 URL | 当前的最终 URL | ✅ 是，丢失原始值 |

> **关键结论 1**：API POST 单条创建和批量创建的逻辑不同！单条创建无论是否已存在都会调用 updateEntry，批量创建只有不存在时才调用。
>
> **关键结论 2**：对已有 Entry 执行**重新抓取**、**API POST 单条（已存在）**或**PATCH 更新内容**时，`given_url` 会被覆盖为当前的最终 URL，丢失用户最初提交的 URL。这是代码的实际行为，与字段注释中的"the url entered by the user"不完全一致。

### 3.5 updateEntry 抓取被跳过的场景分支

`updateEntry` 中调用 Graby 抓取是有条件的，满足以下任一条件时会**跳过抓取**：

```php
// src/Helper/ContentProxy.php:50
if ((empty($content) || false === $this->validateContent($content)) && false === $disableContentUpdate) {
    // 执行抓取：调用 $this->graby->fetchContent($url)
} else {
    // 跳过抓取：使用传入的 $content
}
```

**跳过抓取的条件**（满足任一即可）：
1. `!empty($content) && $this->validateContent($content)` = 传入了完整内容（有 title、html、url）
2. `$disableContentUpdate === true` = 明确禁用内容更新

**内容验证逻辑**：
```php
// src/Helper/ContentProxy.php:400-403
private function validateContent(array $content)
{
    return !empty($content['title']) && !empty($content['html']) && !empty($content['url']);
}
```

**⚠️ 重要：即使跳过抓取，仍然会执行以下操作**：
```php
// 无论是否跳过抓取，以下代码始终执行：
$content['url'] = !empty($content['url']) ? $content['url'] : $url;  // 第67行
if (empty($entry->getUrl()) && !empty($url)) {  // 第72-74行
    $entry->setUrl($url);
}
$entry->setGivenUrl($url);  // 第76行 ⚠️ 始终覆盖 given_url！
$this->stockEntry($entry, $content);  // 第78行 ⚠️ 始终调用 stockEntry！
```

**stockEntry 中的规则检查**：
```php
// src/Helper/ContentProxy.php:243-245
private function stockEntry(Entry $entry, array $content): void
{
    $this->updateOriginUrl($entry, $content['url']);  // ⚠️ 规则检查仍然执行！
    // ...
}
```

**跳过抓取场景的完整执行链**：
```
ContentProxy::updateEntry($entry, $url, $content, true)
    │
    ├─ 跳过 Graby 抓取（使用传入的 $content）
    │
    ├─ $entry->setGivenUrl($url)  ← 仍然覆盖 given_url！
    │
    └─ stockEntry($entry, $content)
        └─ updateOriginUrl($entry, $content['url'])
            └─ 【规则检查点】仍然会执行！
```

**场景示例：API PATCH 带 content 参数**
- `disableContentUpdate = true` → 跳过抓取
- 但 `setGivenUrl($entry->getUrl())` 仍然执行 → given_url 被覆盖
- 且 `stockEntry` → `updateOriginUrl` 仍然执行 → 规则检查仍然生效（如果 `$content['url']` 与 `$entry->getUrl()` 不同）

### 3.6 origin_url 的设置条件

`origin_url` 只有在以下全部条件满足时才会被自动设置：

1. URL 发生了重定向（`$entry->getUrl() !== $url`）
2. 规则不匹配（`ignoreOriginProcessor->process()` 返回 `false`）
3. URL 差异不仅仅是 path/scheme/fragment 变化
4. `origin_url` 当前为空

```php
// src/Helper/ContentProxy.php:386-391
default:
    if (empty($entry->getOriginUrl())) {
        $entry->setOriginUrl($entry->getUrl());
    }
    $entry->setUrl($url);
    break;
```

### 3.7 规则命中后 origin_url 的保留边界

**规则命中时的代码行为**：
```php
// src/Helper/ContentProxy.php:356-360
if ($this->ignoreOriginProcessor->process($entry)) {
    $entry->setUrl($url);  // 只更新 url
    return false;  // 提前返回，跳过后续逻辑
}
```

**关键边界分析**：

| 场景 | 规则匹配前 origin_url 状态 | 规则匹配后 origin_url 状态 | 说明 |
|------|--------------------------|--------------------------|------|
| 1 | `null`（空） | `null`（空） | ✅ 符合预期：不设置 origin_url |
| 2 | 已有值（如之前设置过） | **保持原值不变** | ⚠️ 重要：规则匹配不会清空已有的 origin_url！ |
| 3 | 已有值 + API 传入新值 | API 传入的值 | API 覆盖优先级最高 |

> **边界结论**：忽略规则的作用是**"阻止设置新的 origin_url"**，而不是**"清除已有的 origin_url"**。如果 origin_url 之前已经有值，规则匹配后会保持不变。

**典型场景**：
1. 第一次创建时规则不匹配 → 设置了 origin_url
2. 后来添加了规则
3. 重新抓取时规则匹配 → url 会更新，但 origin_url 仍然保留第一次的值！

### 3.8 origin_url 与 given_url 的边界对照

| 维度 | `origin_url` | `given_url` |
|------|-------------|-------------|
| **设计目的** | 记录"从哪里发现这个链接"（如 RSS 源、Twitter、其他文章） | 记录"用户输入的原始 URL" |
| **设置时机** | URL 重定向且规则不匹配时 | 每次 `updateEntry` 调用时（无条件） |
| **可被规则抑制** | ✅ 是（匹配规则时不设置新值） | ❌ 否（总是设置） |
| **规则匹配后已有值** | ⚠️ 保持不变（不清除） | ⚠️ 仍会被覆盖 |
| **可被 API 覆盖** | ✅ 是（POST/PATCH 都支持 `origin_url` 参数） | ❌ 否（API 无 `given_url` 参数） |
| **重抓时是否变化** | 不变（如已设置且不为空） | ⚠️ 会被覆盖为当前的最终 URL |
| **空值含义** | 没有发生重定向，或规则匹配，或 API 未设置 | 从未调用过 `updateEntry`（极少见） |
| **用于去重检查** | ❌ 否 | ✅ 是（`findByUrlAndUserId` 同时检查 url 和 given_url） |
| **抓取被跳过时** | 仍可能变化（如果 `$content['url']` 不同） | ⚠️ 仍会被覆盖 |

### 3.9 三个 URL 的完整生命周期（以重抓为例）

```
【初次创建】
用户提交 URL: http://feedproxy.google.com/example
    │
    ├─ 1. Entry 创建 → $entry->setUrl('http://feedproxy.google.com/example')
    │
    ├─ 2. ContentProxy::updateEntry($entry, 'http://feedproxy.google.com/example')
    │   └─ $entry->setGivenUrl('http://feedproxy.google.com/example')  ← 初次设置
    │
    ├─ 3. Graby 抓取 → 跟随重定向到 https://example.com/article
    │
    └─ 4. updateOriginUrl() 检查
        ├─ 规则匹配 host = feedproxy.google.com？
        │   ├─ 匹配 → url = https://example.com/article
        │   │          origin_url = null
        │   │          given_url = http://feedproxy.google.com/example
        │   └─ 不匹配 → url = https://example.com/article
        │                origin_url = http://feedproxy.google.com/example
        │                given_url = http://feedproxy.google.com/example
        └─ 状态：url=最终, given_url=原始, origin_url=规则决定

【重新抓取】
用户点击"重新抓取"按钮
    │
    ├─ 1. 从数据库加载 Entry
    │   ├─ url = https://example.com/article（上次的最终 URL）
    │   ├─ given_url = http://feedproxy.google.com/example（原始值）
    │   └─ origin_url = 规则决定的值
    │
    ├─ 2. ContentProxy::updateEntry($entry, 'https://example.com/article')
    │   └─ $entry->setGivenUrl('https://example.com/article')  ← ⚠️ 被覆盖！
    │
    ├─ 3. Graby 抓取 → https://example.com/article（无变化）
    │
    └─ 4. updateOriginUrl() 检查
        ├─ url 未变化，直接返回
        └─ 最终状态：
            ├─ url = https://example.com/article
            ├─ given_url = https://example.com/article  ← 丢失原始值！
            └─ origin_url = 保持不变（如果之前有值的话）
```

---

## 四、抓取入口与规则衔接

### 4.1 抓取入口总览

所有抓取路径最终都会汇聚到 `ContentProxy::updateEntry()` 方法。

| 入口类型 | 调用位置 |
|----------|----------|
| Web 表单提交 | `EntryController::addEntryFormAction()`:192 |
| Bookmarklet | `EntryController::addEntryViaBookmarkletAction()`:221 |
| 重新抓取 | `EntryController::reloadAction()`:412 |
| API 单条创建 | `EntryRestController::postEntriesAction()`:741 |
| API 批量创建 | `EntryRestController::postEntriesListAction()`:563 |
| API 重新抓取 | `EntryRestController::patchEntriesReloadAction()`:1057 |
| API PATCH 更新内容 | `EntryRestController::patchEntriesAction()`:949 |
| 异步消费（RabbitMQ） | `AbstractConsumer::handleMessage()`:56 → `Import::parseEntry()` |
| 异步消费（Redis） | `AbstractConsumer::handleMessage()`:56 → `Import::parseEntry()` |

### 4.2 API 手动写入 origin_url 的影响

在 API 调用中，`origin_url` 可以通过请求参数手动设置，但需要注意**执行顺序**：

#### API POST 创建时的顺序
```php
// src/Controller/Api/EntryRestController.php:740-776
try {
    // 步骤1：调用 ContentProxy 更新 Entry（包含规则检查）
    $contentProxy->updateEntry($entry, $entry->getUrl(), [...]);
} catch (\Exception $e) {
    // ...
}

// ... 其他字段处理 ...

// 步骤2：API 手动设置 origin_url（在 updateEntry 之后！）
if (!empty($data['origin_url'])) {
    $entry->setOriginUrl($data['origin_url']);  // 第775行
}
```

#### API PATCH 更新时的顺序
```php
// src/Controller/Api/EntryRestController.php:947-1008
// 步骤1：仅当有 content 时才调用 updateEntry
if (!empty($data['content'])) {
    try {
        $contentProxy->updateEntry($entry, $entry->getUrl(), [...], true);
    } catch (\Exception $e) { ... }
}

// ... 其他字段处理 ...

// 步骤2：API 手动设置 origin_url（无论是否调用了 updateEntry）
if (!empty($data['origin_url'])) {
    $entry->setOriginUrl($data['origin_url']);  // 第1007行
}
```

**关键结论**：API 手动写入的 `origin_url` **会覆盖**规则检查的结果，且 PATCH 接口**不要求**必须先调用 `updateEntry`。

| 场景 | 规则匹配结果 | API origin_url 参数 | 最终 origin_url |
|------|-------------|-------------------|-----------------|
| API POST 创建-1 | true（忽略） | 未提供 | null |
| API POST 创建-2 | true（忽略） | 提供 = "http://source.com" | "http://source.com" |
| API POST 创建-3 | false（保留） | 未提供 | 自动设置的原始 URL |
| API POST 创建-4 | false（保留） | 提供 = "http://custom.com" | "http://custom.com" |
| API PATCH（无 content）-1 | 不执行规则检查 | 提供 = "http://source.com" | "http://source.com" |
| API PATCH（无 content）-2 | 不执行规则检查 | 未提供 | 保持原值 |

**生效边界**：
- API 的 `origin_url` 设置在 `ContentProxy::updateEntry()` **之后**执行（POST 场景）
- API PATCH 可以**独立设置** `origin_url`，不需要触发抓取
- 手动设置的优先级高于自动逻辑
- 即使规则匹配应该忽略，只要 API 传入了 `origin_url`，最终都会被设置
- 这为导入场景提供了灵活性，但也可能绕过规则

### 4.3 调用链分析

```
抓取入口
    ↓
EntryController::updateEntry()  [src/Controller/EntryController.php:703]
    ↓
ContentProxy::updateEntry()     [src/Helper/ContentProxy.php:43]
    ├─ 步骤1：判断是否需要抓取
    │   ├─ 是 → Graby 抓取网页内容
    │   └─ 否 → 跳过抓取，使用传入的 content
    │
    ├─ 步骤2：设置 given_url（第76行，每次调用都覆盖！无论是否抓取）
    │   $entry->setGivenUrl($url)
    │
    └─ 步骤3：存储抓取结果
        ↓
        ContentProxy::stockEntry()  [src/Helper/ContentProxy.php:243]
            ├─ 【规则生效点】updateOriginUrl() 检查 URL 变化
            │   ├─ 检查 URL 是否变化
            │   ├─ 调用 ignoreOriginProcessor->process()
            │   └─ 根据规则匹配结果决定是否设置 origin_url
            ├─ 设置标题、内容、阅读时间等
            └─ 自动标签
```

---

## 五、规则生效时机详解

### 5.1 核心生效点：`updateOriginUrl()`

`src/Helper/ContentProxy.php:328-393`

```php
private function updateOriginUrl(Entry $entry, $url)
{
    // 前置检查：URL 为空或未变化，直接返回
    if (empty($url) || $entry->getUrl() === $url) {
        return false;
    }

    // 解析两个 URL 并计算差异
    $parsed_entry_url = parse_url($entry->getUrl());
    $parsed_content_url = parse_url($url);
    $diff_ec = array_diff_assoc($parsed_entry_url, $parsed_content_url);
    $diff_ce = array_diff_assoc($parsed_content_url, $parsed_entry_url);
    $diff = array_merge($diff_ec, $diff_ce);
    $diff_keys = array_keys($diff);
    sort($diff_keys);

    // ==================== 规则检查点 ====================
    // 调用规则处理器检查是否匹配忽略规则
    if ($this->ignoreOriginProcessor->process($entry)) {
        // 规则匹配：直接更新 URL，不设置 origin_url
        // ⚠️ 注意：如果 origin_url 已有值，不会被清除！
        $entry->setUrl($url);
        return false;  // 提前返回，跳过后续逻辑
    }
    // ===================================================

    // 规则不匹配：根据 URL 差异进行不同处理
    switch ($diff_keys) {
        case ['path']:
            // 仅路径变化：处理尾部斜杠或 URL 编码问题
            if (($parsed_entry_url['path'] . '/' === $parsed_content_url['path'])
                || ($url === urldecode($entry->getUrl()))) {
                $entry->setUrl($url);  // 直接更新，不设 origin_url
            }
            break;
        case ['scheme']:
            $entry->setUrl($url);  // 仅协议变化，直接更新
            break;
        case ['fragment']:
            // 仅锚点变化：什么都不做
            break;
        default:
            // 其他情况（如主机变化）：设置 origin_url 并更新 url
            if (empty($entry->getOriginUrl())) {
                $entry->setOriginUrl($entry->getUrl());
            }
            $entry->setUrl($url);
            break;
    }
}
```

### 5.2 生效边界条件

规则**仅在以下全部条件满足时**才会被检查和生效：

| 条件 | 说明 | 代码位置 |
|------|------|----------|
| 1. URL 已变化 | 抓取后的 URL 与 Entry 当前 URL 不同 | `ContentProxy.php:330` |
| 2. URL 不为空 | 两个 URL 都有有效值 | `ContentProxy.php:330` |
| 3. 非 API 强制覆盖 | API 没有手动传入 origin_url（仅 API 场景） | `EntryRestController.php:774-776, 1006-1008` |

### 5.3 匹配后的行为

当规则匹配成功（`process()` 返回 `true`）时：

| 行为 | 说明 |
|------|------|
| `$entry->setUrl($url)` | 直接更新为最终抓取到的 URL |
| 不设置新的 `origin_url` | 原始 URL 不被记录为新的来源 |
| ⚠️ 不清除已有的 `origin_url` | 如果之前已有值，保持不变 |
| 跳过 URL 差异分析 | 直接 `return false`，不进入后续 switch 逻辑 |
| 记录日志 | `info` 级别日志记录匹配的规则 |

### 5.4 不匹配时的行为

当规则不匹配时，根据 URL 差异部分决定：

| URL 差异 | 行为 | 是否设置 origin_url |
|----------|------|---------------------|
| 仅 `path` 变化 | 处理特殊情况（尾部斜杠、URL 编码） | 通常不设置 |
| 仅 `scheme` 变化 | 直接更新 URL | 不设置 |
| 仅 `fragment` 变化 | 不做任何处理 | 不设置 |
| 其他情况（如 `host` 变化） | 设置 origin_url 并更新 url | 设置（如果为空） |

---

## 六、默认实例规则的落地链路

### 6.1 完整初始化流程

默认实例规则从配置文件到数据库经历以下链路：

```
配置文件定义
    ↓
容器参数绑定
    ↓
服务注入
    ↓
安装命令执行
    ↓
写入数据库
```

### 6.2 步骤详解

#### 步骤1：配置文件定义
`app/config/wallabag.yml:162-168`

```yaml
wallabag.default_ignore_origin_instance_rules:
    -
        rule: host = "feedproxy.google.com"
    -
        rule: host = "feeds.reuters.com"
    -
        rule: _all ~ "https?://www\.lemonde\.fr/tiny.*"
```

默认包含三条规则，覆盖常见的 RSS 跳转和短链接场景。

#### 步骤2：容器参数绑定
`app/config/services.yml:42`

```yaml
parameters:
    $defaultIgnoreOriginInstanceRules: '%wallabag.default_ignore_origin_instance_rules%'
```

将配置文件中的值绑定为容器参数。

#### 步骤3：服务注入
`app/config/services.yml:275-278`

```yaml
Wallabag\Command\InstallCommand:
    arguments:
        $defaultIgnoreOriginInstanceRules: '%wallabag.default_ignore_origin_instance_rules%'
```

通过构造函数注入到 `InstallCommand`。

#### 步骤4：安装命令执行
`src/Command/InstallCommand.php:340-368`

```php
private function setupConfig()
{
    $this->io->section('Step 4 of 4: Config setup.');

    // 先清空现有规则
    $this->entityManager->createQuery('DELETE FROM Wallabag\Entity\IgnoreOriginInstanceRule')->execute();

    // 遍历默认规则，逐条插入
    foreach ($this->defaultIgnoreOriginInstanceRules as $ignore_origin_instance_rule) {
        $newIgnoreOriginInstanceRule = new IgnoreOriginInstanceRule();
        $newIgnoreOriginInstanceRule->setRule($ignore_origin_instance_rule['rule']);
        $this->entityManager->persist($newIgnoreOriginInstanceRule);
    }

    $this->entityManager->flush();
}
```

**关键特性**：
- ✅ 每次安装都会**清空**现有的实例级规则
- ✅ 然后重新写入默认规则
- ✅ 这意味着手动修改的实例级规则在重新安装时会丢失
- ✅ 仅在 `wallabag:install` 命令执行时触发

### 6.3 触发时机

默认规则初始化仅在以下场景触发：

| 场景 | 是否初始化 | 说明 |
|------|-----------|------|
| 全新安装 | ✅ 是 | 第一次运行 `wallabag:install` |
| 重置安装 | ✅ 是 | 使用 `--reset` 选项重新安装 |
| 正常升级 | ❌ 否 | 升级命令不重新初始化 |
| 日常运行 | ❌ 否 | 仅读取数据库中的规则 |

---

## 七、可复现调用路径

以下是可以用于验证各个场景的具体调用路径和代码步骤。

### 7.1 场景1：创建新 Entry，规则匹配，origin_url 被抑制

**调用路径**：Web 表单提交

1. 用户在 `/new-entry` 页面输入 URL: `http://feedproxy.google.com/example`
2. `EntryController::addEntryFormAction()` 创建新 Entry，设置 `$entry->setUrl('http://feedproxy.google.com/example')`
3. 调用 `$this->updateEntry($entry)` → `ContentProxy::updateEntry($entry, 'http://feedproxy.google.com/example')`
4. 第76行：`$entry->setGivenUrl('http://feedproxy.google.com/example')`
5. Graby 抓取，跟随重定向到 `https://example.com/article`
6. `stockEntry()` → `updateOriginUrl($entry, 'https://example.com/article')`
7. 规则检查：`host = "feedproxy.google.com"` 匹配成功
8. 执行 `$entry->setUrl('https://example.com/article')`，origin_url 保持 null
9. 最终状态：
   - `url` = `https://example.com/article`
   - `given_url` = `http://feedproxy.google.com/example`
   - `origin_url` = `null`

### 7.2 场景2：创建新 Entry，规则不匹配，origin_url 被设置

**调用路径**：API POST 创建

1. POST 请求 `/api/entries.json`，参数 `url=http://othersite.com/link`
2. `EntryRestController::postEntriesAction()` 创建新 Entry
3. 调用 `$contentProxy->updateEntry($entry, 'http://othersite.com/link', ...)`
4. 第76行：`$entry->setGivenUrl('http://othersite.com/link')`
5. Graby 抓取，跟随重定向到 `https://example.com/article`
6. 规则检查：无匹配规则
7. URL 差异：host 变化，进入 default 分支
8. 执行 `$entry->setOriginUrl('http://othersite.com/link')` 和 `$entry->setUrl('https://example.com/article')`
9. 最终状态：
   - `url` = `https://example.com/article`
   - `given_url` = `http://othersite.com/link`
   - `origin_url` = `http://othersite.com/link`

### 7.3 场景3：重新抓取，given_url 被覆盖

**调用路径**：Web 重新抓取

1. 从数据库加载已有 Entry（来自场景1）：
   - `url` = `https://example.com/article`
   - `given_url` = `http://feedproxy.google.com/example`
   - `origin_url` = `null`
2. 用户点击重新抓取按钮，POST 到 `/reload/{id}`
3. `EntryController::reloadAction()` 调用 `$this->updateEntry($entry, 'entry_reloaded')`
4. → `ContentProxy::updateEntry($entry, 'https://example.com/article')` （注意：传递的是当前的 url！）
5. 第76行：`$entry->setGivenUrl('https://example.com/article')` ← **被覆盖！**
6. Graby 抓取，URL 无变化
7. `updateOriginUrl()` 检查到 URL 未变化，直接返回
8. 最终状态：
   - `url` = `https://example.com/article`
   - `given_url` = `https://example.com/article` ← 丢失了原始值！
   - `origin_url` = `null`（保持不变）

### 7.4 场景4：API POST 单条创建命中已存在条目，given_url 被覆盖

**调用路径**：API POST 单条创建（条目已存在）

1. 数据库中已有 Entry：
   - `url` = `https://example.com/article`
   - `given_url` = `http://feedproxy.google.com/example`（原始值）
   - `origin_url` = `null`
2. POST 请求 `/api/entries.json`，参数 `url=http://feedproxy.google.com/example`
3. `EntryRestController::postEntriesAction()` 执行 `findByUrlAndUserId`，找到已存在的 Entry
4. ⚠️ 注意：不会进入 `if (false === $entry)` 块，`$entry->getUrl()` 仍然是数据库中的 `https://example.com/article`
5. 调用 `$contentProxy->updateEntry($entry, 'https://example.com/article', ...)`
6. 第76行：`$entry->setGivenUrl('https://example.com/article')` ← **被覆盖！丢失原始值**
7. 最终状态：
   - `url` = `https://example.com/article`
   - `given_url` = `https://example.com/article` ← 被覆盖为最终 URL
   - `origin_url` = `null`（保持不变）

### 7.5 场景5：API POST 批量创建命中已存在条目，given_url 保持不变

**调用路径**：API POST 批量创建（条目已存在）

1. 数据库中已有 Entry：
   - `url` = `https://example.com/article`
   - `given_url` = `http://feedproxy.google.com/example`（原始值）
   - `origin_url` = `null`
2. POST 请求 `/api/entries/multiple.json`，参数 `urls=["http://feedproxy.google.com/example"]`
3. `EntryRestController::postEntriesListAction()` 执行 `findByUrlAndUserId`，找到已存在的 Entry
4. ⚠️ 注意：跳过了 `if (false === $entry)` 块，**不调用** `updateEntry`！
5. 直接持久化 Entry，given_url 保持不变
6. 最终状态：
   - `url` = `https://example.com/article`
   - `given_url` = `http://feedproxy.google.com/example` ← 保持原始值不变
   - `origin_url` = `null`（保持不变）

### 7.6 场景6：API 手动设置 origin_url 覆盖规则

**调用路径**：API POST 创建，带 origin_url 参数

1. POST 请求 `/api/entries.json`，参数：
   - `url=http://feedproxy.google.com/example`
   - `origin_url=http://twitter.com/user/status/123`
2. `EntryRestController::postEntriesAction()` 创建新 Entry
3. 调用 `$contentProxy->updateEntry($entry, 'http://feedproxy.google.com/example', ...)`
4. Graby 抓取，重定向到 `https://example.com/article`
5. 规则检查：`host = "feedproxy.google.com"` 匹配成功，origin_url 不设置
6. 但在第774-776行：
   ```php
   if (!empty($data['origin_url'])) {
       $entry->setOriginUrl($data['origin_url']);  // 覆盖！
   }
   ```
7. 最终状态：
   - `url` = `https://example.com/article`
   - `given_url` = `http://feedproxy.google.com/example`
   - `origin_url` = `http://twitter.com/user/status/123` ← API 参数优先

### 7.7 场景7：API PATCH 独立设置 origin_url（不触发抓取）

**调用路径**：API PATCH 更新，仅修改 origin_url

1. PATCH 请求 `/api/entries/{id}.json`，参数：
   - `origin_url=http://custom-source.com`
2. `EntryRestController::patchEntriesAction()` 处理请求
3. 无 `content` 参数，**不调用** `ContentProxy::updateEntry()`
4. **不执行**规则检查
5. 第1006-1008行：
   ```php
   if (!empty($data['origin_url'])) {
       $entry->setOriginUrl($data['origin_url']);
   }
   ```
6. 最终状态：
   - `url` = 保持不变
   - `given_url` = 保持不变
   - `origin_url` = `http://custom-source.com` ← 直接设置，绕过所有规则

### 7.8 场景8：API PATCH 更新内容，抓取被跳过但 given_url 仍被覆盖

**调用路径**：API PATCH 更新，带 content 参数

1. PATCH 请求 `/api/entries/{id}.json`，参数：
   - `content=<p>Updated content</p>`
2. `EntryRestController::patchEntriesAction()` 处理请求
3. 有 `content` 参数，调用：
   ```php
   $contentProxy->updateEntry(
       $entry,
       $entry->getUrl(),  // 当前的最终 URL
       ['html' => $data['content']],
       true  // disableContentUpdate = true ← 跳过抓取！
   );
   ```
4. 跳过 Graby 抓取，使用传入的 content
5. 第76行：`$entry->setGivenUrl($entry->getUrl())` ← 仍然被覆盖为当前的最终 URL
6. `stockEntry()` → `updateOriginUrl()` 检查：`$content['url']` 与 `$entry->getUrl()` 相同，不设置 origin_url
7. 最终状态：
   - `url` = 保持不变
   - `given_url` = 被覆盖为当前的 url ← 丢失原始值
   - `origin_url` = 保持不变

### 7.9 场景9：规则匹配后已有 origin_url 保持不变

**调用路径**：先不匹配规则创建，后添加规则再重抓

1. **阶段1：初始创建（无规则）**
   - 创建 Entry: `http://othersite.com/link` → 重定向到 `https://example.com/article`
   - 无规则匹配 → 设置 `origin_url = http://othersite.com/link`
   - 状态：`url=最终, given_url=原始, origin_url=http://othersite.com/link`

2. **阶段2：添加规则**
   - 添加规则：`host = "othersite.com"`

3. **阶段3：重新抓取**
   - 调用 `updateEntry($entry, 'https://example.com/article')`
   - `updateOriginUrl()` 检查 URL 无变化，直接返回
   - ⚠️ 注意：规则检查根本没有执行（因为 URL 未变化）
   - 最终状态：`origin_url` 仍然保持 `http://othersite.com/link`

4. **阶段4：换一个 URL 重新创建（触发规则）**
   - 创建 Entry: `http://othersite.com/another` → 重定向到 `https://example.com/another`
   - 规则匹配成功 → 不设置 origin_url
   - 状态：`origin_url = null`

> **说明**：规则只在 URL 发生变化时才会被检查。已有 Entry 的 origin_url 不会因为后来添加了规则而被清除。

---

## 八、时序图与数据流

### 8.1 完整抓取流程时序

```
用户提交 URL
    │
    ▼
┌─────────────────────────┐
│ EntryController         │
│  - 创建 Entry 实体      │
│  - 调用 contentProxy    │
└─────────────┬───────────┘
              │
              ▼
┌────────────────────────────────────────┐
│ ContentProxy::updateEntry              │
│  1. 判断是否需要 Graby 抓取             │
│     ├─ 是 → 执行抓取                    │
│     └─ 否 → 跳过抓取                    │
│  2. setGivenUrl($url) ← 始终执行！     │
│  3. 获取最终 URL（来自抓取或 content）  │
└─────────────┬──────────────────────────┘
              │
              ▼
┌─────────────────────────┐
│ ContentProxy::stockEntry │
│  ├─ 【规则生效点】updateOriginUrl()    │
│  ├─ 设置标题、内容、阅读时间等         │
│  └─ 自动标签                           │
└─────────────┬───────────┘
              │
              ▼
┌──────────────────────────────────────────────┐
│ ContentProxy::updateOriginUrl                │
│  1. 检查 URL 是否已变化                       │
│  2. 解析 URL 计算差异                        │
│  3. ┌─────────────────────────────────────┐  │
│     │ RuleBasedIgnoreOriginProcessor      │  │
│     │  - 使用 $entry->getUrl()（原始URL） │  │
│     │  - 合并实例+用户规则                │  │
│     │  - 遍历规则 RulerZ 匹配             │  │
│     │  - 返回 true/false                  │  │
│     └─────────────┬───────────────────────┘  │
│  4. 匹配？├─ 是 → 更新 URL，不设 origin     │
│           └─ 否 → 按差异处理，可能设 origin │
└──────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│ API 后置处理（仅 API 场景）      │
│  - 如果传入 origin_url 参数      │
│  - 覆盖自动设置的 origin_url     │
└─────────────────────────────────┘
```

### 8.2 规则优先级

实例级规则（全局）和用户级规则（个人）合并后顺序检查：

```php
// 实例级规则在前，用户级规则在后
$rules = array_merge(
    $this->ignoreOriginInstanceRuleRepository->findAll(),  // 全局规则
    $userRules                                              // 用户规则
);
```

**匹配策略**：只要有任意一条规则匹配，立即返回 `true`（短路逻辑）。

**优先级说明**：
- 实例级规则先检查，用户级规则后检查
- 但由于是短路逻辑，先匹配的规则生效
- 没有"覆盖"概念，只要任意一条匹配就生效

---

## 九、关键代码位置速查

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 规则接口 | `src/Entity/IgnoreOriginRuleInterface.php` | - |
| 用户规则实体 | `src/Entity/IgnoreOriginUserRule.php` | - |
| 实例规则实体 | `src/Entity/IgnoreOriginInstanceRule.php` | - |
| 规则处理器 | `src/Helper/RuleBasedIgnoreOriginProcessor.php` | 24-45 |
| 规则生效点 | `src/Helper/ContentProxy.php` | 356-360 |
| URL 更新逻辑 | `src/Helper/ContentProxy.php` | 328-393 |
| given_url 设置（无条件覆盖） | `src/Helper/ContentProxy.php` | 76 |
| 抓取跳过判断逻辑 | `src/Helper/ContentProxy.php` | 50-63 |
| 内容验证逻辑 | `src/Helper/ContentProxy.php` | 400-403 |
| setGivenUrl 内部实现 | `src/Entity/Entry.php` | 903-908 |
| API POST 单条创建（无论是否存在都调用） | `src/Controller/Api/EntryRestController.php` | 726-754 |
| API POST 批量创建（仅不存在时调用） | `src/Controller/Api/EntryRestController.php` | 552-564 |
| API POST origin_url 覆盖 | `src/Controller/Api/EntryRestController.php` | 774-776 |
| API PATCH origin_url 覆盖 | `src/Controller/Api/EntryRestController.php` | 1006-1008 |
| API PATCH content 触发 updateEntry | `src/Controller/Api/EntryRestController.php` | 947-956 |
| Web 重抓入口 | `src/Controller/EntryController.php` | 412 |
| API PATCH 重抓入口 | `src/Controller/Api/EntryRestController.php` | 1057 |
| 正则匹配操作符 | `src/Operator/PHP/PatternMatches.php` | 17-22 |
| Web 入口控制器方法 | `src/Controller/EntryController.php` | 703-727 |
| API POST 入口 | `src/Controller/Api/EntryRestController.php` | 741 |
| 默认规则配置 | `app/config/wallabag.yml` | 162-168 |
| 默认规则初始化 | `src/Command/InstallCommand.php` | 340-368 |
| 服务配置 | `app/config/services.yml` | 244-257, 275-278 |

---

## 十、测试验证

`tests/unit/Helper/RuleBasedIgnoreOriginProcessorTest.php` 中的测试用例验证了以下场景：

1. 无规则时返回 `false`
2. 规则不匹配时返回 `false`
3. 规则匹配时返回 `true`
4. 多条规则时只要一条匹配即返回 `true`
5. 实例级规则正常工作
6. 实例级规则与用户级规则混合工作

`tests/unit/Helper/ContentProxyTest.php:1020-1067` 中的数据提供器验证了 URL 变化时的完整处理逻辑。

---

## 十一、生效边界总结

### 11.1 规则匹配的基准

> ✅ **规则匹配的是重定向之前的原始 URL**（`$entry->getUrl()`），不是重定向之后的最终 URL。

这意味着：
- 可以针对 RSS 订阅源域名配置规则（如 `feedproxy.google.com`）
- 可以针对短链接服务配置规则（如 `lemonde.fr/tiny`）
- 不能针对最终目标域名配置规则来忽略来源

### 11.2 三个 URL 的边界对照

| 维度 | `url` | `given_url` | `origin_url` |
|------|-------|-------------|-------------|
| 可被规则影响 | ✅ 是（更新为最终 URL） | ❌ 否 | ✅ 是（可被抑制设置新值） |
| 可被 API 覆盖 | ❌ 否 | ❌ 否（无参数） | ✅ 是（POST/PATCH） |
| 重抓时是否变化 | ✅ 是（可能更新） | ⚠️ 是（被覆盖为当前 url） | ❌ 否（如已设置） |
| 用于去重检查 | ✅ 是 | ✅ 是 | ❌ 否 |
| 设计含义 | 最终访问地址 | 用户输入的 URL | 来源发现地址 |
| 实际行为 | 符合设计 | ⚠️ 重抓时会丢失原始值 | 符合设计 |
| 抓取被跳过时 | 可能变化 | ⚠️ 仍会被覆盖 | 可能变化 |

### 11.3 given_url 的覆盖边界

> ⚠️ **`given_url` 会在每次 `updateEntry` 调用时被无条件覆盖**，无论是否执行了抓取。

**不会覆盖的场景**：
- 首次创建 Entry
- API POST 批量创建已存在的 Entry（不调用 updateEntry）
- API PATCH 不包含 content 参数
- 仅更新归档、星标、标签等元数据

**会覆盖的场景**：
- 重新抓取（Web 或 API）
- API POST 单条创建（无论条目是否已存在）
- API PATCH 包含 content 参数

### 11.4 API POST 单条 vs 批量的差异

| 行为 | API POST 单条 | API POST 批量 |
|------|-------------|-------------|
| 条目已存在时是否调用 updateEntry | ✅ 是 | ❌ 否 |
| 条目已存在时 given_url 是否变化 | ⚠️ 被覆盖为最终 URL | ❌ 保持不变 |
| 条目已存在时是否重新抓取 | ✅ 是（如果需要） | ❌ 否 |

> ⚠️ **重要差异**：API POST 单条创建和批量创建对已存在条目的处理逻辑不同！单条创建会重新执行 updateEntry，批量创建不会。

### 11.5 抓取被跳过的场景边界

> ⚠️ **即使跳过了 Graby 抓取，`setGivenUrl` 和 `stockEntry` 仍然会执行**！

**跳过抓取的条件**：
1. 传入了完整的 content（有 title、html、url）
2. 或 `disableContentUpdate = true`

**跳过抓取但仍执行的操作**：
1. `$entry->setGivenUrl($url)` → given_url 仍会被覆盖
2. `$this->stockEntry($entry, $content)` → 仍会调用 stockEntry
3. `updateOriginUrl($entry, $content['url'])` → 规则检查仍可能执行（如果 URL 变化）

### 11.6 规则命中后 origin_url 的保留边界

> ⚠️ **规则匹配只会阻止设置新的 origin_url，不会清除已有的值**。

| 场景 | 规则匹配前 origin_url | 规则匹配后 origin_url |
|------|---------------------|---------------------|
| 首次创建，规则匹配 | null | null |
| 首次创建，规则不匹配 | null | 原始 URL |
| 已有值，规则匹配 | 已有值 | **保持原值** |
| API 传入新值 | 任意值 | API 传入的值 |

> 规则的作用是"不记录新的来源"，而不是"清除已有的来源记录"。

### 11.7 API 手动写入的边界

> ⚠️ **API 手动传入的 `origin_url` 优先级最高**，会覆盖规则检查结果。

> ⚠️ **API PATCH 可以独立设置 `origin_url`**，不需要触发抓取或规则检查。

这是设计如此，为了支持：
- 导入历史数据时保留原始来源
- 第三方应用自定义来源记录
- 但也意味着 API 调用可以完全绕过规则

### 11.8 默认规则的落地边界

> ⚠️ **默认规则仅在安装时初始化**，重新安装会覆盖手动修改的实例级规则。

正常使用时：
- 实例级规则存储在数据库中
- 管理员可以通过后台管理（`IgnoreOriginInstanceRuleController`）增删规则
- 升级系统不会重新初始化规则

---

## 总结

忽略来源规则的核心作用是**控制 URL 重定向时是否保留原始来源信息**。其生效时机和边界非常精确：

> **在抓取完成后、存储 Entry 时，当且仅当 URL 发生变化时，在 `updateOriginUrl()` 方法中，针对重定向之前的原始 URL 检查规则。如果匹配，则直接更新 URL，不记录新的 origin_url（但保留已有的 origin_url）。**

### 关键修正与澄清

1. **`given_url` 并非永不修改**：它会在每次 `updateEntry` 调用时被**无条件覆盖**，重抓、API POST 单条（已存在）或 PATCH 更新内容时都会丢失原始值。

2. **API POST 单条 vs 批量逻辑不同**：单条创建无论条目是否已存在都会调用 updateEntry，批量创建只有不存在时才调用。

3. **抓取被跳过但 given_url 仍被覆盖**：即使跳过了 Graby 抓取，`setGivenUrl` 和 `stockEntry` 仍然会执行。

4. **规则不会清除已有的 origin_url**：规则只阻止设置新值，已有值保持不变。

5. **API PATCH 可以独立设置 origin_url**：不需要触发抓取，也不经过规则检查，直接覆盖。

6. **三个 URL 的职责分工**：
   - `url`：最终访问地址，多次更新
   - `given_url`：本次调用传入的 URL，每次调用覆盖
   - `origin_url`：来源发现地址，可被规则抑制或 API 覆盖

这个设计的意义在于：对于某些已知的、无害的重定向（如 RSS 订阅跳转、短链接跳转、CDN 域名切换等），用户可以通过配置规则来避免生成不必要的 `origin_url` 记录，保持数据的整洁。
