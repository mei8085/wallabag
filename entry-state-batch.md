# 条目状态批量切换代码分析

> 文档目的：梳理条目状态机定义、批量操作入口、权限校验与失败处理。所有代码引用均使用仓库相对路径 + 行号，可直接跳转复核。

---

## 一、状态机定义

### 1.1 三大状态属性

条目（Entry）有三个独立的布尔状态，定义在 `src/Entity/Entry.php`：

| 状态 | 字段 | 类型 | 默认值 | 时间戳字段 |
|------|------|------|---------|-----------|
| 归档（已读/未读） | `isArchived` | bool | `false` | `archivedAt` |
| 星标（收藏） | `isStarred` | bool | `false` | `starredAt` |
| 公开 | （虚拟属性） | - | - | 通过 `uid` 是否为 null 判断 |

**代码引用（均可直接跳转复核）**：
- `isArchived` 属性：`src/Entity/Entry.php:108-111`
- `archivedAt` 属性：`src/Entity/Entry.php:116-118`
- `isStarred` 属性：`src/Entity/Entry.php:123-126`
- `starredAt` 属性：`src/Entity/Entry.php:166-168`
- `uid` 属性：`src/Entity/Entry.php:53-55`
- `isPublic()` 虚拟属性方法：`src/Entity/Entry.php:789-792`

### 1.2 归档状态切换方法

归档状态有 3 个操作方法，行为各不相同：

#### 1.2.1 `setArchived($isArchived)`
- **位置**：`src/Entity/Entry.php:321-326`
- **行为**：仅设置 `isArchived` 布尔值，**不更新** `archivedAt` 时间戳
- **返回**：`$this`（链式调用）

#### 1.2.2 `updateArchived($isArchived = false)`
- **位置**：`src/Entity/Entry.php:335-344`
- **行为**：
  1. 调用 `setArchived()` 设置状态
  2. 先将 `archivedAt` 置为 null
  3. 如果设为归档（true），则将 `archivedAt` 设为当前 `\DateTime()`
- **返回**：`$this`

#### 1.2.3 `toggleArchive()`
- **位置**：`src/Entity/Entry.php:384-389`
- **行为**：调用 `updateArchived((bool) ($this->isArchived() ^ 1))` 异或切换状态，**会同步更新** `archivedAt` 时间戳
- **返回**：`$this`

### 1.3 星标状态切换方法

星标状态同样有 3 个操作方法：

#### 1.3.1 `setStarred($isStarred)`
- **位置**：`src/Entity/Entry.php:398-403`
- **行为**：仅设置 `isStarred` 布尔值，**不更新** `starredAt` 时间戳
- **返回**：`$this`

#### 1.3.2 `updateStar($isStarred = false)`
- **位置**：`src/Entity/Entry.php:539-548`
- **行为**：
  1. 调用 `setStarred()` 设置状态
  2. 先将 `starredAt` 置为 null
  3. 如果设为星标（true），则将 `starredAt` 设为当前 `\DateTime()`
- **返回**：`$this`

#### 1.3.3 `toggleStar()`
- **位置**：`src/Entity/Entry.php:423-428`
- **行为**：直接执行 `$this->isStarred = !$this->isStarred()` 取反，**不调用** `updateStar()`，因此**不会更新** `starredAt` 时间戳
- **返回**：`$this`

> ⚠️ **关键不一致点**：`toggleArchive()` 会调用 `updateArchived()` 更新时间戳，但 `toggleStar()` 不会调用 `updateStar()`。两个 toggle 方法在「是否更新时间戳」这一点上行为不对称。
>
> 交叉复核：
> - 归档 toggle：`src/Entity/Entry.php:384-389` → 调用 `updateArchived()` ✅ 更新时间戳
> - 星标 toggle：`src/Entity/Entry.php:423-428` → 直接赋值，不调用 `updateStar()` ❌ 不更新时间戳

### 1.4 公开状态切换方法

#### 1.4.1 `generateUid()`
- **位置**：`src/Entity/Entry.php:767-773`
- **行为**：如果 `uid` 为 null，生成 `uniqid('', true)`，将条目设为公开
- **返回**：void

#### 1.4.2 `cleanUid()`
- **位置**：`src/Entity/Entry.php:775-778`
- **行为**：将 `uid` 设为 null，将条目设为私有
- **返回**：void

