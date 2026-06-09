# Wallabag 邮件通知系统三层协作机制

## 总览

Wallabag 的邮件通知系统由三层构成：**事件订阅层** → **邮件分发层** → **模板渲染层**。用户操作触发事件，事件监听器/订阅者响应并调用邮件服务，邮件服务使用 Twig 模板渲染邮件内容后通过 Symfony Mailer 发送。

```
用户操作 ──► 事件触发 ──► 事件监听器/订阅者 ──► Mailer服务 ──► Twig模板渲染 ──► 发送邮件
```

---

## 第一层：事件订阅层

### 1.1 事件体系

Wallabag 使用 Symfony EventDispatcher 组件，事件分为两类：

**自定义事件**（定义在 `src/Event/`）：

| 事件类 | 事件名常量 | 触发时机 |
|--------|-----------|----------|
| [EntrySavedEvent](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntrySavedEvent.php) | `entry.saved` | 文章保存时 |
| [EntryDeletedEvent](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntryDeletedEvent.php) | `entry.deleted` | 文章删除时 |
| [ConfigUpdatedEvent](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/ConfigUpdatedEvent.php) | `config.updated` | 用户配置更新时 |

**FOSUserBundle 事件**（来自 `friendsofsymfony/user-bundle`）：

| 事件常量 | 触发时机 |
|----------|----------|
| `FOSUserEvents::REGISTRATION_INITIALIZE` | 注册流程初始化 |
| `FOSUserEvents::REGISTRATION_COMPLETED` | 注册完成（不管是否需要邮箱验证） |
| `FOSUserEvents::REGISTRATION_SUCCESS` | 注册表单提交成功 |
| `FOSUserEvents::USER_CREATED` | 用户被创建（命令行/API/后台） |
| `FOSUserEvents::RESETTING_RESET_SUCCESS` | 密码重置成功 |
| `FOSUserEvents::RESETTING_RESET_REQUEST` | 密码重置请求（发送邮件前） |

### 1.2 事件触发点

**自定义事件触发示例**：

- 文章保存事件在多处触发：
  - [EntryController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Controller/EntryController.php) - 手动创建/编辑文章
  - [EntryRestController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Controller/Api/EntryRestController.php) - API 接口
  - [AbstractImport.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Import/AbstractImport.php) - 导入流程
  - [AbstractConsumer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Consumer/AbstractConsumer.php) - 队列消费
  - [ReloadEntryCommand.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Command/ReloadEntryCommand.php) - 命令行重载

```php
// 典型触发方式
$this->eventDispatcher->dispatch(new EntrySavedEvent($entry), EntrySavedEvent::NAME);
```

