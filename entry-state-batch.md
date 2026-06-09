# 条目状态批量切换代码分析

> 文档目的：梳理条目状态机定义、批量操作入口、权限校验与失败处理。所有代码引用均使用仓库相对路径 + 行号，可直接跳转复核。

---

## 一、状态机定义

### 1.1 三大状态属性

条目（Entry）有三个独立的布尔状态，定义在 `src/Entity/Entry.php:1-958`：

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

公开状态是由 `uid` 字段派生的虚拟属性，没有单独的布尔字段。

#### 1.4.1 `isPublic()` —— 状态读取
- **位置**：`src/Entity/Entry.php:789-792`
- **行为**：返回 `null !== $this->uid`，即只要 `uid` 不为 null 就算公开
- **返回**：bool
- **序列化**：虚拟属性，序列化为 `is_public`（`src/Entity/Entry.php:786-788`）

#### 1.4.2 `generateUid()` —— 设为公开
- **位置**：`src/Entity/Entry.php:767-773`
- **行为**：如果 `uid` 为 null，则用 `uniqid('', true)` 生成一个 23 位的唯一 ID，将条目设为公开
- **幂等性**：如果已经有 `uid`，不重复生成
- **返回**：void

#### 1.4.3 `cleanUid()` —— 设为私有
- **位置**：`src/Entity/Entry.php:775-778`
- **行为**：将 `uid` 设为 null，将条目设为私有（不公开）
- **返回**：void

#### 1.4.4 `uid` 字段定义
- **位置**：`src/Entity/Entry.php:53-55`
- **类型**：string，长度 23，可为 null
- **序列化**：`entries_for_user` + `export_all` 分组均包含

> ⚠️ **注意**：与归档、星标不同，公开状态没有「toggle」方法。设为公开用 `generateUid()`，设为私有用 `cleanUid()`，需要调用方显式判断方向。

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

### 2.3 关于「批量切换公开状态」

**结论：当前代码中没有批量切换公开状态的接口。**

- Web 端 `massAction` 支持 4 种操作：`toggle-read`、`toggle-star`、`delete`、`tag`，不含公开/私有切换（`src/Controller/EntryController.php:66-72`）
- API 端批量接口有 4 个：批量创建、批量删除、批量加标签、批量删标签，不含批量切换公开状态
- 公开状态的切换目前仅支持**单条**操作（见第三章）

---

## 三、公开状态切换入口与权限

公开状态（isPublic）由 `uid` 派生，切换操作均为**单条**操作，目前没有批量切换接口。

### 3.1 Web 端：分享（设为公开）

**方法**：`shareAction()`，位置：`src/Controller/EntryController.php:538-556`
**路由**：`POST /share/{id}`（路由名：`share`）
**权限**：`#[IsGranted('SHARE', subject: 'entry')]` → `src/Controller/EntryController.php:539`

**逻辑**：
1. CSRF token 校验（`share-entry`）→ `src/Controller/EntryController.php:542`
2. 如果 `uid` 为 null，则调用 `$entry->generateUid()` 生成公开链接 → `src/Controller/EntryController.php:546-547`
3. 已经是公开状态则不重复生成（幂等）
4. `persist + flush` 保存
5. 跳转到公开分享页 `share_entry`

### 3.2 Web 端：取消分享（设为私有）

**方法**：`deleteShareAction()`，位置：`src/Controller/EntryController.php:564-579`
**路由**：`POST /share/delete/{id}`（路由名：`delete_share`）
**权限**：`#[IsGranted('UNSHARE', subject: 'entry')]` → `src/Controller/EntryController.php:564`

**逻辑**：
1. CSRF token 校验（`delete-share`）→ `src/Controller/EntryController.php:567`
2. 调用 `$entry->cleanUid()` 清除 uid → `src/Controller/EntryController.php:571`
3. `persist + flush` 保存
4. 跳转到条目详情页 `view`

### 3.3 Web 端：公开访问（只读）

