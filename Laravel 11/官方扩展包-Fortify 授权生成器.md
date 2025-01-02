本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Fortify

+   [介绍](#introduction)
    +   [什么是 Fortify？](#what-is-fortify)
    +   [何时应该使用 Fortify？](#when-should-i-use-fortify)
+   [安装](#installation)
    +   [Fortify 功能](#fortify-features)
    +   [禁用视图](#disabling-views)
+   [身份验证](#authentication)
    +   [自定义用户身份验证](#customizing-user-authentication)
    +   [自定义身份验证管道](#customizing-the-authentication-pipeline)
    +   [自定义重定向](#customizing-authentication-redirects)
+   [双因素身份验证](#two-factor-authentication)
    +   [启用双因素身份验证](#enabling-two-factor-authentication)
    +   [使用双因素身份验证进行身份验证](#authenticating-with-two-factor-authentication)
    +   [禁用双因素身份验证](#disabling-two-factor-authentication)
+   [注册](#registration)
    +   [自定义注册](#customizing-registration)
+   [重置密码](#password-reset)
    +   [请求重置密码链接](#requesting-a-password-reset-link)
    +   [重置密码](#resetting-the-password)
    +   [自定义密码重置](#customizing-password-resets)
+   [电子邮件验证](#email-verification)
    +   [保护路由](#protecting-routes)
+   [密码确认](#password-confirmation)

## 介绍

[Laravel Fortify](https://github.com/laravel/fortify) 是 Laravel 的一个与前端无关的身份验证后端实现。Fortify 注册了实现所有 Laravel 身份验证功能所需的路由和控制器，包括登录、注册、重置密码、电子邮件验证等。安装 Fortify 后，你可以运行 `route:list` Artisan 命令查看 Fortify 注册的路由。

由于 Fortify 不提供自己的用户界面，它旨在与你自己的用户界面配对，该界面向注册的路由发出请求。我们将在本文档的其余部分讨论如何向这些路由发出请求。

> **注意**  
> 请记住，Fortify 是一个旨在帮助你快速实现 Laravel 身份验证功能的包。**你不必使用它。** 你始终可以按照[身份验证](https://learnku.com/docs/laravel/11.x/authentication)、[密码重置](https://learnku.com/docs/laravel/11.x/passwords)和[电子邮件验证](https://learnku.com/docs/laravel/11.x/verification)文档中提供的文档手动与 Laravel 的身份验证服务交互。

### 什么是 Fortify？

正如前面提到的，Laravel Fortify 是 Laravel 的一个与前端无关的身份验证后端实现。Fortify 注册了实现所有 Laravel 身份验证功能所需的路由和控制器，包括登录、注册、重置密码、电子邮件验证等。

**你不必使用 Fortify 来使用 Laravel 的身份验证功能。** 你始终可以按照[身份验证](https://learnku.com/docs/laravel/11.x/authentication)、[密码重置](https://learnku.com/docs/laravel/11.x/passwords)和[电子邮件验证](https://learnku.com/docs/laravel/11.x/verification)文档中提供的文档手动与 Laravel 的身份验证服务交互。

如果你是 Laravel 的新手，你可能希望在尝试使用 Laravel Fortify 之前先探索 [Laravel Breeze](https://learnku.com/docs/laravel/11.x/starter-kits) 应用程序起始套件。Laravel Breeze 为你的应用程序提供了一个身份验证脚手架，包括一个使用 [Tailwind CSS](https://tailwindcss.com/) 构建的用户界面。与 Fortify 不同，Breeze 将其路由和控制器直接发布到你的应用程序中。这使你可以在允许 Laravel Fortify 为你实现这些功能之前，研究和熟悉 Laravel 的身份验证功能。

Laravel Fortify 本质上将 Laravel Breeze 的路由和控制器作为一个不包含用户界面的包提供。这使你仍然可以快速搭建应用程序身份验证层的后端实现，而不受任何特定前端观点的约束。

### 何时应该使用 Fortify？

你可能想知道何时适合使用 Laravel Fortify。首先，如果你正在使用 Laravel 的[应用程序起始套件](https://learnku.com/docs/laravel/11.x/starter-kits)，你无需安装 Laravel Fortify，因为所有 Laravel 的应用程序起始套件已经提供了完整的身份验证实现。

如果你没有使用应用程序起始套件并且你的应用程序需要身份验证功能，你有两个选择：手动实现应用程序的身份验证功能或者使用 Laravel Fortify 提供这些功能的后端实现。

如果你选择安装 Fortify，你的用户界面将向 Fortify 的身份验证路由发出请求，这些路由在本文档中有详细说明，用于对用户进行身份验证和注册。

如果你选择手动与 Laravel 的身份验证服务交互而不是使用 Fortify，你可以按照[身份验证](https://learnku.com/docs/laravel/11.x/authentication)、[密码重置](https://learnku.com/docs/laravel/11.x/passwords)和[电子邮件验证](https://learnku.com/docs/laravel/11.x/verification)文档中提供的文档进行操作。

#### Laravel Fortify 和 Laravel Sanctum

一些开发人员对 [Laravel Sanctum](https://learnku.com/docs/laravel/11.x/sanctum) 和 Laravel Fortify 之间的区别感到困惑。因为这两个包解决了两个不同但相关的问题，Laravel Fortify 和 Laravel Sanctum 不是互斥或竞争的包。

Laravel Sanctum 仅关注管理 API 令牌和使用会话 cookie 或令牌对现有用户进行身份验证。Sanctum 不提供处理用户注册、密码重置等功能的路由。

如果你正在尝试手动构建一个应用程序的身份验证层，该应用程序提供 API 或用作单页面应用程序的后端，那么完全可以同时使用 Laravel Fortify（用于用户注册、密码重置等）和 Laravel Sanctum（API 令牌管理、会话身份验证）。

## 安装

要开始使用，使用 Composer 包管理器安装 Fortify：

```shell
composer require laravel/fortify
```

接下来，使用 `fortify:install` Artisan 命令发布 Fortify 的资源：

```shell
php artisan fortify:install
```

该命令将发布 Fortify 的操作到你的 `app/Actions` 目录中，如果该目录不存在则会创建。此外，`FortifyServiceProvider`、配置文件和所有必要的数据库迁移也将被发布。

接下来，你应该迁移数据库：

### Fortify 功能

`fortify` 配置文件包含一个 `features` 配置数组。该数组定义了 Fortify 默认会暴露的后端路由 / 功能。如果你没有将 Fortify 与 [Laravel Jetstream](https://jetstream.laravel.com/) 结合使用，我们建议你只启用以下功能，这些功能是大多数 Laravel 应用程序提供的基本身份验证功能：

```php
'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::emailVerification(),
],
```

### 禁用视图

默认情况下，Fortify 定义了用于返回视图的路由，比如登录界面或注册界面。但是，如果你正在构建一个由 JavaScript 驱动的单页面应用程序，你可能不需要这些路由。因此，你可以通过在你的应用程序的 `config/fortify.php` 配置文件中将 `views` 配置值设置为 `false` 来完全禁用这些路由：

#### 禁用视图和密码重置

如果你选择禁用 Fortify 的视图，并且你将为你的应用程序实现密码重置功能，你仍然应该定义一个名为 `password.reset` 的路由，该路由负责显示你的应用程序的 “重置密码” 视图。这是必要的，因为 Laravel 的 `Illuminate\Auth\Notifications\ResetPassword` 通知将通过 `password.reset` 命名路由生成密码重置 URL。

## 身份验证

首先，我们需要指示 Fortify 如何返回我们的 “登录” 视图。请记住，Fortify 是一个无头身份验证库。如果你想要一个已经为你完成的 Laravel 身份验证功能的前端实现，你应该使用一个[应用程序起始套件](https://learnku.com/docs/laravel/11.x/starter-kits)。

所有身份验证视图的渲染逻辑可以使用 `Laravel\Fortify\Fortify` 类中提供的适当方法进行自定义。通常情况下，你应该在你的应用程序的 `App\Providers\FortifyServiceProvider` 类的 `boot` 方法中调用这个方法。Fortify 将负责定义返回此视图的 `/login` 路由：

```php
use Laravel\Fortify\Fortify;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Fortify::loginView(function () {
        return view('auth.login');
    });

    // ...
}
```

你的登录模板应包含一个提交 POST 请求到 `/login` 的表单。`/login` 端点期望一个字符串 `email` / `username` 和一个 `password`。邮箱 / 用户名字段的名称应与 `config/fortify.php` 配置文件中的 `username` 值匹配。此外，可以提供一个布尔型 `remember` 字段，以指示用户是否希望使用 Laravel 提供的 “记住我” 功能。

如果登录尝试成功，Fortify 将重定向你到通过应用程序的 `fortify` 配置文件中的 `home` 配置选项配置的 URI。如果登录请求是 XHR 请求，将返回一个 200 HTTP 响应。

如果请求不成功，用户将被重定向回登录界面，并且验证错误将通过共享的 `$errors` [Blade 模板变量](https://learnku.com/docs/laravel/11.x/validation#quick-displaying-the-validation-errors)提供给你。或者，在 XHR 请求的情况下，验证错误将在 422 HTTP 响应中返回。

### 自定义用户认证

Fortify 将根据提供的凭据和为你的应用程序配置的身份验证守卫自动检索和验证用户。然而，有时你可能希望完全定制登录凭据的验证方式和用户的检索方式。幸运的是，Fortify 允许你使用 `Fortify::authenticateUsing` 方法轻松实现这一点。

这个方法接受一个闭包，该闭包接收传入的 HTTP 请求。该闭包负责验证附加到请求的登录凭据并返回相应的用户实例。如果凭据无效或找不到用户，则闭包应返回 `null` 或 `false`。通常情况下，这个方法应该从你的 `FortifyServiceProvider` 的 `boot` 方法中调用：

```php
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Laravel\Fortify\Fortify;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Fortify::authenticateUsing(function (Request $request) {
        $user = User::where('email', $request->email)->first();

        if ($user &&
            Hash::check($request->password, $user->password)) {
            return $user;
        }
    });

    // ...
}
```

#### 身份验证守卫

你可以在应用程序的 `fortify` 配置文件中自定义 Fortify 使用的身份验证守卫。但是，你应确保配置的守卫是 `Illuminate\Contracts\Auth\StatefulGuard` 的实现。如果你尝试使用 Laravel Fortify 来验证单页面应用程序（SPA），你应该结合使用 Laravel 的默认 `web` 守卫和 [Laravel Sanctum](https://laravel.com/docs/sanctum)。

### 自定义身份验证流程

Laravel Fortify 通过一系列可调用类来验证登录请求。如果你愿意，你可以定义一个自定义的类流程，用于处理登录请求。每个类应该有一个接收传入的 `Illuminate\Http\Request` 实例的 `__invoke` 方法，并且像 [中间件](https://learnku.com/docs/laravel/11.x/middleware) 一样，一个 `$next` 变量，用于调用以将请求传递给管道中的下一个类。

要定义你的自定义流程，你可以使用 `Fortify::authenticateThrough` 方法。这个方法接受一个闭包，该闭包应返回一个类数组，用于处理登录请求。通常情况下，这个方法应该从你的 `App\Providers\FortifyServiceProvider` 类的 `boot` 方法中调用。

下面的示例包含了默认的流程定义，你可以将其用作制作自定义修改的起点：

```php
use Laravel\Fortify\Actions\AttemptToAuthenticate;
use Laravel\Fortify\Actions\EnsureLoginIsNotThrottled;
use Laravel\Fortify\Actions\PrepareAuthenticatedSession;
use Laravel\Fortify\Actions\RedirectIfTwoFactorAuthenticatable;
use Laravel\Fortify\Fortify;
use Illuminate\Http\Request;

Fortify::authenticateThrough(function (Request $request) {
    return array_filter([
            config('fortify.limiters.login') ? null : EnsureLoginIsNotThrottled::class,
            Features::enabled(Features::twoFactorAuthentication()) ? RedirectIfTwoFactorAuthenticatable::class : null,
            AttemptToAuthenticate::class,
            PrepareAuthenticatedSession::class,
    ]);
});
```

#### 身份验证节流

默认情况下，Fortify 将使用 `EnsureLoginIsNotThrottled` 中间件来节流身份验证尝试。这个中间件会对与用户名和 IP 地址组合唯一的尝试进行节流。

一些应用程序可能需要不同的身份验证尝试节流方法，比如仅按 IP 地址节流。因此，Fortify 允许你通过 `fortify.limiters.login` 配置选项指定你自己的[速率限制器](https://learnku.com/docs/laravel/11.x/routing#rate-limiting)。当然，这个配置选项位于你的应用程序的 `config/fortify.php` 配置文件中。

> **注意**  
> 结合使用节流、[双因素身份验证](https://learnku.com/docs/laravel/11.x/fortify#two-factor-authentication)和外部网络应用程序防火墙（WAF）将为你的合法应用程序用户提供最强大的防御。

### 自定义重定向

如果登录尝试成功，Fortify 将重定向你到通过应用程序的 `fortify` 配置文件中的 `home` 配置选项配置的 URI。如果登录请求是 XHR 请求，将返回一个 200 HTTP 响应。用户登出应用程序后，用户将被重定向到 `/` URI。

如果你需要对这个行为进行高级定制，你可以将 `LoginResponse` 和 `LogoutResponse` 接口的实现绑定到 Laravel 的[服务容器](https://learnku.com/docs/laravel/11.x/container)中。通常情况下，这应该在你的应用程序的 `App\Providers\FortifyServiceProvider` 类的 `register` 方法中完成：

```php
use Laravel\Fortify\Contracts\LogoutResponse;

/**
 * 注册任何应用服务。
 */
public function register(): void
{
    $this->app->instance(LogoutResponse::class, new class implements LogoutResponse {
        public function toResponse($request)
        {
            return redirect('/');
        }
    });
}
```

## 双因素身份验证

当启用 Fortify 的双因素身份验证功能时，用户在身份验证过程中需要输入一个六位数字令牌。这个令牌是使用基于时间的一次性密码（TOTP）生成的，可以从任何支持 TOTP 的移动身份验证应用程序（如 Google Authenticator）中获取。

在开始之前，你应该首先确保你的应用程序的 `App\Models\User` 模型使用了 `Laravel\Fortify\TwoFactorAuthenticatable` 特性：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Fortify\TwoFactorAuthenticatable;

class User extends Authenticatable
{
    use Notifiable, TwoFactorAuthenticatable;
}
```

接下来，你应该在你的应用程序中构建一个屏幕，让用户可以管理他们的双因素身份验证设置。这个屏幕应该允许用户启用和禁用双因素身份验证，以及重新生成他们的双因素身份验证恢复码。

> 默认情况下，`fortify` 配置文件中的 `features` 数组指示 Fortify 的双因素身份验证设置在修改前需要密码确认。因此，在继续之前，你的应用程序应该实现 Fortify 的[密码确认](#password-confirmation)功能。

### 启用双因素身份验证

要开始启用双因素身份验证，你的应用程序应该向 Fortify 定义的 `/user/two-factor-authentication` 端点发起 POST 请求。如果请求成功，用户将被重定向回先前的 URL，并且 `status` 会话变量将被设置为 `two-factor-authentication-enabled`。你可以在模板中检测这个 `status` 会话变量，以显示适当的成功消息。如果请求是 XHR 请求，将返回 `200` HTTP 响应。

在选择启用双因素身份验证后，用户仍需要通过提供有效的双因素身份验证代码来 “确认” 他们的双因素身份验证配置。因此，你的 “成功” 消息应该指示用户仍然需要双因素身份验证确认：

```html
@if (session('status') == 'two-factor-authentication-enabled')
    <div class="mb-4 font-medium text-sm">
        请在下面完成双因素身份验证配置。
    </div>
@endif
```

接下来，你应该显示双因素身份验证的 QR 码，以便用户可以扫描到他们的认证器应用程序中。如果你在 Blade 中渲染你的应用程序的前端，你可以使用用户实例上可用的 `twoFactorQrCodeSvg` 方法来检索 QR 码的 SVG：

```php
$request->user()->twoFactorQrCodeSvg();
```

如果你正在构建一个由 JavaScript 驱动的前端，你可以向 `/user/two-factor-qr-code` 端点发起 XHR GET 请求，以检索用户的双因素身份验证 QR 码。这个端点将返回一个包含 `svg` 键的 JSON 对象。

#### 确认双因素身份验证

除了显示用户的双因素身份验证 QR 码之外，你还应该提供一个文本输入框，用户可以在其中提供有效的身份验证代码来 “确认” 他们的双因素身份验证配置。这个代码应该通过向 Fortify 定义的 `/user/confirmed-two-factor-authentication` 端点发起 POST 请求提供给 Laravel 应用程序。

如果请求成功，用户将被重定向回先前的 URL，并且 `status` 会话变量将被设置为 `two-factor-authentication-confirmed`：

```html
@if (session('status') == 'two-factor-authentication-confirmed')
    <div class="mb-4 font-medium text-sm">
        双因素身份验证已成功确认和启用。
    </div>
@endif
```

如果通过 XHR 请求向双因素身份验证确认端点发起请求，将返回 `200` HTTP 响应。

#### 显示恢复码

你还应该显示用户的双因素身份验证恢复码。这些恢复码允许用户在丢失移动设备访问权限时进行身份验证。如果你在 Blade 中渲染你的应用程序的前端，你可以通过认证用户实例访问恢复码：

```php
(array) $request->user()->recoveryCodes()
```

如果你正在构建一个由 JavaScript 驱动的前端，你可以向 `/user/two-factor-recovery-codes` 端点发起 XHR GET 请求。这个端点将返回一个包含用户恢复码的 JSON 数组。

要重新生成用户的恢复码，你的应用程序应该向 `/user/two-factor-recovery-codes` 端点发起 POST 请求。

### 使用双因素身份验证进行认证

在身份验证过程中，Fortify 将自动将用户重定向到你的应用程序的双因素身份验证挑战屏幕。然而，如果你的应用程序正在进行 XHR 登录请求，成功认证尝试后返回的 JSON 响应将包含一个具有 `two_factor` 布尔属性的 JSON 对象。你应该检查这个值，以确定是否应该重定向到你的应用程序的双因素身份验证挑战屏幕。

要开始实现双因素身份验证功能，我们需要指示 Fortify 如何返回我们的双因素身份验证挑战视图。所有 Fortify 的身份验证视图渲染逻辑都可以使用 `Laravel\Fortify\Fortify` 类提供的适当方法进行自定义。通常情况下，你应该从你的应用程序的 `App\Providers\FortifyServiceProvider` 类的 `boot` 方法中调用这个方法：

```php
use Laravel\Fortify\Fortify;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Fortify::twoFactorChallengeView(function () {
        return view('auth.two-factor-challenge');
    });

    // ...
}
```

Fortify 会负责定义返回此视图的 `/two-factor-challenge` 路由。你的 `two-factor-challenge` 模板应包含一个表单，该表单会向 `/two-factor-challenge` 端点发起 POST 请求。`/two-factor-challenge` 操作需要一个包含有效 TOTP 令牌的 `code` 字段，或者一个包含用户恢复码之一的 `recovery_code` 字段。

如果登录尝试成功，Fortify 将重定向用户到通过你的应用程序 `fortify` 配置文件中的 `home` 配置选项配置的 URI。如果登录请求是一个 XHR 请求，则将返回一个 204 HTTP 响应。

如果请求不成功，用户将被重定向回双因素身份验证挑战屏幕，并且验证错误将通过共享的 `$errors` Blade 模板变量提供给你。或者，在 XHR 请求的情况下，验证错误将在 422 HTTP 响应中返回。

### 禁用双因素身份验证

要禁用双因素身份验证，你的应用程序应该向 `/user/two-factor-authentication` 端点发起 DELETE 请求。请记住，Fortify 的双因素身份验证端点在调用之前需要[密码确认](#password-confirmation)。

## 注册

要开始实现我们应用程序的注册功能，我们需要指示 Fortify 如何返回我们的 “register” 视图。请记併，Fortify 是一个无头身份验证库。如果你想要一个已经为你完成的 Laravel 身份验证功能的前端实现，你应该使用一个[应用程序起始工具包](https://learnku.com/docs/laravel/11.x/starter-kits)。

所有 Fortify 的视图渲染逻辑都可以使用 `Laravel\Fortify\Fortify` 类提供的适当方法进行自定义。通常情况下，你应该从你的 `App\Providers\FortifyServiceProvider` 类的 `boot` 方法中调用这个方法：

```php
use Laravel\Fortify\Fortify;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Fortify::registerView(function () {
        return view('auth.register');
    });

    // ...
}
```

Fortify 会负责定义返回此视图的 `/register` 路由。你的 `register` 模板应包含一个表单，该表单会向 Fortify 定义的 `/register` 端点发起 POST 请求。

`/register` 端点需要一个字符串 `name` 、字符串电子邮件地址 / 用户名、`password` 和 `password_confirmation` 字段。电子邮件 / 用户名字段的名称应与你的应用程序 `fortify` 配置文件中定义的 `username` 配置值匹配。

如果注册尝试成功，Fortify 将重定向用户到通过你的应用程序 `fortify` 配置文件中的 `home` 配置选项配置的 URI 。如果请求是一个 XHR 请求，则将返回一个 201 HTTP 响应。

如果请求不成功，用户将被重定向回注册屏幕，并且验证错误将通过共享的 `$errors` Blade 模板变量提供给你。或者，在 XHR 请求的情况下，验证错误将在 422 HTTP 响应中返回。

### 自定义注册

用户验证和创建过程可以通过修改安装 Laravel Fortify 时生成的 `App\Actions\Fortify\CreateNewUser` 动作来进行自定义。

## 密码重置

### 请求密码重置链接

要开始实现我们应用程序的密码重置功能，我们需要指示 Fortify 如何返回我们的「forgot password」视图。请记併，Fortify 是一个无头身份验证库。如果你想要一个已经为你完成的 Laravel 身份验证功能的前端实现，你应该使用一个[应用程序起始工具包](https://learnku.com/docs/laravel/11.x/starter-kits)。

所有 Fortify 的视图渲染逻辑都可以使用 `Laravel\Fortify\Fortify` 类提供的适当方法进行自定义。通常情况下，你应该从你的应用程序 `App\Providers\FortifyServiceProvider` 类的 `boot` 方法中调用这个方法：

```php
use Laravel\Fortify\Fortify;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Fortify::requestPasswordResetLinkView(function () {
        return view('auth.forgot-password');
    });

    // ...
}
```

Fortify 会负责定义返回此视图的 `/forgot-password` 路由。你的 `forgot-password` 模板应包含一个表单，该表单会向 `/forgot-password` 端点发起 POST 请求。

`/forgot-password` 端点需要一个字符串 `email` 字段。这个字段 / 数据库列的名称应与你的应用程序 `fortify` 配置文件中定义的 `email` 配置值匹配。

#### 处理密码重置链接请求响应

如果密码重置链接请求成功，Fortify 将重定向用户回到 `/forgot-password` 端点，并向用户发送一封包含安全链接的电子邮件，用户可以使用该链接重置密码。如果请求是一个 XHR 请求，则将返回一个 200 HTTP 响应。

在成功请求后重定向回 `/forgot-password` 端点后，`status` 会话变量可用于显示密码重置链接请求尝试的状态。

`$status` 会话变量的值将匹配你的应用程序 `passwords` [语言文件](https://learnku.com/docs/laravel/11.x/localization)中定义的翻译字符串之一。如果你想自定义此值并且尚未发布 Laravel 的语言文件，你可以通过 `lang:publish` Artisan 命令来执行此操作：

```html
@if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
@endif
```

如果请求不成功，用户将被重定向回请求密码重置链接屏幕，并且验证错误将通过共享的 `$errors`[Blade 模板变量](https://learnku.com/docs/laravel/11.x/validation#quick-displaying-the-validation-errors) 提供给你。或者，在 XHR 请求的情况下，验证错误将以 422 HTTP 响应返回。

### 重置密码

要完成我们应用程序的密码重置功能的实现，我们需要指示 Fortify 如何返回我们的「重置密码」视图。

所有 Fortify 的视图渲染逻辑都可以使用 `Laravel\Fortify\Fortify` 类提供的适当方法进行自定义。通常情况下，你应该从你的应用程序 `App\Providers\FortifyServiceProvider` 类的 `boot` 方法中调用这个方法：

```php
use Laravel\Fortify\Fortify;
use Illuminate\Http\Request;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Fortify::resetPasswordView(function (Request $request) {
        return view('auth.reset-password', ['request' => $request]);
    });

    // ...
}
```

Fortify 会负责定义显示此视图的路由。你的 `reset-password` 模板应包含一个表单，该表单会向 `/reset-password` 发起 POST 请求。

`/reset-password` 端点需要一个字符串 `email` 字段，一个 `password` 字段，一个 `password_confirmation` 字段，以及一个名为 `token` 的隐藏字段，其中包含 `request()->route('token')` 的值。 「email」字段 / 数据库列的名称应与你的应用程序 `fortify` 配置文件中定义的 `email` 配置值匹配。

#### 处理密码重置响应

如果密码重置请求成功，Fortify 将重定向回 `/login` 路由，以便用户可以使用新密码登录。此外，将设置一个 `status` 会话变量，以便在登录屏幕上显示重置的成功状态：

```blade
@if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
@endif
```

如果请求是一个 XHR 请求，将返回一个 200 HTTP 响应。

如果请求不成功，用户将被重定向回重置密码屏幕，并且验证错误将通过共享的 `$errors` [Blade 模板变量](https://learnku.com/docs/laravel/11.x/validation#quick-displaying-the-validation-errors)提供给你。或者，在 XHR 请求的情况下，验证错误将以 422 HTTP 响应返回。

### 自定义密码重置

密码重置流程可以通过修改在安装 Laravel Fortify 时生成的 `App\Actions\ResetUserPassword` 操作来进行自定义。

## 电子邮件验证

注册后，你可能希望用户在继续访问你的应用程序之前验证他们的电子邮件地址。要开始，请确保在你的 `fortify` 配置文件的 `features` 数组中启用 `emailVerification` 功能。接下来，你应该确保你的 `App\Models\User` 类实现了 `Illuminate\Contracts\Auth\MustVerifyEmail` 接口。

完成这两个设置步骤后，新注册用户将收到一封提示他们验证电子邮件地址所有权的电子邮件。然而，我们需要告诉 Fortify 如何显示电子邮件验证屏幕，提示用户需要点击电子邮件中的验证链接。

所有 Fortify 视图的渲染逻辑都可以使用 `Laravel\Fortify\Fortify` 类提供的适当方法进行自定义。通常情况下，你应该从你的应用程序 `App\Providers\FortifyServiceProvider` 类的 `boot` 方法中调用这个方法：

```php
use Laravel\Fortify\Fortify;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Fortify::verifyEmailView(function () {
        return view('auth.verify-email');
    });

    // ...
}
```

Fortify 会负责定义路由，当用户被 Laravel 内置的 `verified` 中间件重定向到 `/email/verify` 端点时，显示此视图。

你的 `verify-email` 模板应包含一条信息性消息，指示用户点击发送到他们电子邮件地址的电子邮件验证链接。

#### 重新发送电子邮件验证链接

如果你希望，在你的应用程序的 `verify-email` 模板中添加一个按钮，触发一个向 `/email/verification-notification` 端点的 POST 请求。当此端点收到请求时，将向用户发送一个新的验证电子邮件链接，允许用户获取新的验证链接，如果之前的链接被意外删除或丢失。

如果重新发送验证链接电子邮件的请求成功，Fortify 将重定向用户回 `/email/verify` 端点，并设置一个 `status` 会话变量，允许你向用户显示一条信息性消息，告知操作成功。如果请求是一个 XHR 请求，则将返回一个 202 HTTP 响应：

```blade
@if (session('status') == 'verification-link-sent')
    <div class="mb-4 font-medium text-sm text-green-600">
        一个新的电子邮件验证链接已发送到您的邮箱！
    </div>
@endif
```

### 保护路由

要指定一个路由或一组路由要求用户已验证他们的电子邮件地址，你应该将 Laravel 内置的 `verified` 中间件附加到路由上。`verified` 中间件别名由 Laravel 自动注册，并作为 `Illuminate\Routing\Middleware\ValidateSignature` 中间件的别名：

```php
Route::get('/dashboard', function () {
    // ...
})->middleware(['verified']);
```

## 密码确认

在构建你的应用程序时，你可能偶尔会有需要用户在执行操作之前确认他们的密码的情况。通常情况下，这些路由受 Laravel 内置的 `password.confirm` 中间件保护。

要开始实现密码确认功能，我们需要指示 Fortify 如何返回我们应用程序的 “密码确认” 视图。请记住，Fortify 是一个无头身份验证库。如果你想要一个已为你完成的 Laravel 身份验证功能的前端实现，你应该使用一个[应用程序起始工具包](https://learnku.com/docs/laravel/11.x/starter-kits)。

所有 Fortify 的视图渲染逻辑都可以使用 `Laravel\Fortify\Fortify` 类提供的适当方法进行自定义。通常情况下，你应该从你的应用程序 `App\Providers\FortifyServiceProvider` 类的 `boot` 方法中调用这个方法：

```php
use Laravel\Fortify\Fortify;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Fortify::confirmPasswordView(function () {
        return view('auth.confirm-password');
    });

    // ...
}
```

Fortify 会负责定义返回此视图的 `/user/confirm-password` 路由。你的 `confirm-password` 模板应包含一个表单，该表单会向 `/user/confirm-password` 端点发起 POST 请求。`/user/confirm-password` 端点期望一个包含用户当前密码的 `password` 字段。

如果密码与用户的当前密码匹配，Fortify 将重定向用户到他们试图访问的路由。如果请求是一个 XHR 请求，则将返回一个 201 HTTP 响应。

如果请求不成功，用户将被重定向回确认密码屏幕，并且验证错误将通过共享的 `$errors` Blade 模板变量提供给你。或者，在 XHR 请求的情况下，验证错误将以 422 HTTP 响应返回。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/fo...](https://learnku.com/docs/laravel/11.x/fortifymd/16718)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/fo...](https://learnku.com/docs/laravel/11.x/fortifymd/16718)