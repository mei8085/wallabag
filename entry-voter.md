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

## 三、三条 API 路径的权限关系分析

### 3.1 三条路径总览

Entry API 有三条核心路径，分别对应不同的权限属性和投票器：

| 路径 | 路由 | 权限属性 | 所属 Voter | subject | 是否修改 uid |
|------|------|---------|-----------|---------|-------------|
| 列表查询 | `GET /api/entries` | `LIST_ENTRIES` | MainVoter | 无 | ❌ 只读 |
| 创建/更新（按 URL） | `POST /api/entries` | `CREATE_ENTRIES` | MainVoter | 无 | ✅ 通过 `public` 参数 |
| 更新（按 ID） | `PATCH /api/entries/{entry}` | `EDIT` | EntryVoter | Entry 实例 | ✅ 通过 `public` 参数 |

**关键区分**：
- `LIST_ENTRIES` / `CREATE_ENTRIES` 走 **MainVoter**（无 subject，基于角色）
- `EDIT` 走 **EntryVoter**（有 Entry subject，基于归属判定）

---

### 3.2 GET /api/entries — LIST_ENTRIES 路径

**代码位置**：[EntryRestController.php:L312-L379](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L312-L379)

```php
#[Route(path: '/api/entries.{_format}', name: 'api_get_entries', methods: ['GET'])]
#[IsGranted('LIST_ENTRIES')]
public function getEntriesAction(Request $request, EntryRepository $entryRepository)
```

**权限路径**：
1. `#[IsGranted('LIST_ENTRIES')]` → MainVoter → 检查 `ROLE_USER` 角色
2. 业务逻辑：`$entryRepository->findEntries($this->getUser()->getId(), ...)`
3. 查询时限定 `userId = 当前用户`，天然数据隔离

**与 public/uid 的关系**：
- 请求参数 `public` 仅作为**过滤条件**（[L318](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L318)）
- 不修改 uid，只用于筛选"已公开/未公开"的条目
- 因为查询已限定 userId，所以只能看到自己条目的公开状态

---

### 3.3 POST /api/entries — CREATE_ENTRIES 路径

**代码位置**：[EntryRestController.php:L715-L811](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L715-L811)

```php
#[Route(path: '/api/entries.{_format}', name: 'api_post_entries', methods: ['POST'])]
#[IsGranted('CREATE_ENTRIES')]
public function postEntriesAction(...)
```

#### 3.3.1 权限路径

1. `#[IsGranted('CREATE_ENTRIES')]` → MainVoter → 检查 `ROLE_USER` 角色
2. **没有**使用 `EDIT` 或 `SHARE` 权限
3. 业务层通过 `findByUrlAndUserId($url, $this->getUser()->getId())` 确保数据隔离（[L728-L731](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L728-L731)）

#### 3.3.2 Upsert 语义

```php
$entry = $entryRepository->findByUrlAndUserId($url, $this->getUser()->getId());

if (false === $entry) {
    $entry = new Entry($this->getUser());
    $entry->setUrl($url);
}
```

- 按 URL + 用户 ID 查找已存在的条目
- 存在则更新，不存在则创建
- 因为限定了 userId，所以只会更新自己的条目，不会越权

#### 3.3.3 public/uid 修改逻辑

[EntryRestController.php:L778-L784](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L778-L784)：

```php
if (null !== $data['isPublic']) {
    if (true === (bool) $data['isPublic'] && null === $entry->getUid()) {
        $entry->generateUid();
    } elseif (false === (bool) $data['isPublic']) {
        $entry->cleanUid();
    }
}
```

| 输入 `isPublic` | 当前 uid 状态 | 结果 |
|----------------|-------------|------|
| `true` / `1` | 为 null | 生成新 uid |
| `true` / `1` | 已存在 | **不重新生成**，保持原值 |
| `false` / `0` | 任何状态 | 清空 uid |
| `null` / 不传 | 任何状态 | **不做任何修改** |

---

### 3.4 PATCH /api/entries/{entry} — EDIT 路径

**代码位置**：[EntryRestController.php:L938-L1025](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L938-L1025)

