本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Socialite

+   [简介](#introduction)
+   [安装](#installation)
+   [升级](#upgrading-socialite)
+   [配置](#configuration)
+   [身份验证](#authentication)
    +   [路由](#routing)
    +   [身份验证和存储](#authentication-and-storage)
    +   [访问范围](#access-scopes)
    +   [机器人访问范围](#slack-bot-scopes)
    +   [可选参数](#optional-parameters)
+   [获取用户详细信息](#retrieving-user-details)

## 简介

除了典型的基于表单的身份验证外，Laravel 还提供了一种简单、方便的方式，使用 [Laravel Socialite](https://github.com/laravel/socialite) 与 OAuth 提供者进行身份验证。Socialite 目前支持通过 Facebook、Twitter、LinkedIn、Google、GitHub、GitLab、Bitbucket 和 Slack 进行身份验证。

> \[!NOTE\]  
> 社区驱动的 [Socialite Providers](https://socialiteproviders.com/) 网站提供了其他平台的适配器。

## 安装

要开始使用 Socialite，请使用 Composer 包管理器将包添加到项目的依赖项中：

```shell
composer require laravel/socialite
```

## 升级

升级到 Socialite 的新主要版本时，重要的是仔细查看[升级指南](https://github.com/laravel/socialite/blob/master/UPGRADE.md)。

## 配置

在使用 Socialite 之前，您需要为您的应用程序使用的 OAuth 提供者添加凭据。通常，这些凭据可以通过在您将要进行身份验证的服务的仪表板中创建一个 “开发人员应用程序” 来检索。

这些凭据应放置在您的应用程序的 `config/services.php` 配置文件中，并且应使用键 `facebook`、`twitter`（OAuth 1.0）、`twitter-oauth-2`（OAuth 2.0）、`linkedin-openid`、`google`、`github`、`gitlab`、`bitbucket`、`slack` 或 `slack-openid`，具体取决于您的应用程序所需的提供者：

```php
'github' => [
    'client_id' => env('GITHUB_CLIENT_ID'),
    'client_secret' => env('GITHUB_CLIENT_SECRET'),
    'redirect' => 'http://example.com/callback-url',
],
```

> \[!NOTE\]  
> 如果 `redirect` 选项包含相对路径，它将自动解析为完全合格的 URL。

## 身份验证

### 路由

要使用 OAuth 提供者对用户进行身份验证，你需要两个路由：一个用于将用户重定向到 OAuth 提供者，另一个用于在身份验证后接收提供者的回调。下面的示例路由演示了这两个路由的实现：

```php
use Laravel\Socialite\Facades\Socialite;

Route::get('/auth/redirect', function () {
    return Socialite::driver('github')->redirect();
});

Route::get('/auth/callback', function () {
    $user = Socialite::driver('github')->user();

    // $user->token
});
```

`Socialite` 门面提供的 `redirect` 方法负责将用户重定向到 OAuth 提供者，而 `user` 方法将检查传入的请求并在用户批准身份验证请求后从提供者那里检索用户信息。

### 身份验证和存储

一旦从 OAuth 提供者检索到用户，你可以确定用户是否存在于您的应用程序数据库中，并[对用户进行身份验证](https://learnku.com/docs/laravel/11.x/authentication#authenticate-a-user-instance)。如果用户不存在于您的应用程序数据库中，通常会在数据库中创建一个新记录来表示该用户：

```php
use App\Models\User;
use Illuminate\Support\Facades\Auth;
use Laravel\Socialite\Facades\Socialite;

Route::get('/auth/callback', function () {
    $githubUser = Socialite::driver('github')->user();

    $user = User::updateOrCreate([
        'github_id' => $githubUser->id,
    ], [
        'name' => $githubUser->name,
        'email' => $githubUser->email,
        'github_token' => $githubUser->token,
        'github_refresh_token' => $githubUser->refreshToken,
    ]);

    Auth::login($user);

    return redirect('/dashboard');
});
```

> \[!NOTE\]  
> 有关从特定 OAuth 提供者检索用户信息的更多信息，请参阅[获取用户详细信息](#retrieving-user-details)上的文档。

### 访问范围

在重定向用户之前，你可以使用 `scopes` 方法来指定应包含在身份验证请求中的「范围」。此方法将会将之前指定的所有范围与你指定的范围合并：

```php
use Laravel\Socialite\Facades\Socialite;

return Socialite::driver('github')
    ->scopes(['read:user', 'public_repo'])
    ->redirect();
```

你可以使用 `setScopes` 方法重写身份验证请求中的所有现有范围：

```php
return Socialite::driver('github')
    ->setScopes(['read:user', 'public_repo'])
    ->redirect();
```

### Slack 机器人范围

Slack 的 API 提供了[不同类型的访问令牌](https://api.slack.com/authentication/token-types)，每种类型都有其自己的一组[权限范围](https://api.slack.com/scopes)。Socialite 兼容以下两种 Slack 访问令牌类型：

+   机器人（以 `xoxb-` 为前缀）
+   用户（以 `xoxp-` 为前缀）

默认情况下，`slack` 驱动程序将生成一个 `user` 令牌，并调用驱动程序的 `user` 方法将返回用户的详细信息。

如果你的应用程序将向由你的应用程序用户拥有的外部 Slack 工作区发送通知，则机器人令牌非常有用。要生成机器人令牌，请在将用户重定向到 Slack 进行身份验证之前调用 `asBotUser` 方法：

```php
return Socialite::driver('slack')
    ->asBotUser()
    ->setScopes(['chat:write', 'chat:write.public', 'chat:write.customize'])
    ->redirect();
```

此外，在 Slack 将用户重定向回你的应用程序进行身份验证后，必须在调用 `user` 方法之前调用 `asBotUser` 方法：

```php
$user = Socialite::driver('slack')->asBotUser()->user();
```

在生成机器人令牌时，`user` 方法仍将返回一个 `Laravel\Socialite\Two\User` 实例；但是，只会填充 `token` 属性。可以存储此令牌，以便[向经过身份验证的用户的 Slack 工作区发送通知](https://learnku.com/docs/laravel/11.x/notifications#notifying-external-slack-workspaces)。

### 可选参数

许多 OAuth 提供程序支持在重定向请求中使用其他可选参数。要在请求中包含任何可选参数，请使用带有关联数组的 `with` 方法：

```php
use Laravel\Socialite\Facades\Socialite;

return Socialite::driver('google')
    ->with(['hd' => 'example.com'])
    ->redirect();
```

> \[!WARNING\]  
> 使用 `with` 方法时，请注意不要传递任何保留关键字，如 `state` 或 `response_type`。

## 获取用户详细信息

在用户被重定向回你的应用程序的身份验证回调路由后，你可以使用 Socialite 的 `user` 方法来检索用户的详细信息。`user` 方法返回的用户对象提供了各种属性和方法，你可以使用它们来存储关于用户的信息到你自己的数据库中。

根据你正在进行身份验证的 OAuth 提供程序是否支持 OAuth 1.0 或 OAuth 2.0，此对象上可能可用不同的属性和方法：

```php
use Laravel\Socialite\Facades\Socialite;

Route::get('/auth/callback', function () {
    $user = Socialite::driver('github')->user();

    // OAuth 2.0 提供程序...
    $token = $user->token;
    $refreshToken = $user->refreshToken;
    $expiresIn = $user->expiresIn;

    // OAuth 1.0 提供程序...
    $token = $user->token;
    $tokenSecret = $user->tokenSecret;

    // 所有提供程序...
    $user->getId();
    $user->getNickname();
    $user->getName();
    $user->getEmail();
    $user->getAvatar();
});
```

如果你已经拥有用户的有效访问令牌，你可以使用 Socialite 的 `userFromToken` 方法检索他们的用户详细信息：

```php
use Laravel\Socialite\Facades\Socialite;

$user = Socialite::driver('github')->userFromToken($token);
```

如果你正在使用 iOS 应用程序通过 Facebook 有限登录，Facebook 将返回一个 OIDC 令牌而不是访问令牌。类似于访问令牌，OIDC 令牌可以提供给 `userFromToken` 方法以检索用户详细信息。

#### 从令牌和密钥检索用户详细信息（OAuth1）

如果你已经拥有用户的有效令牌和密钥，你可以使用 Socialite 的 `userFromTokenAndSecret` 方法检索他们的用户详细信息：

```php
use Laravel\Socialite\Facades\Socialite;

$user = Socialite::driver('twitter')->userFromTokenAndSecret($token, $secret);
```

#### 无状态身份验证

`stateless` 方法可用于禁用会话状态验证。当向不使用基于 cookie 的会话的无状态 API 添加社交身份验证时，这很有用：

```php
use Laravel\Socialite\Facades\Socialite;

return Socialite::driver('google')->stateless()->user();
```

> \[!WARNING\]  
> 无状态身份验证不适用于 Twitter OAuth 1.0 驱动程序。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/so...](https://learnku.com/docs/laravel/11.x/socialitemd/16734)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/so...](https://learnku.com/docs/laravel/11.x/socialitemd/16734)