#### 1.4.3 `isPublic()`
- **位置**：`src/Entity/Entry.php:789-792`
- **行为**：返回 `null !== $this->uid`
- **返回**：bool

---

## 二、批量操作入口

### 2.1 Web 端批量操作：`massAction()`

**入口方法**：`src/Controller/EntryController.php:53-135`
**路由**：`POST /mass`（路由名：`mass_action`）

#### 2.1.1 全局权限
- 注解：`#[IsGranted('EDIT_ENTRIES')]`
- 对应 Voter 常量：`MainVoter::EDIT_ENTRIES`
- 实际校验：`ROLE_USER` 角色

**代码引用**：`src/Controller/EntryController.php:53-55`

#### 2.1.2 CSRF 校验
- Token ID：`mass-action`
- 失败行为：抛出 `BadRequestHttpException('Bad CSRF token.')`

**代码引用**：`src/Controller/EntryController.php:57-59`

#### 2.1.3 支持的操作类型

通过请求参数判断操作类型，判断逻辑在 `src/Controller/EntryController.php:66-101`：

| 操作类型 | 触发条件 | 对应实体方法 | 单条校验权限 |
|---------|---------|------------|------------|
| `toggle-read`（默认） | 无特殊参数 | `$entry->toggleArchive()` | `EDIT` |
| `toggle-star` | 请求中有 `toggle-star` 键 | `$entry->toggleStar()` | `EDIT` |
| `delete` | 请求中有 `delete` 键 | `$em->remove($entry)` | `EDIT` |
| `tag` | 请求中有 `tag` 键 | `addTag()` / `removeTag()` | `EDIT` |

操作判断代码：`src/Controller/EntryController.php:66-72`
标签解析代码：`src/Controller/EntryController.php:74-100`

#### 2.1.4 批量循环与权限校验

**位置**：`src/Controller/EntryController.php:103-130`

**循环流程**：
1. 遍历 `entry-checkbox` 数组中的条目 ID（`src/Controller/EntryController.php:103-104`）
2. 通过 `findById([$id])[0]` 查询条目（`src/Controller/EntryController.php:106`）
3. 每条执行 `$this->security->isGranted('EDIT', $entry)` 权限检查（`src/Controller/EntryController.php:108`）
4. 权限不足则 `throw $this->createAccessDeniedException(...)` —— **立即终止，不继续处理后续条目**（`src/Controller/EntryController.php:109`）
5. 执行对应操作（`src/Controller/EntryController.php:112-126`）
6. 全部循环结束后一次性 `flush()`（`src/Controller/EntryController.php:129`）

#### 2.1.5 批量星标不更新时间戳（关键 Bug）

**批量星标代码**：`src/Controller/EntryController.php:114-115`

```php
} elseif ('toggle-star' === $action) {
    $entry->toggleStar();
}
```

由于 `toggleStar()` 不会更新 `starredAt`（见 1.3.3 与 `src/Entity/Entry.php:423-428`），因此**批量切换星标时，`starredAt` 时间戳不会更新**。

**对比：单条星标会更新时间戳**

单条星标代码：`src/Controller/EntryController.php:474-475`

```php
$entry->toggleStar();       // 切换布尔值
$entry->updateStar($entry->isStarred());  // 更新时间戳
```

单条星标显式调用了 `updateStar()`，因此会更新 `starredAt`。

交叉复核：
- 单条（正确）：`src/Controller/EntryController.php:466-491` → 调用 `toggleStar()` + `updateStar()`
- 批量（问题）：`src/Controller/EntryController.php:112-115` → 只调用 `toggleStar()`

---

### 2.2 API 端批量操作

API 端没有专门的「批量状态切换（归档/星标）」接口，但有 4 个批量操作接口。

#### 2.2.1 批量创建条目

**方法**：`src/Controller/Api/EntryRestController.php:536-576`
**路由**：`POST /api/entries/lists.{_format}`（路由名：`api_post_entries_list`）
**全局权限**：`#[IsGranted('CREATE_ENTRIES')]`（`src/Controller/Api/EntryRestController.php:537`）