```php
#[Route(path: '/api/entries/{entry}.{_format}', name: 'api_patch_entries', methods: ['PATCH'])]
#[IsGranted('EDIT', subject: 'entry')]
public function patchEntriesAction(Entry $entry, ...)
```

#### 3.4.1 权限路径

1. `{entry}` 参数由 Doctrine ParamConverter 按 ID 自动查找 Entry 实例
2. `#[IsGranted('EDIT', subject: 'entry')]` → EntryVoter → 归属判定
3. 必须是条目所有者才能通过，否则直接 403

#### 3.4.2 public/uid 修改逻辑

[EntryRestController.php:L998-L1004](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/Api/EntryRestController.php#L998-L1004)：

```php
if (null !== $data['isPublic']) {
    if (true === (bool) $data['isPublic'] && null === $entry->getUid()) {
        $entry->generateUid();
    } elseif (false === (bool) $data['isPublic']) {
        $entry->cleanUid();
    }
}
```

**代码与 POST 路径完全相同**，行为矩阵也一致。

---

### 3.5 CREATE_ENTRIES 路径 vs EDIT 路径对比

| 维度 | POST /api/entries（CREATE_ENTRIES） | PATCH /api/entries/{entry}（EDIT） |
|------|-------------------------------------|------------------------------------|
| 权限属性 | `CREATE_ENTRIES` | `EDIT` |
| 所属 Voter | MainVoter | EntryVoter |
| subject | 无 | Entry 实例 |
| 判定方式 | `ROLE_USER` 角色检查 | 归属判定（`$user === $entry->getUser()`） |
| 标识条目方式 | URL + userId（业务层查询） | 路径参数 `{entry}`（ParamConverter） |
| 数据隔离方式 | 业务层查询限定 userId | 投票器层归属校验 |
| 是否修改 uid | 是（通过 `public` 参数） | 是（通过 `public` 参数） |
| uid 修改逻辑 | 完全相同 | 完全相同 |
| 能否操作他人条目 | 不能（查询限定了自己） | 不能（EntryVoter 拒绝） |

**两条路径的安全效果等价**：都只能操作自己的条目。但实现方式不同：
- POST 路径靠**业务层查询过滤**（隐性安全）
- PATCH 路径靠**投票器层归属判定**（显性安全）

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

### 4.4 uid 生成算法

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

- `LIST_ENTRIES` / `CREATE_ENTRIES` 等 `*_ENTRIES`（MainVoter）：回答"**能不能做某类操作**"，基于角色，无 subject
- `EDIT` / `VIEW` / `DELETE` 等（EntryVoter）：回答"**能不能操作这条特定的条目**"，基于归属，有 Entry subject
- POST /api/entries（CREATE_ENTRIES）的 upsert 语义不会导致越权，因为查询时限定了 `userId = 当前用户`
- PATCH /api/entries/{entry}（EDIT）走 EntryVoter 归属判定，天然只有所有者能操作
- 两条路径都支持 `public` 参数控制 uid，但权限路径完全不同

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
| API POST 设置 public=1 生成 uid | MainVoter + 业务层查询 | `CREATE_ENTRIES` 权限 + `findByUrlAndUserId` 限定自己的条目 | 允许 |
| API POST 设置 public=0 清理 uid | MainVoter + 业务层查询 | `CREATE_ENTRIES` 权限 + `findByUrlAndUserId` 限定自己的条目 | 允许 |
| API PATCH 设置 public=1 生成 uid | EntryVoter（EDIT） | 必须是所有者 + uid 为空才生成 | 允许 |
| API PATCH 设置 public=0 清理 uid | EntryVoter（EDIT） | 必须是所有者 | 允许 |
| 通过 feedToken 获取列表 | UsernameFeedTokenConverter | username + feedToken 匹配 | 允许（只读列表） |
| feedToken 错误或已撤销 | UsernameFeedTokenConverter | 查不到用户 | 拒绝（404） |
| feedToken 为 null（未生成） | UsernameFeedTokenConverter | 查不到用户 | 拒绝（404） |

> **特别注意**：管理员（ROLE_ADMIN、ROLE_SUPER_ADMIN）也无法通过 EntryVoter 操作他人条目。EntryVoter 只认"所有者"这一个条件，不考虑角色层级。
