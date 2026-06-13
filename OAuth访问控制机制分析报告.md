# Wallabag OAuth 令牌访问控制机制深度分析

## 一、整体架构概览

Wallabag 采用 **Symfony Security 组件 + FOSOAuthServerBundle** 实现 OAuth2 认证，并通过 **Voter 投票者机制 + Repository 数据层过滤 + 异常转换** 构建多层访问控制体系，确保调用方只能访问其授权范围内的条目资源。

### 完整访问控制链路图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      第一阶段：OAuth 令牌申请                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 用户登录 Web → /developer/client/create 创建 Client                 │
│     → [Client 实体通过 user_id 绑定到创建用户]                           │
│                                                                         │
│  2. 调用方 POST /oauth/v2/token                                         │
│     grant_type=password                                                 │
│     + client_id + client_secret (Client 凭证)                           │
│     + username + password (用户凭证)                                     │
│     → FOSOAuthServerBundle 验证通过后                                    │
│       生成 AccessToken → [通过 user_id 外键绑定用户]                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ 携带 Authorization: Bearer {access_token}
┌─────────────────────────────────────────────────────────────────────────┐
│                      第二阶段：请求进入条目接口鉴权                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  3. API 防火墙 (fos_oauth: true)                                        │
│     → 从 Bearer Token 查 oauth2_access_tokens 表                        │
│     → 找到对应用户 → 注入 Security Token                                │
│     → 无有效 Token → 返回 401 (否决点 #1)                                │
│                                                                         │
│  4. ParamConverter 加载 Entry 实体                                       │
│     → 按 {id} 查 entry 表（不过滤 user_id，可加载他人条目）               │
│                                                                         │
│  5. #[IsGranted] 注解触发 Voter 投票                                     │
│     → EntryVoter::voteOnAttribute()                                     │
│     → $user === $subject->getUser() 校验所有权                           │
│     → 非所有者 → 抛出 AccessDeniedHttpException (否决点 #2)              │
│                                                                         │
│  6. AccessDeniedToNotFoundSubscriber 异常转换                            │
│     → 捕获 AccessDeniedHttpException                                    │
│     → 替换为 NotFoundHttpException                                      │
│     → 最终表现为 404 而非 403 (否决点 #2.5 - 伪装层)                    │
│                                                                         │
│  7. 控制器代码运行时二次检查 (批量操作)                                   │
│     → authorizationChecker->isGranted()                                 │
│     → 跳过未授权条目 (否决点 #3)                                         │
│                                                                         │
│  8. Repository 强制按 user_id 过滤                                       │
│     → 所有查询 WHERE user_id = 当前用户                                  │
│     → 兜底保障，仅返回本人数据 (否决点 #4)                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、OAuth 令牌从申请到绑定用户的完整链路

### 2.1 第一步：创建 OAuth Client（与用户绑定）

用户首先需要登录 Web 界面，在开发者页面创建一个 API Client（即一个"应用"）。

**控制器代码**：[DeveloperController.php#L39-L66](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/DeveloperController.php#L39-L66)

```php
#[Route(path: '/developer/client/create', name: 'developer_create_client', methods: ['GET', 'POST'])]
public function createClientAction(Request $request, EntityManagerInterface $entityManager, TranslatorInterface $translator)
{
    $client = new Client($this->getUser());  // ★ Client 构造时绑定当前登录用户
    $clientForm = $this->createForm(ClientType::class, $client);
    // ...
    if ($clientForm->isSubmitted() && $clientForm->isValid()) {
        $client->setAllowedGrantTypes(['token', 'authorization_code', 'password', 'refresh_token']);
        $entityManager->persist($client);
        $entityManager->flush();
        // 返回 client_id 和 client_secret
    }
}
```

**Client 实体结构**：[Client.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Entity/Api/Client.php)

```php
#[ORM\Entity]
class Client extends BaseClient
{
    #[ORM\JoinColumn(nullable: false)]
    #[ORM\ManyToOne(targetEntity: User::class, inversedBy: 'clients')]
    private $user;  // ★ Client 绑定到创建它的用户

    public function __construct(User $user)
    {
        parent::__construct();
        $this->user = $user;
    }

    public function getUser(): User
    {
        return $this->user;
    }
}
```

**绑定原理**：Client 实体通过 `user_id` 外键严格归属某个用户。但此绑定仅用于开发者管理（查看/删除自己的 Client），真正的令牌-用户绑定发生在 AccessToken 级别。

### 2.2 第二步：调用方申请 Access Token

调用方通过 FOSOAuthServerBundle 提供的标准 OAuth2 端点 `/oauth/v2/token` 申请令牌。

**官方文档说明**：[oauth.md](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/doc/content/developer/api/oauth.md)

```bash
# 申请令牌（password 授权模式）
http POST http://localhost:8000/oauth/v2/token \
    grant_type=password \
    client_id=1_3o53gl30vhgk0c8ks4cocww08o84448osgo40wgw4gwkoo8skc \
    client_secret=636ocbqo978ckw0gsw4gcwwocg8044sco0w8w84cws48ggogs4 \
    username=wallabag \
    password=wallabag
```

**功能测试验证**：[DeveloperControllerTest.php#L39-L59](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/tests/functional/Controller/Api/DeveloperControllerTest.php#L39-L59)

```php
public function testCreateToken(): void
{
    $apiClient = $this->createApiClientForUser('admin');

    $client->request('POST', '/oauth/v2/token', [
        'grant_type' => 'password',
        'client_id' => $apiClient->getPublicId(),
        'client_secret' => $apiClient->getSecret(),
        'username' => 'admin',       // 用户凭证
        'password' => 'mypassword',  // 用户凭证
    ]);

    $data = json_decode((string) $client->getResponse()->getContent(), true);
    $this->assertArrayHasKey('access_token', $data);
    // 返回: { "access_token": "...", "expires_in": 3600, "refresh_token": "...", "scope": null, "token_type": "bearer" }
}
```

**FOSOAuthServerBundle 内部令牌颁发流程**（由第三方包实现，配置见 [config.yml#L203-L213](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/app/config/config.yml#L203-L213)）：

```
POST /oauth/v2/token
    │
    ▼
1. 校验 client_id + client_secret → 查 oauth2_clients 表
    │
    ▼
2. 校验 username + password → 通过 fos_user.user_provider.username_email
    │
    ▼
3. 生成随机 token 字符串
    │
    ▼
4. 写入 oauth2_access_tokens 表
   ├─ token: 随机字符串
   ├─ client_id: 关联的 Client
   ├─ user_id: ★ 关联的用户（从 username 反查得到的 User ID）★
   ├─ expires_at: 过期时间戳
   └─ scope: null（Wallabag 未使用）
    │
    ▼
5. 返回 JSON: { access_token, expires_in, refresh_token, scope, token_type }
```

### 2.3 第三步：AccessToken 与用户的绑定关系

**AccessToken 实体**：[AccessToken.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Entity/Api/AccessToken.php)

```php
#[ORM\Table('oauth2_access_tokens')]
#[ORM\Entity]
class AccessToken extends BaseAccessToken
{
    #[ORM\JoinColumn(nullable: false)]
    #[ORM\ManyToOne(targetEntity: Client::class, inversedBy: 'accessTokens')]
    protected $client;          // 关联 OAuth 客户端

    #[ORM\JoinColumn(name: 'user_id', referencedColumnName: 'id', onDelete: 'CASCADE')]
    #[ORM\ManyToOne(targetEntity: User::class)]
    protected $user;            // ★ 关联资源所有者（用户）★
}
```

**四种 OAuth 实体均与 User 绑定**：

| 实体类 | 表名 | 绑定字段 |
|--------|------|---------|
| [AccessToken.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Entity/Api/AccessToken.php) | `oauth2_access_tokens` | `user_id` |
| [RefreshToken.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Entity/Api/RefreshToken.php) | `oauth2_refresh_tokens` | `user_id` |
| [AuthCode.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Entity/Api/AuthCode.php) | `oauth2_auth_codes` | `user_id` |
| [Client.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Entity/Api/Client.php) | `oauth2_clients` | `user_id` |

### 2.4 第四步：请求鉴权时如何从 Token 还原用户

**防火墙配置**：[security.yml#L29-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/app/config/security.yml#L29-L34)

```yaml
firewalls:
    api:
        pattern: /api/.*
        fos_oauth: true          # ★ 启用 FOS OAuth 认证监听器
        stateless: true
        anonymous: true
        provider: fos_userbundle
```

**FOSOAuthServerBundle 认证流程（由第三方包实现）**：

```
HTTP 请求到达 /api/entries/123
    │
    ▼
1. 提取 Authorization: Bearer {token} 头
    │
    ▼
2. 查询 oauth2_access_tokens 表 WHERE token = ?
    │
    ▼
3. 检查 token 是否过期（expires_at > 当前时间）
    │
    ▼
4. 通过 token 记录的 user_id 查 User 表
    │
    ▼
5. 将 User 对象注入 Symfony Security Token
    → 后续 $this->getUser() 即返回该用户
    │
    ▼
6. 若任何步骤失败 → 返回 401 Unauthorized (否决点 #1)
```

**关键结论**：每个 OAuth AccessToken 通过数据库中的 `user_id` 外键**严格绑定到且仅绑定到一个用户**。调用方使用该令牌发起的所有请求，其身份即等同于令牌绑定的用户——这是后续所有访问控制的身份根基。

### 2.5 Scope 机制分析（未实际使用）

数据库层面存在 scope 字段（见 [Version20160401000000.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/migrations/Version20160401000000.php#L38-L46)）：

```sql
CREATE TABLE oauth2_access_tokens (
    ...
    scope VARCHAR(255) DEFAULT NULL,
    ...
);
```

**但是，代码分析表明 Wallabag 并未实际使用 OAuth scope 做细粒度权限控制**：
- `config.yml` 中未配置 `supported_scopes`
- 所有 Voter 和控制器均未检查令牌的 scope 字段
- 权限判断完全基于 **用户身份（user_id）+ 角色（ROLE_USER）**，而非 scope
- 令牌申请响应中 `"scope": null` 也印证了这一点

---

## 三、越权访问为何表现为 404 而非 403

这是 Wallabag 安全设计中非常精妙的一点——通过**异常转换**隐藏资源存在性。

### 3.1 现象描述

功能测试验证：[EntryRestControllerTest.php#L99-L113](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php#L99-L113)

```php
public function testGetOneEntryWrongUser(): void
{
    // 获取属于 bob 的一条记录
    $entry = $this->client->getContainer()
        ->get(EntityManagerInterface::class)
        ->getRepository(Entry::class)
        ->findOneBy(['user' => $this->getUserId('bob'), 'isArchived' => false]);

    // 使用 admin 用户（而非 bob）请求访问 bob 的条目
    $this->client->request('GET', '/api/entries/' . $entry->getId() . '.json');

    // ★ 预期结果是 404，而不是 403
    $this->assertSame(404, $this->client->getResponse()->getStatusCode());
}
```

同样的 404 行为出现在越权删除、越权 PATCH 等所有单条目操作中：
- [EntryRestControllerTest.php#L644-L647](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php#L644-L647)：删除已删除条目 → 404
- [EntryRestControllerTest.php#L672-L675](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/tests/functional/Controller/Api/EntryRestControllerTest.php#L672-L675)：再次删除已删除条目 → 404

### 3.2 核心机制：AccessDeniedToNotFoundSubscriber

**异常转换事件订阅器**：[AccessDeniedToNotFoundSubscriber.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Event/Subscriber/AccessDeniedToNotFoundSubscriber.php)

```php
class AccessDeniedToNotFoundSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::EXCEPTION => 'onKernelException',  // ★ 监听异常事件
        ];
    }

    public function onKernelException(ExceptionEvent $event): void
    {
        $exception = $event->getThrowable();

        if ($exception instanceof AccessDeniedHttpException) {
            // ★ 将 403 异常替换为 404 异常 ★
            $notFoundException = new NotFoundHttpException('', $exception);
            $event->setThrowable($notFoundException);
        }
    }
}
```

### 3.3 完整时序：越权单条目的执行流程

以 `GET /api/entries/123` 为例（用户 A 访问属于用户 B 的条目 123）：

```
步骤 1: ParamConverter 加载 Entry
─────────────────────────────────────────
[EntryRestController.php#L405-L407]
    #[Route(path: '/api/entries/{entry}.{_format}', ...)]
    #[IsGranted('VIEW', subject: 'entry')]
    public function getEntryAction(Entry $entry)

↓ Symfony SensioFrameworkExtraBundle 的 DoctrineParamConverter 自动执行:
  SELECT * FROM entry WHERE id = 123
  → 找到记录（虽然属于用户 B），注入 $entry 对象

步骤 2: #[IsGranted] 触发 EntryVoter 检查
─────────────────────────────────────────
[EntryVoter.php#L40-L54]
    protected function voteOnAttribute(string $attribute, $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();   // 当前认证用户 = A
        // $subject = 条目 123，其所属用户 = B

        return $user === $subject->getUser();  // A === B ? → false
    }

↓ Symfony Security 组件判定 ACCESS_DENIED
  → 抛出 AccessDeniedHttpException (HTTP 403)

步骤 3: 异常转换（关键伪装步骤）
─────────────────────────────────────────
[AccessDeniedToNotFoundSubscriber.php#L20-L28]
    public function onKernelException(ExceptionEvent $event): void
    {
        $exception = $event->getThrowable();  // AccessDeniedHttpException

        if ($exception instanceof AccessDeniedHttpException) {
            // 替换异常对象
            $notFoundException = new NotFoundHttpException('', $exception);
            $event->setThrowable($notFoundException);
        }
    }

↓ 最终返回给客户端:
  HTTP/1.1 404 Not Found
```

### 3.4 设计意图：防止资源存在性探测（防枚举攻击）

这种 403→404 的转换是一种经典安全策略，目的是**不向攻击者泄露资源是否存在**：

| 场景 | 无转换时响应 | 有转换时响应 |
|------|-------------|-------------|
| 条目不存在 | 404 Not Found | 404 Not Found |
| 条目存在但不属于当前用户 | 403 Forbidden | 404 Not Found |
| 条目存在且属于当前用户 | 200 OK | 200 OK |

**攻击者视角**：无论目标 ID 是否真实存在，只要无权访问就统一返回 404，无法通过响应码差异来枚举出有效的条目 ID。

### 3.5 ParamConverter 加载他人数据是否构成信息泄露？

**答案：不构成。** 原因如下：

1. **ParamConverter 仅在内存中加载**：Doctrine 查询出的 Entry 对象仅存在于 PHP 内存中，未序列化输出
2. **#[IsGranted] 在控制器方法执行前触发**：SensioFrameworkExtraBundle 的 SecurityListener 在控制器调用前执行权限检查，若失败则控制器方法**根本不会执行**，$entry 对象不会被使用
3. **异常转换在响应前执行**：AccessDenied→NotFound 转换发生在 KernelEvents::EXCEPTION，此时响应体尚未构建

执行顺序由 Symfony HTTP Kernel 保证：

```
kernel.request
  → RouterListener (匹配路由)
  → ParamConverterListener (加载 Entry 到 $request->attributes)
  → SecurityListener (#[IsGranted] 检查 → 失败则抛异常)
  → 控制器方法调用 (被跳过，因为异常已抛出)

kernel.exception
  → AccessDeniedToNotFoundSubscriber (403 → 404)
  → 渲染异常响应
```

---

## 四、Voter 投票机制——越权否决的核心

Wallabag 使用 Symfony Voter 实现基于属性的访问控制（ABAC），这是**越权请求被否决的核心位置**。

### 4.1 MainVoter：集合级操作权限判断

文件位置：[MainVoter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/MainVoter.php)

**支持的权限属性（无 Subject）**：

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
```

**投票逻辑（[voteOnAttribute#L42-L48](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/MainVoter.php#L42-L48)）**：

```php
protected function voteOnAttribute(string $attribute, $subject, TokenInterface $token): bool
{
    return match ($attribute) {
        self::LIST_ENTRIES, self::CREATE_ENTRIES, self::EDIT_ENTRIES, 
        self::EXPORT_ENTRIES, self::IMPORT_ENTRIES, self::DELETE_ENTRIES,
        self::LIST_TAGS, self::CREATE_TAGS, self::DELETE_TAGS,
        => $this->security->isGranted('ROLE_USER'),
        default => false,
    };
}
```

**判断规则**：
- 只要用户拥有 `ROLE_USER` 角色即授权通过
- 这是**粗粒度**的入口级权限检查

### 4.2 EntryVoter：单条目级操作权限判断

文件位置：[EntryVoter.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/EntryVoter.php)

**支持的权限属性（需 Subject = Entry 实例）**：

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

**核心投票逻辑（[voteOnAttribute#L40-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/EntryVoter.php#L40-L54)）**：

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

**否决点 #2 —— 条目所有权检查**：
- **`$user === $subject->getUser()`** 这行代码是越权否决的关键
- 比较当前认证用户与条目的所属用户是否为同一对象（PHP 对象比较，因为 Doctrine 同一 EntityManager 内同一 ID 始终返回同一对象引用）
- 若不匹配（即越权访问他人条目），返回 `false`，Voter 投出 `ACCESS_DENIED`
- 随后 AccessDeniedToNotFoundSubscriber 将其伪装成 404

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

### 5.3 运行时二次权限检查

除了注解检查外，部分方法在代码逻辑中进行二次检查，例如批量操作：

[EntryRestController.php#L499](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php#L499)
```php
if (false !== $entry && $this->authorizationChecker->isGranted('DELETE', $entry)) {
    // 执行删除操作
}
```

**否决点 #3 —— 运行时二次检查**：
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

**控制器调用方式**（[EntryRestController.php#L333-L334](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php#L333-L334)）：
```php
$pager = $entryRepository->findEntries(
    $this->getUser()->getId(),  // 传入当前认证用户的 ID
    // ...
);
```

**否决点 #4 —— 数据层过滤**：
- 即使攻击者设法绕过 Voter 检查，Repository 查询也不会返回其他用户的数据
- 所有查询方法要求显式传入 `$userId`，避免上下文泄漏

---

## 七、五层越权否决机制总结

| 层级 | 位置 | 否决条件 | 效果 |
|------|------|---------|------|
| **第1层：防火墙** | [security.yml#L29-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/app/config/security.yml#L29-L34) | 无有效 OAuth Bearer Token | 401 Unauthorized |
| **第2层：Voter 所有权检查** | `#[IsGranted]` + [EntryVoter.php#L51](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Security/Voter/EntryVoter.php#L51) | 当前用户 ≠ 条目所属用户 | 抛出 AccessDeniedHttpException (内部 403) |
| **第2.5层：异常伪装** | [AccessDeniedToNotFoundSubscriber.php#L20-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Event/Subscriber/AccessDeniedToNotFoundSubscriber.php#L20-L28) | 捕获 AccessDeniedHttpException | 替换为 NotFoundHttpException → 404 Not Found |
| **第3层：运行时二次检查** | [EntryRestController.php#L499](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Controller/Api/EntryRestController.php#L499) | 批量操作中逐条检查所有权 | 跳过未授权条目 |
| **第4层：Repository 强制过滤** | [EntryRepository.php#L753-L757](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Repository/EntryRepository.php#L753-L757) | 所有查询强制 `WHERE user_id = ?` | 空结果集 / 仅返回本人数据 |

---

## 八、设计亮点与潜在风险

### 8.1 设计亮点

1. **纵深防御**：五层否决机制层层递进，即使单层被绕过也不会导致越权
2. **存在性隐藏**：通过 AccessDenied→NotFound 异常转换，防止攻击者通过 403/404 差异枚举出有效资源 ID
3. **令牌-用户强绑定**：AccessToken 通过数据库外键绑定唯一用户，身份映射清晰可靠
4. **所有权优先**：EntryVoter 中所有条目操作权限统一归结为「用户是否为所有者」，逻辑简单一致
5. **无状态认证**：API 防火墙使用 `stateless: true`，避免 Session 劫持风险
6. **Repository 封装**：所有数据查询通过统一构造器强制用户过滤，难以遗漏

### 8.2 潜在风险与改进空间

1. **OAuth Scope 未实际使用**：数据库有 scope 字段但代码未利用，无法对不同客户端授予不同 API 权限（如只读客户端）
2. **缺少 Rate Limiting**：配置中未见针对 OAuth 令牌的速率限制配置
3. **缺少审计日志**：越权尝试未被记录审计日志（仅标准访问日志）
4. **Scope 为 null**：令牌响应中 scope 固定为 null，丧失了 OAuth2 协议的细粒度授权能力

---

## 九、核心问题回答总结

### Q1: OAuth 令牌如何与用户绑定？

**绑定发生在两个层面**：

1. **令牌颁发时**：调用方通过 `POST /oauth/v2/token` 提交 `username + password`，FOSOAuthServerBundle 验证用户凭证后，在 `oauth2_access_tokens` 表中创建记录时将 `user_id` 字段设置为对应用户的 ID（[AccessToken.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Entity/Api/AccessToken.php)）

2. **请求鉴权时**：API 请求携带 `Authorization: Bearer {token}`，FOSOAuthServerBundle 的防火墙监听器查询 `oauth2_access_tokens` 表，通过 token 找到 `user_id`，再通过该外键加载 User 对象并注入 Symfony Security Token 中。后续 `$this->getUser()` 即返回此用户

**数据链路**：`Bearer Token → oauth2_access_tokens.token → oauth2_access_tokens.user_id → user.id → User 对象`

### Q2: 越权访问条目为何表现为 404 而非 403？

**核心机制**：[AccessDeniedToNotFoundSubscriber.php](file:///d:/fz/0601-1/solo-dogfeeding/code/43-wallabag/src/Event/Subscriber/AccessDeniedToNotFoundSubscriber.php)

**执行流程**：

1. ParamConverter 按 ID 加载 Entry（无论所属用户）
2. `#[IsGranted('VIEW', subject: 'entry')]` 触发 EntryVoter 检查
3. `$user === $subject->getUser()` 返回 false → 抛出 `AccessDeniedHttpException`（原生应为 403）
4. `AccessDeniedToNotFoundSubscriber` 监听 `KernelEvents::EXCEPTION`，检测到 `AccessDeniedHttpException` 后将其替换为 `NotFoundHttpException`
5. 最终响应为 404 Not Found

**设计目的**：防止资源存在性探测。攻击者无法通过 403/404 响应差异判断某个 ID 是否真实存在，避免被枚举攻击。