**特点**：
- 通过 URL 数组（JSON）批量创建条目
- 有数量限制：`apiLimitMassActions`，超限抛 `BadRequestHttpException('API limit reached')`（`src/Controller/Api/EntryRestController.php:542-544`）
- 逐条创建，逐条 `flush()`（`src/Controller/Api/EntryRestController.php:566-567`）
- 不做单条权限检查（因为是创建新条目，属于当前用户）

`apiLimitMassActions` 定义在基类：`src/Controller/Api/WallabagRestController.php:32`

#### 2.2.2 批量删除条目

**方法**：`src/Controller/Api/EntryRestController.php:478-511`
**路由**：`DELETE /api/entries/list.{_format}`（路由名：`api_delete_entries_list`）
**全局权限**：`#[IsGranted('DELETE_ENTRIES')]`（`src/Controller/Api/EntryRestController.php:479`）

**批量循环**：`src/Controller/Api/EntryRestController.php:491-508`

**单条权限检查**：`src/Controller/Api/EntryRestController.php:499`

```php
if (false !== $entry && $this->authorizationChecker->isGranted('DELETE', $entry)) {
    // 执行删除...
}
```

**权限失败处理**：
- 无权限的条目**静默跳过**，不抛异常
- 继续处理下一个条目
- 返回结果中 `entry` 字段只标记条目是否存在，不标记是否有权限

**flush 时机**：逐条 flush（`src/Controller/Api/EntryRestController.php:504`）

#### 2.2.3 批量添加标签

**方法**：`src/Controller/Api/EntryRestController.php:1345-1378`
**路由**：`POST /api/entries/tags/lists.{_format}`（路由名：`api_post_entries_tags_list`）
**全局权限**：`#[IsGranted('CREATE_TAGS')]`（`src/Controller/Api/EntryRestController.php:1346`）

**单条权限**：`$this->authorizationChecker->isGranted('TAG', $entry)`（`src/Controller/Api/EntryRestController.php:1369`）
**权限失败**：静默跳过，继续处理
**flush 时机**：逐条 flush（`src/Controller/Api/EntryRestController.php:1373`）

#### 2.2.4 批量删除标签

**方法**：`src/Controller/Api/EntryRestController.php:1280-1322`
**路由**：`DELETE /api/entries/tags/list.{_format}`（路由名：`api_delete_entries_tags_list`）
**全局权限**：`#[IsGranted('DELETE_TAGS')]`（`src/Controller/Api/EntryRestController.php:1281`）

**单条权限**：`$this->authorizationChecker->isGranted('UNTAG', $entry)`（`src/Controller/Api/EntryRestController.php:1304`）
**权限失败**：静默跳过，继续处理
**flush 时机**：逐条 flush（`src/Controller/Api/EntryRestController.php:1317`）

---

## 三、权限校验体系

### 3.1 两层 Voter 架构

项目有两个独立的 Voter，分别处理不同级别的权限。

#### 3.1.1 MainVoter（全局操作权限）

**文件**：`src/Security/Voter/MainVoter.php`
**适用场景**：不需要具体主体（subject）的全局/批量操作

**支持的属性常量**：`src/Security/Voter/MainVoter.php:11-22`

| 属性常量 | 用途 |
|---------|------|
| `LIST_ENTRIES` | 列出条目 |
| `CREATE_ENTRIES` | 创建条目 |
| `EDIT_ENTRIES` | 编辑条目（Web 批量操作用） |
| `EXPORT_ENTRIES` | 导出条目 |
| `IMPORT_ENTRIES` | 导入条目 |
| `DELETE_ENTRIES` | 删除条目（API 批量删除用） |
| `LIST_TAGS` | 列出标签 |
| `CREATE_TAGS` | 创建标签（API 批量加标签用） |
| `DELETE_TAGS` | 删除标签（API 批量删标签用） |
| `LIST_SITE_CREDENTIALS` | 列出站点凭证 |
| `CREATE_SITE_CREDENTIALS` | 创建站点凭证 |
| `EDIT_CONFIG` | 编辑配置 |

**supports 条件**：`src/Security/Voter/MainVoter.php:29-40`
- `$subject` 必须为 null（即不针对具体实体）
- `$attribute` 必须在上述常量列表中

**校验逻辑**：`src/Security/Voter/MainVoter.php:42-48`

所有这些权限最终都校验同一个条件：`$this->security->isGranted('ROLE_USER')`。

