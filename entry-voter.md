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

| 投票器类 | 文件路径 | 管辖范围 |
|---------|---------|---------|
| EntryVoter | [EntryVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/EntryVoter.php) | 单条目操作权限 |
| MainVoter | [MainVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/MainVoter.php) | 全局操作权限（列表、创建等） |
| AnnotationVoter | [AnnotationVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/AnnotationVoter.php) | 注释操作权限 |
| TagVoter | [TagVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/TagVoter.php) | 标签操作权限 |
| UserVoter | [UserVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/UserVoter.php) | 用户相关权限 |
| AdminVoter | [AdminVoter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Security/Voter/AdminVoter.php) | 管理员权限 |
| ... | ... | ... |

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

## 三、共享 Token 校验机制

共享访问**不通过 EntryVoter**，而是通过两条独立的旁路机制实现。

### 3.1 单条目共享（uid 机制）

#### 3.1.1 uid 字段

[Entry.php:L53-L55](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Entity/Entry.php#L53-L55) 定义了 `uid` 字段：

```php
#[ORM\Column(name: 'uid', type: 'string', length: 23, nullable: true)]
private $uid;
```

相关方法（[Entry.php:L767-L792](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Entity/Entry.php#L767-L792)）：
- `generateUid()`：调用 `uniqid('', true)` 生成 23 位随机 ID
- `cleanUid()`：置空 uid，取消共享
- `isPublic()`：返回 `null !== $this->uid`，即"有 uid = 已共享"

#### 3.1.2 共享访问路由

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
2. 方法级 `#[IsGranted('PUBLIC_ACCESS')]` —— 这是 Symfony 内置属性，不经过 EntryVoter
3. 全局配置校验：`share_public` 开关
4. Entry 通过 `uid` 由 Doctrine ParamConverter 自动查找

#### 3.1.3 共享管理

- **开启共享** `shareAction`（[L538-L556](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php#L538-L556)）：
  - 需 `SHARE` 权限 → 经 EntryVoter → 只有所有者可操作
  - 生成 uid 持久化后跳转到共享页面

- **关闭共享** `deleteShareAction`（[L563-L579](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Controller/EntryController.php#L563-L579)）：
  - 需 `UNSHARE` 权限 → 经 EntryVoter → 只有所有者可操作
  - 清除 uid

### 3.2 Feed Token 机制

#### 3.2.1 feedToken 字段

用户的 `Config` 实体中存储 `feedToken`（[Config.php:L52-L54](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Entity/Config.php#L52-L54)）：

```php
#[ORM\Column(name: 'feed_token', type: 'string', nullable: true)]
private $feedToken;
```

#### 3.2.2 ParamConverter 校验

[UsernameFeedTokenConverter.php](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/ParamConverter/UsernameFeedTokenConverter.php) 是自定义参数转换器：

```php
public function apply(Request $request, ParamConverter $configuration): bool
{
    $username = $request->attributes->get('username');
    $feedToken = $request->attributes->get('token');
    // ...
    $user = $userRepository->findOneByUsernameAndFeedtoken($username, $feedToken);
    // ...
}
```

查询逻辑在 [UserRepository.php:L28-L36](file:///d:/fz/0508-2/solo-dogfeeding/code/110-wallabag/src/Repository/UserRepository.php#L28-L36)：
- 用 `username` + `feedToken` 联合查询用户
- 查不到则抛 `NotFoundHttpException`

#### 3.2.3 使用场景

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

## 四、三者关系全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                    访问请求                                      │
└─────────┬───────────────────────────────────┬───────────────────┘
          │                                   │
          ▼                                   ▼
┌─────────────────────┐             ┌─────────────────────┐
│  登录用户操作        │             │  匿名共享访问        │
│  (普通路由)          │             │  (/share, /feed)    │
└─────────┬───────────┘             └─────────┬───────────┘
          │                                   │
          ▼                                   ▼
┌─────────────────────┐             ┌─────────────────────┐
│  投票器链            │             │  ParamConverter     │
│  (affirmative)      │             │  或 PUBLIC_ACCESS   │
└─────────┬───────────┘             └─────────┬───────────┘
          │                                   │
    ┌─────┴──────┐                     ┌──────┴──────┐
    ▼            ▼                     ▼             ▼
┌─────────┐ ┌─────────┐         ┌─────────┐   ┌─────────┐
│EntryVoter│ │MainVoter│         │uid 校验 │   │feedToken│
│  归属判定 │ │ 角色判定 │         │(单条目) │   │(用户级) │
└────┬────┘ └────┬────┘         └────┬────┘   └────┬────┘
     │           │                  │            │
     ▼           ▼                  ▼            ▼
  owner?     ROLE_USER?         uid 存在?   token 匹配?
  (全有或全无)  (全局操作)        + 全局开关    + 用户存在
```

---

## 五、容易混淆的关键点

### 5.1 SHARE 权限 ≠ 共享访问权限

| 概念 | 权限属性 | 经过 Voter | 使用者 |
|------|---------|-----------|--------|
| 管理共享状态 | `SHARE` / `UNSHARE` | ✅ EntryVoter | 条目所有者 |
| 访问共享内容 | `PUBLIC_ACCESS` | ❌ 不经过 | 匿名用户 |

`SHARE` 是"是否可以生成/删除共享链接"的权限，归 EntryVoter 管；而实际通过共享链接查看内容走的是独立的旁路，与 EntryVoter 无关。

### 5.2 两种 Token 的本质区别

| 维度 | Entry.uid | Config.feedToken |
|------|-----------|-----------------|
| 级别 | 单条目 | 用户级 |
| 用途 | 公开分享单篇文章 | RSS/Atom Feed 订阅 |
| 校验方式 | Doctrine ParamConverter 按 uid 查找 | 自定义 ParamConverter 联合查询 |
| 权限属性 | `PUBLIC_ACCESS` | `PUBLIC_ACCESS` |
| 绕过 EntryVoter | ✅ 是 | ✅ 是 |
| 全局开关 | `share_public` | 无（由用户自行保管 token） |

### 5.3 归属判定的"全有或全无"特性

EntryVoter 对所有 14 种操作使用完全相同的判定逻辑——只要你是所有者，就可以执行任何操作；不是所有者，就什么都不能做。没有更细粒度的权限区分（比如"可以看但不能改"）。

### 5.4 两条独立的查看路径

查看同一条 Entry 有两条完全不同的安全路径：

1. **`/view/{id}`** → `IsGranted('VIEW')` → EntryVoter 归属判定 → 仅所有者
2. **`/share/{uid}`** → `IsGranted('PUBLIC_ACCESS')` → uid 存在性 + 全局开关 → 匿名可访问

这两条路径在代码中没有交叉，也没有共享权限逻辑。

---

## 六、安全边界总结

| 场景 | 判定主体 | 判定逻辑 | 结果 |
|------|---------|---------|------|
| 所有者查看自己的条目 | EntryVoter | `$user === $entry->getUser()` | 允许 |
| 非所有者尝试查看他人条目 | EntryVoter | `$user !== $entry->getUser()` | 拒绝 |
| 匿名用户访问共享链接 | 无（PUBLIC_ACCESS） | uid 有效 + share_public 开启 | 允许 |
| 所有者开启/关闭共享 | EntryVoter | 必须是所有者 | 允许/拒绝 |
| 通过 feedToken 获取列表 | UsernameFeedTokenConverter | username + feedToken 匹配 | 允许/拒绝 |
| 管理员操作他人条目 | EntryVoter | `$user !== $entry->getUser()` | 拒绝（即使是管理员） |

> **特别注意**：管理员（ROLE_ADMIN、ROLE_SUPER_ADMIN）也无法通过 EntryVoter 操作他人条目。EntryVoter 只认"所有者"这一个条件，不考虑角色层级。
