# 条目状态批量切换代码分析

> 文档目的：梳理条目状态机定义、批量操作入口、权限校验机制，以及多接口行为差异。所有结论均标注仓库相对路径，便于直接复核。

---

## 一、状态机定义

### 1.1 三大状态属性

条目（Entry）有三个独立的布尔状态，定义在 `src/Entity/Entry.php`：

| 状态 | 字段 | 类型 | 默认值 | 时间戳字段 |
|------|------|------|---------|-----------|
| 归档（已读/未读） | `isArchived` | bool | `false` | `archivedAt` |
| 星标（收藏） | `isStarred` | bool | `false` | `starredAt` |
| 公开 | （虚拟属性） | - | - | 通过 `uid` 是否为 null 判断 |

**代码位置**：
- `isArchived` 字段：`src/Entity/Entry.php:108-111`
- `archivedAt` 字段：`src/Entity/Entry.php:116-118`
- `isStarred` 字段：`src/Entity/Entry.php:123-126`
- `starredAt` 字段：`src/Entity/Entry.php:166-168`
- `uid` 字段：`src/Entity/Entry.php:53-55`
- `isPublic()` 虚拟属性：`src/Entity/Entry.php:789-792`

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
  3. 如果设为归档（true），则将 `archivedAt` 设为当前时间
- **返回**：`$this`

#### 1.2.3 `toggleArchive()`
- **位置**：`src/Entity/Entry.php:384-389`
- **行为**：调用 `updateArchived()` 异或切换状态，**会更新** `archivedAt` 时间戳
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
  3. 如果设为星标（true），则将 `starredAt` 设为当前时间
- **返回**：`$this`

#### 1.3.3 `toggleStar()`
- **位置**：`src/Entity/Entry.php:423-428`
- **行为**：直接取反 `isStarred`，**不调用** `updateStar()`，因此**不会更新** `starredAt` 时间戳
- **返回**：`$this`

> ⚠️ **关键不一致点**：`toggleArchive()` 会调用 `updateArchived()` 更新时间戳，但 `toggleStar()` 不会调用 `updateStar()`。两个 toggle 方法的行为不对称。

### 1.4 公开状态切换方法

#### 1.4.1 `generateUid()`
- **位置**：`src/Entity/Entry.php:767-773`
- **行为**：如果 `uid` 为 null，则生成一个唯一 ID，将条目设为公开
- **返回**：void

#### 1.4.2 `cleanUid()`
- **位置**：`src/Entity/Entry.php:775-778`
- **行为**：将 `uid` 设为 null，将条目设为私有
- **返回**：void

#### 1.4.3 `isPublic()`
- **位置**：`src/Entity/Entry.php:789-792`
- **行为**：虚拟属性，返回 `null !== $this->uid`

---

## 二、批量操作入口

### 2.1 Web 端批量操作：`massAction()`

**入口方法**：`src/Controller/EntryController.php:53-135`
**路由**：`POST /mass`（路由名：`mass_action`）

#### 2.1.1 全局权限
- 注解：`#[IsGranted('EDIT_ENTRIES')]`
- 对应 Voter：`MainVoter::EDIT_ENTRIES`
- 实际要求：`ROLE_USER` 角色

**代码位置**：`src/Controller/EntryController.php:53-55`

#### 2.1.2 CSRF 校验
- Token ID：`mass-action`
- 失败行为：抛出 `BadRequestHttpException`

**代码位置**：`src/Controller/EntryController.php:57-59`

#### 2.1.3 支持的操作类型

通过请求参数判断操作类型，判断逻辑在 `src/Controller/EntryController.php:66-101`：

| 操作类型 | 触发条件 | 对应方法 | 单条权限 |
|---------|---------|---------|---------|
| `toggle-read`（默认） | 无特殊参数 | `$entry->toggleArchive()` | `EDIT` |
| `toggle-star` | 请求中有 `toggle-star` 参数 | `$entry->toggleStar()` | `EDIT` |
| `delete` | 请求中有 `delete` 参数 | `$em->remove($entry)` | `EDIT` |
| `tag` | 请求中有 `tag` 参数 | `addTag()` / `removeTag()` | `EDIT` |

#### 2.1.4 批量循环逻辑

**位置**：`src/Controller/EntryController.php:103-130`

