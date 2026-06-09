# Entry 投票器注册、归属判定与共享 Token 校验关系分析

## 一、投票器注册机制

### 1.1 自动注册

投票器通过 Symfony 的 `autoconfigure` 机制自动注册，无需手动打标签。在 [services.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/app/config/services.yml#L12-L14) 中：

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
```

`Wallabag\` 命名空间下的类被自动扫描（[services.yml:L46-L48](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/app/config/services.yml#L46-L48)），其中继承自 `Voter` 抽象类的投票器会被自动识别并注册到安全系统中。

### 1.2 投票器清单

| 投票器类 | 文件路径 | 管辖范围 | subject 类型 |
|---------|---------|---------|-------------|
| EntryVoter | [EntryVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/EntryVoter.php) | 单条目操作权限 | Entry 实例 |
| MainVoter | [MainVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/MainVoter.php) | 全局操作权限（列表、创建等） | null（无 subject） |
| AnnotationVoter | [AnnotationVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/AnnotationVoter.php) | 注释操作权限 | Annotation 实例 |
| TagVoter | [TagVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/TagVoter.php) | 标签操作权限 | Tag 实例 |
| UserVoter | [UserVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/UserVoter.php) | 用户相关权限 | User 实例 |
| AdminVoter | [AdminVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/AdminVoter.php) | 管理员权限 | 无 |

**关键区分**：
- **EntryVoter**：针对**具体条目**（有 subject），做归属判定
- **MainVoter**：针对**全局操作**（无 subject），做角色判定

### 1.3 决策策略

项目未显式配置 `access_decision_manager`，使用 Symfony 默认的 **affirmative** 策略：只要有一个投票器投 `ACCESS_GRANTED`，即授权通过。

---

## 二、归属判定逻辑（EntryVoter）

### 2.1 支持的权限属性

[EntryVoter.php:L27-L38](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/EntryVoter.php#L27-L38) 定义了支持的属性：

```
VIEW, EDIT, RELOAD, STAR, ARCHIVE, SHARE, UNSHARE, EXPORT, DELETE,
LIST_ANNOTATIONS, CREATE_ANNOTATIONS, LIST_TAGS, TAG, UNTAG
```

适用条件：`$subject` 必须是 `Entry` 实例。

### 2.2 核心判定逻辑

[EntryVoter.php:L40-L54](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/EntryVoter.php#L40-L54) 的核心逻辑：

```php
return match ($attribute) {
    self::VIEW, self::EDIT, self::RELOAD, self::STAR, self::ARCHIVE, 
    self::SHARE, self::UNSHARE, self::EXPORT, self::DELETE, 
    self::LIST_ANNOTATIONS, self::CREATE_ANNOTATIONS, 
    self::LIST_TAGS, self::TAG, self::UNTAG => $user === $subject->getUser(),
    default => false,
};
```

**关键结论**：所有 14 种操作权限共用同一条判定规则——**当前登录用户必须是条目的所有者**（严格对象同一性比较 `===`）。

### 2.3 典型调用场景

在 [EntryController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php) 中：

- `viewAction` → `#[IsGranted('VIEW', subject: 'entry')]` （[L388-L389](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php#L388-L389)）
- `editEntryAction` → `#[IsGranted('EDIT', subject: 'entry')]`
- `shareAction` → `#[IsGranted('SHARE', subject: 'entry')]` （[L538-L539](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php#L538-L539)）
- `deleteShareAction` → `#[IsGranted('UNSHARE', subject: 'entry')]` （[L563-L564](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php#L563-L564)）

> **注意**：`SHARE` 和 `UNSHARE` 权限也走归属判定——只有所有者才能开启/关闭共享。

---

## 三、CREATE_ENTRIES 与 EDIT 权限控制对比

### 3.1 两层权限体系

项目对 Entry 操作的权限控制分为**两个层级**，分别由不同 Voter 负责：

| 权限属性 | 所属 Voter | subject | 判定逻辑 | 适用场景 |
|---------|-----------|---------|---------|---------|
| `CREATE_ENTRIES` | MainVoter | 无 | `ROLE_USER` 角色 | 创建条目、批量操作、列表查询 |
| `EDIT` / `VIEW` / `DELETE` 等 | EntryVoter | Entry 实例 | 必须是所有者 | 对具体条目的读写删 |

### 3.2 CREATE_ENTRIES 的职责范围

[MainVoter.php:L42-L48](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/MainVoter.php#L42-L48) 中：

```php
self::LIST_ENTRIES, self::CREATE_ENTRIES, self::EDIT_ENTRIES, 
self::EXPORT_ENTRIES, self::IMPORT_ENTRIES, self::DELETE_ENTRIES, ...
    => $this->security->isGranted('ROLE_USER'),
```

所有 `*_ENTRIES` 类权限的判定逻辑完全相同——只要有 `ROLE_USER` 角色就通过。

**使用 `CREATE_ENTRIES` 的接口**：

| 接口 | 路由 | 说明 |
|------|------|------|
| Web 新建 | `GET/POST /new-entry` | [EntryController.php:L170-L172](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php#L170-L172) |
| API 创建 | `POST /api/entries` | [EntryRestController.php:L715-L717](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L715-L717) |
| API 批量创建 | `POST /api/entries/lists` | [EntryRestController.php:L536-L538](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L536-L538) |
| API 列表查询 | `GET /api/entries` | [EntryRestController.php:L312-L314](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L312-L314) |

### 3.3 POST /api/entries 的 upsert 语义——边界点

**重要发现**：`POST /api/entries` 并非纯"创建"接口，而是 **upsert**（存在则更新）语义。

[EntryRestController.php:L728-L736](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L728-L736)：

```php
$entry = $entryRepository->findByUrlAndUserId(
    $url,
    $this->getUser()->getId()
);

if (false === $entry) {
    $entry = new Entry($this->getUser());
    $entry->setUrl($url);
}
```

**权限与数据安全分析**：
- 权限检查只用了 `CREATE_ENTRIES`（MainVoter → ROLE_USER），没有用 `EDIT`
- 但查询时限定了 `userId = 当前用户`，所以只能查到自己的条目
- 因此即使是"更新已存在的条目"，也不会越权——因为只能更新自己的

### 3.4 EDIT 权限的职责范围

`EDIT` 及同类属性（VIEW/DELETE/STAR/ARCHIVE 等）都由 EntryVoter 管辖，必须提供 Entry 实例作为 subject。

**使用 `EDIT` / 同类权限的接口**：

| 接口 | 路由 | 权限属性 |
|------|------|---------|
| Web 查看 | `GET /view/{id}` | `VIEW` |
| Web 编辑 | `GET/POST /edit/{id}` | `EDIT` |
| API 单条查询 | `GET /api/entries/{id}` | `VIEW` |
| API 更新 | `PATCH /api/entries/{id}` | `EDIT` |
| API 删除 | `DELETE /api/entries/{id}` | `DELETE` |
| API 重新抓取 | `PATCH /api/entries/{id}/reload` | `RELOAD` |

### 3.5 两层权限如何配合

以 API 创建接口为例，完整的安全链路：

```
请求到达
   │
   ▼
Firewall 认证（OAuth token / Session）
   │
   ▼
#[IsGranted('CREATE_ENTRIES')]  → MainVoter → ROLE_USER?
   │
   ▼
业务逻辑：按 url + userId 查找条目
   │  （限定 userId = 当前用户，天然隔离）
   ▼
不存在 → 创建新条目（归属当前用户）
存在   → 更新已有条目（必然是自己的）
```

---

## 四、API 中 isPublic 生成与清理 uid 的代码流程

### 4.1 isPublic 属性的序列化

`isPublic` 不是数据库字段，而是一个**虚拟属性**（[Entry.php:L786-L792](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Entity/Entry.php#L786-L792)）：

```php
#[VirtualProperty]
#[SerializedName('is_public')]
#[Groups(['entries_for_user'])]
public function isPublic()
{
    return null !== $this->uid;
}
```

即：**uid 非空 = 已公开**。

### 4.2 三条修改 uid 的代码路径

#### 路径一：Web 端开启共享

[EntryController.php:L538-L556](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php#L538-L556)

```php
#[IsGranted('SHARE', subject: 'entry')]  // 先过 EntryVoter 归属判定
public function shareAction(Request $request, Entry $entry)
{
    if (null === $entry->getUid()) {
        $entry->generateUid();  // 仅在 uid 为空时生成
        $this->entityManager->persist($entry);
        $this->entityManager->flush();
    }
    // 重定向到共享页面
}
```

**特点**：
- 幂等：多次调用不会重新生成 uid
- 必须是所有者才能操作

#### 路径二：Web 端关闭共享

[EntryController.php:L563-L579](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php#L563-L579)

```php
#[IsGranted('UNSHARE', subject: 'entry')]  // 先过 EntryVoter 归属判定
public function deleteShareAction(Request $request, Entry $entry)
{
    $entry->cleanUid();  // 直接置空
    $this->entityManager->persist($entry);
    $this->entityManager->flush();
}
```

#### 路径三：API 创建/更新时指定 public 参数

API 的 `POST`（创建）和 `PATCH`（更新）接口都支持通过 `public` 参数控制 uid。

**创建时（POST /api/entries）** [EntryRestController.php:L778-L784](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L778-L784)：

```php
if (null !== $data['isPublic']) {
    if (true === (bool) $data['isPublic'] && null === $entry->getUid()) {
        $entry->generateUid();
    } elseif (false === (bool) $data['isPublic']) {
        $entry->cleanUid();
    }
}
```

**更新时（PATCH /api/entries/{id}）** [EntryRestController.php:L998-L1004](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L998-L1004)：

```php
if (null !== $data['isPublic']) {
    if (true === (bool) $data['isPublic'] && null === $entry->getUid()) {
        $entry->generateUid();
    } elseif (false === (bool) $data['isPublic']) {
        $entry->cleanUid();
    }
}
```

**两段代码完全相同**，行为逻辑：

| 请求参数 | 当前 uid 状态 | 结果 |
|---------|-------------|------|
| `public=1` | uid 为 null | 生成新 uid |
| `public=1` | uid 已存在 | **不重新生成**，保持原值 |
| `public=0` | 任何状态 | 清空 uid |
| 不传 public | 任何状态 | **不做任何修改** |

### 4.3 uid 生成算法

[Entry.php:L767-L773](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Entity/Entry.php#L767-L773)：

```php
public function generateUid(): void
{
    if (null === $this->uid) {
        $this->uid = uniqid('', true);
    }
}
```

使用 PHP 内置的 `uniqid('', true)`：
- 长度：23 位字符串
- 前缀：基于微秒时间戳
- 后缀：`true` 参数增加额外的熵（基于 `lcg_value()`）
- **不是加密安全**的随机数

---

## 五、共享 Token 校验机制

共享访问**不通过 EntryVoter**，而是通过两条独立的旁路机制实现。

### 5.1 单条目共享（uid 机制）

#### 5.1.1 uid 字段

[Entry.php:L53-L55](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Entity/Entry.php#L53-L55) 定义了 `uid` 字段：

```php
#[ORM\Column(name: 'uid', type: 'string', length: 23, nullable: true)]
private $uid;
```

#### 5.1.2 共享访问路由

[EntryController.php:L586-L599](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php#L586-L599)：

```php
#[Route(path: '/share/{uid}', name: 'share_entry', methods: ['GET'])]
#[Cache(maxage: 25200, smaxage: 25200, public: true)]
#[IsGranted('PUBLIC_ACCESS')]
public function shareEntryAction(Entry $entry, Config $craueConfig)
{
    if (!$craueConfig->get('share_public')) {
        throw $this->createAccessDeniedException('Sharing an entry is disabled for this user.');
    }
    // ...
}
```

**权限路径**：
1. `security.yml` 中 `/share` 路径允许匿名访问（[security.yml:L75](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/app/config/security.yml#L75)）
2. 方法级 `#[IsGranted('PUBLIC_ACCESS')]` —— Symfony 内置属性，不经过 EntryVoter
3. 全局配置校验：`share_public` 开关（实例级总开关）
4. Entry 通过 `uid` 由 Doctrine ParamConverter 自动查找

#### 5.1.3 共享管理（Web 端 + API 端）

- **开启共享**：需 `SHARE` 权限 → EntryVoter → 只有所有者可操作
- **关闭共享**：需 `UNSHARE` 权限 → EntryVoter → 只有所有者可操作
- **API 控制**：创建/更新时传 `public=1` 或 `public=0`

### 5.2 Feed Token 机制

#### 5.2.1 feedToken 字段

用户的 `Config` 实体中存储 `feedToken`（[Config.php:L52-L54](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Entity/Config.php#L52-L54)）：

```php
#[ORM\Column(name: 'feed_token', type: 'string', nullable: true)]
private $feedToken;
```

**初始状态**：用户注册时**不自动生成** feedToken，默认为 `null`。
- 证据：[CreateConfigListener.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Event/Listener/CreateConfigListener.php) 的 `createConfig` 方法中没有设置 feedToken

#### 5.2.2 feedToken 的生成与撤销

用户在设置页面主动管理：

- **生成** `generateTokenAction`（[ConfigController.php:L431-L451](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/ConfigController.php#L431-L451)）：
  - 权限：`EDIT_CONFIG` → MainVoter → `ROLE_USER`
  - 算法：`Utils::generateToken()` → 15 位 base64（去掉 `+` 和 `/`）

- **撤销** `revokeTokenAction`（[ConfigController.php:L456-L476](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/ConfigController.php#L456-L476)）：
  - 权限：`EDIT_CONFIG`
  - 操作：`$config->setFeedToken(null)`

**生成算法** [Utils.php:L14-L20](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Tools/Utils.php#L14-L20)：

```php
public static function generateToken($length = 15)
{
    $token = substr(base64_encode(random_bytes($length)), 0, $length);
    return str_replace(['+', '/'], '', $token);
}
```

使用 `random_bytes()` —— **加密安全**的随机数。

#### 5.2.3 ParamConverter 校验

[UsernameFeedTokenConverter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/ParamConverter/UsernameFeedTokenConverter.php) 是自定义参数转换器：

```php
public function apply(Request $request, ParamConverter $configuration): bool
{
    $username = $request->attributes->get('username');
    $feedToken = $request->attributes->get('token');
    // ...
    $user = $userRepository->findOneByUsernameAndFeedtoken($username, $feedToken);
    // 查不到则抛 NotFoundHttpException
}
```

查询逻辑在 [UserRepository.php:L28-L36](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Repository/UserRepository.php#L28-L36)：
- 用 `username` + `feedToken` 联合查询用户
- 查不到则抛 `NotFoundHttpException`（注意：是 404，不是 403）

#### 5.2.4 使用场景

[FeedController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/FeedController.php) 中的所有 feed 路由：

```php
#[Route(path: '/feed/{username}/{token}/unread/{page}', name: 'unread_feed')]
#[IsGranted('PUBLIC_ACCESS')]
#[ParamConverter('user', class: User::class, converter: 'username_feed_token_converter')]
public function showUnreadFeedAction(User $user, $page)
```

**权限路径**：
1. `security.yml` 中 `/feed` 路径为 `PUBLIC_ACCESS`（[security.yml:L73](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/app/config/security.yml#L73)）
2. ParamConverter 在校验 `username + feedToken` 时完成身份验证
3. 方法级 `#[IsGranted('PUBLIC_ACCESS')]` 放行
4. **完全绕过 EntryVoter**，直接查询该用户的条目列表

---

## 六、公开 uid 访问与 feedToken 校验的边界情况

### 6.1 uid 共享访问的边界场景

| 场景 | 行为 | 状态码 / 结果 |
|------|------|--------------|
| uid 在数据库中存在 + share_public=1 | 正常显示共享页面 | 200 OK |
| uid 在数据库中存在 + share_public=0 | 抛 `AccessDeniedException` | 403 Forbidden |
| uid 不存在 | Doctrine ParamConverter 找不到 → 404 | 404 Not Found |
| uid 为 null（未共享）的条目 | 同上，按 uid 查不到 | 404 Not Found |
| 已登录用户访问共享页面 | 正常通过（PUBLIC_ACCESS 对所有人有效） | 200 OK |
| 管理员访问他人的共享页面 | 同上，正常通过 | 200 OK |
| 条目被删除后 | uid 随之消失，访问不到 | 404 Not Found |
| 多次点击"生成共享" | 幂等，uid 不变 | 200 / 302 |

**关于 share_public 的层级**：
- `share_public` 是**实例级**全局开关，存储在 `craue_config_setting` 表中
- 不是用户级的，对所有用户统一生效
- 管理员可在后台设置中开启/关闭

### 6.2 feedToken 校验的边界场景

| 场景 | 行为 | 状态码 / 结果 |
|------|------|--------------|
| username 存在 + feedToken 匹配 | ParamConverter 返回 User 对象 | 正常访问 |
| username 不存在 | 查不到用户 → NotFoundHttpException | 404 Not Found |
| username 存在但 feedToken 为 null | 查不到（= null 不匹配任何值）| 404 Not Found |
| feedToken 错误 | 查不到用户 | 404 Not Found |
| token 泄露 | 可访问该用户的所有条目列表（只读） | 安全风险 |
| 撤销 token 后立即访问 | 旧 token 失效 | 404 Not Found |
| 重新生成 token | 旧 token 失效，新 token 生效 | / |

**feedToken 的权限范围**：
- ✅ 可访问：条目列表（unread/starred/archive/all/tags）
- ❌ 不可访问：创建、修改、删除条目
- ❌ 不可访问：用户设置、标签管理等
- 本质：**只读的列表访问权限**，绕过 EntryVoter

### 6.3 两种共享机制的风险对比

| 风险维度 | Entry.uid 共享 | Config.feedToken 共享 |
|---------|---------------|---------------------|
| 泄露影响范围 | 单篇文章 | 该用户**所有**条目列表 |
| 可执行操作 | 查看文章内容 | 查看条目元数据列表（标题、URL、状态等） |
| 实例级开关 | `share_public` 全局开关 | 无（用户自主管理） |
| 生成算法 | `uniqid()`（非加密安全） | `random_bytes()`（加密安全） |
| 能否撤销 | 能（cleanUid） | 能（revoke / regenerate） |
| 匿名访问 | 是 | 是 |
| 绕过 EntryVoter | 是 | 是 |

---

## 七、三者关系全景图

```
┌───────────────────────────────────────────────────────────────────┐
│                        访问请求                                    │
└─────────┬───────────────────────────────┬─────────────────────────┘
          │                               │
          ▼                               ▼
┌─────────────────────┐         ┌─────────────────────┐
│  登录用户操作        │         │  匿名共享访问        │
│  (普通路由/API)      │         │  (/share, /feed)    │
└─────────┬───────────┘         └─────────┬───────────┘
          │                               │
          ▼                               ▼
┌─────────────────────┐         ┌─────────────────────┐
│  投票器链            │         │  ParamConverter     │
│  (affirmative)      │         │  或 PUBLIC_ACCESS   │
└─────┬─────────┬─────┘         └───────┬─────────────┘
      │         │                       │
      ▼         ▼                       ▼
┌─────────┐ ┌─────────┐           ┌─────────┐
│EntryVoter│ │MainVoter│           │  旁路校验  │
│  归属判定 │ │ 角色判定 │           │           │
└────┬────┘ └────┬────┘           ┌───┴───────┐
     │           │                ▼           ▼
     ▼           ▼           ┌─────────┐ ┌─────────┐
  owner?     ROLE_USER?      │uid 校验 │ │feedToken│
  (全有或全无)  (全局操作)    │(单条目) │ │(用户级) │
                             └─────────┘ └─────────┘
```

---

## 八、容易混淆的关键点

### 8.1 SHARE 权限 ≠ 共享访问权限

| 概念 | 权限属性 | 经过 Voter | 使用者 |
|------|---------|-----------|--------|
| 管理共享状态 | `SHARE` / `UNSHARE` | ✅ EntryVoter | 条目所有者 |
| 访问共享内容 | `PUBLIC_ACCESS` | ❌ 不经过 | 匿名用户 |

`SHARE` 是"是否可以生成/删除共享链接"的权限，归 EntryVoter 管；而实际通过共享链接查看内容走的是独立的旁路，与 EntryVoter 无关。

### 8.2 两种 Token 的本质区别

| 维度 | Entry.uid | Config.feedToken |
|------|-----------|-----------------|
| 级别 | 单条目 | 用户级 |
| 用途 | 公开分享单篇文章 | RSS/Atom Feed 订阅 |
| 校验方式 | Doctrine ParamConverter 按 uid 查找 | 自定义 ParamConverter 联合查询 |
| 权限属性 | `PUBLIC_ACCESS` | `PUBLIC_ACCESS` |
| 绕过 EntryVoter | ✅ 是 | ✅ 是 |
| 全局开关 | `share_public` | 无（由用户自行保管 token） |
| 生成算法 | `uniqid('', true)` | `random_bytes()` + base64 |

### 8.3 归属判定的"全有或全无"特性

EntryVoter 对所有 14 种操作使用完全相同的判定逻辑——只要你是所有者，就可以执行任何操作；不是所有者，就什么都不能做。没有更细粒度的权限区分（比如"可以看但不能改"）。

### 8.4 两条独立的查看路径

查看同一条 Entry 有两条完全不同的安全路径：

1. **`/view/{id}`** → `IsGranted('VIEW')` → EntryVoter 归属判定 → 仅所有者
2. **`/share/{uid}`** → `IsGranted('PUBLIC_ACCESS')` → uid 存在性 + 全局开关 → 匿名可访问

这两条路径在代码中没有交叉，也没有共享权限逻辑。

### 8.5 CREATE_ENTRIES 与 EDIT 的分工

- `CREATE_ENTRIES`（MainVoter）：回答"**能不能创建条目**"，基于角色
- `EDIT`（EntryVoter）：回答"**能不能修改这条特定的条目**"，基于归属
- 创建接口的 upsert 语义不会导致越权，因为查询时限定了 `userId = 当前用户`

---

## 九、安全边界总结

| 场景 | 判定主体 | 判定逻辑 | 结果 |
|------|---------|---------|------|
| 所有者查看/编辑自己的条目 | EntryVoter | `$user === $entry->getUser()` | 允许 |
| 非所有者尝试查看他人条目 | EntryVoter | `$user !== $entry->getUser()` | 拒绝 |
| 管理员操作他人条目 | EntryVoter | `$user !== $entry->getUser()` | 拒绝（即使是管理员） |
| 登录用户创建新条目 | MainVoter | `ROLE_USER` 角色 | 允许 |
| POST /api/entries 更新已存在的自己的条目 | MainVoter + 查询限定 | `ROLE_USER` + 只能查到自己的 | 允许（安全） |
| 匿名用户访问共享链接（uid 有效 + 开关开） | 无（PUBLIC_ACCESS） | uid 存在 + share_public 开启 | 允许 |
| 匿名用户访问共享链接（开关关） | 业务逻辑 | share_public = 0 | 拒绝（403） |
| 访问不存在的 uid | ParamConverter | 数据库查不到 | 拒绝（404） |
| 所有者开启/关闭共享 | EntryVoter | 必须是所有者 | 允许/拒绝 |
| API 设置 public=1 生成 uid | EntryVoter（EDIT） | 必须是所有者 + uid 为空才生成 | 允许 |
| API 设置 public=0 清理 uid | EntryVoter（EDIT） | 必须是所有者 | 允许 |
| 通过 feedToken 获取列表 | UsernameFeedTokenConverter | username + feedToken 匹配 | 允许（只读列表） |
| feedToken 错误或已撤销 | UsernameFeedTokenConverter | 查不到用户 | 拒绝（404） |
| feedToken 为 null（未生成） | UsernameFeedTokenConverter | 查不到用户 | 拒绝（404） |

> **特别注意**：管理员（ROLE_ADMIN、ROLE_SUPER_ADMIN）也无法通过 EntryVoter 操作他人条目。EntryVoter 只认"所有者"这一个条件，不考虑角色层级。