#### 3.1.2 EntryVoter（单条目权限）

**文件**：`src/Security/Voter/EntryVoter.php`
**适用场景**：针对具体 Entry 实体的单条操作

**支持的属性常量**：`src/Security/Voter/EntryVoter.php:12-25`

| 属性常量 | 用途 |
|---------|------|
| `VIEW` | 查看条目 |
| `EDIT` | 编辑条目 |
| `RELOAD` | 重新抓取 |
| `STAR` | 星标操作 |
| `ARCHIVE` | 归档操作 |
| `SHARE` | 分享 |
| `UNSHARE` | 取消分享 |
| `EXPORT` | 导出 |
| `DELETE` | 删除 |
| `LIST_ANNOTATIONS` | 列出注释 |
| `CREATE_ANNOTATIONS` | 创建注释 |
| `LIST_TAGS` | 列出标签 |
| `TAG` | 添加标签 |
| `UNTAG` | 移除标签 |

**supports 条件**：`src/Security/Voter/EntryVoter.php:27-38`
- `$subject` 必须是 `Entry` 实例
- `$attribute` 必须在上述常量列表中

**校验逻辑**：`src/Security/Voter/EntryVoter.php:40-54`

所有 14 个单条目权限的校验条件完全相同：`$user === $subject->getUser()` —— 当前用户必须是条目的所有者。

### 3.2 批量操作中的权限校验模式

#### 模式 A：Web 端 massAction

**入口全局权限**：`EDIT_ENTRIES`（MainVoter）→ `src/Controller/EntryController.php:54`
**单条循环权限**：`EDIT`（EntryVoter）→ `src/Controller/EntryController.php:108`
**失败处理**：遇到无权限条目立即 `throw createAccessDeniedException()`，整体终止 → `src/Controller/EntryController.php:109`

流程图：
```
入口 @IsGranted('EDIT_ENTRIES')         ← MainVoter 全局检查
        ↓
循环每条 entry（entry-checkbox）：
  isGranted('EDIT', $entry)              ← EntryVoter 单条检查
        ↓
  无权限 → throw AccessDeniedException   ← 立即终止，全部回滚
        ↓
  有权限 → 执行操作
        ↓
最后统一 flush()
```

**特点**：
- 全局权限用 `EDIT_ENTRIES`
- 单条权限**统一用 `EDIT`**（而非 `STAR`/`ARCHIVE`/`DELETE` 等细粒度权限）
- 权限失败立即终止，原子性强但容错性差
- 一次 flush，性能好

#### 模式 B：API 端批量操作（以删除为例）

**入口全局权限**：`DELETE_ENTRIES`（MainVoter）→ `src/Controller/Api/EntryRestController.php:479`
**单条循环权限**：`DELETE`（EntryVoter）→ `src/Controller/Api/EntryRestController.php:499`
**失败处理**：无权限条目静默跳过，继续处理下一条 → `src/Controller/Api/EntryRestController.php:499`

流程图：
```
入口 @IsGranted('DELETE_ENTRIES')       ← MainVoter 全局检查
        ↓
循环每条 entry（by URL）：
  isGranted('DELETE', $entry)           ← EntryVoter 单条检查
        ↓
  无权限 → 跳过，继续下一条               ← 静默忽略，部分成功
        ↓
  有权限 → 执行操作 + flush()
        ↓
返回结果数组
```

**特点**：
- 全局权限用对应的操作权限（`DELETE_ENTRIES`/`CREATE_TAGS`/`DELETE_TAGS`）
- 单条权限用对应的操作权限（`DELETE`/`TAG`/`UNTAG`）
- 权限失败静默跳过，容错性强但原子性差
- 逐条 flush，性能较低

### 3.3 权限校验粒度对比

| 操作类型 | Web 单条 | Web 批量 | API 单条 | API 批量 |
|---------|---------|---------|---------|---------|
| 归档 | `ARCHIVE` | `EDIT` | `EDIT` | —（无接口） |
| 星标 | `STAR` | `EDIT` | `EDIT` | —（无接口） |
| 删除 | `DELETE` | `EDIT` | `DELETE` | `DELETE` |
| 加标签 | `TAG` | `EDIT` | `TAG` | `TAG` |
| 删标签 | `UNTAG` | `EDIT` | `UNTAG` | `UNTAG` |