流程：
1. 遍历 `entry-checkbox` 数组中的条目 ID
2. 通过 `findById()` 查询条目
3. 对每条执行 `$this->security->isGranted('EDIT', $entry)` 权限检查
4. 权限不足则 `throw $this->createAccessDeniedException()` —— **立即终止，不继续处理后续条目**
5. 执行对应操作
6. 全部循环结束后一次性 `flush()`

#### 2.1.5 批量星标不更新时间戳

**关键代码**：`src/Controller/EntryController.php:114-115`

```php
} elseif ('toggle-star' === $action) {
    $entry->toggleStar();
}
```

由于 `toggleStar()` 不会更新 `starredAt`（见 1.3.3），因此**批量切换星标时，`starredAt` 时间戳不会更新**。

作为对比，单条星标切换会更新时间戳（见 3.1.2）。

---

### 2.2 API 端批量操作

API 端没有专门的「批量状态切换」接口，但有 4 个批量操作接口。

#### 2.2.1 批量创建条目

**方法**：`src/Controller/Api/EntryRestController.php:536-576`
**路由**：`POST /api/entries/lists.{_format}`（路由名：`api_post_entries_list`）
**全局权限**：`#[IsGranted('CREATE_ENTRIES')]`

**特点**：
- 通过 URL 数组（JSON）批量创建条目
- 有数量限制：`apiLimitMassActions`，超限抛 `BadRequestHttpException`
- 逐条创建，逐条 `flush()`
- 不做单条权限检查（因为是创建新条目，属于当前用户）

#### 2.2.2 批量删除条目

**方法**：`src/Controller/Api/EntryRestController.php:478-511`
**路由**：`DELETE /api/entries/list.{_format}`（路由名：`api_delete_entries_list`）
**全局权限**：`#[IsGranted('DELETE_ENTRIES')]`

**关键逻辑**：`src/Controller/Api/EntryRestController.php:491-508`

```php
foreach ($urls as $key => $url) {
    $entry = $entryRepository->findByUrlAndUserId($url, $this->getUser()->getId());
    $results[$key]['url'] = $url;

    if (false !== $entry && $this->authorizationChecker->isGranted('DELETE', $entry)) {
        // 删除操作...
        $this->entityManager->remove($entry);
        $this->entityManager->flush();
    }

    $results[$key]['entry'] = $entry instanceof Entry ? true : false;
}
```

**权限失败处理**：
- 无权限的条目**静默跳过**，不抛异常
- 继续处理下一个条目
- 返回结果中 `entry` 字段标记是否存在该条目（但不标记是否有权限）

#### 2.2.3 批量添加标签

**方法**：`src/Controller/Api/EntryRestController.php:1345-1378`
**路由**：`POST /api/entries/tags/lists.{_format}`（路由名：`api_post_entries_tags_list`）
**全局权限**：`#[IsGranted('CREATE_TAGS')]`

**单条权限**：`$this->authorizationChecker->isGranted('TAG', $entry)`
**权限失败**：静默跳过，继续处理

**代码位置**：`src/Controller/Api/EntryRestController.php:1369`

#### 2.2.4 批量删除标签

**方法**：`src/Controller/Api/EntryRestController.php:1280-1322`
**路由**：`DELETE /api/entries/tags/list.{_format}`（路由名：`api_delete_entries_tags_list`）
**全局权限**：`#[IsGranted('DELETE_TAGS')]`

**单条权限**：`$this->authorizationChecker->isGranted('UNTAG', $entry)`
**权限失败**：静默跳过，继续处理

**代码位置**：`src/Controller/Api/EntryRestController.php:1304`

---

## 三、权限校验体系

### 3.1 两层 Voter 架构

项目有两个独立的 Voter，分别处理不同级别的权限。

#### 3.1.1 MainVoter（全局操作权限）

**文件**：`src/Security/Voter/MainVoter.php`
**适用场景**：不需要具体主体（subject）的全局操作

**支持的属性**：`src/Security/Voter/MainVoter.php:11-22`

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

**校验逻辑**：`src/Security/Voter/MainVoter.php:42-48`

所有这些权限最终都要求 `$this->security->isGranted('ROLE_USER')`。

**supports 条件**：`src/Security/Voter/MainVoter.php:29-40`
- `$subject` 必须为 null（即不针对具体实体）
- `$attribute` 必须在上述列表中

#### 3.1.2 EntryVoter（单条目权限）

