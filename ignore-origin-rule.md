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
        $parsed_url['_all'] = $url;  // 完整 URL 作为 _all 字段

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

### 2.2 RulerZ 规则引擎

项目使用 [RulerZ](https://github.com/K-Phoen/rulerz) 作为规则引擎，在 `services.yml` 中配置：

```yaml
RulerZ\RulerZ:
    alias: rulerz
```

### 2.3 自定义操作符

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

### 2.4 规则示例

| 规则 | 匹配逻辑 |
|------|----------|
| `host = "example.com"` | 精确匹配主机名为 example.com |
| `host ~ "example\.(com|org)"` | 正则匹配主机名为 example.com 或 example.org |
| `_all ~ "^https://blog\."` | 正则匹配 URL 以 https://blog. 开头 |

---

## 三、抓取入口与规则衔接

### 3.1 抓取入口总览

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

### 3.2 调用链分析

```
抓取入口
    ↓
EntryController::updateEntry()  [src/Controller/EntryController.php:703]
    ↓
ContentProxy::updateEntry()     [src/Helper/ContentProxy.php:43]
    ├─ 步骤1：使用 Graby 抓取网页内容
    │   $this->graby->fetchContent($url)
    │
    └─ 步骤2：存储抓取结果
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

## 四、规则生效时机详解

### 4.1 核心生效点：`updateOriginUrl()`

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

### 4.2 生效条件

规则**仅在以下全部条件满足时**才会被检查和生效：

1. ✅ 抓取后的 URL 与原始 URL **不相同**（发生了重定向）
2. ✅ URL **不为空**

### 4.3 匹配后的行为

当规则匹配成功（`process()` 返回 `true`）时：

| 行为 | 说明 |
|------|------|
| `$entry->setUrl($url)` | 直接更新为最终抓取到的 URL |
| 不设置 `origin_url` | 原始 URL 被丢弃，不记录来源 |
| 跳过 URL 差异分析 | 直接 `return false`，不进入后续 switch 逻辑 |
| 记录日志 | `info` 级别日志记录匹配的规则 |

### 4.4 不匹配时的行为

当规则不匹配时，根据 URL 差异部分决定：

| URL 差异 | 行为 | 是否设置 origin_url |
|----------|------|---------------------|
| 仅 `path` 变化 | 处理特殊情况（尾部斜杠、URL 编码） | 通常不设置 |
| 仅 `scheme` 变化 | 直接更新 URL | 不设置 |
| 仅 `fragment` 变化 | 不做任何处理 | 不设置 |
| 其他情况（如 `host` 变化） | 设置 origin_url 并更新 url | 设置 |

---

## 五、时序图与数据流

### 5.1 完整抓取流程时序

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
┌─────────────────────────┐
│ ContentProxy::updateEntry │
│  - Graby 抓取网页        │
│  - 获取最终 URL          │
└─────────────┬───────────┘
              │
              ▼
┌─────────────────────────┐
│ ContentProxy::stockEntry │
│  - 设置标题、内容等      │
│  - 调用 updateOriginUrl  │
└─────────────┬───────────┘
              │
              ▼
┌──────────────────────────────────────┐
│ ContentProxy::updateOriginUrl        │
│  1. 检查 URL 是否变化                │
│  2. 解析 URL 计算差异                │
│  3. ┌─────────────────────────────┐  │
│     │ RuleBasedIgnoreOriginProcessor │
│     │  - 合并实例+用户规则        │  │
│     │  - 遍历规则 RulerZ 匹配     │  │
│     │  - 返回 true/false          │  │
│     └─────────────┬───────────────┘  │
│  4. 匹配？├─ 是 → 更新 URL，返回    │
│           └─ 否 → 按差异处理        │
└──────────────────────────────────────┘
```

### 5.2 规则优先级

实例级规则（全局）和用户级规则（个人）合并后顺序检查：

```php
// 实例级规则在前，用户级规则在后
$rules = array_merge(
    $this->ignoreOriginInstanceRuleRepository->findAll(),  // 全局规则
    $userRules                                              // 用户规则
);
```

**匹配策略**：只要有任意一条规则匹配，立即返回 `true`（短路逻辑）。

---

## 六、关键代码位置速查

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 规则接口 | `src/Entity/IgnoreOriginRuleInterface.php` | - |
| 用户规则实体 | `src/Entity/IgnoreOriginUserRule.php` | - |
| 实例规则实体 | `src/Entity/IgnoreOriginInstanceRule.php` | - |
| 规则处理器 | `src/Helper/RuleBasedIgnoreOriginProcessor.php` | 24-45 |
| 规则生效点 | `src/Helper/ContentProxy.php` | 356-360 |
| URL 更新逻辑 | `src/Helper/ContentProxy.php` | 328-393 |
| 正则匹配操作符 | `src/Operator/PHP/PatternMatches.php` | 17-22 |
| Web 入口 | `src/Controller/EntryController.php` | 703-727 |
| API 入口 | `src/Controller/Api/EntryRestController.php` | 741 |
| 服务配置 | `app/config/services.yml` | 244-257 |

---

## 七、测试验证

`tests/unit/Helper/RuleBasedIgnoreOriginProcessorTest.php` 中的测试用例验证了以下场景：

1. 无规则时返回 `false`
2. 规则不匹配时返回 `false`
3. 规则匹配时返回 `true`
4. 多条规则时只要一条匹配即返回 `true`
5. 实例级规则正常工作
6. 实例级规则与用户级规则混合工作

---

## 总结

忽略来源规则的核心作用是**控制 URL 重定向时是否保留原始来源信息**。其生效时机非常精确：

> **在抓取完成后、存储 Entry 时，当且仅当 URL 发生变化时，在 `updateOriginUrl()` 方法中检查规则。如果匹配，则直接更新 URL，不记录 `origin_url`。**

这个设计的意义在于：对于某些已知的、无害的重定向（如短链接跳转、CDN 域名切换等），用户可以通过配置规则来避免生成不必要的 `origin_url` 记录，保持数据的整洁。