**代码交叉复核**：
- Web 单条归档权限：`src/Controller/EntryController.php:436` → `ARCHIVE`
- Web 单条星标权限：`src/Controller/EntryController.php:467` → `STAR`
- Web 单条删除权限：`src/Controller/EntryController.php:499` → `DELETE`
- Web 批量权限：`src/Controller/EntryController.php:54 + :108` → `EDIT_ENTRIES` + 单条 `EDIT`
- API PATCH 单条权限：`src/Controller/Api/EntryRestController.php:939` → `EDIT`
- API 批量删除权限：`src/Controller/Api/EntryRestController.php:479 + :499` → `DELETE_ENTRIES` + 单条 `DELETE`
- API 批量加标签权限：`src/Controller/Api/EntryRestController.php:1346 + :1369` → `CREATE_TAGS` + 单条 `TAG`
- API 批量删标签权限：`src/Controller/Api/EntryRestController.php:1281 + :1304` → `DELETE_TAGS` + 单条 `UNTAG`

> **说明**：虽然目前 `EntryVoter` 中所有权限的校验条件相同（都是所有者判断），但权限常量的粒度设计不一致，未来如果扩展更细的权限（如「只能归档不能删除」），这些不一致可能导致问题。

---

## 四、不统一点汇总（快速索引）

### 4.1 状态切换方法不一致

| 场景 | 归档方法 | 星标方法 | 星标时间戳 |
|------|---------|---------|-----------|
| Web 单条归档 | `toggleArchive()` | - | - |
| Web 单条星标 | - | `toggleStar()` + `updateStar()` | ✅ 更新 |
| Web 批量 toggle-read | `toggleArchive()` | - | - |
| Web 批量 toggle-star | - | `toggleStar()` | ❌ 不更新 |
| API PATCH 单条 | `updateArchived()` | `updateStar()` | ✅ 更新 |
| API POST 创建 | `updateArchived()` | `updateStar()` | ✅ 更新 |

**核心问题**：Web 批量切换星标时 `starredAt` 不更新。
**Bug 位置**：`src/Controller/EntryController.php:115`

### 4.2 权限失败处理不一致

| 接口 | 权限失败行为 | 代码位置 |
|------|-------------|---------|
| Web 批量 massAction | 抛异常，整体终止 | `src/Controller/EntryController.php:109` |
| API 批量删除 | 静默跳过，继续处理 | `src/Controller/Api/EntryRestController.php:499` |
| API 批量加标签 | 静默跳过，继续处理 | `src/Controller/Api/EntryRestController.php:1369` |
| API 批量删标签 | 静默跳过，继续处理 | `src/Controller/Api/EntryRestController.php:1304` |

### 4.3 权限粒度不一致

- Web 批量统一用 `EDIT` 权限（粒度过粗）
- API 批量用对应操作权限（粒度较细）

### 4.4 flush 时机不一致

- Web 批量：循环结束后一次 flush（`src/Controller/EntryController.php:129`）
- API 批量：逐条 flush（性能较低）

### 4.5 标识方式不一致

- Web 批量：条目 ID 数组（`entry-checkbox`）
- API 批量：URL 数组（JSON）

### 4.6 数量限制不一致

- API 批量创建：受 `apiLimitMassActions` 限制（`src/Controller/Api/EntryRestController.php:542`）
- Web 批量及其他 API 批量：无显式数量限制

---

## 五、代码文件索引

| 文件 | 仓库相对路径 | 说明 |
|------|-------------|------|
| Entry 实体 | `src/Entity/Entry.php` | 状态属性及切换方法 |
| Web 入口控制器 | `src/Controller/EntryController.php` | 单条 + 批量 Web 操作 |
| API 入口控制器 | `src/Controller/Api/EntryRestController.php` | API 单条 + 批量操作 |
| API 基类 | `src/Controller/Api/WallabagRestController.php` | `apiLimitMassActions` 等公共属性 |
| 条目权限 Voter | `src/Security/Voter/EntryVoter.php` | 单条目权限校验 |
| 全局权限 Voter | `src/Security/Voter/MainVoter.php` | 全局/批量操作权限校验 |
| 前端批量控制器 | `assets/controllers/batch_edit_controller.js` | 前端批量选择交互 |
