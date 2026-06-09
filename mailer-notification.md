# Wallabag 邮件通知系统三层协作机制

## 版本依据

| 依赖包 | 版本 | 来源 |
|--------|------|------|
| friendsofsymfony/user-bundle | v3.4.0 | [composer.lock L2416-L2417](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/composer.lock#L2416-L2417) |
| scheb/2fa-email | v5.13.2 | [composer.lock L7601-L7602](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/composer.lock#L7601-L7602) |

---

## 三条邮件链路总览

| 链路 | 触发方式 | 触发入口 | 邮件器服务 | 模板路径 | 模板结构 |
|------|---------|---------|-----------|---------|---------|
| **注册确认邮件** | 事件监听器 | `EmailConfirmationListener` 监听 `REGISTRATION_SUCCESS` | `fos_user.mailer.twig_symfony` | `@FOSUser/Registration/email.txt.twig` | subject / body_text / body_html |
| **密码重置邮件** | 控制器直接调用 | `ResettingController::sendEmailAction()` | `fos_user.mailer.twig_symfony` | `@FOSUser/Resetting/email.txt.twig` | subject / body_text / body_html |
| **双因素验证码邮件** | 2FA 认证流程触发 | 登录成功后由 scheb/2fa-email 认证器触发 | `Wallabag\Mailer\AuthCodeMailer` | `TwoFactor/email_auth_code.html.twig` | subject / body_text / body_html |

> **核心区别**：注册确认走**事件监听器**（可选功能），密码重置走**控制器直接调用**（核心功能），双因素走**2FA 认证流程**（独立安全模块）。

---

## 第一层：事件订阅层

### 1.1 Wallabag 自定义事件

定义在 `src/Event/` 目录，**都不直接触发邮件**：

| 事件类 | 事件名常量 | 触发时机 | 用途 |
|--------|-----------|----------|------|
| [EntrySavedEvent](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntrySavedEvent.php) | `entry.saved` | 文章保存时 | 触发图片下载等 |
| [EntryDeletedEvent](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntryDeletedEvent.php) | `entry.deleted` | 文章删除时 | 清理图片等 |
| [ConfigUpdatedEvent](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/ConfigUpdatedEvent.php) | `config.updated` | 配置更新时 | 重新生成 CSS 等 |

### 1.2 FOSUserBundle 事件体系

来自 FOSUserBundle v3.4.0，所有事件常量定义在 `FOS\UserBundle\FOSUserEvents` 类中。

**密码重置相关事件（发送邮件流程）**：

| 事件常量 | 触发时机 | 事件类 | 作用 |
|----------|----------|--------|------|
| `RESETTING_SEND_EMAIL_INITIALIZE` | 发送重置邮件流程最开始（尚未查找用户） | `GetResponseNullableUserEvent` | 可在此修改用户查找逻辑或提前拦截 |
| `RESETTING_RESET_REQUEST` | 查找到用户后、冷却时间检查前 | `GetResponseUserEvent` | 可在此做额外的用户状态检查 |
| `RESETTING_SEND_EMAIL_CONFIRM` | 生成 token 后、**发送邮件前** | `GetResponseUserEvent` | 发送前确认 — 可拦截发送或修改用户 |
| `RESETTING_SEND_EMAIL_COMPLETED` | 发送邮件且保存用户后 | `GetResponseUserEvent` | 发送后通知 — 可用于日志、审计等 |

> **源码事实**：`RESETTING_SEND_EMAIL_CONFIRM` 确实存在，位于生成 token 之后、发送邮件之前。这是发送邮件前的最后一个拦截点，如果监听器设置了 response，会直接返回而不发送邮件。

**密码重置相关事件（重置密码流程）**：

| 事件常量 | 触发时机 | 事件类 |
|----------|----------|--------|
| `RESETTING_RESET_INITIALIZE` | 用户点击重置链接、重置页面初始化时 | `GetResponseUserEvent` |
| `RESETTING_RESET_SUCCESS` | 重置密码表单验证通过、保存前 | `FormEvent` |
| `RESETTING_RESET_COMPLETED` | 重置密码完成后 | `FilterUserResponseEvent` |

**注册相关事件**：

| 事件常量 | 触发时机 | 事件类 |
|----------|----------|--------|
| `REGISTRATION_INITIALIZE` | 注册页面初始化时 | `UserEvent` |
| `REGISTRATION_SUCCESS` | 注册表单验证通过、用户保存前 | `FormEvent` |
| `REGISTRATION_COMPLETED` | 注册完成后 | `FilterUserResponseEvent` |
| `REGISTRATION_CONFIRM` | 用户点击确认链接、确认页面初始化时 | `GetResponseUserEvent` |
| `REGISTRATION_CONFIRMED` | 用户确认成功后 | `FilterUserResponseEvent` |
| `REGISTRATION_FAILURE` | 注册表单验证失败时 | `FormEvent` |

**事件命名约定**（FOSUserBundle 通用模式）：
- `*_INITIALIZE` — 流程初始化后、表单创建前
- `*_SUCCESS` — 表单验证通过后、持久化前
- `*_COMPLETED` — 所有操作完成后
- `*_CONFIRM` — 确认操作前（如点击邮件中的确认链接时）
- `*_CONFIRMED` — 确认操作完成后

### 1.3 Wallabag 自身的事件监听器/订阅者

位于 `src/Event/Listener/` 和 `src/Event/Subscriber/`，**都不直接发送邮件**：

| 监听器/订阅者 | 订阅的事件 | 作用 |
|--------------|-----------|------|
| [RegistrationListener](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/RegistrationListener.php) | `REGISTRATION_INITIALIZE` | 注册开关控制，禁用时重定向 |
| [PasswordResettingListener](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/PasswordResettingListener.php) | `RESETTING_RESET_SUCCESS` | 重置成功后跳转到首页 |
| [CreateConfigListener](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/CreateConfigListener.php) | `REGISTRATION_COMPLETED`, `USER_CREATED` | 创建用户默认配置 |
| [DownloadImagesSubscriber](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php) | `entry.saved`, `entry.deleted` | 下载/清理文章图片 |
| [GenerateCustomCSSSubscriber](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Subscriber/GenerateCustomCSSSubscriber.php) | `config.updated` | 生成自定义 CSS |

**FOSUserBundle 内置监听器中与邮件相关的**：

| 监听器 | 订阅的事件 | 作用 |
|--------|-----------|------|
| `EmailConfirmationListener` | `REGISTRATION_SUCCESS` | 注册确认邮件发送（仅在 confirmation.enabled=true 时注册） |
| `ResettingListener` | `RESETTING_RESET_INITIALIZE`, `RESETTING_RESET_SUCCESS`, `RESETTING_RESET_REQUEST` | 密码重置流程控制（token 过期检查、状态清理等） |
| `FlashListener` | 各类 `*_COMPLETED` 事件 | 操作成功后添加 flash 消息 |

> **重要事实**：Wallabag 自身代码中**没有任何事件监听器直接发送邮件**。邮件发送逻辑主要封装在 FOSUserBundle 的 `EmailConfirmationListener` 监听器和 `ResettingController` 控制器中，以及 scheb/2fa-email 的认证流程中。

---

## 第二层：邮件分发层

### 2.1 配置总览

在 [config.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml) 中配置了三个邮件相关组件：

```yaml
# 1. Symfony 基础邮件传输层
framework:
    mailer:
        dsn: "%env(MAILER_DSN)%"

# 2. FOSUserBundle 邮件服务配置
fos_user:
    from_email:
        address: "%env(WALLABAG_FROM_EMAIL)%"
        sender_name: wallabag
    service:
        mailer: fos_user.mailer.twig_symfony
    registration:
        confirmation:
            enabled: "%env(bool:WALLABAG_CONFIRMATION_ENABLED)%"

# 3. SchebTwoFactor 邮件服务配置
scheb_two_factor:
    email:
        enabled: true
        sender_email: "%env(WALLABAG_TWOFACTOR_SENDER)%"
        digits: 6
        mailer: Wallabag\Mailer\AuthCodeMailer
```

### 2.2 邮件器一：FOSUserBundle TwigSymfonyMailer

**服务 ID**：`fos_user.mailer.twig_symfony`

**类名**：`FOS\UserBundle\Mailer\TwigSymfonyMailer`

**实现接口**：`FOS\UserBundle\Mailer\MailerInterface`

**接口方法**：
```php
interface MailerInterface
{
    public function sendConfirmationEmailMessage(UserInterface $user);
    public function sendResettingEmailMessage(UserInterface $user);
}
```

**负责发送**：
1. 注册确认邮件 — `sendConfirmationEmailMessage()`
2. 密码重置邮件 — `sendResettingEmailMessage()`

**依赖注入**：
- `Symfony\Component\Mailer\MailerInterface` — 底层邮件传输
- `Twig\Environment` — Twig 模板引擎
- `Symfony\Component\Routing\Generator\UrlGeneratorInterface` — URL 生成器
- 发件人配置（`fos_user.from_email`）
- 模板路径配置（注册和重置各一个）

**命名含义**：
- `twig` — 用 Twig 模板引擎渲染邮件内容
- `symfony` — 用 Symfony Mailer 组件发送邮件
- 这是 FOSUserBundle 3.x 推荐的邮件器实现

**在 ResettingController 中的注入**：
`fos_user.resetting.controller` 服务的第 5 个构造参数是 `fos_user.mailer`，证实控制器直接持有邮件器引用，邮件由控制器直接调用发送。

### 2.3 邮件器二：Wallabag AuthCodeMailer

**位置**：[src/Mailer/AuthCodeMailer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Mailer/AuthCodeMailer.php)

**实现接口**：`Scheb\TwoFactorBundle\Mailer\AuthCodeMailerInterface`

**负责发送**：双因素认证验证码邮件

**核心代码**：

```php
class AuthCodeMailer implements AuthCodeMailerInterface
{
    public function __construct(
        private readonly MailerInterface $mailer,   // Symfony Mailer
        private readonly Environment $twig,         // Twig 模板引擎
        private $senderEmail,    // 发件人邮箱
        private $senderName,     // 发件人名称
        private $supportUrl,     // 支持页面 URL
    ) {}

    public function sendAuthCode(TwoFactorInterface $user): void
    {
        $template = $this->twig->load('TwoFactor/email_auth_code.html.twig');
        
        $subject = $template->renderBlock('subject', []);
        $bodyHtml = $template->renderBlock('body_html', [
            'user' => $user->getName(),
            'code' => $user->getEmailAuthCode(),
            'support_url' => $this->supportUrl,
        ]);
        $bodyText = $template->renderBlock('body_text', [
            'user' => $user->getName(),
            'code' => $user->getEmailAuthCode(),
            'support_url' => $this->supportUrl,
        ]);
        
        $email = (new Email())
            ->from(new Address($this->senderEmail, $this->senderName ?: $this->senderEmail))
            ->to($user->getEmailAuthRecipient())
            ->subject($subject)
            ->text($bodyText)
            ->html($bodyHtml);
        
        $this->mailer->send($email);
    }
}
```

**依赖注入参数**（在 [services.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/services.yml#L37-L40) 中配置）：

| 参数 | 来源 | 配置项 |
|------|------|--------|
| `$senderEmail` | 参数绑定 | `scheb_two_factor.email.sender_email` |
| `$senderName` | 参数绑定 | `scheb_two_factor.email.sender_name` |
| `$supportUrl` | 表达式服务 | `craue_config.get('wallabag_support_url')` |

### 2.4 邮件发送架构图

```
┌───────────────────────────────────────────────────────────────────┐
│                        触发层 (3 种方式)                           │
│                                                                   │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │ 注册确认邮件        │  │ 密码重置邮件     │  │ 双因素验证码  │  │
│  │ EmailConfirmation   │  │ ResettingControl- │  │ scheb/2fa-email│ │
│  │ Listener            │  │ ler::sendEmail()   │  │ 认证流程      │  │
│  │ (事件监听器)        │  │ (控制器直接调用)  │  │ (登录成功后触发)│  │
│  └──────────┬──────────┘  └────────┬─────────┘  └───────┬───────┘  │
└─────────────┼──────────────────────┼──────────────────────┼──────────┘
              │                      │                      │
              ▼                      ▼                      ▼
┌───────────────────────────────────────────────────────────────────┐
│                        邮件分发层                                  │
│                                                                   │
│  ┌──────────────────────────────────┐  ┌───────────────────────┐  │
│  │  fos_user.mailer.twig_symfony    │  │ Wallabag\Mailer\      │  │
│  │  TwigSymfonyMailer               │  │   AuthCodeMailer      │  │
│  │  • sendConfirmationEmailMessage  │  │  • sendAuthCode()     │  │
│  │  • sendResettingEmailMessage     │  │                       │  │
│  └──────────────┬───────────────────┘  └──────────┬────────────┘  │
│                 │                                   │               │
│                 └─────────────────┬─────────────────┘               │
│                                   │                                 │
│                                   ▼                                 │
│                    Symfony\Component\Mailer\MailerInterface        │
│                    (底层邮件传输：SMTP / sendmail 等)               │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
                                    ▼
                          邮件送达用户邮箱
```

---

## 第三层：模板渲染层

### 3.1 邮件模板清单

| 模板文件 | 用途 | 使用者 | 块结构 | 来源 |
|---------|------|--------|--------|------|
| `@FOSUser/Registration/email.txt.twig` | 注册确认邮件 | TwigSymfonyMailer | subject / body_text / body_html | FOSUserBundle 内置 |
| `@FOSUser/Resetting/email.txt.twig` | 密码重置邮件 | TwigSymfonyMailer | subject / body_text / body_html | FOSUserBundle 内置 |
| [templates/TwoFactor/email_auth_code.html.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/TwoFactor/email_auth_code.html.twig) | 双因素验证码邮件 | AuthCodeMailer | subject / body_text / body_html | Wallabag 自定义 |
| [templates/Mail/forgotPassword.txt.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/Mail/forgotPassword.txt.twig) | 密码重置邮件（遗留） | 未使用 | 无明确块结构 | Wallabag 旧代码 |

> **关于遗留模板**：`templates/Mail/forgotPassword.txt.twig` 是早期版本遗留下来的文件。当前版本使用 FOSUserBundle 的 `fos_user.mailer.twig_symfony` 邮件器，该文件已不再被任何代码引用。

### 3.2 块模式（Block-based）模板设计

所有邮件模板都采用 **Twig 块模式** 设计，即一个模板文件中包含多个命名块，分别渲染邮件的不同部分。

**FOSUserBundle 邮件模板结构**（纯文本邮件）：

```twig
{% trans_default_domain 'FOSUserBundle' %}

{% block subject %}
{%- autoescape false -%}
{{ 'resetting.email.subject'|trans({'%username%': user.username}) }}
{%- endautoescape -%}
{% endblock %}

{% block body_text %}
{% autoescape false %}
{{ 'resetting.email.message'|trans({'%username%': user.username, '%confirmationUrl%': confirmationUrl}) }}
{% endautoescape %}
{% endblock %}

{% block body_html %}{% endblock %}
```

> **注意**：FOSUserBundle 默认的邮件模板只有纯文本版本，`body_html` 块是空的。

**双因素邮件模板结构**（HTML + 纯文本）：

```twig
{# 主题行 #}
{% block subject %}
{{ "auth_code.mailer.subject"|trans({}, 'wallabag_user') }}
{% endblock %}

{# 纯文本版本 #}
{% block body_text %}
{{ "auth_code.mailer.body.hello"|trans({'%user%': user}, 'wallabag_user') }}
...
{{ code }}
{% endblock %}

{# HTML 版本（完整样式） #}
{% block body_html %}
<!DOCTYPE html>
<html>
    <!-- 完整的 HTML 邮件，包含内联 CSS 样式 -->
    <!-- 有 logo、卡片布局、页脚等 -->
</html>
{% endblock %}
```

### 3.3 模板渲染流程

```php
// 1. 加载模板文件
$template = $this->twig->load('TwoFactor/email_auth_code.html.twig');

// 2. 分别渲染每个块（传入不同的上下文变量）
$subject = $template->renderBlock('subject', []);
$bodyHtml = $template->renderBlock('body_html', $context);
$bodyText = $template->renderBlock('body_text', $context);

// 3. 组装到 Email 对象
$email = (new Email())
    ->subject($subject)
    ->text($bodyText)    // 纯文本版本（邮件客户端降级显示）
    ->html($bodyHtml);   // HTML 版本（富媒体显示）
```

### 3.4 模板覆盖机制

FOSUserBundle 的模板可以通过在 `templates/bundles/FOSUserBundle/` 下创建同名文件来覆盖。

**Wallabag 当前覆盖情况**：
- ✅ 覆盖了页面模板（如 `Resetting/check_email.html.twig`）
- ❌ 没有覆盖邮件模板（如 `Registration/email.txt.twig`）

如需自定义注册确认或密码重置邮件内容，需创建：
- `templates/bundles/FOSUserBundle/Registration/email.txt.twig`
- `templates/bundles/FOSUserBundle/Resetting/email.txt.twig`

---

## 链路详解：三条邮件的完整代码路径

### 链路一：注册确认邮件（事件监听器触发）

**前置条件**：`fos_user.registration.confirmation.enabled = true`

**调用链依据**：
- `fos_user.registration.confirmation.enabled` 配置项控制是否启用确认
- 启用时注册 `EmailConfirmationListener` 服务，监听 `REGISTRATION_SUCCESS` 事件
- `REGISTRATION_SUCCESS` 在注册表单验证通过后、用户保存前触发

**完整调用链**：

```
1. 用户提交注册表单
   │
   ▼
2. FOSUserBundle RegistrationController::registerAction()
   │
   ├─► 验证表单数据
   ├─► 创建 User 对象
   └─► 触发 FOSUserEvents::REGISTRATION_SUCCESS 事件 (FormEvent)
   │
   ▼
3. EmailConfirmationListener::onRegistrationSuccess()
   （FOSUserBundle 内置监听器，仅在 confirmation.enabled=true 时注册）
   │
   ├─► 生成确认 token：$tokenGenerator->generateToken()
   ├─► 保存到用户：$user->setConfirmationToken($token)
   ├─► 设置用户为未启用：$user->setEnabled(false)
   ├─► 保存用户：$userManager->updateUser($user)
   └─► 调用邮件器：$mailer->sendConfirmationEmailMessage($user)
   │
   ▼
4. TwigSymfonyMailer::sendConfirmationEmailMessage($user)
   │
   ├─► 生成确认 URL：$router->generate('fos_user_registration_confirm', ...)
   ├─► 加载模板：@FOSUser/Registration/email.txt.twig
   ├─► 渲染 subject 块
   ├─► 渲染 body_text 块（含 confirmationUrl）
   ├─► 渲染 body_html 块（默认空）
   ├─► 构建 Email 对象
   │   └─► 发件人：fos_user.from_email
   └─► 调用 $this->mailer->send($email)
   │
   ▼
5. Symfony Mailer 发送邮件
   │
   ▼
6. EmailConfirmationListener 设置重定向响应
   └─► 跳转到 fos_user_registration_check_email 路由
```

**配置依据**：
- 开关：[config.yml L196](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L196-L196) `fos_user.registration.confirmation.enabled`
- 邮件器：[config.yml L201](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L201-L201) `fos_user.service.mailer`
- 发件人：[config.yml L197-L199](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L197-L199) `fos_user.from_email`

**为什么用事件监听器而不是控制器直接调用**：
- 注册确认是**可选功能**，由 `confirmation.enabled` 控制
- 通过事件监听器可以有条件地注册服务，不修改核心控制器代码
- 监听器服务只在启用确认功能时才会被注册到容器中

**关于 REGISTRATION_SUCCESS 事件监听器的顺序**：
- `EmailConfirmationListener` 由 FOSUserBundle 内部注册
- 由于确认邮件发送后会设置重定向响应，后续监听器无法改变跳转目标
- 自定义监听器如需在确认邮件发送前执行逻辑，需设置合适的事件优先级

---

### 链路二：密码重置邮件（控制器直接调用）

**调用链依据**：
- `fos_user.resetting.controller` 服务定义中直接注入了 `fos_user.mailer`（第 5 个构造参数）
- 邮件发送是 `sendEmailAction` 的核心职责，不由事件触发
- `RESETTING_SEND_EMAIL_INITIALIZE` → `RESETTING_RESET_REQUEST` → `RESETTING_SEND_EMAIL_CONFIRM` → 发送邮件 → `RESETTING_SEND_EMAIL_COMPLETED`

**ResettingController 构造函数参数**（依据 FOSUserBundle v3.4.0 的 resetting.xml）：
1. `event_dispatcher` — 事件分发器
2. `fos_user.resetting.form.factory` — 重置密码表单工厂
3. `fos_user.user_manager` — 用户管理器
4. `fos_user.util.token_generator` — Token 生成器
5. `fos_user.mailer` — **邮件器（直接依赖，证明控制器直接调用）**
6. `%fos_user.resetting.retry_ttl%` — 重发冷却时间

**完整调用链（sendEmailAction）**：

```
1. 用户在"忘记密码"页面提交邮箱
   │
   ▼
2. FOSUserBundle ResettingController::sendEmailAction()
   │
   ├─► 根据用户名或邮箱查找用户（$user 可能为 null）
   │
   ├─► 触发 RESETTING_SEND_EMAIL_INITIALIZE 事件
   │   (GetResponseNullableUserEvent)
   │   扩展点：可修改用户查找结果或提前拦截
   │
   ├─► 如果用户不存在，直接跳转到 check_email
   │   （安全考量：不暴露邮箱是否已注册）
   │
   ├─► 触发 RESETTING_RESET_REQUEST 事件
   │   (GetResponseUserEvent)
   │   扩展点：可做额外的用户状态检查
   │
   ├─► 检查是否在冷却时间内
   │   (isPasswordRequestNonExpired())
   │   └─► 如果在冷却期内，直接跳转 check_email
   │       （防止恶意频繁发送）
   │
   ├─► 如果没有 token，生成重置 token
   │   $tokenGenerator->generateToken()
   │   $user->setConfirmationToken($token)
   │
   ├─► 触发 RESETTING_SEND_EMAIL_CONFIRM 事件
   │   (GetResponseUserEvent)
   │   扩展点：发送前的最后确认，可拦截发送
   │   （如果监听器设置了 response，直接返回不发送）
   │
   ├─► 直接调用邮件器发送：
   │   $this->mailer->sendResettingEmailMessage($user)
   │   ↑ 注意：这是控制器直接调用，不是通过事件触发！
   │
   ├─► 设置密码请求时间：
   │   $user->setPasswordRequestedAt(new \DateTime())
   │
   ├─► 保存用户：$userManager->updateUser($user)
   │
   ├─► 触发 RESETTING_SEND_EMAIL_COMPLETED 事件
   │   (GetResponseUserEvent)
   │   扩展点：发送后通知，可用于日志、审计等
   │
   └─► 设置 session 标记，重定向到 check_email 页面
   │
   ▼
3. TwigSymfonyMailer::sendResettingEmailMessage($user)
   │
   ├─► 生成重置 URL：$router->generate('fos_user_resetting_reset', ...)
   ├─► 加载模板：@FOSUser/Resetting/email.txt.twig
   ├─► 渲染 subject 块
   ├─► 渲染 body_text 块（含 confirmationUrl）
   ├─► 渲染 body_html 块（默认空）
   ├─► 构建 Email 对象
   │   └─► 发件人：fos_user.from_email
   └─► 调用 $this->mailer->send($email)
   │
   ▼
4. Symfony Mailer 发送邮件
   │
   ▼
5. 用户收到包含重置链接的邮件
```

**关键顺序总结**（sendEmailAction 内）：

```
查找用户（可能为 null）
  ↓
RESETTING_SEND_EMAIL_INITIALIZE 事件
  ↓
用户不存在 → 跳转 check_email
  ↓
RESETTING_RESET_REQUEST 事件
  ↓
检查冷却时间 → 在冷却期内 → 跳转 check_email
  ↓
生成 token → setConfirmationToken
  ↓
RESETTING_SEND_EMAIL_CONFIRM 事件 ← 发送前最后拦截点
  ↓
sendResettingEmailMessage() ← 发送邮件（控制器直接调用）
  ↓
setPasswordRequestedAt()
  ↓
updateUser() ← 持久化
  ↓
RESETTING_SEND_EMAIL_COMPLETED 事件
  ↓
跳转 check_email
```

**源码事实澄清**：
- **发送邮件在 setPasswordRequestedAt 和 updateUser 之前**，而不是之后
- `RESETTING_SEND_EMAIL_CONFIRM` 是发送前的确认事件，不是发送后的完成事件
- 四个事件的顺序是：INITIALIZE → RESET_REQUEST → CONFIRM → COMPLETED

**配置依据**：
- 邮件器：[config.yml L201](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L201-L201) `fos_user.service.mailer`
- 发件人：[config.yml L197-L199](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L197-L199) `fos_user.from_email`

**为什么用控制器直接调用而不是事件监听器**：
- 发送重置邮件是密码重置流程的**核心职责**，不是可选扩展
- 邮件发送本身就是控制器业务逻辑的一部分
- 相比注册确认，密码重置不需要在事件中做"是否发送"的决策

**四个 RESETTING_SEND_EMAIL_* 事件的作用**：
- `RESETTING_SEND_EMAIL_INITIALIZE` — 流程最开始，用户尚未查找到。可用于自定义用户查找逻辑。
- `RESETTING_RESET_REQUEST` — 查找到用户后，冷却检查前。可用于额外的用户状态校验。
- `RESETTING_SEND_EMAIL_CONFIRM` — **发送前确认**。最后一个可以拦截发送的点。如果监听器设置了 response，发送会被取消。
- `RESETTING_SEND_EMAIL_COMPLETED` — 发送完成后。可用于日志记录、审计追踪等。

> **澄清**：这四个事件是**扩展点**，不是**触发点**。邮件发送是控制器直接调用的，事件只是在各个阶段提供拦截机会。

---

### 链路三：双因素验证码邮件（2FA 认证流程触发）

**调用链依据**：
- `scheb/2fa-email` v5.13.2 是 scheb/2fa-bundle 的邮箱 2FA 扩展包
- 登录成功后（第一因素认证通过），2FA 框架会检查是否需要第二因素认证
- 如果用户启用了邮箱 2FA，系统会生成验证码并调用配置的 mailer 发送邮件
- Wallabag 配置了自定义 mailer：`scheb_two_factor.email.mailer: Wallabag\Mailer\AuthCodeMailer`

**User 实体需实现的接口**：
`Scheb\TwoFactorBundle\Model\Email\TwoFactorInterface`

| 方法 | 作用 |
|------|------|
| `isEmailAuthEnabled(): bool` | 是否启用邮箱双因素认证 |
| `getEmailAuthRecipient(): string` | 验证码接收邮箱 |
| `getEmailAuthCode(): string` | 获取当前验证码 |
| `setEmailAuthCode(string $authCode): void` | 设置验证码 |

**AuthCodeMailerInterface 接口**：

```php
interface AuthCodeMailerInterface
{
    public function sendAuthCode(TwoFactorInterface $user): void;
}
```

**完整调用链**：

```
1. 用户提交登录表单（用户名 + 密码）
   │
   ▼
2. Symfony Security 认证成功（第一因素通过）
   │
   ▼
3. SchebTwoFactorBundle 2FA 防火墙拦截
   │
   ├─► 遍历所有启用的 2FA 提供者
   └─► 对每个提供者检查是否需要对用户进行 2FA
   │
   ▼
4. 邮箱 2FA 提供者检查
   │
   ├─► 检查用户是否实现 Email\TwoFactorInterface
   └─► 调用 isEmailAuthEnabled() 判断是否启用
   │
   ▼
5. 如果需要邮箱 2FA，进入准备阶段
   │
   ├─► 生成随机验证码（digits 配置控制位数，默认 6 位）
   ├─► 保存到用户对象：setEmailAuthCode($code)
   └─► 调用配置的 mailer 发送邮件
       $this->mailer->sendAuthCode($user)
   │
   ▼
6. Wallabag\Mailer\AuthCodeMailer::sendAuthCode($user)
   │
   ├─► 加载模板：TwoFactor/email_auth_code.html.twig
   ├─► 渲染 subject 块
   │   └─► 使用 wallabag_user 翻译域
   ├─► 渲染 body_html 块（完整 HTML 邮件）
   │   └─► 变量：user, code, support_url
   ├─► 渲染 body_text 块（纯文本版本）
   │   └─► 变量：user, code, support_url
   ├─► 构建 Email 对象
   │   ├─► 发件人：scheb_two_factor.email.sender_email
   │   ├─► 收件人：getEmailAuthRecipient()
   │   ├─► text() + html() 双版本
   │   └─► subject
   └─► 调用 $this->mailer->send($email)
   │
   ▼
7. Symfony Mailer 发送邮件
   │
   ▼
8. 用户跳转到 2FA 输入页面（/2fa）
   │
   ▼
9. 用户查收邮件，输入验证码
```

**配置依据**：
- 开关：[config.yml L230](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L230-L230) `scheb_two_factor.email.enabled`
- 发件人：[config.yml L231](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L231-L231) `scheb_two_factor.email.sender_email`
- 自定义邮件器：[config.yml L234](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L234-L234) `scheb_two_factor.email.mailer`
- 验证码位数：[config.yml L232](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L232-L232) `scheb_two_factor.email.digits`

**代码依据**：
- 自定义邮件器实现：[AuthCodeMailer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Mailer/AuthCodeMailer.php)
- 邮件模板：[email_auth_code.html.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/TwoFactor/email_auth_code.html.twig)
- User 实体实现：[User.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Entity/User.php#L285-L303)

**为什么用 2FA 认证流程而不是事件**：
- SchebTwoFactorBundle 是独立的安全认证模块，遵循 Symfony Security 的设计模式
- 双因素认证有自己的完整流程（开始 → 准备 → 验证 → 完成）
- 通过 `TwoFactorProviderInterface` 接口实现策略模式，支持多种 2FA 方式
- 邮件发送是邮箱 2FA 提供者准备阶段的标准操作

**触发时机说明**：
- 不是通过事件触发，而是在 2FA 认证流程中被框架调用
- 默认在登录成功后立即触发准备阶段

---

## 关键文件索引

### Wallabag 自身代码

| 文件 | 作用 |
|------|------|
| [src/Mailer/AuthCodeMailer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Mailer/AuthCodeMailer.php) | 双因素认证邮件发送器 |
| [src/Event/EntrySavedEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntrySavedEvent.php) | 文章保存事件 |
| [src/Event/EntryDeletedEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntryDeletedEvent.php) | 文章删除事件 |
| [src/Event/ConfigUpdatedEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/ConfigUpdatedEvent.php) | 配置更新事件 |
| [src/Event/Listener/CreateConfigListener.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/CreateConfigListener.php) | 用户创建时创建配置 |
| [src/Event/Listener/RegistrationListener.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/RegistrationListener.php) | 注册开关控制 |
| [src/Event/Listener/PasswordResettingListener.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/PasswordResettingListener.php) | 密码重置成功跳转 |

### 模板文件

| 文件 | 作用 |
|------|------|
| [templates/TwoFactor/email_auth_code.html.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/TwoFactor/email_auth_code.html.twig) | 双因素邮件模板 |
| [templates/Mail/forgotPassword.txt.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/Mail/forgotPassword.txt.twig) | 遗留的密码重置模板（未使用） |

### 配置文件

| 文件 | 作用 |
|------|------|
| [app/config/config.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml) | 主配置（mailer、fos_user、scheb_two_factor） |
| [app/config/services.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/services.yml) | 服务配置与依赖注入参数 |
| [composer.lock](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/composer.lock) | 依赖版本锁定 |

---

## 常见疑问解答

**Q: 密码重置邮件到底是事件触发还是控制器直接调用？**
A: **控制器直接调用**。FOSUserBundle 的 `ResettingController::sendEmailAction()` 方法中直接调用 `$this->mailer->sendResettingEmailMessage($user)` 发送邮件。虽然有四个相关事件（INITIALIZE、RESET_REQUEST、CONFIRM、COMPLETED），但它们是各个阶段的扩展钩子，不是触发点。

**Q: RESETTING_SEND_EMAIL_CONFIRM 存在吗？它的作用是什么？**
A: **存在**。`RESETTING_SEND_EMAIL_CONFIRM` 是发送邮件前的确认事件，位于生成 token 之后、发送邮件之前。这是发送前的最后一个拦截点，如果监听器在事件上设置了 response，邮件将不会发送，直接返回那个响应。

**Q: 四个 RESETTING_SEND_EMAIL_* 事件的顺序是什么？**
A: 完整顺序是：
1. `RESETTING_SEND_EMAIL_INITIALIZE` — 流程开始，用户尚未查找
2. `RESETTING_RESET_REQUEST` — 查找到用户后，冷却检查前
3. `RESETTING_SEND_EMAIL_CONFIRM` — 生成 token 后，发送邮件前
4. `RESETTING_SEND_EMAIL_COMPLETED` — 发送邮件且保存用户后

**Q: sendResettingEmailMessage 和 setPasswordRequestedAt 哪个先执行？**
A: **先发送邮件，再设置 passwordRequestedAt**。准确顺序是：
1. `sendResettingEmailMessage()` — 发送邮件
2. `setPasswordRequestedAt()` — 设置请求时间
3. `updateUser()` — 持久化到数据库

这样设计的原因可能是：即使邮件发送失败，也不会记录为已发送过，方便用户重试。

**Q: 注册确认邮件为什么用事件监听器？**
A: 因为注册确认是**可选功能**，由 `confirmation.enabled` 开关控制。通过事件监听器模式，可以在不修改控制器代码的情况下，有条件地注册服务。监听器服务只在启用确认功能时才会被注册到容器中。

**Q: 双因素验证码邮件是事件触发的吗？**
A: **不是**。它由 scheb/2fa-email 的认证流程在登录成功后触发。当第一因素（密码）认证成功后，2FA 框架检测到用户启用了邮箱 2FA，就会生成验证码并调用配置的 mailer 服务发送邮件。

**Q: Wallabag 自身代码里为什么找不到直接发邮件的地方？**
A: 因为邮件发送逻辑主要封装在两个第三方 Bundle 内部。Wallabag 只负责：
1. 在配置中指定使用哪个邮件器服务
2. 为双因素认证提供自定义的 `AuthCodeMailer` 实现
3. 为双因素邮件提供自定义模板

**Q: `templates/Mail/forgotPassword.txt.twig` 是做什么用的？**
A: 这个文件是早期版本遗留下来的。当前版本使用 FOSUserBundle 的 `fos_user.mailer.twig_symfony` 邮件器，它使用 bundle 内部的 `@FOSUser/Resetting/email.txt.twig` 模板。该文件目前没有被任何代码引用，可以视为死代码。

**Q: 如何添加新的邮件通知类型？**
A: 标准做法：
1. 如果需要解耦，创建自定义事件类
2. 在业务逻辑的合适位置触发事件
3. 创建事件订阅者监听该事件
4. 在订阅者中注入 `MailerInterface` 并发送邮件
5. 创建对应的 Twig 邮件模板（使用块模式）

简单做法：直接在控制器/服务中注入 `MailerInterface` 并发送邮件，不经过事件层。

**Q: 邮件发送是同步还是异步？**
A: 默认是**同步发送**的。如果需要异步，可以配置 Symfony Messenger 组件将邮件发送推送到队列中处理。当前代码中没有看到 Messenger 的相关配置。
