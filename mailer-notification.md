# Wallabag 邮件通知系统三层协作机制

## 总览

Wallabag 的邮件通知系统由三层构成：**事件订阅层** → **邮件分发层** → **模板渲染层**。

但核心要点是：**Wallabag 自身代码中没有直接发送邮件的监听器或控制器**。所有邮件发送都由两个第三方 Bundle 内部完成：
- **FOSUserBundle** — 负责注册确认邮件和密码重置邮件
- **SchebTwoFactorBundle** — 负责双因素验证码邮件

```
用户操作 ──► Bundle内部控制器/监听器 ──► Mailer服务 ──► Twig模板渲染 ──► Symfony Mailer发送
```

---

## 三条邮件链路对照表

| 链路 | 触发方式 | 触发事件/时机 | 邮件器服务 | 模板路径 | 模板来源 |
|------|---------|--------------|-----------|---------|---------|
| **注册确认邮件** | 事件监听器 | `FOSUserEvents::REGISTRATION_SUCCESS` | `fos_user.mailer.twig_symfony` | `@FOSUserBundle/Registration/email.txt.twig` | FOSUserBundle 内置 |
| **密码重置邮件** | 控制器直接调用 | `ResettingController::sendEmailAction` | `fos_user.mailer.twig_symfony` | `@FOSUserBundle/Resetting/email.txt.twig` | FOSUserBundle 内置 |
| **双因素验证码邮件** | Bundle 内部直接调用 | 登录成功后检测到启用邮箱 2FA | `Wallabag\Mailer\AuthCodeMailer` | `TwoFactor/email_auth_code.html.twig` | Wallabag 自定义 |

> **关键区别**：注册确认走**事件监听器**模式，密码重置走**控制器直接调用**模式，双因素走**Bundle 内部服务调用**模式。

---

## 第一层：事件订阅层

### 1.1 事件体系

Wallabag 中的事件分为两类：

#### Wallabag 自定义事件

定义在 `src/Event/` 目录下，**都不直接触发邮件**：

| 事件类 | 事件名常量 | 触发时机 | 用途 |
|--------|-----------|----------|------|
| [EntrySavedEvent](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntrySavedEvent.php) | `entry.saved` | 文章保存时 | 触发图片下载等 |
| [EntryDeletedEvent](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/EntryDeletedEvent.php) | `entry.deleted` | 文章删除时 | 清理图片等 |
| [ConfigUpdatedEvent](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/ConfigUpdatedEvent.php) | `config.updated` | 配置更新时 | 重新生成 CSS 等 |

#### FOSUserBundle 事件

来自 `friendsofsymfony/user-bundle`，其中部分事件**间接触发邮件**：

| 事件常量 | 触发时机 | 是否触发邮件 |
|----------|----------|-------------|
| `FOSUserEvents::REGISTRATION_INITIALIZE` | 注册流程初始化 | 否（Wallabag 用来控制注册开关） |
| `FOSUserEvents::REGISTRATION_SUCCESS` | 注册表单提交成功、用户保存前 | **是**（注册确认邮件） |
| `FOSUserEvents::REGISTRATION_COMPLETED` | 注册完成后 | 否（Wallabag 用来创建用户配置） |
| `FOSUserEvents::USER_CREATED` | 用户被手动创建时 | 否 |
| `FOSUserEvents::RESETTING_RESET_REQUEST` | 密码重置请求（发送邮件前） | 否 |
| `FOSUserEvents::RESETTING_SEND_EMAIL_INITIALIZE` | 发送重置邮件前 | 否 |
| `FOSUserEvents::RESETTING_SEND_EMAIL_COMPLETED` | 发送重置邮件后 | 否 |
| `FOSUserEvents::RESETTING_RESET_SUCCESS` | 密码重置成功 | 否（Wallabag 用来跳转首页） |

### 1.2 自定义事件触发点