**文件**：`src/Security/Voter/EntryVoter.php`
**适用场景**：针对具体 Entry 实体的操作

**支持的属性**：`src/Security/Voter/EntryVoter.php:12-25`

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

**校验逻辑**：`src/Security/Voter/EntryVoter.php:40-54`

所有单条目权限的校验条件完全相同：`$user === $subject->getUser()` —— 当前用户必须是条目的所有者。

**supports 条件**：`src/Security/Voter/EntryVoter.php:27-38`
- `$subject` 必须是 `Entry` 实例
- `$attribute` 必须在上述列表中

### 3.2 批量操作中的权限校验模式对比

#### 模式 A：Web 端 massAction

```
入口：@IsGranted('EDIT_ENTRIES')         ← MainVoter 全局检查
        ↓
循环每条 entry：
  isGranted('EDIT', $entry)              ← EntryVoter 单条检查
        ↓
  无权限 → throw AccessDeniedException   ← 立即终止，整体失败
        ↓
  有权限 → 执行操作
        ↓
最后统一 flush()
```

**特点**：
- 全局权限用 `EDIT_ENTRIES`
- 单条权限统一用 `EDIT`（而非 `STAR`/`ARCHIVE`/`DELETE` 等细粒度权限）
- 权限失败立即终止，原子性强但容错性差

**代码位置**：`src/Controller/EntryController.php:108-110`

#### 模式 B：API 端批量操作（以删除为例）

```
入口：@IsGranted('DELETE_ENTRIES')       ← MainVoter 全局检查
        ↓
循环每条 entry：
  isGranted('DELETE', $entry)            ← EntryVoter 单条检查
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
- 逐条 `flush()`，性能较低

---

## 四、单条操作接口（对比基准）

为了看清批量操作的差异，这里列出单条状态切换接口作为对比基准。

### 4.1 Web 端单条操作

#### 4.1.1 单条归档切换

**方法**：`src/Controller/EntryController.php:435-459`
**路由**：`POST /archive/{id}`（路由名：`archive_entry`）
**权限**：`#[IsGranted('ARCHIVE', subject: 'entry')]`
**调用方法**：`$entry->toggleArchive()`
**时间戳**：✅ 更新（因为 `toggleArchive()` 调用了 `updateArchived()）

#### 4.1.2 单条星标切换

**方法**：`src/Controller/EntryController.php:466-491`
**路由**：`POST /star/{id}`（路由名：`star_entry`）
**权限**：`#[IsGranted('STAR', subject: 'entry')]`
**调用方法**：
```php
$entry->toggleStar();       // 切换布尔值
$entry->updateStar($entry->isStarred());  // 更新时间戳
```
**时间戳**：✅ 更新（因为显式调用了 `updateStar()`）

**代码位置**：`src/Controller/EntryController.php:474-475`

> ⚠️ **注意**：单条星标切换调用了两个方法：先 `toggleStar()` 切换状态，再 `updateStar()` 更新时间戳。而批量星标只调用了 `toggleStar()`，不会更新时间戳。

#### 4.1.3 单条删除

**方法**：`src/Controller/EntryController.php:498-531`
**路由**：`POST /delete/{id}`（路由名：`delete_entry`）
**权限**：`#[IsGranted('DELETE', subject: 'entry')]`

### 4.2 API 端单条操作

#### 4.2.1 PATCH 修改条目

**方法**：`src/Controller/Api/EntryRestController.php:938-1025`
**路由**：`PATCH /api/entries/{entry}.{_format}`（路由名：`api_patch_entries`）
**权限**：`#[IsGranted('EDIT', subject: 'entry')]`

**状态切换方式**：
- 归档：`$entry->updateArchived((bool) $data['isArchived'])` —— ✅ 更新时间戳
- 星标：`$entry->updateStar((bool) $data['isStarred'])` —— ✅ 更新时间戳

**代码位置**：`src/Controller/Api/EntryRestController.php:985-991`

#### 4.2.2 POST 创建条目

**方法**：`src/Controller/Api/EntryRestController.php:715-811`
**路由**：`POST /api/entries.{_format}`（路由名：`api_post_entries`）
**权限**：`#[IsGranted('CREATE_ENTRIES')]`

**状态设置方式**：同 PATCH，使用 `updateArchived()` 和 `updateStar()`，均更新时间戳。

