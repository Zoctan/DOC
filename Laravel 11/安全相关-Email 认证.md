本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## 邮件认证

+   [介绍](#introduction)
    +   [准备模型](#model-preparation)
    +   [准备数据库](#database-preparation)
+   [路由](#verification-routing)
    +   [邮件验证通知](#the-email-verification-notice)
    +   [邮件验证处理](#the-email-verification-handler)
    +   [重新发送验证邮件](#resending-the-verification-email)
    +   [保护路由](#protecting-routes)
+   [自定义](#customization)
+   [事件](#events)

## 介绍

很多 Web 应用会要求用户在使用之前进行 Email 地址验证。Laravel 不会强迫你在每个应用中手动重复实现该特性，而是提供了方便的内置服务来发送和校验电子邮件的验证请求。

> 注意  
> 想快速上手吗？你可以在全新的应用中安装 [Laravel 应用入门套件](https://learnku.com/docs/laravel/11.x/starter-kits)。入门套件将帮助你搭建整个身份验证系统，包括电子邮件验证支持。

### 准备模型

在开始之前，需要检查你的 `App\Models\User` 模型是否实现了 `Illuminate\Contracts\Auth\MustVerifyEmail` 契约：

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable implements MustVerifyEmail
{
    use Notifiable;

    // ...
}
```

一旦这一接口被添加到模型中，新注册的用户将自动收到一封包含电子邮件验证链接的电子邮件。这是无缝发生的，因为 Laravel 自动为 `Illuminate\Auth\Events\Registered` 事件注册了 `Illuminate\Auth\Listeners\SendEmailVerificationNotification` [监听器](https://learnku.com/docs/laravel/11.x/events)。

如果你没有在应用中使用[入门套件](https://learnku.com/docs/laravel/11.x/starter-kits)而是手动实现的注册，你需要确保在用户注册成功后手动分发 `Illuminate\Auth\Events\Registered` 事件：

```php
use Illuminate\Auth\Events\Registered;

event(new Registered($user));
```

### 数据库准备

接下来，`users` 表中必须有一个 `email_verified_at` 字段，用以保存用户邮箱验证日期和时间。通常，这个字段在 Laravel 自带的 `0001_01_01_000000_create_users_table.php` 数据库迁移文件已经默认包含了。

## 路由

为了实现邮件认证，需要定义三个路由。首先，需要一个路由来向用户显示通知，通知用户在注册后，应该点击 Laravel 发送给他们的验证邮件中的链接。

其次，需要一个路由来处理用户点击邮件中验证链接时发来的请求。

第三，如果用户没有收到验证邮件，则需要一路由来重新发送验证邮件。

### 邮箱验证通知

如前所述，应该定义一条路由，该路由将返回一个视图，引导用户点击注册后 Laravel 发送给他们邮件中的验证链接。当用户尝试访问网站的其它页面而没有先完成邮箱验证时，将向用户显示此视图。请注意，只要你的 `App\Models\User` 模型实现了 `MustVerifyEmail` 接口，就会自动将该链接发邮件给用户：

```php
Route::get('/email/verify', function () {
    return view('auth.verify-email');
})->middleware('auth')->name('verification.notice');
```

返回邮件验证通知的路由应该命名为 `verification.notice`。将这个路由分配这个命名很重要，因为如果用户邮箱验证未通过，[Laravel 自带的 `verified` 中间件](#protecting-routes)将会自动重定向到该路由名上。

> 注意  
> 手动实现邮箱验证过程时，你需要自己定义验证通知视图。如果你希望包含所有必要的身份认证和验证视图，请查看 [Laravel 应用入门套件](https://learnku.com/docs/laravel/11.x/starter-kits)。

### 邮件验证处理

接下来，我们需要定义一个路由，它将在用户点击邮件验证链接时处理生成的请求。该路由应该命名为 `verification.verify`，并且为其分配 `auth` 和 `signed` 中间件：

```php
use Illuminate\Foundation\Auth\EmailVerificationRequest;

Route::get('/email/verify/{id}/{hash}', function (EmailVerificationRequest $request) {
    $request->fulfill();

    return redirect('/home');
})->middleware(['auth', 'signed'])->name('verification.verify');
```

在继续之前，让我们仔细看一下这个路由。首先，你会注意到我们使用的是 `EmailVerificationRequest` 请求类型，而不是通常的 `Illuminate\Http\Request` 实例。 `EmailVerificationRequest` 是 Laravel 中包含的[表单请求](https://learnku.com/docs/laravel/11.x/validation#form-request-validation)。此请求将自动处理验证请求的 `id` 和 `hash` 参数。

接下来，我们可以直接在请求上调用 `fulfill` 方法。该方法将在身份认证过的用户上调用 `markEmailAsVerified` 方法，并会触发 `Illuminate\Auth\Events\Verified` 事件。通过 `Illuminate\Foundation\Auth\User` 基类，`markEmailAsVerified` 方法可用于默认的 `App\Models\User` 模型。验证用户的电子邮件地址后，你可以将其重定向到任意位置。

### 重发验证邮件

有时候，用户可能输错了电子邮件地址或者不小心删除了验证邮件。为了解决这种问题，你可能会想定义一个路由允许用户请求重新发送验证邮件。你可以在[验证通知视图](#the-email-verification-notice)中放置一个简单的表单来实现此功能：

```php
use Illuminate\Http\Request;

Route::post('/email/verification-notification', function (Request $request) {
    $request->user()->sendEmailVerificationNotification();

    return back()->with('message', 'Verification link sent!');
})->middleware(['auth', 'throttle:6,1'])->name('verification.send');
```

### 保护路由

[路由中间件](https://learnku.com/docs/laravel/11.x/middleware)可用作只允许认证过的用户访问给定路由。Laravel 自带了一个 `verified` [中间件别名](https://learnku.com/docs/laravel/11.x/middleware#middleware-alias)，它是 `Illuminate\Auth\Middleware\EnsureEmailIsVerified` 中间件类的别名。由于该别名已经由 Laraval 自动注册，所以你只需将中间件附加到路由定义即可。通常，此中间件与 `auth` 中间件配对使用：

```php
Route::get('/profile', function () {
    // Only verified users may access this route...
})->middleware(['auth', 'verified']);
```

如果未经验证的用户尝试访问已经分配了该中间件的路由，将会被自动重定向到 `verification.notice` [命名路由](https://learnku.com/docs/laravel/11.x/routing#named-routes)。

## 自定义

#### 自定义验证邮件

尽管默认的邮件验证通知可以满足大部分应用的需求，Laravel 还是允许你自定义验证邮件消息的构建。

要开始自定义验证邮件，请将一个闭包传递给 `Illuminate\Auth\Notifications\VerifyEmail` 通知提供的 `toMailUsing` 方法。该闭包接收 notifiable 模型实例，该模型实例接收通知以及用户必须访问以验证邮箱地址的签名邮箱验证 URL。该闭包返回一个 `Illuminate\Notifications\Messages\MailMessage` 实例。通常，你应该在应用的 `AppServiceProvider` 类的 `boot` 方法中调用 `toMailUsing` 方法：

```php
use Illuminate\Auth\Notifications\VerifyEmail;
use Illuminate\Notifications\Messages\MailMessage;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    // ...

    VerifyEmail::toMailUsing(function (object $notifiable, string $url) {
        return (new MailMessage)
            ->subject('Verify Email Address')
            ->line('Click the button below to verify your email address.')
            ->action('Verify Email Address', $url);
    });
}
```

> 注意  
> 要了解更多关于邮件通知的内容，请参考[邮件通知文档](https://learnku.com/docs/laravel/11.x/notifications#mail-notifications)。

## 事件

使用 [Laravel 入门套件](https://learnku.com/docs/laravel/11.x/starter-kits)时，Laravel 在电子邮件验证过程中会派发 `Illuminate\Auth\Events\Verified` [事件](https://learnku.com/docs/laravel/11.x/events) 。如果你想让你的应用手动处理邮箱验证，你可以在验证完成后手动分发这些事件。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/ve...](https://learnku.com/docs/laravel/11.x/verificationmd/16692)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/ve...](https://learnku.com/docs/laravel/11.x/verificationmd/16692)