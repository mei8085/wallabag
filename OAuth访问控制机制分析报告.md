# Wallabag OAuth 令牌访问控制机制深度分析

## 一、整体架构概览

Wallabag 采用 **Symfony Security 组件 + FOSOAuthServerBundle** 实现 OAuth2 认证，并通过 **Voter 投票者机制 + Repository 数据层过滤** 构建双层访问控制体系，确保调用方只能访问其授权范围内的条目资源。

### 访问控制链路图

```
HTTP 请求
    │
    ▼
┌─────────────────────────────────┐
│  API 防火墙 (security.yml)      │
│  fos_oauth: true                │
│  → 验证 OAuth Access Token      │
│  → 加载用户身份到 Security Token│
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  #[IsGranted] 注解 (控制器层)    │
│  → 触发 Voter 投票机制           │
└─────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────────────┐
│  Symfony Voter 投票链                           │
│  ├─ MainVoter (无subject: LIST_ENTRIES 等)    │
│  └─ EntryVoter (有Entry subject: VIEW/EDIT 等) │
└───────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  Repository 数据层过滤           │
│  → 强制 WHERE user_id = 当前用户│
└─────────────────────────────────┘
```

---

## 二、OAuth 令牌颁发机制

### 2.1 OAuth 服务端配置

配置位置：[config.yml#L203-L213](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/app/config/config.yml#L203-L213)

```yaml
fos_oauth_server:
    db_driver:           orm
    client_class:        Wallabag\Entity\Api\Client
    access_token_class:  Wallabag\Entity\Api\AccessToken
    refresh_token_class: Wallabag\Entity\Api\RefreshToken
    auth_code_class:     Wallabag\Entity\Api\AuthCode
    service:
        user_provider: fos_user.user_provider.username_email
        options:
            refresh_token_lifetime: "%env(int:WALLABAG_OAUTH_REFRESH_TOKEN_LIFETIME)%"
            access_token_lifetime: "%env(int:WALLABAG_OAUTH_ACCESS_TOKEN_LIFETIME)%"
```

**关键特性：**
- 使用 FOSOAuthServerBundle 作为 OAuth2 服务端实现
- 令牌存储在数据库中（ORM 驱动）
- 令牌与用户（User）和客户端（Client）双向关联

### 2.2 Access Token 实体结构

实体位置：[AccessToken.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Entity/Api/AccessToken.php)

```php
class AccessToken extends BaseAccessToken
{
    #[ORM\Id]
    #[ORM\Column(type: 'integer')]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    protected $id;

    #[ORM\JoinColumn(nullable: false)]
    #[ORM\ManyToOne(targetEntity: Client::class, inversedBy: 'accessTokens')]
    protected $client;          // 关联 OAuth 客户端

    #[ORM\JoinColumn(name: 'user_id', referencedColumnName: 'id', onDelete: 'CASCADE')]
    #[ORM\ManyToOne(targetEntity: User::class)]
    protected $user;            // 关联资源所有者（用户）
}
```

**核心要点：**
- 每个 AccessToken 通过 `user_id` 外键**严格绑定到一个用户**
- 这是后续访问控制的身份基础——令牌即代表特定用户

### 2.3 Scope 机制分析

数据库层面存在 scope 字段（见 [Version20160401000000.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/migrations/Version20160401000000.php#L38-L46)）：

```sql
CREATE TABLE oauth2_access_tokens (
    ...
    scope VARCHAR(255) DEFAULT NULL,
    ...
);
```

**但是，代码分析表明 Wallabag 并未实际使用 OAuth scope 做细粒度权限控制：**
- `config.yml` 中未配置 `supported_scopes`
- 所有 Voter 和控制器均未检查令牌的 scope 字段
- 权限判断完全基于 **用户身份（user_id）+ 角色（ROLE_USER）**，而非 scope

---

## 三、API 防火墙与认证层

### 3.1 防火墙配置

配置位置：[security.yml#L29-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/app/config/security.yml#L29-L34)

```yaml
firewalls:
    api:
        pattern: /api/.*
        fos_oauth: true          # 启用 FOS OAuth 认证
        stateless: true          # 无状态，不使用 Session
        anonymous: true          # 允许匿名访问部分接口
        provider: fos_userbundle
```

**作用：**
- 匹配所有 `/api/.*` 路径的请求
- `fos_oauth: true` 触发 FOSOAuthServerBundle 的认证监听器
- 从 `Authorization: Bearer {token}` 头中解析并验证 Access Token
- 验证通过后将对应用户加载到 Symfony Security Token 中

### 3.2 访问控制（access_control）

配置位置：[security.yml#L62-L78](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/app/config/security.yml#L62-L78)

```yaml
access_control:
    - { path: ^/api/(doc|version|info|user), roles: IS_AUTHENTICATED_ANONYMOUSLY }
    # ... 其他公开路径 ...
    - { path: ^/, roles: ROLE_USER }
```

**否决点 #1 —— 防火墙级别的认证否决：**
- 未携带有效 OAuth 令牌的请求访问受保护路径时，在此层被直接拒绝
- 返回 401 Unauthorized

---

## 四、Voter 投票机制——越权否决的核心

Wallabag 使用 Symfony Voter 实现基于属性的访问控制（ABAC），这是**越权请求被否决的核心位置**。

### 4.1 MainVoter：集合级操作权限判断

文件位置：[MainVoter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/MainVoter.php)

**支持的权限属性（无 Subject）：**

```php
public const LIST_ENTRIES = 'LIST_ENTRIES';
public const CREATE_ENTRIES = 'CREATE_ENTRIES';
public const EDIT_ENTRIES = 'EDIT_ENTRIES';
public const EXPORT_ENTRIES = 'EXPORT_ENTRIES';
public const IMPORT_ENTRIES = 'IMPORT_ENTRIES';
public const DELETE_ENTRIES = 'DELETE_ENTRIES';
public const LIST_TAGS = 'LIST_TAGS';
public const CREATE_TAGS = 'CREATE_TAGS';
public const DELETE_TAGS = 'DELETE_TAGS';
// ...
```

**投票逻辑（[voteOnAttribute#L42-L48](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/MainVoter.php#L42-L48)）：**

```php
protected function voteOnAttribute(string $attribute, $subject, TokenInterface $token): bool
{
    return match ($attribute) {
        self::LIST_ENTRIES, self::CREATE_ENTRIES, self::EDIT_ENTRIES, 
        self::EXPORT_ENTRIES, self::IMPORT_ENTRIES, self::DELETE_ENTRIES,
        self::LIST_TAGS, self::CREATE_TAGS, self::DELETE_TAGS,
        // ...
        => $this->security->isGranted('ROLE_USER'),
        default => false,
    };
}
```

**判断规则：**
- 只要用户拥有 `ROLE_USER` 角色即授权通过
- 这是**粗粒度**的入口级权限检查

### 4.2 EntryVoter：单条目级操作权限判断

文件位置：[EntryVoter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/EntryVoter.php)

**支持的权限属性（需 Subject = Entry 实例）：**

```php
public const VIEW = 'VIEW';
public const EDIT = 'EDIT';
public const RELOAD = 'RELOAD';
public const STAR = 'STAR';
public const ARCHIVE = 'ARCHIVE';
public const SHARE = 'SHARE';
public const UNSHARE = 'UNSHARE';
public const EXPORT = 'EXPORT';
public const DELETE = 'DELETE';
public const LIST_ANNOTATIONS = 'LIST_ANNOTATIONS';
public const CREATE_ANNOTATIONS = 'CREATE_ANNOTATIONS';
public const LIST_TAGS = 'LIST_TAGS';
public const TAG = 'TAG';
public const UNTAG = 'UNTAG';
```

**核心投票逻辑（[voteOnAttribute#L40-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/EntryVoter.php#L40-L54)）：**

```php
protected function voteOnAttribute(string $attribute, $subject, TokenInterface $token): bool
{
    \assert($subject instanceof Entry);

    $user = $token->getUser();

    if (!$user instanceof User) {
        return false;       // 未认证用户 → 拒绝
    }

    return match ($attribute) {
        self::VIEW, self::EDIT, self::RELOAD, self::STAR, self::ARCHIVE,
        self::SHARE, self::UNSHARE, self::EXPORT, self::DELETE,
        self::LIST_ANNOTATIONS, self::CREATE_ANNOTATIONS,
        self::LIST_TAGS, self::TAG, self::UNTAG
            => $user === $subject->getUser(),  // ★ 关键：用户必须是条目的所有者
        default => false,
    };
}
```

**否决点 #2 —— 条目所有权检查：**
- **`$user === $subject->getUser()`** 这行代码是越权否决的关键
- 比较当前认证用户与条目的所属用户是否为同一对象
- 若不匹配（即越权访问他人条目），返回 `false`，Voter 投出 `ACCESS_DENIED`
- 效果：用户只能查看/编辑/删除自己的条目

**单元测试验证**（[EntryVoterTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/tests/unit/Security/Voter/EntryVoterTest.php)）：

```php
// 非条目的所属用户访问 → 返回 ACCESS_DENIED
public function testVoteReturnsDeniedForNonEntryUserView(): void
{
    $this->token->method('getUser')->willReturn(new User());  // 不同用户对象
    $this->assertSame(VoterInterface::ACCESS_DENIED, 
        $this->entryVoter->vote($this->token, $this->entry, [EntryVoter::VIEW]));
}
```

---

## 五、控制器层：#[IsGranted] 注解的应用

文件位置：[EntryRestController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php)

### 5.1 集合级操作（无 Subject）

使用 `MainVoter` 判断：

| 方法 | 路由 | #[IsGranted] | Voter |
|------|------|--------------|-------|
| `getEntriesExistsAction` | `GET /api/entries/exists` | `#[IsGranted('LIST_ENTRIES')]` | MainVoter |
| `getEntriesAction` | `GET /api/entries` | `#[IsGranted('LIST_ENTRIES')]` | MainVoter |
| `postEntriesAction` | `POST /api/entries` | `#[IsGranted('CREATE_ENTRIES')]` | MainVoter |
| `deleteEntriesListAction` | `DELETE /api/entries/list` | `#[IsGranted('DELETE_ENTRIES')]` | MainVoter |
| `postEntriesListAction` | `POST /api/entries/lists` | `#[IsGranted('CREATE_ENTRIES')]` | MainVoter |

示例（[EntryRestController.php#L312-L314](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php#L312-L314)）：
```php
#[Route(path: '/api/entries.{_format}', name: 'api_get_entries', methods: ['GET'])]
#[IsGranted('LIST_ENTRIES')]
public function getEntriesAction(Request $request, EntryRepository $entryRepository)
```

### 5.2 单条目操作（带 Entry Subject）

使用 `EntryVoter` 判断所有权：

| 方法 | 路由 | #[IsGranted] | Voter | Subject |
|------|------|--------------|-------|---------|
| `getEntryAction` | `GET /api/entries/{entry}` | `#[IsGranted('VIEW', subject: 'entry')]` | EntryVoter | Entry |
| `patchEntriesAction` | `PATCH /api/entries/{entry}` | `#[IsGranted('EDIT', subject: 'entry')]` | EntryVoter | Entry |
| `deleteEntriesAction` | `DELETE /api/entries/{entry}` | `#[IsGranted('DELETE', subject: 'entry')]` | EntryVoter | Entry |
| `patchEntriesReloadAction` | `PATCH /api/entries/{entry}/reload` | `#[IsGranted('RELOAD', subject: 'entry')]` | EntryVoter | Entry |
| `getEntriesTagsAction` | `GET /api/entries/{entry}/tags` | `#[IsGranted('LIST_TAGS', subject: 'entry')]` | EntryVoter | Entry |
| `postEntriesTagsAction` | `POST /api/entries/{entry}/tags` | `#[IsGranted('TAG', subject: 'entry')]` | EntryVoter | Entry |
| `deleteEntriesTagsAction` | `DELETE /api/entries/{entry}/tags/{tag}` | `#[IsGranted('UNTAG', subject: 'entry')]` | EntryVoter | Entry |

示例（[EntryRestController.php#L405-L407](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php#L405-L407)）：
```php
#[Route(path: '/api/entries/{entry}.{_format}', name: 'api_get_entry', methods: ['GET'])]
#[IsGranted('VIEW', subject: 'entry')]
public function getEntryAction(Entry $entry)
```

**工作流程：**
1. Symfony ParamConverter 根据路由参数 `{entry}` 自动从数据库加载 Entry 实体
2. `#[IsGranted]` 注解在控制器方法执行前触发安全检查
3. Security 组件调用 EntryVoter 进行投票
4. 若投票结果为 ACCESS_DENIED，抛出 `AccessDeniedException`，返回 403

### 5.3 运行时二次权限检查

除了注解检查外，部分方法在代码逻辑中进行二次检查，例如批量操作：

[EntryRestController.php#L499](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php#L499)
```php
if (false !== $entry && $this->authorizationChecker->isGranted('DELETE', $entry)) {
    // 执行删除操作
}
```

[EntryRestController.php#L1304](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php#L1304)
```php
if (false !== $entry && !(empty($tags)) && $this->authorizationChecker->isGranted('UNTAG', $entry)) {
    // 执行移除标签操作
}
```

**否决点 #3 —— 运行时二次检查：**
- 在批量操作循环中对每个条目单独调用 `isGranted()`
- 跳过不属于当前用户的条目，避免批量越权

---

## 六、Repository 数据层：强制用户过滤

文件位置：[EntryRepository.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Repository/EntryRepository.php)

这是**最后一道防线**，即使上层 Voter 被绕过，Repository 层也会强制只返回当前用户的数据。

### 6.1 私有基础查询构造器

[EntryRepository.php#L753-L757](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Repository/EntryRepository.php#L753-L757)

```php
private function getQueryBuilderByUser($userId)
{
    return $this->createQueryBuilder('e')
        ->andWhere('e.user = :userId')->setParameter('userId', $userId);
}
```

**所有查询方法均基于此构造器**，确保 WHERE 条件中始终包含 `e.user = :userId`。

### 6.2 典型查询示例

**查询条目列表**（[EntryRepository.php#L282-L291](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Repository/EntryRepository.php#L282-L291)）：
```php
public function findEntries($userId, ...)
{
    $qb = $this->createQueryBuilder('e')
        ->leftJoin('e.tags', 't')
        ->where('e.user = :userId')->setParameter('userId', $userId);  // 强制过滤
    // ...
}
```

**按 URL 查询**（[EntryRepository.php#L535-L560](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Repository/EntryRepository.php#L535-L560)）：
```php
public function findByHashedUrlAndUserId($hashedUrl, $userId)
{
    $res = $this->createQueryBuilder('e')
        ->where('e.hashedUrl = :hashed_url')->setParameter('hashed_url', $hashedUrl)
        ->andWhere('e.user = :user_id')->setParameter('user_id', $userId)  // 强制过滤
        ->getQuery()
        ->getResult();
    // ...
}
```

**控制器调用方式**（[EntryRestController.php#L333-L334](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php#L333-L334)）：
```php
$pager = $entryRepository->findEntries(
    $this->getUser()->getId(),  // 传入当前认证用户的 ID
    // ...
);
```

**否决点 #4 —— 数据层过滤：**
- 即使攻击者设法绕过 Voter 检查，Repository 查询也不会返回其他用户的数据
- 所有查询方法要求显式传入 `$userId`，避免上下文泄漏

---

## 七、四层越权否决机制总结

| 否决层级 | 位置 | 否决条件 | 效果 |
|---------|------|---------|------|
| **第1层：防火墙** | [security.yml#L29-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/app/config/security.yml#L29-L34) | 无有效 OAuth Bearer Token | 401 Unauthorized |
| **第2层：Voter 注解检查** | `#[IsGranted]` + [EntryVoter.php#L51](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/EntryVoter.php#L51) | 当前用户 ≠ 条目所属用户 | 403 Forbidden |
| **第3层：运行时二次检查** | [EntryRestController.php#L499](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php#L499) | 批量操作中逐条检查所有权 | 跳过未授权条目 |
| **第4层：Repository 强制过滤** | [EntryRepository.php#L753-L757](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Repository/EntryRepository.php#L753-L757) | 所有查询强制 `WHERE user_id = ?` | 空结果集 / 仅返回本人数据 |

---

## 八、设计亮点与潜在风险

### 8.1 设计亮点

1. **纵深防御**：四层否决机制，层层递进，即使单层被绕过也不会导致越权
2. **所有权优先**：EntryVoter 中所有条目操作权限统一归结为「用户是否为所有者」，逻辑简单一致
3. **无状态认证**：API 防火墙使用 `stateless: true`，避免 Session 劫持风险
4. **Repository 封装**：所有数据查询通过 `getQueryBuilderByUser()` 统一添加用户过滤，难以遗漏

### 8.2 潜在风险与改进空间

1. **OAuth Scope 未实际使用**：数据库有 scope 字段但代码未利用，无法对不同客户端授予不同 API 权限（如只读客户端）
2. **缺少 Rate Limiting**：配置中未见针对 OAuth 令牌的速率限制配置
3. **缺少审计日志**：越权尝试未被记录审计日志（仅标准访问日志）
4. **ParamConverter 加载顺序**：Entry 实体在 `#[IsGranted]` 之前已被加载，存在潜在的信息泄露风险（虽然通过 Voter 阻止了后续操作）

---

## 九、调用方访问范围约束方式总结

Wallabag 通过以下方式约束 OAuth 调用方对条目接口的访问范围：

1. **令牌-用户绑定**：每个 Access Token 绑定唯一用户，调用方的身份等同于令牌所属用户
2. **角色约束**：必须拥有 `ROLE_USER` 角色才能调用条目接口
3. **所有权约束**：单条目操作（VIEW/EDIT/DELETE 等）通过 EntryVoter 严格校验 `当前用户 === 条目用户`
4. **数据隔离**：Repository 层所有查询强制附加 `user_id = 当前用户` 条件，实现行级数据隔离

**最终结论：** 越权请求在 EntryVoter 的 `voteOnAttribute()` 方法中通过 `$user === $subject->getUser()` 判断被否决（返回 ACCESS_DENIED），并由 Repository 层的强制用户过滤作为兜底保障。