**方法**：`shareEntryAction()`，位置：`src/Controller/EntryController.php:588-608`
**路由**：`GET /share/{uid}`（路由名：`share_entry`）
**权限**：`#[IsGranted('PUBLIC_ACCESS')]` → `src/Controller/EntryController.php:588`（公开访问，无需登录）

**注意**：这是只读的公开页面，不是状态切换入口，但它是公开状态的消费端。

### 3.4 API 端：创建时设置 public

**方法**：`postEntriesAction()`，位置：`src/Controller/Api/EntryRestController.php:717-811`
**路由**：`POST /api/entries.{_format}`（路由名：`api_post_entries`）
**全局权限**：`#[IsGranted('CREATE_ENTRIES')]`
**参数**：`public`（query 参数，值为 `"1"` 或 `"0"`）

**public 处理逻辑**：`src/Controller/Api/EntryRestController.php:778-784`

```php
if (null !== $data['isPublic']) {
    if (true === (bool) $data['isPublic'] && null === $entry->getUid()) {
        $entry->generateUid();
    } elseif (false === (bool) $data['isPublic']) {
        $entry->cleanUid();
    }
}
```

**行为**：
- `isPublic = true` 且 uid 为空 → `generateUid()`（设为公开）
- `isPublic = true` 且 uid 已存在 → 不操作（幂等）
- `isPublic = false` → `cleanUid()`（设为私有）
- `isPublic = null` → 不操作

### 3.5 API 端：修改时设置 public

**方法**：`patchEntriesAction()`，位置：`src/Controller/Api/EntryRestController.php:940-1040`
**路由**：`PATCH /api/entries/{entry}.{_format}`（路由名：`api_patch_entries`）
**单条权限**：`#[IsGranted('EDIT', subject: 'entry')]` → `src/Controller/Api/EntryRestController.php:939`

**public 处理逻辑**：`src/Controller/Api/EntryRestController.php:998-1004`

与创建时完全相同的三分支逻辑：
- `isPublic = true` 且 uid 为空 → `generateUid()`
- `isPublic = false` → `cleanUid()`
- `isPublic = null` → 不操作

### 3.6 API 端：按 public 过滤列表

**方法**：`getEntriesAction()`，位置：`src/Controller/Api/EntryRestController.php:314-406`
**权限**：`#[IsGranted('LIST_ENTRIES')]`
**参数**：`public`（query 参数）

**解析**：`src/Controller/Api/EntryRestController.php:318` —— `$isPublic = ... (bool) $request->query->get('public')`
**传入查询**：`src/Controller/Api/EntryRestController.php:337` —— 作为参数传给 `findEntries()`

注意：这是查询过滤，不是状态切换。

### 3.7 公开状态权限校验汇总

| 入口 | 权限常量 | 位置 | 类型 |
|------|---------|------|------|
| Web 分享 | `SHARE` | `src/Controller/EntryController.php:539` | 单条权限 |
| Web 取消分享 | `UNSHARE` | `src/Controller/EntryController.php:564` | 单条权限 |
| Web 公开访问 | `PUBLIC_ACCESS` | `src/Controller/EntryController.php:588` | 公开访问 |
| API 创建设置 public | `CREATE_ENTRIES` | （全局权限） | 全局权限 |
| API 修改设置 public | `EDIT` | `src/Controller/Api/EntryRestController.php:939` | 单条权限 |
| API 按 public 过滤 | `LIST_ENTRIES` | （全局权限） | 全局权限 |

**Voter 中的常量定义**：
- `SHARE`：`src/Security/Voter/EntryVoter.php:17`
- `UNSHARE`：`src/Security/Voter/EntryVoter.php:18`
- `EDIT`：`src/Security/Voter/EntryVoter.php:13`
- `CREATE_ENTRIES`：`src/Security/Voter/MainVoter.php:12`
- `LIST_ENTRIES`：`src/Security/Voter/MainVoter.php:11`

