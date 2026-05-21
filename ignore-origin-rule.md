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
| `given_url` | `given_url` | `ContentProxy::updateEntry()`:76 | **用户提交的 URL** - 用户最初输入的原始地址，永不修改 |
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

### 3.2 given_url 的设置时机

`given_url` 在 `ContentProxy::updateEntry()` 开始时就被设置，**早于**抓取和规则检查：

```php
// src/Helper/ContentProxy.php:43-78
public function updateEntry(Entry $entry, $url, array $content = [], $disableContentUpdate = false): void
{
    // ... Graby 抓取网页内容 ...

    // 无论后续如何处理，先设置 given_url
    $entry->setGivenUrl($url);  // 第76行

    $this->stockEntry($entry, $content);
}
```

**特性**：
- ✅ 始终设置，不受规则影响
- ✅ 永不修改，保留用户最初输入的 URL
- ✅ 可用于重复检查（`findByUrlAndUserId` 会同时检查 url 和 given_url）

### 3.3 origin_url 的设置条件

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

### 3.4 三个 URL 的完整生命周期

```
用户提交 URL: http://feedproxy.google.com/example
    │
    ├─ 1. Entry 创建 → $entry->setUrl('http://feedproxy.google.com/example')
    │
    ├─ 2. ContentProxy::updateEntry() 开始
    │   └─ $entry->setGivenUrl('http://feedproxy.google.com/example')  ← given_url 固定
    │
    ├─ 3. Graby 抓取 → 跟随重定向到 https://example.com/article
    │
    └─ 4. updateOriginUrl() 检查
        ├─ 比较：原始 URL ≠ 最终 URL ✓
        ├─ 规则匹配？（检查原始 URL host = feedproxy.google.com）
        │   ├─ 匹配 → $entry->setUrl('https://example.com/article')
        │   │          origin_url = null（不设置）
        │   └─ 不匹配 → $entry->setOriginUrl('http://feedproxy.google.com/example')
        │                $entry->setUrl('https://example.com/article')
        └─ 最终状态
            ├─ url = https://example.com/article
            ├─ given_url = http://feedproxy.google.com/example
            └─ origin_url = 匹配规则时为 null，否则为 http://feedproxy.google.com/example
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
| 异步消费（RabbitMQ） | `AbstractConsumer::handleMessage()`:56 → `Import::parseEntry()` |
| 异步消费（Redis） | `AbstractConsumer::handleMessage()`:56 → `Import::parseEntry()` |

### 4.2 API 手动写入 origin_url 的影响

在 API 调用中，`origin_url` 可以通过请求参数手动设置，但需要注意**执行顺序**：

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

**关键结论**：API 手动写入的 `origin_url` **会覆盖**规则检查的结果。

| 场景 | 规则匹配结果 | API origin_url 参数 | 最终 origin_url |
|------|-------------|-------------------|-----------------|
| 1 | true（忽略） | 未提供 | null |
| 2 | true（忽略） | 提供 = "http://source.com" | "http://source.com" |
| 3 | false（保留） | 未提供 | 自动设置的原始 URL |
| 4 | false（保留） | 提供 = "http://custom.com" | "http://custom.com" |

**生效边界**：
- API 的 `origin_url` 设置在 `ContentProxy::updateEntry()` **之后**执行
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
    ├─ 步骤1：设置 given_url（第76行，固定不变）
    ├─ 步骤2：使用 Graby 抓取网页内容
    │   $this->graby->fetchContent($url)
    │
    └─ 步骤3：存储抓取结果
        ↓
        ContentProxy::stockEntry()  [src/Helper/ContentProxy.php:243]
            ├─ 设置标题、内容、阅读时间等
            ├─ 自动标签
            └─ 更新 URL
                ↓
                ContentProxy::updateOriginUrl()  [src/Helper/ContentProxy.php:328]
                    ├─ 【规则生效点】检查 URL 是否变化
                    ├─ 【规则生效点】调用 ignoreOriginProcessor->process()
                    └─ 根据规则匹配结果决定是否设置 origin_url
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
| 3. 非 API 强制覆盖 | API 没有手动传入 origin_url（仅 API 场景） | `EntryRestController.php:774-776` |

### 5.3 匹配后的行为

当规则匹配成功（`process()` 返回 `true`）时：

| 行为 | 说明 |
|------|------|
| `$entry->setUrl($url)` | 直接更新为最终抓取到的 URL |
| 不设置 `origin_url` | 原始 URL 被丢弃，不记录来源 |
| 跳过 URL 差异分析 | 直接 `return false`，不进入后续 switch 逻辑 |
| 记录日志 | `info` 级别日志记录匹配的规则 |

### 5.4 不匹配时的行为

当规则不匹配时，根据 URL 差异部分决定：

| URL 差异 | 行为 | 是否设置 origin_url |
|----------|------|---------------------|
| 仅 `path` 变化 | 处理特殊情况（尾部斜杠、URL 编码） | 通常不设置 |
| 仅 `scheme` 变化 | 直接更新 URL | 不设置 |
| 仅 `fragment` 变化 | 不做任何处理 | 不设置 |
| 其他情况（如 `host` 变化） | 设置 origin_url 并更新 url | 设置 |

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

## 七、时序图与数据流

### 7.1 完整抓取流程时序

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
┌─────────────────────────────────┐
│ ContentProxy::updateEntry       │
│  1. setGivenUrl($url) ← 固定   │
│  2. Graby 抓取网页              │
│  3. 获取最终 URL                │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────┐
│ ContentProxy::stockEntry │
│  - 设置标题、内容等      │
│  - 调用 updateOriginUrl  │
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

### 7.2 规则优先级

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

## 八、关键代码位置速查

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 规则接口 | `src/Entity/IgnoreOriginRuleInterface.php` | - |
| 用户规则实体 | `src/Entity/IgnoreOriginUserRule.php` | - |
| 实例规则实体 | `src/Entity/IgnoreOriginInstanceRule.php` | - |
| 规则处理器 | `src/Helper/RuleBasedIgnoreOriginProcessor.php` | 24-45 |
| 规则生效点 | `src/Helper/ContentProxy.php` | 356-360 |
| URL 更新逻辑 | `src/Helper/ContentProxy.php` | 328-393 |
| given_url 设置 | `src/Helper/ContentProxy.php` | 76 |
| API origin_url 覆盖 | `src/Controller/Api/EntryRestController.php` | 774-776 |
| 正则匹配操作符 | `src/Operator/PHP/PatternMatches.php` | 17-22 |
| Web 入口 | `src/Controller/EntryController.php` | 703-727 |
| API 入口 | `src/Controller/Api/EntryRestController.php` | 741 |
| 默认规则配置 | `app/config/wallabag.yml` | 162-168 |
| 默认规则初始化 | `src/Command/InstallCommand.php` | 340-368 |
| 服务配置 | `app/config/services.yml` | 244-257, 275-278 |

---

## 九、测试验证

`tests/unit/Helper/RuleBasedIgnoreOriginProcessorTest.php` 中的测试用例验证了以下场景：

1. 无规则时返回 `false`
2. 规则不匹配时返回 `false`
3. 规则匹配时返回 `true`
4. 多条规则时只要一条匹配即返回 `true`
5. 实例级规则正常工作
6. 实例级规则与用户级规则混合工作

`tests/unit/Helper/ContentProxyTest.php:1020-1067` 中的数据提供器验证了 URL 变化时的完整处理逻辑。

---

## 十、生效边界总结

### 10.1 规则匹配的基准

> ✅ **规则匹配的是重定向之前的原始 URL**（`$entry->getUrl()`），不是重定向之后的最终 URL。

这意味着：
- 可以针对 RSS 订阅源域名配置规则（如 `feedproxy.google.com`）
- 可以针对短链接服务配置规则（如 `lemonde.fr/tiny`）
- 不能针对最终目标域名配置规则来忽略来源

### 10.2 三个 URL 的边界

| 字段 | 可被规则影响 | 可被 API 覆盖 | 始终保留 |
|------|-------------|--------------|----------|
| `url` | ✅ 是（更新为最终 URL） | ❌ 否 | ❌ 否 |
| `given_url` | ❌ 否 | ❌ 否 | ✅ 是 |
| `origin_url` | ✅ 是（可被抑制） | ✅ 是（可被覆盖） | ❌ 否 |

### 10.3 API 手动写入的边界

> ⚠️ **API 手动传入的 `origin_url` 优先级最高**，会覆盖规则检查结果。

这是设计如此，为了支持：
- 导入历史数据时保留原始来源
- 第三方应用自定义来源记录
- 但也意味着 API 调用可以绕过规则

### 10.4 默认规则的落地边界

> ⚠️ **默认规则仅在安装时初始化**，重新安装会覆盖手动修改的实例级规则。

正常使用时：
- 实例级规则存储在数据库中
- 管理员可以通过后台管理（`IgnoreOriginInstanceRuleController`）增删规则
- 升级系统不会重新初始化规则

---

## 总结

忽略来源规则的核心作用是**控制 URL 重定向时是否保留原始来源信息**。其生效时机和边界非常精确：

> **在抓取完成后、存储 Entry 时，当且仅当 URL 发生变化时，在 `updateOriginUrl()` 方法中，针对重定向之前的原始 URL 检查规则。如果匹配，则直接更新 URL，不记录 `origin_url`。**

这个设计的意义在于：对于某些已知的、无害的重定向（如 RSS 订阅跳转、短链接跳转、CDN 域名切换等），用户可以通过配置规则来避免生成不必要的 `origin_url` 记录，保持数据的整洁。同时，`given_url` 始终保留用户最初输入的地址，确保可追溯性。