**EntrySavedEvent 触发位置**（共 12 处）：
- [EntryController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Controller/EntryController.php) — 手动创建/编辑文章
- [EntryRestController.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Controller/Api/EntryRestController.php) — API 接口（5处）
- [AbstractImport.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Import/AbstractImport.php) — 导入流程
- [HtmlImport.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Import/HtmlImport.php) — HTML 导入
- [BrowserImport.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Import/BrowserImport.php) — 浏览器导入
- [AbstractConsumer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Consumer/AbstractConsumer.php) — 队列消费
- [ReloadEntryCommand.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Command/ReloadEntryCommand.php) — 命令行重载

典型触发方式：
```php
$this->eventDispatcher->dispatch(new EntrySavedEvent($entry), EntrySavedEvent::NAME);
```

**FOSUserEvents::USER_CREATED 手动触发位置**：
- [UserController::newAction](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Controller/UserController.php#L58) — 后台创建用户
- [UserRestController](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Controller/Api/UserRestController.php#L172) — API 创建用户
- [InstallCommand](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Command/InstallCommand.php#L331) — 安装命令
- [UserFixtures](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/fixtures/UserFixtures.php#L34) — 数据夹具

### 1.3 Wallabag 自身的事件监听器/订阅者

位于 `src/Event/Listener/` 和 `src/Event/Subscriber/`，**都不直接发送邮件**：

| 监听器/订阅者 | 订阅的事件 | 作用 |
|--------------|-----------|------|
| [RegistrationListener](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/RegistrationListener.php) | `REGISTRATION_INITIALIZE` | 注册开关控制，禁用时重定向 |
| [PasswordResettingListener](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/PasswordResettingListener.php) | `RESETTING_RESET_SUCCESS` | 重置成功后跳转到首页 |
| [CreateConfigListener](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Listener/CreateConfigListener.php) | `REGISTRATION_COMPLETED`, `USER_CREATED` | 创建用户默认配置 |
| [DownloadImagesSubscriber](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Subscriber/DownloadImagesSubscriber.php) | `entry.saved`, `entry.deleted` | 下载/清理文章图片 |
| [GenerateCustomCSSSubscriber](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Event/Subscriber/GenerateCustomCSSSubscriber.php) | `config.updated` | 生成自定义 CSS |

> **重点**：Wallabag 自身代码中**没有任何事件监听器直接发送邮件**。邮件发送逻辑全部封装在 FOSUserBundle 和 SchebTwoFactorBundle 内部。

---

## 第二层：邮件分发层

### 2.1 配置总览

在 [config.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml) 中配置了三个邮件相关组件：

```yaml
# 1. Symfony 基础邮件传输层
framework:
    mailer:
        dsn: "%env(MAILER_DSN)%"

# 2. FOSUserBundle 邮件服务（用户注册/密码重置）
fos_user:
    from_email:
        address: "%env(WALLABAG_FROM_EMAIL)%"
        sender_name: wallabag
    service:
        mailer: fos_user.mailer.twig_symfony  # 选用 Twig + Symfony Mailer 实现
    registration:
        confirmation:
            enabled: "%env(bool:WALLABAG_CONFIRMATION_ENABLED)%"

# 3. SchebTwoFactor 邮件服务（双因素认证）
scheb_two_factor:
    email:
        enabled: true
        sender_email: "%env(WALLABAG_TWOFACTOR_SENDER)%"
        digits: 6
        mailer: Wallabag\Mailer\AuthCodeMailer  # 自定义邮件器
```

### 2.2 邮件器一：FOSUserBundle TwigMailer (`fos_user.mailer.twig_symfony`)

**类型**：FOSUserBundle 内置邮件器

**负责发送**：
1. 注册确认邮件
2. 密码重置邮件

**工作原理**：
- 实现 `FOS\UserBundle\Mailer\MailerInterface` 接口
- 使用 Twig 模板引擎渲染邮件内容
- 底层调用 Symfony `MailerInterface` 发送邮件
- 通过 `fos_user.from_email` 配置发件人

**调用方式**：
- 注册确认：通过 **事件监听器** `EmailConfirmationListener` 触发
- 密码重置：通过 **控制器直接调用** 邮件器

**为什么叫 twig_symfony**：
- `twig` 表示用 Twig 模板渲染邮件内容
- `symfony` 表示用 Symfony Mailer 组件发送邮件
- 这是 FOSUserBundle 3.x 推荐的邮件器实现

### 2.3 邮件器二：Wallabag AuthCodeMailer

**位置**：[src/Mailer/AuthCodeMailer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Mailer/AuthCodeMailer.php)

**类型**：自定义邮件器，实现 `Scheb\TwoFactorBundle\Mailer\AuthCodeMailerInterface` 接口

**负责发送**：双因素认证验证码邮件

**核心代码结构**：

```php
class AuthCodeMailer implements AuthCodeMailerInterface
{
    public function __construct(
        private readonly MailerInterface $mailer,   // Symfony Mailer 底层传输
        private readonly Environment $twig,         // Twig 模板引擎
        private $senderEmail,    // 发件人邮箱
        private $senderName,     // 发件人名称
        private $supportUrl,     // 支持页面 URL
    ) {}

    public function sendAuthCode(TwoFactorInterface $user): void
    {
        // 1. 加载自定义邮件模板
        $template = $this->twig->load('TwoFactor/email_auth_code.html.twig');
        
        // 2. 分别渲染三个 Twig 块
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
        
        // 3. 构建 Email 对象（同时包含 HTML 和纯文本版本）
        $email = (new Email())
            ->from(new Address($this->senderEmail, $this->senderName ?: $this->senderEmail))
            ->to($user->getEmailAuthRecipient())
            ->subject($subject)
            ->text($bodyText)
            ->html($bodyHtml);
        
        // 4. 调用 Symfony Mailer 发送
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

**调用方式**：由 SchebTwoFactorBundle 内部的认证流程直接调用，**不通过事件**。

### 2.4 邮件发送架构图

```
┌───────────────────────────────────────────────────────────────────┐
│                        触发层 (3 种方式)                           │
│                                                                   │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │ 注册确认邮件        │  │ 密码重置邮件     │  │ 双因素验证码  │  │
│  │ (事件监听器触发)    │  │ (控制器直接调用)  │  │ (服务直接调用) │  │
│  └──────────┬──────────┘  └────────┬─────────┘  └───────┬───────┘  │
└─────────────┼──────────────────────┼──────────────────────┼──────────┘
              │                      │                      │
              ▼                      ▼                      ▼
┌───────────────────────────────────────────────────────────────────┐
│                        邮件分发层                                  │
│                                                                   │
│  ┌──────────────────────────────────┐  ┌───────────────────────┐  │
│  │  fos_user.mailer.twig_symfony    │  │ Wallabag\Mailer\      │  │
│  │  (FOSUserBundle TwigMailer)      │  │   AuthCodeMailer      │  │
│  │  • 注册确认邮件                  │  │  • 双因素验证码邮件    │  │
│  │  • 密码重置邮件                  │  │                       │  │
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
| `@FOSUserBundle/Registration/email.txt.twig` | 注册确认邮件 | FOSUserBundle | subject / body_text | FOSUserBundle 内置 |
| `@FOSUserBundle/Resetting/email.txt.twig` | 密码重置邮件 | FOSUserBundle | subject / body_text | FOSUserBundle 内置 |
| [templates/TwoFactor/email_auth_code.html.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/TwoFactor/email_auth_code.html.twig) | 双因素验证码邮件 | AuthCodeMailer | subject / body_text / body_html | Wallabag 自定义 |
| [templates/Mail/forgotPassword.txt.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/Mail/forgotPassword.txt.twig) | 密码重置邮件（遗留） | 未使用 | 无 | Wallabag 旧代码 |

> **关于遗留模板**：`templates/Mail/forgotPassword.txt.twig` 是早期版本遗留下来的文件。当前版本使用 FOSUserBundle 的 `fos_user.mailer.twig_symfony` 邮件器，该文件已不再被使用。

### 3.2 块模式（Block-based）模板设计

所有邮件模板都采用 **Twig 块模式** 设计，即一个模板文件中包含多个命名块，分别渲染邮件的不同部分。

**FOSUserBundle 模板结构**（纯文本邮件）：
```twig
{# 只有 subject 和 body_text 两个块 #}
{% block subject %}...{% endblock %}
{% block body_text %}...{% endblock %}
```

**双因素邮件模板结构**（HTML + 纯文本）：
```twig
{% block subject %}
{{ "auth_code.mailer.subject"|trans({}, 'wallabag_user') }}
{% endblock %}

{% block body_text %}
{{ "auth_code.mailer.body.hello"|trans({'%user%': user}, 'wallabag_user') }}
{{ "auth_code.mailer.body.second_para"|trans({}, 'wallabag_user') }} {{ code }}
{% endblock %}

{% block body_html %}
<!DOCTYPE html>
<html>
    <!-- 完整的 HTML 邮件，包含内联 CSS 样式 -->
    <!-- 有 logo、卡片布局、页脚等 -->
</html>
{% endblock %}
```

### 3.3 模板渲染方式

**渲染流程**：

```php
// 1. 加载模板文件
$template = $this->twig->load('TwoFactor/email_auth_code.html.twig');

// 2. 分别渲染每个块
$subject = $template->renderBlock('subject', []);
$bodyHtml = $template->renderBlock('body_html', $context);
$bodyText = $template->renderBlock('body_text', $context);

// 3. 组装到 Email 对象
$email = (new Email())
    ->subject($subject)
    ->text($bodyText)
    ->html($bodyHtml);
```

### 3.4 模板覆盖机制

FOSUserBundle 的邮件模板可以通过在 `templates/bundles/FOSUserBundle/` 下创建同名文件来覆盖。

**Wallabag 当前覆盖情况**：
- ✅ 覆盖了页面模板（如 `Registration/check_email.html.twig`）
- ❌ 没有覆盖邮件模板（如 `Registration/email.txt.twig`）

如果需要自定义注册确认或密码重置邮件的内容，需要创建以下文件：
- `templates/bundles/FOSUserBundle/Registration/email.txt.twig`
- `templates/bundles/FOSUserBundle/Resetting/email.txt.twig`

---

## 链路详解：三条邮件的完整流程

### 链路一：注册确认邮件（事件监听器触发）

**前置条件**：`fos_user.registration.confirmation.enabled = true`

**完整流程**：

```
1. 用户在注册页面提交表单
   │
   ▼
2. FOSUserBundle RegistrationController::registerAction 处理
   │
   ├─► 验证表单数据
   ├─► 创建 User 对象
   └─► 触发 FOSUserEvents::REGISTRATION_SUCCESS 事件
   │
   ▼
3. FOSUserBundle 内部的 EmailConfirmationListener 监听到事件
   │
   ├─► 生成确认 token 并保存到用户对象
   └─► 调用 fos_user.mailer.twig_symfony 邮件服务
   │
   ▼
4. TwigMailer 加载模板 @FOSUserBundle/Registration/email.txt.twig
   │
   ├─► 渲染 subject 块
   └─► 渲染 body_text 块（包含确认链接）
   │
   ▼
5. 通过 Symfony MailerInterface 发送邮件
   │
   ▼
6. 用户收到包含确认链接的邮件
```

**关键配置**：
- 开关：[config.yml L196](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L196-L196) `fos_user.registration.confirmation.enabled`
- 邮件器：[config.yml L201](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L201-L201) `fos_user.service.mailer`
- 发件人：[config.yml L197-L199](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L197-L199) `fos_user.from_email`

**为什么是事件监听器而不是控制器直接调用**：
- 因为确认邮件是可选功能（由 `confirmation.enabled` 控制）
- 通过事件监听器可以在不修改控制器的情况下启用/禁用该功能
- 这是 Symfony 生态的典型设计：核心控制器触发事件，可选功能通过监听器扩展

---

### 链路二：密码重置邮件（控制器直接调用）

**完整流程**：

```
1. 用户在"忘记密码"页面提交邮箱
   │
   ▼
2. FOSUserBundle ResettingController::sendEmailAction 处理
   │
   ├─► 验证邮箱对应的用户是否存在
   ├─► 检查是否在冷却时间内（防止频繁发送）
   ├─► 生成重置 token 并保存到用户对象
   └─► 直接调用 $this->mailer->sendResettingEmailMessage($user)
   │
   ▼
3. fos_user.mailer.twig_symfony 邮件器发送邮件
   │
   ├─► 加载模板 @FOSUserBundle/Resetting/email.txt.twig
   ├─► 渲染 subject 块
   └─► 渲染 body_text 块（包含重置链接）
   │
   ▼
4. 通过 Symfony MailerInterface 发送邮件
   │
   ▼
5. 跳转到 "check_email" 页面提示用户查收邮件
   │
   ▼
6. 用户收到包含重置链接的邮件
```

**关键配置**：
- 邮件器：[config.yml L201](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L201-L201) `fos_user.service.mailer`
- 发件人：[config.yml L197-L199](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L197-L199) `fos_user.from_email`

**为什么是控制器直接调用而不是事件监听器**：
- 密码重置邮件是核心功能，不是可选扩展
- 发送邮件本身就是控制器的核心职责之一
- 相比注册确认，密码重置不需要在事件中决策是否发送

**相关事件**（仅供扩展，不直接触发邮件）：
- `FOSUserEvents::RESETTING_SEND_EMAIL_INITIALIZE` — 发送前
- `FOSUserEvents::RESETTING_SEND_EMAIL_COMPLETED` — 发送后

---

### 链路三：双因素验证码邮件（Bundle 内部服务调用）

**完整流程**：

```
1. 用户在登录页面提交用户名密码
   │
   ▼
2. Symfony Security 认证成功（第一因素通过）
   │
   ▼
3. SchebTwoFactorBundle 检测是否需要第二因素
   │
   ├─► 检查用户是否启用了邮箱双因素认证
   │   (User::isEmailAuthEnabled() === true)
   └─► 是，进入邮箱 2FA 流程
   │
   ▼
4. SchebTwoFactorBundle 生成随机验证码
   │
   ├─► 调用 AuthCodeManager::generateAndSend()
   ├─► 生成 6 位数字验证码
   ├─► 保存到用户对象（setEmailAuthCode）
   └─► 调用配置的 mailer 发送邮件
   │
   ▼
5. Wallabag\Mailer\AuthCodeMailer::sendAuthCode() 被调用
   │
   ├─► 加载模板 TwoFactor/email_auth_code.html.twig
   ├─► 渲染 subject 块
   ├─► 渲染 body_html 块（完整 HTML 邮件）
   └─► 渲染 body_text 块（纯文本版本）
   │
   ▼
6. 构建 Email 对象（同时包含 HTML + 纯文本）
   │
   ▼
7. 通过 Symfony MailerInterface 发送邮件
   │
   ▼
8. 用户跳转到 2FA 输入页面
   │
   ▼
9. 用户收到验证码邮件，在 2FA 页面输入
```

**关键配置**：
- 开关：[config.yml L230](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L230-L230) `scheb_two_factor.email.enabled`
- 发件人：[config.yml L231](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L231-L231) `scheb_two_factor.email.sender_email`
- 自定义邮件器：[config.yml L234](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L234-L234) `scheb_two_factor.email.mailer`
- 验证码位数：[config.yml L232](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml#L232-L232) `scheb_two_factor.email.digits`

**User 实体相关接口**：
- 实现 `Scheb\TwoFactorBundle\Model\Email\TwoFactorInterface`
- 关键方法：`isEmailAuthEnabled()`、`getEmailAuthCode()`、`setEmailAuthCode()`、`getEmailAuthRecipient()`

**相关文件**：
- 邮件器：[AuthCodeMailer.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Mailer/AuthCodeMailer.php)
- 模板：[email_auth_code.html.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/TwoFactor/email_auth_code.html.twig)
- 用户实体：[User.php](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/src/Entity/User.php#L285-L303)

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
| [templates/Mail/forgotPassword.txt.twig](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/templates/Mail/forgotPassword.txt.twig) | 遗留的密码重置模板（未使用） |

### 配置相关

| 文件 | 作用 |
|------|------|
| [app/config/config.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/config.yml) | 主配置（mailer、fos_user、scheb_two_factor） |
| [app/config/services.yml](file:///d:/fz/0508-2/solo-dogfeeding/code/109-wallabag/app/config/services.yml) | 服务配置与依赖注入参数 |

---

## 常见疑问解答

**Q: 为什么 Wallabag 自身代码里找不到直接发邮件的地方？**
A: 因为邮件发送逻辑全部封装在 FOSUserBundle 和 SchebTwoFactorBundle 内部。Wallabag 只负责：
1. 在配置中指定使用哪个邮件器服务
2. 为双因素认证提供自定义的 AuthCodeMailer
3. 为双因素邮件提供自定义模板

**Q: 密码重置邮件到底是事件触发还是控制器直接调用？**
A: 控制器直接调用。FOSUserBundle 的 `ResettingController::sendEmailAction` 方法中直接调用邮件器的 `sendResettingEmailMessage()` 方法发送邮件。虽然也有 `RESETTING_SEND_EMAIL_INITIALIZE` 和 `RESETTING_SEND_EMAIL_COMPLETED` 事件，但这些事件是在发送前后用于扩展的，不是用来触发邮件发送的。

**Q: 注册确认邮件为什么用事件监听器而不是控制器直接调用？**
A: 因为注册确认是可选功能（由 `confirmation.enabled` 控制）。通过事件监听器模式，可以在不修改控制器代码的情况下，根据配置决定是否启用邮件确认功能。这是 Symfony 生态中"可选功能通过事件扩展"的典型设计。

**Q: 双因素验证码邮件是怎么触发的？**
A: 由 SchebTwoFactorBundle 的认证流程直接调用。当用户第一因素（密码）认证成功后，Bundle 检测到用户启用了邮箱双因素认证，就会生成验证码并调用配置的 mailer 服务发送邮件。这个过程不经过 Symfony EventDispatcher。

**Q: `templates/Mail/forgotPassword.txt.twig` 是做什么用的？**
A: 这个文件是早期版本遗留下来的模板。当前版本使用 FOSUserBundle 的 `fos_user.mailer.twig_symfony` 邮件器，它使用 bundle 内部的模板。该文件目前没有被任何代码引用，可以视为死代码。

**Q: 如何添加新的邮件通知类型？**
A: 标准做法是：
1. 创建自定义事件类（如果需要）
2. 在业务逻辑的合适位置触发事件
3. 创建事件订阅者监听该事件
4. 在订阅者中注入邮件服务并发送邮件
5. 创建对应的邮件模板

也可以简化为：直接在控制器/服务中注入 `MailerInterface` 并发送邮件，不经过事件层。

**Q: 邮件发送是同步还是异步？**
A: 默认是同步发送的。如果需要异步，可以配置 Symfony Messenger 组件将邮件发送推送到队列中处理。当前代码中没有看到 Messenger 的配置。