---

## 四、权限校验体系

### 4.1 两层 Voter 架构

项目有两个独立的 Voter，分别处理不同级别的权限。

#### 4.1.1 MainVoter（全局操作权限）

**文件**：`src/Security/Voter/MainVoter.php:1-49`
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

#### 4.1.2 EntryVoter（单条目权限）

**文件**：`src/Security/Voter/EntryVoter.php:1-55`
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

### 4.2 批量操作中的权限校验模式

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

### 4.3 权限校验粒度对比

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
- Web 批量入口权限：`src/Controller/EntryController.php:54` → `EDIT_ENTRIES`（全局）
- Web 批量单条权限：`src/Controller/EntryController.php:108` → `EDIT`（单条）
- API PATCH 单条权限：`src/Controller/Api/EntryRestController.php:939` → `EDIT`
- API 批量删除入口权限：`src/Controller/Api/EntryRestController.php:479` → `DELETE_ENTRIES`（全局）
- API 批量删除单条权限：`src/Controller/Api/EntryRestController.php:499` → `DELETE`（单条）
- API 批量加标签入口权限：`src/Controller/Api/EntryRestController.php:1346` → `CREATE_TAGS`（全局）
- API 批量加标签单条权限：`src/Controller/Api/EntryRestController.php:1369` → `TAG`（单条）
- API 批量删标签入口权限：`src/Controller/Api/EntryRestController.php:1281` → `DELETE_TAGS`（全局）
- API 批量删标签单条权限：`src/Controller/Api/EntryRestController.php:1304` → `UNTAG`（单条）

> **说明**：虽然目前 `EntryVoter` 中所有权限的校验条件相同（都是所有者判断），但权限常量的粒度设计不一致，未来如果扩展更细的权限（如「只能归档不能删除」），这些不一致可能导致问题。

---

## 五、三部分交叉引用对照表

以下表格将状态机、批量入口、权限失败处理三部分的对应点一一列出，每个引用均为独立的仓库相对路径，可直接跳转复核。

### 5.1 Web 批量 toggle-read（归档切换）

| 维度 | 代码位置 | 说明 |
|------|---------|------|
| 状态机方法 | `src/Entity/Entry.php:384-389` | `toggleArchive()` 方法，调用 `updateArchived()` 更新时间戳 |
| 状态机时间戳更新 | `src/Entity/Entry.php:335-344` | `updateArchived()` 方法，同步更新 `archivedAt` |
| 批量入口调用 | `src/Controller/EntryController.php:112-113` | massAction 中调用 `$entry->toggleArchive()` |
| 全局权限入口 | `src/Controller/EntryController.php:54` | `#[IsGranted('EDIT_ENTRIES')]` |
| 单条权限校验 | `src/Controller/EntryController.php:108` | `$this->security->isGranted('EDIT', $entry)` |
| 权限失败处理 | `src/Controller/EntryController.php:109` | 抛 `AccessDeniedException`，整体终止 |
| 权限常量定义（全局） | `src/Security/Voter/MainVoter.php:13` | `EDIT_ENTRIES` 常量 |
| 权限常量定义（单条） | `src/Security/Voter/EntryVoter.php:13` | `EDIT` 常量 |
| 权限校验逻辑 | `src/Security/Voter/EntryVoter.php:40-54` | 所有者校验 |

### 5.2 Web 批量 toggle-star（星标切换）

| 维度 | 代码位置 | 说明 |
|------|---------|------|
| 状态机方法 | `src/Entity/Entry.php:423-428` | `toggleStar()` 方法，**不更新**时间戳 |
| 状态机时间戳方法 | `src/Entity/Entry.php:539-548` | `updateStar()` 方法，本应被调用但未被调用 |
| 批量入口调用 | `src/Controller/EntryController.php:114-115` | 仅调用 `$entry->toggleStar()`（Bug 位置） |
| 单条对比（正确） | `src/Controller/EntryController.php:474-475` | 单条调用 `toggleStar()` + `updateStar()` |
| 全局权限入口 | `src/Controller/EntryController.php:54` | `#[IsGranted('EDIT_ENTRIES')]` |
| 单条权限校验 | `src/Controller/EntryController.php:108` | `$this->security->isGranted('EDIT', $entry)` |
| 权限失败处理 | `src/Controller/EntryController.php:109` | 抛 `AccessDeniedException`，整体终止 |

