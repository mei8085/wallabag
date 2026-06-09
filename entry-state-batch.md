# 条目状态批量切换行为分析

## 一、状态机定义

### 1.1 核心状态属性

条目（Entry）有三个主要的布尔状态属性，定义在 [Entry.php](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php) 中：

| 状态 | 属性字段 | 默认值 | 对应时间戳字段 | 说明 |
|------|---------|--------|--------------|------|
| 归档（已读） | `isArchived` | `false` | `archivedAt` | 标记条目是否已读/归档 |
| 星标（收藏） | `isStarred` | `false` | `starredAt` | 标记条目是否被收藏 |
| 公开 | （虚拟属性） | - | - | 通过 `uid` 是否为 null 判断，`uid` 存在则为公开 |

此外还有 `isNotParsed` 属性表示条目是否未解析。

### 1.2 状态切换方法

#### 归档状态
- `toggleArchive()` [L384-L389](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php#L384-L389)：异或切换归档状态，调用 `updateArchived()` 更新时间戳
- `updateArchived($isArchived = false)` [L335-L344](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php#L335-L344)：设置归档状态并更新 `archivedAt` 时间戳
- `setArchived($isArchived)` [L321-L326](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php#L321-L326)：仅设置状态，不更新时间戳

#### 星标状态
- `toggleStar()` [L423-L428](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php#L423-L428)：取反切换星标状态，**不更新时间戳**
- `updateStar($isStarred = false)` [L539-L548](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php#L539-L548)：设置星标状态并更新 `starredAt` 时间戳
- `setStarred($isStarred)` [L398-L403](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php#L398-L403)：仅设置状态，不更新时间戳

> **注意不一致点**：`toggleStar()` 只切换状态布尔值，但不会更新 `starredAt` 时间戳；而 `toggleArchive()` 会调用 `updateArchived()` 同时更新时间戳。

#### 公开状态
- `generateUid()` [L767-L773](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php#L767-L773)：生成 uid，将条目设为公开
- `cleanUid()` [L775-L778](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php#L775-L778)：清除 uid，将条目设为私有
- `isPublic()` [L789-L792](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php#L789-L792)：虚拟属性，通过 `uid !== null` 判断是否公开

---

## 二、批量操作入口

### 2.1 Web 端批量操作

**入口方法**：[EntryController::massAction()](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/EntryController.php#L53-L135)

**路由**：`POST /mass` （名称：`mass_action`）

**支持的操作类型**：

| 操作 | 触发条件 | 调用方法 | 权限检查 |
|------|---------|---------|---------|
| 切换归档（toggle-read） | 默认操作 | `$entry->toggleArchive()` | `EDIT` |
| 切换星标（toggle-star） | `toggle-star` 参数存在 | `$entry->toggleStar()` | `EDIT` |
| 标签操作（tag） | `tag` 参数存在 | `addTag()` / `removeTag()` | `EDIT` |
| 删除（delete） | `delete` 参数存在 | `remove()` | `EDIT` |

**工作流程**：
1. CSRF token 校验 (`mass-action`)
2. 解析请求参数，确定操作类型
3. 遍历 `entry-checkbox` 数组中的条目 ID
4. 对每个条目先做 `EDIT` 权限检查
5. 执行对应操作
6. 一次性 `flush()`

### 2.2 API 端批量操作

API 端没有专门的「批量状态切换」接口，但有以下批量操作接口：

#### 2.2.1 批量创建条目
- **方法**：[EntryRestController::postEntriesListAction()](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/Api/EntryRestController.php#L536-L576)
- **路由**：`POST /api/entries/lists.{_format}`
- **权限**：`CREATE_ENTRIES`（全局）
- **限制**：受 `apiLimitMassActions` 数量限制
- **特点**：通过 URL 查找或创建条目，创建时可设置 `archive`、`starred`、`public` 等状态

#### 2.2.2 批量删除条目
- **方法**：[EntryRestController::deleteEntriesListAction()](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/Api/EntryRestController.php#L478-L511)
- **路由**：`DELETE /api/entries/list.{_format}`
- **权限**：`DELETE_ENTRIES`（全局） + 单条 `DELETE` 权限
- **特点**：通过 URL 查找条目，逐条检查权限后删除

#### 2.2.3 批量添加标签
- **方法**：[EntryRestController::postEntriesTagsListAction()](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/Api/EntryRestController.php#L1345-L1378)
- **路由**：`POST /api/entries/tags/lists.{_format}`
- **权限**：`CREATE_TAGS`（全局） + 单条 `TAG` 权限
- **特点**：通过 URL 查找条目

#### 2.2.4 批量删除标签
- **方法**：[EntryRestController::deleteEntriesTagsListAction()](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/Api/EntryRestController.php#L1280-L1322)
- **路由**：`DELETE /api/entries/tags/list.{_format}`
- **权限**：`DELETE_TAGS`（全局） + 单条 `UNTAG` 权限
- **特点**：通过 URL 查找条目

### 2.3 单条状态切换接口（对比用）

| 接口 | 方法 | 路由 | 权限 | 调用方法 |
|------|------|------|------|---------|
| Web 切换归档 | `toggleArchiveAction()` | `POST /archive/{id}` | `ARCHIVE` | `toggleArchive()` |
| Web 切换星标 | `toggleStarAction()` | `POST /star/{id}` | `STAR` | `toggleStar()` + `updateStar()` |
| API 修改条目 | `patchEntriesAction()` | `PATCH /api/entries/{entry}` | `EDIT` | `updateArchived()` / `updateStar()` |
| API 创建条目 | `postEntriesAction()` | `POST /api/entries` | `CREATE_ENTRIES` | `updateArchived()` / `updateStar()` |

> **注意不一致点**：
> 1. Web 单条 `toggleStarAction()` 同时调用了 `toggleStar()` 和 `updateStar()`，而批量 `massAction()` 只调用 `toggleStar()`，**导致批量切换星标时不会更新 `starredAt` 时间戳**。
> 2. Web 批量操作统一使用 `EDIT` 权限，而单条操作有专门的 `ARCHIVE`、`STAR`、`DELETE` 等细粒度权限。

---

## 三、权限校验体系

### 3.1 两层权限模型

#### 3.1.1 全局操作权限（MainVoter）

定义在 [MainVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Security/Voter/MainVoter.php) 中，针对「资源类型」级别的操作，不需要具体主体（subject）：

| 权限常量 | 说明 | 所需角色 |
|---------|------|---------|
| `LIST_ENTRIES` | 列出条目 | `ROLE_USER` |
| `CREATE_ENTRIES` | 创建条目 | `ROLE_USER` |
| `EDIT_ENTRIES` | 编辑条目（用于批量操作） | `ROLE_USER` |
| `DELETE_ENTRIES` | 删除条目（用于批量操作） | `ROLE_USER` |
| `EXPORT_ENTRIES` | 导出条目 | `ROLE_USER` |
| `IMPORT_ENTRIES` | 导入条目 | `ROLE_USER` |
| `CREATE_TAGS` | 创建标签（批量） | `ROLE_USER` |
| `DELETE_TAGS` | 删除标签（批量） | `ROLE_USER` |

所有这些权限最终都映射到 `ROLE_USER` 角色。

#### 3.1.2 单条目权限（EntryVoter）

定义在 [EntryVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Security/Voter/EntryVoter.php) 中，针对具体 Entry 实例：

| 权限常量 | 说明 | 校验条件 |
|---------|------|---------|
| `VIEW` | 查看 | `$user === $entry->getUser()` |
| `EDIT` | 编辑 | 同上 |
| `RELOAD` | 重新抓取 | 同上 |
| `STAR` | 星标操作 | 同上 |
| `ARCHIVE` | 归档操作 | 同上 |
| `SHARE` | 分享 | 同上 |
| `UNSHARE` | 取消分享 | 同上 |
| `EXPORT` | 导出 | 同上 |
| `DELETE` | 删除 | 同上 |
| `LIST_ANNOTATIONS` | 列出注释 | 同上 |
| `CREATE_ANNOTATIONS` | 创建注释 | 同上 |
| `LIST_TAGS` | 列出标签 | 同上 |
| `TAG` | 添加标签 | 同上 |
| `UNTAG` | 移除标签 | 同上 |

所有单条目权限的校验条件都是：**当前用户必须是条目的所有者**。

### 3.2 批量操作中的权限校验模式

#### Web 端 massAction 模式

```
入口注解 @IsGranted('EDIT_ENTRIES')  →  全局检查
        ↓
循环每条 entry:
  $security->isGranted('EDIT', $entry)  →  单条检查
        ↓
  执行操作
```

特点：
- 第一层用全局权限 `EDIT_ENTRIES`
- 第二层每条都用 `EDIT` 权限（而不是对应的 `STAR`/`ARCHIVE`/`DELETE` 等细粒度权限）
- 任何一条无权限则抛出异常，整个操作终止

#### API 端批量模式（以删除为例）

```
入口注解 @IsGranted('DELETE_ENTRIES')  →  全局检查
        ↓
循环每条 entry:
  $authorizationChecker->isGranted('DELETE', $entry)  →  单条检查
        ↓
  有权限则执行，无权限则跳过（静默忽略）
```

特点：
- 第一层用对应的全局权限（`DELETE_ENTRIES`/`CREATE_TAGS`/`DELETE_TAGS`）
- 第二层用对应的单条权限（`DELETE`/`TAG`/`UNTAG`）
- 无权限的条目静默跳过，不抛出异常，继续处理其他条目

---

## 四、多接口行为不统一点汇总

### 4.1 状态切换方法不一致

| 场景 | 归档状态 | 星标状态 | 时间戳更新 |
|------|---------|---------|-----------|
| Web 单条归档 | `toggleArchive()` | - | ✅ 更新 |
| Web 单条星标 | - | `toggleStar()` + `updateStar()` | ✅ 更新 |
| Web 批量 toggle-read | `toggleArchive()` | - | ✅ 更新 |
| Web 批量 toggle-star | - | `toggleStar()` | ❌ 不更新 |
| API PATCH 单条 | `updateArchived()` | `updateStar()` | ✅ 更新 |
| API POST 创建 | `updateArchived()` | `updateStar()` | ✅ 更新 |

**问题**：Web 批量星标切换时，`starredAt` 时间戳不会更新。

### 4.2 权限校验粒度不一致

| 操作 | Web 单条 | Web 批量 | API 单条 | API 批量 |
|------|---------|---------|---------|---------|
| 归档 | `ARCHIVE` | `EDIT` | `EDIT` | -（无批量接口） |
| 星标 | `STAR` | `EDIT` | `EDIT` | -（无批量接口） |
| 删除 | `DELETE` | `EDIT` | `DELETE` | `DELETE` |
| 添加标签 | `TAG` | `EDIT` | `TAG` | `TAG` |
| 删除标签 | `UNTAG` | `EDIT` | `UNTAG` | `UNTAG` |

**问题**：
1. Web 批量操作统一使用 `EDIT` 权限，粒度过粗，无法区分不同操作类型的权限。
2. API 单条 PATCH 也使用 `EDIT` 权限，没有单独的 `ARCHIVE`/`STAR` 权限校验。

### 4.3 无权限时的行为不一致

| 场景 | 行为 |
|------|------|
| Web 批量 massAction | 遇到无权限条目立即抛出异常，整体终止 |
| API 批量操作（删除/标签） | 无权限条目静默跳过，继续处理其他条目 |

### 4.4 批量操作的标识方式不一致

| 接口 | 标识方式 |
|------|---------|
| Web massAction | 条目 ID 数组（`entry-checkbox`） |
| API 批量创建/删除 | URL 数组 |
| API 批量标签操作 | URL 数组 |

### 4.5 数量限制不一致

| 接口 | 数量限制 |
|------|---------|
| Web massAction | 无显式限制 |
| API 批量创建 | 受 `apiLimitMassActions` 限制 |
| API 批量删除 | 无显式限制 |
| API 批量标签 | 无显式限制 |

---

## 五、代码参考

### 核心文件
- 实体定义：[Entry.php](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Entity/Entry.php)
- Web 控制器：[EntryController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/EntryController.php)
- API 控制器：[EntryRestController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/Api/EntryRestController.php)
- 条目权限：[EntryVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Security/Voter/EntryVoter.php)
- 全局权限：[MainVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Security/Voter/MainVoter.php)

### 关键代码位置
- Web 批量入口：[EntryController::massAction() L53-L135](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/EntryController.php#L53-L135)
- 星标切换不一致：[EntryController::toggleStarAction() L474-L476](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/EntryController.php#L474-L476) vs [EntryController::massAction() L115](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/EntryController.php#L115)
- API 批量删除：[EntryRestController::deleteEntriesListAction() L478-L511](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/Api/EntryRestController.php#L478-L511)
- API 批量标签：[EntryRestController::postEntriesTagsListAction() L1345-L1378](file:///d:/fz/0508-2/solo-dogfeeding/code/108-wallabag/src/Controller/Api/EntryRestController.php#L1345-L1378)