**FOSUserBundle 事件触发**：
- 由 FOSUserBundle 内部控制器自动触发（注册、登录、重置密码流程）
- 也可手动触发，如：
  - [UserController::newAction](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Controller/UserController.php#L58) - 后台创建用户
  - [UserRestController](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Controller/Api/UserRestController.php#L172) - API 创建用户
  - [InstallCommand](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Command/InstallCommand.php#L331) - 安装命令

### 1.3 事件监听器/订阅者

项目中的监听器位于 `src/Event/Listener/`，订阅者位于 `src/Event/Subscriber/`。

**与邮件相关的监听器**：

> 注意：Wallabag 自身代码中没有直接监听事件来发送邮件的监听器。邮件发送主要由 FOSUserBundle 和 SchebTwoFactorBundle 内部完成。

| 监听器类 | 订阅的事件 | 作用 |
|---------|-----------|------|
| [RegistrationListener](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/RegistrationListener.php) | `REGISTRATION_INITIALIZE` | 注册开关控制，不直接发邮件 |
| [PasswordResettingListener](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/PasswordResettingListener.php) | `RESETTING_RESET_SUCCESS` | 重置成功后的跳转，不直接发邮件 |
| [CreateConfigListener](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/CreateConfigListener.php) | `REGISTRATION_COMPLETED`, `USER_CREATED` | 创建用户配置，不发邮件 |

**与邮件无关的事件订阅者**（供参考）：

| 订阅者类 | 订阅的事件 | 作用 |
|---------|-----------|------|
| [DownloadImagesSubscriber](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php) | `entry.saved`, `entry.deleted` | 下载/清理文章图片 |
| [GenerateCustomCSSSubscriber](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Subscriber/GenerateCustomCSSSubscriber.php) | `config.updated` | 生成自定义 CSS |

---

## 第二层：邮件分发层

### 2.1 邮件服务配置

在 [config.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml) 中配置了两类邮件服务：

```yaml
# Symfony 基础邮件器配置
framework:
    mailer:
        dsn: "%env(MAILER_DSN)%"

# FOSUserBundle 邮件器配置
fos_user:
    from_email:
        address: "%env(WALLABAG_FROM_EMAIL)%"
        sender_name: wallabag
    service:
        mailer: fos_user.mailer.twig_symfony  # 使用 Twig + Symfony Mailer

# SchebTwoFactor 双因素邮件器配置
scheb_two_factor:
    email:
        enabled: true
        sender_email: "%env(WALLABAG_TWOFACTOR_SENDER)%"
        mailer: Wallabag\Mailer\AuthCodeMailer  # 自定义邮件器
```

### 2.2 邮件器分类

#### 2.2.1 FOSUserBundle 邮件器 (`fos_user.mailer.twig_symfony`)

这是 FOSUserBundle 内置的邮件器，负责发送：
- **注册确认邮件** - 用户注册后需要邮箱验证时
- **密码重置邮件** - 用户请求重置密码时

该邮件器由 FOSUserBundle 内部调用，监听相关事件并自动发送邮件。它使用 Twig 模板渲染邮件内容。

关键配置点：
- 通过 `fos_user.service.mailer` 指定使用 `fos_user.mailer.twig_symfony`
- 发件人信息通过 `fos_user.from_email` 配置
- 模板由 bundle 内置，可在 `templates/bundles/FOSUserBundle/` 下覆盖

#### 2.2.2 自定义双因素邮件器 ([AuthCodeMailer](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Mailer/AuthCodeMailer.php))

实现了 `Scheb\TwoFactorBundle\Mailer\AuthCodeMailerInterface` 接口，专门用于发送双因素认证验证码邮件。

**核心代码结构**：

```php
class AuthCodeMailer implements AuthCodeMailerInterface
{
    public function __construct(
        private readonly MailerInterface $mailer,   // Symfony Mailer
        private readonly Environment $twig,         // Twig 环境
        private $senderEmail,
        private $senderName,
        private $supportUrl,
    ) {}

    public function sendAuthCode(TwoFactorInterface $user): void
    {
        // 1. 加载模板
        $template = $this->twig->load('TwoFactor/email_auth_code.html.twig');
        
        // 2. 渲染三个区块：subject, body_html, body_text
        $subject = $template->renderBlock('subject', []);
        $bodyHtml = $template->renderBlock('body_html', [...] %);
        $bodyText = $template->renderBlock('body_text', [...]);
        
        // 3. 构建 Email 对象
        $email = (new Email())
            ->from(new Address($this->senderEmail, $this->senderName))
            ->to($user->getEmailAuthRecipient())
            ->subject($subject)
            ->text($bodyText)
            ->html($bodyHtml);
        
        // 4. 发送
        $this->mailer->send($email);
    }
}
```

**依赖注入参数**（在 [services.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/services.yml) 中配置）：

- `$senderEmail`: 来自 `scheb_two_factor.email.sender_email`
- `$senderName`: 来自 `scheb_two_factor.email.sender_name`
- `$supportUrl`: 来自 `craue_config` 的 `wallabag_support_url` 配置项

### 2.3 邮件发送流程图

```
┌─────────────────────────────────────────────────────────────┐
│                     事件触发层                               │
│  - 用户注册 / 密码重置 / 双因素认证                           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                     邮件分发层                               │
│                                                             │
│  ┌──────────────────────┐    ┌──────────────────────────┐   │
│  │ FOSUserBundle Mailer │    │  AuthCodeMailer          │   │
│  │  (注册确认/密码重置)  │    │  (双因素验证码)           │   │
│  │  fos_user.mailer.    │    │  Wallabag\Mailer\        │   │
│  │  twig_symfony        │    │  AuthCodeMailer          │   │
│  └──────────┬───────────┘    └────────────┬─────────────┘   │
│             │                             │                 │
│             └─────────────┬───────────────┘                 │
│                           │                                 │
│                           ▼                                 │
│              Symfony MailerInterface                        │
│              (底层邮件传输，支持 SMTP 等)                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     模板渲染层                               │
│                  Twig 模板渲染引擎                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 第三层：模板渲染层

### 3.1 模板位置与结构

项目中的邮件模板分布在以下位置：

| 模板路径 | 用途 | 由谁使用 |
|---------|------|---------|
| [templates/TwoFactor/email_auth_code.html.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/TwoFactor/email_auth_code.html.twig) | 双因素验证码邮件 | AuthCodeMailer |
| [templates/bundles/FOSUserBundle/...](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/bundles/FOSUserBundle/) | 覆盖 FOSUserBundle 的页面模板（非邮件） | FOSUserBundle |
| [templates/Mail/forgotPassword.txt.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/Mail/forgotPassword.txt.twig) | 密码重置邮件（疑似遗留文件，未被使用） | 未知 |

### 3.2 双因素邮件模板结构

[email_auth_code.html.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/TwoFactor/email_auth_code.html.twig) 采用 **块模式（Block-based）** 设计，包含三个 Twig 块：

```twig
{# 1. 邮件主题 #}
{% block subject %}
{{ "auth_code.mailer.subject"|trans({}, 'wallabag_user') }}
{% endblock %}

{# 2. 纯文本版本 #}
{% block body_text %}
{{ "auth_code.mailer.body.hello"|trans({'%user%': user}, 'wallabag_user') }}
...
{% endblock %}

{# 3. HTML 版本（完整样式） #}
{% block body_html %}
<!DOCTYPE html>
<html>...完整的 HTML 邮件模板...</html>
{% endblock %}
```

**模板变量**：
- `user`: 用户名
- `code`: 验证码
- `support_url`: 支持页面 URL
- `wallabag_url`: 站点 URL（通过全局 Twig 变量）

### 3.3 FOSUserBundle 邮件模板

FOSUserBundle 的邮件模板由 bundle 内部提供，遵循相同的块模式：

- `@FOSUserBundle/Registration/email.txt.twig` - 注册确认邮件（纯文本）
- `@FOSUserBundle/Resetting/email.txt.twig` - 密码重置邮件（纯文本）

这些模板可以通过在 `templates/bundles/FOSUserBundle/` 下创建同名文件来覆盖。

> **注意**：Wallabag 项目中目前只覆盖了 FOSUserBundle 的页面模板（check_email.html.twig 等），没有覆盖邮件模板。

### 3.4 模板渲染方式

**AuthCodeMailer 的渲染方式**：

```php
$template = $this->twig->load('TwoFactor/email_auth_code.html.twig');

// 分别渲染三个块
$subject = $template->renderBlock('subject', []);
$bodyHtml = $template->renderBlock('body_html', $context);
$bodyText = $template->renderBlock('body_text', $context);
```

**FOSUserBundle 的渲染方式**：

FOSUserBundle 的 `TwigSymfonyMailer` 使用类似的块模式渲染邮件，但它是从 bundle 内部模板加载的。

---

## 具体场景：邮件通知完整流程

### 场景一：用户注册并需要邮件确认

```
1. 用户提交注册表单
   ↓
2. FOSUserBundle RegistrationController 处理
   ↓
3. 触发 FOSUserEvents::REGISTRATION_SUCCESS 事件
   ↓
4. FOSUserBundle 内部的邮件监听器捕获事件
   ↓
5. 调用 fos_user.mailer.twig_symfony 邮件服务
   ↓
6. 加载注册确认邮件模板（@FOSUserBundle/Registration/email.txt.twig）
   ↓
7. 使用 Twig 渲染邮件内容
   ↓
8. 通过 Symfony MailerInterface 发送邮件
   ↓
9. 用户收到包含确认链接的邮件
```

**配置开关**：`fos_user.registration.confirmation.enabled` 控制是否需要邮件确认

### 场景二：用户请求密码重置

```
1. 用户在登录页点击"忘记密码"
   ↓
2. FOSUserBundle ResettingController 处理请求
   ↓
3. 触发 FOSUserEvents::RESETTING_RESET_REQUEST 事件
   ↓
4. FOSUserBundle 内部邮件监听器捕获事件
   ↓
5. 调用 fos_user.mailer.twig_symfony 邮件服务
   ↓
6. 加载密码重置邮件模板（@FOSUserBundle/Resetting/email.txt.twig）
   ↓
7. 使用 Twig 渲染（包含重置链接）
   ↓
8. 通过 Symfony MailerInterface 发送邮件
   ↓
9. 用户收到包含重置链接的邮件
```

### 场景三：双因素认证验证码

```
1. 用户登录成功（第一因素）
   ↓
2. SchebTwoFactorBundle 检测用户启用了邮箱双因素认证
   ↓
3. 生成随机验证码并保存到用户对象
   ↓
4. 调用 Wallabag\Mailer\AuthCodeMailer::sendAuthCode()
   ↓
5. 加载 TwoFactor/email_auth_code.html.twig 模板
   ↓
6. 渲染 subject / body_html / body_text 三个块
   ↓
7. 构建 Email 对象（同时包含 HTML 和纯文本版本）
   ↓
8. 通过 Symfony MailerInterface 发送邮件
   ↓
9. 用户跳转到 2FA 输入页，同时收到验证码邮件
```

### 场景四：管理员手动创建用户

```
1. 管理员在后台提交新用户表单
   ↓
2. UserController::newAction 处理
   ↓
3. 通过 UserManager 创建用户（设置 enabled=true）
   ↓
4. 手动触发 FOSUserEvents::USER_CREATED 事件
   ↓
5. CreateConfigListener 捕获事件，创建用户配置
   ↓
   （注意：不会发送注册确认邮件，因为用户是被管理员创建的）
```

---

## 关键文件索引

### 事件相关

| 文件 | 作用 |
|------|------|
| [src/Event/EntrySavedEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntrySavedEvent.php) | 文章保存事件 |
| [src/Event/EntryDeletedEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntryDeletedEvent.php) | 文章删除事件 |
| [src/Event/ConfigUpdatedEvent.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/ConfigUpdatedEvent.php) | 配置更新事件 |
| [src/Event/Listener/CreateConfigListener.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/CreateConfigListener.php) | 用户创建时创建配置 |
| [src/Event/Listener/RegistrationListener.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/RegistrationListener.php) | 注册开关控制 |
| [src/Event/Listener/PasswordResettingListener.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/PasswordResettingListener.php) | 密码重置成功跳转 |

### 邮件相关

| 文件 | 作用 |
|------|------|
| [src/Mailer/AuthCodeMailer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Mailer/AuthCodeMailer.php) | 双因素认证邮件发送器 |
| [tests/unit/Mailer/AuthCodeMailerTest.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/tests/unit/Mailer/AuthCodeMailerTest.php) | 邮件发送器单元测试 |

### 模板相关

| 文件 | 作用 |
|------|------|
| [templates/TwoFactor/email_auth_code.html.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/TwoFactor/email_auth_code.html.twig) | 双因素邮件模板 |
| [templates/Mail/forgotPassword.txt.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/Mail/forgotPassword.txt.twig) | 疑似遗留的密码重置模板 |

### 配置相关

| 文件 | 作用 |
|------|------|
| [app/config/config.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml) | 主配置（mailer、fos_user、scheb_two_factor） |
| [app/config/services.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/services.yml) | 服务配置与依赖注入参数 |

---

## 常见疑问解答

**Q: 为什么代码中找不到直接监听事件来发邮件的地方？**
A: 因为注册确认和密码重置邮件由 FOSUserBundle 内部的事件监听器和邮件服务完成，双因素邮件由 SchebTwoFactorBundle 内部触发。Wallabag 自身代码只负责自定义双因素邮件器的实现。

**Q: `templates/Mail/forgotPassword.txt.twig` 是做什么用的？**
A: 这个文件看起来是早期版本遗留下来的模板。当前版本使用 FOSUserBundle 的 `fos_user.mailer.twig_symfony` 邮件器，它使用 bundle 内部的模板。该文件可能已不再被使用。

**Q: 如何添加新的邮件通知类型？**
A: 1) 创建自定义事件类；2) 在合适的位置触发事件；3) 创建事件订阅者监听该事件；4) 在订阅者中调用邮件服务；5) 创建对应邮件模板。

**Q: 邮件发送使用的是同步还是异步？**
A: 默认是同步发送的。如果配置了 RabbitMQ 或 Redis，可以使用 Messenger 组件实现异步邮件，但当前代码中没有看到相关配置。