### 5.3 Web 批量 delete（删除）

| 维度 | 代码位置 | 说明 |
|------|---------|------|
| 批量入口调用 | `src/Controller/EntryController.php:123-125` | 调用 `remove()` + 事件派发 |
| 全局权限入口 | `src/Controller/EntryController.php:54` | `#[IsGranted('EDIT_ENTRIES')]`（粒度过粗） |
| 单条权限校验 | `src/Controller/EntryController.php:108` | `$this->security->isGranted('EDIT', $entry)`（粒度过粗） |
| 单条对比（正确） | `src/Controller/EntryController.php:499` | 单条使用 `#[IsGranted('DELETE', ...)]` |
| 权限失败处理 | `src/Controller/EntryController.php:109` | 抛 `AccessDeniedException`，整体终止 |

### 5.4 API 批量删除

| 维度 | 代码位置 | 说明 |
|------|---------|------|
| 批量入口方法 | `src/Controller/Api/EntryRestController.php:478-511` | `deleteEntriesListAction()` |
| 全局权限入口 | `src/Controller/Api/EntryRestController.php:479` | `#[IsGranted('DELETE_ENTRIES')]` |
| 单条权限校验 | `src/Controller/Api/EntryRestController.php:499` | `isGranted('DELETE', $entry)`（粒度正确） |
| 权限失败处理 | `src/Controller/Api/EntryRestController.php:499` | if 判断静默跳过，不抛异常 |
| flush 时机 | `src/Controller/Api/EntryRestController.php:504` | 逐条 flush |
| 权限常量定义（全局） | `src/Security/Voter/MainVoter.php:16` | `DELETE_ENTRIES` 常量 |
| 权限常量定义（单条） | `src/Security/Voter/EntryVoter.php:20` | `DELETE` 常量 |

### 5.5 API 批量加标签

| 维度 | 代码位置 | 说明 |
|------|---------|------|
| 批量入口方法 | `src/Controller/Api/EntryRestController.php:1345-1378` | `postEntriesTagsListAction()` |
| 全局权限入口 | `src/Controller/Api/EntryRestController.php:1346` | `#[IsGranted('CREATE_TAGS')]` |
| 单条权限校验 | `src/Controller/Api/EntryRestController.php:1369` | `isGranted('TAG', $entry)`（粒度正确） |
| 权限失败处理 | `src/Controller/Api/EntryRestController.php:1369` | if 判断静默跳过 |
| flush 时机 | `src/Controller/Api/EntryRestController.php:1373` | 逐条 flush |

### 5.6 API 批量删标签

| 维度 | 代码位置 | 说明 |
|------|---------|------|
| 批量入口方法 | `src/Controller/Api/EntryRestController.php:1280-1322` | `deleteEntriesTagsListAction()` |
| 全局权限入口 | `src/Controller/Api/EntryRestController.php:1281` | `#[IsGranted('DELETE_TAGS')]` |
| 单条权限校验 | `src/Controller/Api/EntryRestController.php:1304` | `isGranted('UNTAG', $entry)`（粒度正确） |
| 权限失败处理 | `src/Controller/Api/EntryRestController.php:1304` | if 判断静默跳过 |
| flush 时机 | `src/Controller/Api/EntryRestController.php:1317` | 逐条 flush |

### 5.7 Web 单条分享（设为公开）