---

## 五、多接口行为不统一点汇总

### 5.1 状态切换方法不一致（时间戳）

| 场景 | 归档 | 星标 | 星标时间戳 |
|------|------|------|-----------|
| Web 单条归档 | `toggleArchive()` | - | - |
| Web 单条星标 | - | `toggleStar()` + `updateStar()` | ✅ 更新 |
| Web 批量 toggle-read | `toggleArchive()` | - | - |
| Web 批量 toggle-star | - | `toggleStar()` | ❌ 不更新 |
| API PATCH 单条 | `updateArchived()` | `updateStar()` | ✅ 更新 |
| API POST 创建 | `updateArchived()` | `updateStar()` | ✅ 更新 |

**问题**：Web 批量切换星标时，`starredAt` 不会更新。这是一个明显的 bug。

**复现代码**：
- 单条（正确）：`src/Controller/EntryController.php:474-475`
- 批量（错误）：`src/Controller/EntryController.php:115`

### 5.2 权限校验粒度不一致

| 操作 | Web 单条 | Web 批量 | API 单条 | API 批量 |
|------|---------|---------|---------|---------|
| 归档 | `ARCHIVE` | `EDIT` | `EDIT` | —（无接口） |
| 星标 | `STAR` | `EDIT` | `EDIT` | —（无接口） |
| 删除 | `DELETE` | `EDIT` | `DELETE` | `DELETE` |
| 加标签 | `TAG` | `EDIT` | `TAG` | `TAG` |
| 删标签 | `UNTAG` | `EDIT` | `UNTAG` | `UNTAG` |

**问题**：
1. Web 批量操作统一用 `EDIT` 权限，粒度过粗
2. API PATCH 单条也用 `EDIT` 权限，没有单独的 `ARCHIVE`/`STAR` 权限

虽然目前 `EntryVoter` 中所有权限的校验条件相同（都是所有者），但权限粒度设计不一致，未来扩展权限时可能产生问题。

### 5.3 权限失败处理不一致

| 接口 | 权限失败行为 |
|------|-------------|
| Web 批量 massAction | 抛 `AccessDeniedException`，整体终止 |
| API 批量删除 | 静默跳过，继续处理 |
| API 批量加标签 | 静默跳过，继续处理 |
| API 批量删标签 | 静默跳过，继续处理 |

**两种策略的权衡**：
- 终止策略：原子性好，用户明确知道操作失败
- 跳过策略：容错性好，不会因为一条失败影响全部，但用户可能不知道部分失败

### 5.4 批量标识方式不一致

| 接口 | 标识方式 |
|------|---------|
| Web massAction | 条目 ID 数组（`entry-checkbox`） |
| API 批量创建 | URL 数组（JSON） |
| API 批量删除 | URL 数组（JSON） |
| API 批量标签 | URL + 标签（JSON 对象数组） |

### 5.5 数量限制不一致

| 接口 | 数量限制 |
|------|---------|
| Web massAction | 无显式限制 |
| API 批量创建 | 受 `apiLimitMassActions` 限制 |
| API 批量删除 | 无显式限制 |
| API 批量标签 | 无显式限制 |

### 5.6 flush 时机不一致

| 接口 | flush 时机 |
|------|-----------|
| Web massAction | 全部循环结束后一次 flush |
| API 批量创建 | 逐条 flush |
| API 批量删除 | 逐条 flush |
| API 批量标签 | 逐条 flush |

Web 批量性能更好（单次 flush），但如果中间出错会全部回滚。

---

## 六、代码文件索引

| 文件 | 仓库相对路径 | 说明 |
|------|-------------|------|
| Entry 实体 | `src/Entity/Entry.php` | 状态属性及切换方法 |
| Web 入口控制器 | `src/Controller/EntryController.php` | 单条 + 批量操作 |
| API 入口控制器 | `src/Controller/Api/EntryRestController.php` | API 单条 + 批量操作 |
| API 基类 | `src/Controller/Api/WallabagRestController.php` | `apiLimitMassActions` 等 |
| 条目权限 | `src/Security/Voter/EntryVoter.php` | 单条目权限校验 |
| 全局权限 | `src/Security/Voter/MainVoter.php` | 全局操作权限校验 |
| 前端批量控制器 | `assets/controllers/batch_edit_controller.js` | 前端批量选择交互 |