| 维度 | 代码位置 | 说明 |
|------|---------|------|
| 状态机方法 | `src/Entity/Entry.php:767-773` | `generateUid()`，幂等生成 uid |
| 入口方法 | `src/Controller/EntryController.php:538-556` | `shareAction()` |
| 入口路由 | `src/Controller/EntryController.php:538` | `POST /share/{id}`，路由名 `share` |
| 权限入口 | `src/Controller/EntryController.php:539` | `#[IsGranted('SHARE', subject: 'entry')]` |
| 权限常量 | `src/Security/Voter/EntryVoter.php:17` | `SHARE` 常量 |
| CSRF 校验 | `src/Controller/EntryController.php:542` | token id：`share-entry` |
| 核心调用 | `src/Controller/EntryController.php:546-547` | `$entry->generateUid()` |
| flush 时机 | `src/Controller/EntryController.php:550` | 单次 flush |

### 5.8 Web 单条取消分享（设为私有）

| 维度 | 代码位置 | 说明 |
|------|---------|------|
| 状态机方法 | `src/Entity/Entry.php:775-778` | `cleanUid()`，清除 uid |
| 入口方法 | `src/Controller/EntryController.php:564-579` | `deleteShareAction()` |
| 入口路由 | `src/Controller/EntryController.php:563` | `POST /share/delete/{id}`，路由名 `delete_share` |
| 权限入口 | `src/Controller/EntryController.php:564` | `#[IsGranted('UNSHARE', subject: 'entry')]` |
| 权限常量 | `src/Security/Voter/EntryVoter.php:18` | `UNSHARE` 常量 |
| CSRF 校验 | `src/Controller/EntryController.php:567` | token id：`delete-share` |
| 核心调用 | `src/Controller/EntryController.php:571` | `$entry->cleanUid()` |
| flush 时机 | `src/Controller/EntryController.php:574` | 单次 flush |

### 5.9 API 单条设置 public（创建 & 修改）

| 维度 | 代码位置（创建） | 代码位置（修改） | 说明 |
|------|----------------|----------------|------|
| 状态机方法 | `src/Entity/Entry.php:767-773` | `src/Entity/Entry.php:775-778` | `generateUid()` / `cleanUid()` |
| 入口方法 | `src/Controller/Api/EntryRestController.php:717-811` | `src/Controller/Api/EntryRestController.php:940-1040` | POST 创建 / PATCH 修改 |
| 路由名 | `api_post_entries` | `api_patch_entries` | |
| 全局权限 | `#[IsGranted('CREATE_ENTRIES')]` | — | 创建用全局权限 |
| 单条权限 | — | `src/Controller/Api/EntryRestController.php:939` | 修改用 `EDIT` 单条权限 |
| public 处理 | `src/Controller/Api/EntryRestController.php:778-784` | `src/Controller/Api/EntryRestController.php:998-1004` | 三分支逻辑完全相同 |
| 参数名 | `isPublic`（来自 `public` query 参数） | `isPublic`（来自 `public` query 参数） | |
| 权限失败 | 401/403 | 401/403 | 标准 HTTP 错误，不会静默跳过 |

---

## 六、不统一点汇总（快速索引）

### 6.1 状态切换方法不一致

| 场景 | 归档方法 | 星标方法 | 公开方法 | 星标时间戳 | 公开时间戳 |
|------|---------|---------|---------|-----------|-----------|
| Web 单条 | `toggleArchive()` | `toggleStar()` + `updateStar()` | `generateUid()` / `cleanUid()` | ✅ 更新 | —（无） |
| Web 批量 | `toggleArchive()` | `toggleStar()` | —（无批量） | ❌ 不更新 | — |
| API POST 创建 | `updateArchived()` | `updateStar()` | `generateUid()` / `cleanUid()` | ✅ 更新 | —（无） |
| API PATCH 修改 | `updateArchived()` | `updateStar()` | `generateUid()` / `cleanUid()` | ✅ 更新 | —（无） |

**核心问题 1**：Web 批量切换星标时 `starredAt` 不更新。
**Bug 位置**：`src/Controller/EntryController.php:115`

**核心问题 2**：公开状态完全没有时间戳字段（没有 `publicAt` 之类的字段），无法追踪何时设为公开。

### 6.2 状态机设计不对称

| 维度 | 归档 | 星标 | 公开 |
|------|------|------|------|
| 是否有 toggle 方法 | ✅ `toggleArchive()` | ✅ `toggleStar()` | ❌ 无 toggle |
| 是否有时间戳 | ✅ `archivedAt` | ✅ `starredAt` | ❌ 无时间戳 |
| 是否有 updateXxx 方法 | ✅ `updateArchived()` | ✅ `updateStar()` | ❌ 无对应方法 |
| 状态字段 | bool 字段 `isArchived` | bool 字段 `isStarred` | 派生自 `uid` |

公开状态的特殊性：
- 由 `uid` 是否为 null 派生，不是独立布尔字段
- 设为公开用 `generateUid()`，设为私有用 `cleanUid()`，不是对称的方法名
- 没有「切换」语义的方法（toggle）

### 6.3 批量操作覆盖度不一致

| 操作类型 | Web 批量 | API 批量 |
|---------|---------|---------|
| 切换归档 | ✅ `toggle-read` | ❌ 无 |
| 切换星标 | ✅ `toggle-star` | ❌ 无 |
| 删除 | ✅ `delete` | ✅ `DELETE /api/entries/list` |
| 标签操作 | ✅ `tag` | ✅ 加标签 / 删标签 |
| 切换公开 | ❌ 无 | ❌ 无 |

**关键发现**：API 端完全没有「批量切换归档/星标」接口，Web 端完全没有「批量切换公开」接口。三个状态的批量操作覆盖度都不完整。

### 6.4 权限失败处理不一致

| 接口 | 权限失败行为 | 代码位置 |
|------|-------------|---------|
| Web 批量 massAction | 抛异常，整体终止 | `src/Controller/EntryController.php:109` |
| API 批量删除 | 静默跳过，继续处理 | `src/Controller/Api/EntryRestController.php:499` |
| API 批量加标签 | 静默跳过，继续处理 | `src/Controller/Api/EntryRestController.php:1369` |
| API 批量删标签 | 静默跳过，继续处理 | `src/Controller/Api/EntryRestController.php:1304` |

### 6.5 权限粒度不一致

- Web 批量统一用 `EDIT` 权限（粒度过粗）
- API 批量用对应操作权限（粒度较细）

### 6.6 flush 时机不一致

- Web 批量：循环结束后一次 flush（`src/Controller/EntryController.php:129`）
- API 批量：逐条 flush（性能较低）

### 6.7 标识方式不一致

- Web 批量：条目 ID 数组（`entry-checkbox`）
- API 批量：URL 数组（JSON）

### 6.8 数量限制不一致

- API 批量创建：受 `apiLimitMassActions` 限制（`src/Controller/Api/EntryRestController.php:542`）
- Web 批量及其他 API 批量：无显式数量限制

---

## 七、代码文件索引

| 文件 | 仓库相对路径（可跳转） | 说明 |
|------|----------------------|------|
| Entry 实体 | `src/Entity/Entry.php:1-958` | 状态属性及切换方法 |
| Web 入口控制器 | `src/Controller/EntryController.php:1-738` | 单条 + 批量 Web 操作 |
| API 入口控制器 | `src/Controller/Api/EntryRestController.php:1-1420` | API 单条 + 批量操作 |
| API 基类 | `src/Controller/Api/WallabagRestController.php:1-123` | `apiLimitMassActions` 等公共属性 |
| 条目权限 Voter | `src/Security/Voter/EntryVoter.php:1-55` | 单条目权限校验 |
| 全局权限 Voter | `src/Security/Voter/MainVoter.php:1-49` | 全局/批量操作权限校验 |
| 前端批量控制器 | `assets/controllers/batch_edit_controller.js:1-15` | 前端批量选择交互 |
