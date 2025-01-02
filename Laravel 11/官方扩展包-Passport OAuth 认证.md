本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Passport

+   [介绍](#introduction)
    +   [Passport 还是 Sanctum？](#passport-or-sanctum)
+   [安装](#installation)
    +   [部署 Passport](#deploying-passport)
    +   [升级 Passport](#upgrading-passport)
+   [配置](#configuration)
    +   [客户端密钥哈希](#client-secret-hashing)
    +   [令牌生命周期](#token-lifetimes)
    +   [覆盖默认模型](#overriding-default-models)
    +   [覆盖路由](#overriding-routes)
+   [发放访问令牌](#issuing-access-tokens)
    +   [管理客户端](#managing-clients)
    +   [请求令牌](#requesting-tokens)
    +   [刷新令牌](#refreshing-tokens)
    +   [撤销令牌](#revoking-tokens)
    +   [清除令牌](#purging-tokens)
+   [授权码授权与 PKCE](#code-grant-pkce)
    +   [创建客户端](#creating-a-auth-pkce-grant-client)
    +   [请求令牌](#requesting-auth-pkce-grant-tokens)
+   [密码授权令牌](#password-grant-tokens)
    +   [创建密码授权客户端](#creating-a-password-grant-client)
    +   [请求令牌](#requesting-password-grant-tokens)
    +   [请求所有范围](#requesting-all-scopes)
    +   [自定义用户提供程序](#customizing-the-user-provider)
    +   [自定义用户名字段](#customizing-the-username-field)
    +   [自定义密码验证](#customizing-the-password-validation)
+   [隐式授权令牌](#implicit-grant-tokens)
+   [客户端凭据授权令牌](#client-credentials-grant-tokens)
+   [个人访问令牌](#personal-access-tokens)
    +   [创建个人访问客户端](#creating-a-personal-access-client)
    +   [管理个人访问令牌](#managing-personal-access-tokens)
+   [保护路由](#protecting-routes)
    +   [通过中间件](#via-middleware)
    +   [传递访问令牌](#passing-the-access-token)
+   [令牌范围](#token-scopes)
    +   [定义范围](#defining-scopes)
    +   [默认范围](#default-scope)
    +   [为令牌分配范围](#assigning-scopes-to-tokens)
    +   [检查范围](#checking-scopes)
+   [使用 JavaScript 消费你的 API](#consuming-your-api-with-javascript)
+   [事件](#events)
+   [测试](#testing)

## 介绍

[Laravel Passport](https://github.com/laravel/passport) 在几分钟内为你的 Laravel 应用程序提供了完整的 OAuth2 服务器实现。Passport 构建在由 Andy Millington 和 Simon Hamp 维护的 [League OAuth2 server](https://github.com/thephpleague/oauth2-server) 之上。

> **警告**  
> 本文档假定你已经熟悉 OAuth2。如果你对 OAuth2 一无所知，请在继续之前熟悉一下 OAuth2 的一般 [术语](https://oauth2.thephpleague.com/terminology/) 和特性。

### Passport 还是 Sanctum？

在开始之前，你可能希望确定你的应用程序更适合使用 Laravel Passport 还是 [Laravel Sanctum](https://learnku.com/docs/laravel/11.x/sanctum)。如果你的应用程序绝对需要支持 OAuth2，则应该使用 Laravel Passport。

然而，如果你尝试对单页面应用、移动应用程序进行身份验证，或者发放 API 令牌，你应该使用 [Laravel Sanctum](https://learnku.com/docs/laravel/11.x/sanctum)。Laravel Sanctum 不支持 OAuth2；但它提供了一个更简单的 API 身份验证开发体验。

## 安装

你可以通过 `install:api` Artisan 命令安装 Laravel Passport：

```shell
php artisan install:api --passport
```

这个命令将发布并运行数据库迁移，用于创建你的应用程序需要存储 OAuth2 客户端和访问令牌的表。该命令还将创建生成安全访问令牌所需的加密密钥。

此外，该命令将询问你是否想要将 UUID 作为 Passport `Client` 模型的主键值，而不是自增整数。

运行 `install:api` 命令后，将 `Laravel\Passport\HasApiTokens` trait 添加到你的 `App\Models\User` 模型中。这个 trait 将为你的模型提供一些辅助方法，允许你检查经过身份验证的用户的令牌和范围：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

最后，在你的应用程序的 `config/auth.php` 配置文件中，你应该定义一个 `api` 认证守卫，并将 `driver` 选项设置为 `passport`。这将指示你的应用程序在验证传入的 API 请求时使用 Passport 的 `TokenGuard`：

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],
```

### 部署 Passport

当首次将 Passport 部署到应用程序的服务器时，你可能需要运行 `passport:keys` 命令。这个命令生成 Passport 需要的加密密钥，以便生成访问令牌。通常，生成的密钥不会保存在源代码控制中：

```shell
php artisan passport:keys
```

如果需要，你可以定义 Passport 密钥应从哪个路径加载。你可以使用 `Passport::loadKeysFrom` 方法来实现这一点。通常情况下，这个方法应该从应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中调用：

```php
/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Passport::loadKeysFrom(__DIR__.'/../secrets/oauth');
}
```

#### 从环境加载密钥

或者，你可以使用 `vendor:publish` Artisan 命令发布 Passport 的配置文件：

```shell
php artisan vendor:publish --tag=passport-config
```

配置文件发布后，你可以将应用程序的加密密钥定义为环境变量来加载：

```ini
PASSPORT_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
<private key here>
-----END RSA PRIVATE KEY-----"

PASSPORT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
<public key here>
-----END PUBLIC KEY-----"
```

### 升级 Passport

升级到 Passport 的新主要版本时，重要的是仔细查阅[升级指南](https://github.com/laravel/passport/blob/master/UPGRADE.md)。

## 配置

### 客户端密钥哈希

如果希望在数据库中存储客户端密钥时对其进行哈希处理，你应该在 `App\Providers\AppServiceProvider` 类的 `boot` 方法中调用 `Passport::hashClientSecrets` 方法：

```php
use Laravel\Passport\Passport;

Passport::hashClientSecrets();
```

启用后，所有客户端密钥只能在创建后立即显示给用户。由于明文客户端密钥值从不存储在数据库中，如果丢失，则无法恢复密钥的值。

### 令牌生命周期

默认情况下，Passport 发放的访问令牌具有长期有效性，在一年后过期。如果你想配置更长 / 更短的令牌生命周期，你可以使用 `tokensExpireIn`、`refreshTokensExpireIn` 和 `personalAccessTokensExpireIn` 方法。这些方法应该从应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中调用：

```php
/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Passport::tokensExpireIn(now()->addDays(15));
    Passport::refreshTokensExpireIn(now()->addDays(30));
    Passport::personalAccessTokensExpireIn(now()->addMonths(6));
}
```

> **警告**  
> Passport 数据库表上的 `expires_at` 列是只读的，仅用于显示目的。在发放令牌时，Passport 会将过期信息存储在签名和加密令牌中。如果需要使令牌失效，你应该[撤销它](#revoking-tokens)。

### 覆盖默认模型

你可以通过定义自己的模型并扩展相应的 Passport 模型来自由扩展 Passport 内部使用的模型：

```php
use Laravel\Passport\Client as PassportClient;

class Client extends PassportClient
{
    // ...
}
```

定义完你的模型后，你可以通过 `Laravel\Passport\Passport` 类指示 Passport 使用你的自定义模型。通常情况下，你应该在应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中告知 Passport 有关你的自定义模型：

```php
use App\Models\Passport\AuthCode;
use App\Models\Passport\Client;
use App\Models\Passport\PersonalAccessClient;
use App\Models\Passport\RefreshToken;
use App\Models\Passport\Token;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Passport::useTokenModel(Token::class);
    Passport::useRefreshTokenModel(RefreshToken::class);
    Passport::useAuthCodeModel(AuthCode::class);
    Passport::useClientModel(Client::class);
    Passport::usePersonalAccessClientModel(PersonalAccessClient::class);
}
```

### 覆盖路由

有时你可能希望自定义 Passport 定义的路由。为实现此目的，首先需要在应用程序的 `AppServiceProvider` 的 `register` 方法中添加 `Passport::ignoreRoutes` 来忽略 Passport 注册的路由：

```php
use Laravel\Passport\Passport;

/**
 * 注册任何应用服务。
 */
public function register(): void
{
    Passport::ignoreRoutes();
}
```

然后，你可以将 Passport 在[其路由文件](https://github.com/laravel/passport/blob/11.x/routes/web.php)中定义的路由复制到你的应用程序的 `routes/web.php` 文件中，并根据需要进行修改：

```php
Route::group([
    'as' => 'passport.',
    'prefix' => config('passport.path', 'oauth'),
    'namespace' => '\Laravel\Passport\Http\Controllers',
], function () {
    // Passport routes...
});
```

## 发放访问令牌

通过授权码使用 OAuth2 是大多数开发人员熟悉的 OAuth2 方式。在使用授权码时，客户端应用程序将用户重定向到你的服务器，在那里用户将批准或拒绝向客户端发放访问令牌的请求。

### 管理客户端

首先，需要与需要与你的应用程序 API 交互的应用程序开发人员注册他们的应用程序，创建一个 “客户端”。通常，这包括提供他们应用程序的名称和一个 URL，你的应用程序可以在用户批准他们的授权请求后重定向到该 URL。

#### `passport:client` 命令

创建客户端的最简单方法是使用 `passport:client` Artisan 命令。此命令可用于为测试你的 OAuth2 功能创建自己的客户端。当运行 `client` 命令时，Passport 将提示你提供有关你的客户端的更多信息，并为你提供客户端 ID 和密钥：

```shell
php artisan passport:client
```

**重定向 URL**

如果你想为你的客户端允许多个重定向 URL，你可以在 `passport:client` 命令提示的 URL 处使用逗号分隔的列表来指定它们。任何包含逗号的 URL 都应该进行 URL 编码：

```shell
http://example.com/callback,http://examplefoo.com/callback
```

#### JSON API

由于你的应用程序用户无法使用 `client` 命令，Passport 提供了一个 JSON API，你可以用来创建客户端。这样可以避免手动编写用于创建、更新和删除客户端的控制器。

但是，你需要将 Passport 的 JSON API 与你自己的前端配对，为用户提供一个仪表板，让他们管理他们的客户端。下面，我们将回顾所有用于管理客户端的 API 端点。为了方便起见，我们将使用 [Axios](https://github.com/axios/axios) 来演示如何向这些端点发出 HTTP 请求。

JSON API 受到 `web` 和 `auth` 中间件的保护；因此，它只能从你自己的应用程序调用，无法从外部来源调用。

#### `GET /oauth/clients`

此路由返回经过身份验证的用户的所有客户端。这主要用于列出用户的所有客户端，以便他们可以编辑或删除它们：

```js
axios.get('/oauth/clients')
    .then(response => {
        console.log(response.data);
    });
```

#### `POST /oauth/clients`

此路由用于创建新的客户端。它需要两个数据：客户端的 `name` 和 `redirect` URL。`redirect` URL 是用户在批准或拒绝授权请求后将被重定向的地方。

当创建客户端时，将会分配一个客户端 ID 和客户端密钥。在从你的应用程序请求访问令牌时，将使用这些值。客户端创建路由将返回新的客户端实例：

```js
const data = {
    name: '客户端名称',
    redirect: 'http://example.com/callback'
};

axios.post('/oauth/clients', data)
    .then(response => {
        console.log(response.data);
    })
    .catch(response => {
        // 在响应中列出错误...
    });
```

#### `PUT /oauth/clients/{client-id}`

此路由用于更新客户端。它需要两个数据：客户端的 `name` 和 `redirect` URL。`redirect` URL 是用户在批准或拒绝授权请求后将被重定向的地方。路由将返回更新后的客户端实例：

```js
const data = {
    name: '新的客户端名称',
    redirect: 'http://example.com/callback'
};

axios.put('/oauth/clients/' + clientId, data)
    .then(response => {
        console.log(response.data);
    })
    .catch(response => {
        // 在响应中列出错误...
    });
```

#### `DELETE /oauth/clients/{client-id}`

此路由用于删除客户端：

```js
axios.delete('/oauth/clients/' + clientId)
    .then(response => {
        // ...
    });
```

### 请求令牌

#### 重定向进行授权

一旦创建了客户端，开发人员可以使用他们的客户端 ID 和密钥从你的应用程序请求授权码和访问令牌。首先，消费应用程序应该向你的应用程序的 `/oauth/authorize` 路由发起重定向请求，如下所示：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Str;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => '',
        'state' => $state,
        // 'prompt' => '', // "none", "consent", or "login"
    ]);

    return redirect('http://passport-app.test/oauth/authorize?' . $query);
});
```

`prompt` 参数可用于指定 Passport 应用程序的身份验证行为。

如果 `prompt` 值为 `none`，如果用户尚未通过 Passport 应用程序进行身份验证，Passport 将始终抛出身份验证错误。如果值为 `consent`，Passport 将始终显示授权批准屏幕，即使所有权限之前已经授予给消费应用程序。当值为 `login` 时，Passport 应用程序将始终提示用户重新登录应用程序，即使他们已经有一个现有会话。

如果未提供 `prompt` 值，则仅当用户之前未授权访问请求的范围时，用户将被提示进行授权。

> **注意**  
> 请记住，`/oauth/authorize` 路由已经由 Passport 定义。你不需要手动定义这个路由。

#### 批准请求

在接收授权请求时，Passport 将根据 `prompt` 参数的值（如果存在）自动做出响应，并可能向用户显示一个模板，让他们批准或拒绝授权请求。如果他们批准请求，他们将被重定向回消费应用程序指定的 `redirect_uri`。`redirect_uri` 必须与创建客户端时指定的 `redirect` URL 匹配。

如果你想自定义授权批准屏幕，你可以使用 `vendor:publish` Artisan 命令发布 Passport 的视图。发布的视图将放置在 `resources/views/vendor/passport` 目录中：

```shell
php artisan vendor:publish --tag=passport-views
```

有时你可能希望跳过授权提示，比如在授权第一方客户端时。你可以通过[扩展 `Client` 模型](#overriding-default-models)并定义一个 `skipsAuthorization` 方法来实现这一点。如果 `skipsAuthorization` 返回 `true`，客户端将被批准，并且用户将立即被重定向回 `redirect_uri`，除非消费应用程序在重定向授权时明确设置了 `prompt` 参数：

```php
<?php

namespace App\Models\Passport;

use Laravel\Passport\Client as BaseClient;

class Client extends BaseClient
{
    /**
     * Determine if the client should skip the authorization prompt.
     */
    public function skipsAuthorization(): bool
    {
        return $this->firstParty();
    }
}
```

#### 将授权码转换为访问令牌

如果用户批准授权请求，他们将被重定向回消费应用程序。消费者应首先根据重定向前存储的值验证 `state` 参数。如果 state 参数匹配，则消费者应发出 `POST` 请求到你的应用程序请求访问令牌。请求应包括用户批准授权请求时你的应用程序发放的授权码：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;

Route::get('/callback', function (Request $request) {
    $state = $request->session()->pull('state');

    throw_unless(
        strlen($state) > 0 && $state === $request->state,
        InvalidArgumentException::class,
        'Invalid state value.'
    );

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'authorization_code',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'code' => $request->code,
    ]);

    return $response->json();
});
```

这个 `/oauth/token` 路由将返回一个包含 `access_token`、`refresh_token` 和 `expires_in` 属性的 JSON 响应。`expires_in` 属性包含访问令牌过期前的秒数。

> **注意**  
> 像 `/oauth/authorize` 路由一样，`/oauth/token` 路由由 Passport 为你定义。你不需要手动定义这个路由。

#### JSON API

Passport 还包括一个用于管理授权访问令牌的 JSON API。你可以将其与自己的前端配对，为用户提供一个管理访问令牌的仪表板。为了方便起见，我们将使用 [Axios](https://github.com/mzabriskie/axios) 来演示如何向端点发出 HTTP 请求。JSON API 受 `web` 和 `auth` 中间件保护；因此，它只能从你自己的应用程序中调用。

#### `GET /oauth/tokens`

这个路由返回经过授权的所有访问令牌，这些令牌是认证用户创建的。这主要用于列出用户的所有令牌，以便他们可以撤销它们：

```js
axios.get('/oauth/tokens')
    .then(response => {
        console.log(response.data);
    });
```

#### `DELETE /oauth/tokens/{token-id}`

这个路由可用于撤销经过授权的访问令牌及其相关的刷新令牌：

```js
axios.delete('/oauth/tokens/' + tokenId);
```

### 刷新令牌

如果你的应用程序发放短期访问令牌，用户将需要通过在发放访问令牌时提供的刷新令牌来刷新他们的访问令牌：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('http://passport-app.test/oauth/token', [
    'grant_type' => 'refresh_token',
    'refresh_token' => 'the-refresh-token',
    'client_id' => 'client-id',
    'client_secret' => 'client-secret',
    'scope' => '',
]);

return $response->json();
```

这个 `/oauth/token` 路由将返回一个包含 `access_token`、`refresh_token` 和 `expires_in` 属性的 JSON 响应。`expires_in` 属性包含访问令牌过期前的秒数。

### 撤销令牌

你可以使用 `Laravel\Passport\TokenRepository` 上的 `revokeAccessToken` 方法来撤销一个令牌。你可以使用 `Laravel\Passport\RefreshTokenRepository` 上的 `revokeRefreshTokensByAccessTokenId` 方法来撤销一个令牌的刷新令牌。这些类可以通过 Laravel 的[服务容器](https://learnku.com/docs/laravel/11.x/container)来解析：

```php
use Laravel\Passport\TokenRepository;
use Laravel\Passport\RefreshTokenRepository;

$tokenRepository = app(TokenRepository::class);
$refreshTokenRepository = app(RefreshTokenRepository::class);

// 撤销一个访问令牌...
$tokenRepository->revokeAccessToken($tokenId);

// 撤销所有令牌的刷新令牌...
$refreshTokenRepository->revokeRefreshTokensByAccessTokenId($tokenId);
```

### 清理令牌

当令牌被撤销或过期时，你可能希望从数据库中清除它们。Passport 附带的 `passport:purge` Artisan 命令可以帮助你实现这一点：

```shell
# 清除被撤销和过期的令牌和授权码...
php artisan passport:purge

# 仅清除超过6小时过期的令牌...
php artisan passport:purge --hours=6

# 仅清除被撤销的令牌和授权码...
php artisan passport:purge --revoked

# 仅清除过期的令牌和授权码...
php artisan passport:purge --expired
```

你还可以在应用程序的 `routes/console.php` 文件中配置一个[定时任务](https://learnku.com/docs/laravel/11.x/scheduling)，以便在计划时间自动清理你的令牌：

```php
use Laravel\Support\Facades\Schedule;

Schedule::command('passport:purge')->hourly();
```

## 使用 PKCE 的授权码授予

具有「Proof Key for Code Exchange」（PKCE）的授权码授予是一种安全的方式，用于验证单页应用程序或原生应用程序以访问你的 API。当无法保证客户端密钥会被保密存储，或者为了减轻授权码被攻击者拦截的威胁时，应使用这种授权方式。在交换授权码获取访问令牌时，一个 “代码验证器” 和一个 “代码挑战” 组合取代了客户端密钥。

### 创建客户端

在你的应用程序可以通过带有 PKCE 的授权码授予发放令牌之前，你需要创建一个启用了 PKCE 的客户端。你可以使用 `passport:client` Artisan 命令并加上 `--public` 选项来实现：

```shell
php artisan passport:client --public
```

### 请求令牌

#### 代码验证器和代码挑战

由于这种授权方式不提供客户端密钥，开发人员需要生成一个代码验证器和一个代码挑战的组合来请求令牌。

代码验证器应该是一个包含字母、数字和 `"-"`、`"."`、`"_"`、`"~"` 字符的随机字符串，长度在 43 到 128 个字符之间，如 [RFC 7636 规范](https://tools.ietf.org/html/rfc7636)中定义。

代码挑战应该是一个使用 Base64 编码的字符串，其中包含 URL 和文件名安全字符。末尾的`'='` 字符应该被移除，不应包含换行、空格或其他额外字符。

```php
$encoded = base64_encode(hash('sha256', $code_verifier, true));

$codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');
```

#### 重定向进行授权

一旦客户端被创建，你可以使用客户端 ID 以及生成的代码验证器和代码挑战来从你的应用程序请求授权码和访问令牌。首先，消费应用程序应该向你的应用程序的 `/oauth/authorize` 路由发出重定向请求：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Str;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $request->session()->put(
        'code_verifier', $code_verifier = Str::random(128)
    );

    $codeChallenge = strtr(rtrim(
        base64_encode(hash('sha256', $code_verifier, true))
    , '='), '+/', '-_');

    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => '',
        'state' => $state,
        'code_challenge' => $codeChallenge,
        'code_challenge_method' => 'S256',
        // 'prompt' => '', // "none", "consent", or "login"
    ]);

    return redirect('http://passport-app.test/oauth/authorize?' . $query);
});
```

#### 将授权码转换为访问令牌

如果用户批准授权请求，他们将被重定向回消费应用程序。消费者应该根据重定向前存储的值验证 `state` 参数，就像标准的授权码授予方式一样。

如果 state 参数匹配，消费者应该向你的应用程序发出 `POST` 请求来请求一个访问令牌。请求应该包括用户批准授权请求时你的应用程序发放的授权码，以及最初生成的代码验证器：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;

Route::get('/callback', function (Request $request) {
    $state = $request->session()->pull('state');

    $codeVerifier = $request->session()->pull('code_verifier');

    throw_unless(
        strlen($state) > 0 && $state === $request->state,
        InvalidArgumentException::class
    );

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'authorization_code',
        'client_id' => 'client-id',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'code_verifier' => $codeVerifier,
        'code' => $request->code,
    ]);

    return $response->json();
});
```

## 密码授权令牌

> **警告**  
> 我们不再推荐使用密码授权令牌。相反，你应该选择 [OAuth2 服务器当前推荐的授权类型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

OAuth2 密码授权允许你的其他第一方客户端（如移动应用程序）使用电子邮件地址 / 用户名和密码获取访问令牌。这使你能够向你的第一方客户端安全地发放访问令牌，而无需要求用户经历整个 OAuth2 授权码重定向流程。

要启用密码授权，可以在应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中调用 `enablePasswordGrant` 方法：

```php
/**
 * 启动任何应用程序服务。
 */
public function boot(): void
{
    Passport::enablePasswordGrant();
}
```

### 创建密码授权客户端

在你的应用程序可以通过密码授权发放令牌之前，你需要创建一个密码授权客户端。你可以使用 `passport:client` Artisan 命令并加上 `--password` 选项来实现。**如果你已经运行过 `passport:install` 命令，则无需运行此命令：**

```shell
php artisan passport:client --password
```

### 请求令牌

一旦你创建了一个密码授权客户端，你可以通过向 `/oauth/token` 路由发出 `POST` 请求，并提供用户的电子邮件地址和密码来请求访问令牌。请记住，Passport 已经注册了这个路由，因此无需手动定义。如果请求成功，你将从服务器的 JSON 响应中收到 `access_token` 和 `refresh_token`：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('http://passport-app.test/oauth/token', [
    'grant_type' => 'password',
    'client_id' => 'client-id',
    'client_secret' => 'client-secret',
    'username' => 'taylor@laravel.com',
    'password' => 'my-password',
    'scope' => '',
]);

return $response->json();
```

> **注意**  
> 请记住，默认情况下访问令牌具有长寿命。但是，如果需要，你可以[配置最大访问令牌寿命](#configuration)。

### 请求所有范围

当使用密码授权或客户端凭据授权时，你可能希望为所有受应用程序支持的范围授权令牌。你可以通过请求 `*` 范围来实现这一点。如果请求 `*` 范围，令牌实例上的 `can` 方法将始终返回 `true`。此范围只能分配给使用 `password` 或 `client_credentials` 授权发放的令牌：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('http://passport-app.test/oauth/token', [
    'grant_type' => 'password',
    'client_id' => 'client-id',
    'client_secret' => 'client-secret',
    'username' => 'taylor@laravel.com',
    'password' => 'my-password',
    'scope' => '*',
]);
```

### 自定义用户提供程序

如果你的应用程序使用多个[身份验证用户提供程序](https://learnku.com/docs/laravel/11.x/authentication#introduction)，你可以在通过 `artisan passport:client --password` 命令创建客户端时提供 `--provider` 选项，以指定密码授权客户端使用哪个用户提供程序。给定的提供程序名称应与你的应用程序 `config/auth.php` 配置文件中定义的有效提供程序匹配。然后，你可以通过[使用中间件保护路由](#via-middleware)来确保只有来自指定提供程序的用户被授权。

### 自定义用户名字段

在使用密码授权进行身份验证时，Passport 将使用你的可认证模型的 `email` 属性作为 “用户名”。但是，你可以通过在模型上定义一个 `findForPassport` 方法来自定义此行为：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;

    /**
     * 为给定的用户名查找用户实例。
     */
    public function findForPassport(string $username): User
    {
        return $this->where('username', $username)->first();
    }
}
```

### 自定义密码验证

在使用密码授权进行身份验证时，Passport 将使用你的模型的 `password` 属性来验证给定的密码。如果你的模型没有 `password` 属性，或者你希望自定义密码验证逻辑，你可以在模型上定义一个 `validateForPassportPasswordGrant` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Support\Facades\Hash;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;

    /**
     * 验证用户的密码是否符合Passport密码授权要求。
     */
    public function validateForPassportPasswordGrant(string $password): bool
    {
        return Hash::check($password, $this->password);
    }
}
```

## 隐式授权令牌

> **警告**  
> 我们不再推荐使用隐式授权令牌。相反，你应该选择 [OAuth2 服务器当前推荐的授权类型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

隐式授权类似于授权码授权；但是，令牌会在不交换授权码的情况下返回给客户端。这种授权通常用于 JavaScript 或移动应用程序，其中客户端凭据无法安全存储。要启用这种授权，可以在应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中调用 `enableImplicitGrant` 方法：

```php
/**
 * 启动任何应用程序服务。
 */
public function boot(): void
{
    Passport::enableImplicitGrant();
}
```

一旦授权被启用，开发人员可以使用他们的客户端 ID 从你的应用程序请求访问令牌。消费应用程序应该向你的应用程序的 `/oauth/authorize` 路由发出重定向请求，如下所示：

```php
use Illuminate\Http\Request;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://third-party-app.com/callback',
        'response_type' => 'token',
        'scope' => '',
        'state' => $state,
        // 'prompt' => '', // "none", "consent", or "login"
    ]);

    return redirect('http://passport-app.test/oauth/authorize?' . $query);
});
```

> **注意**  
> 请记住，`/oauth/authorize` 路由已经被 Passport 定义。你无需手动定义此路由。

## 客户端凭据授权令牌

客户端凭据授权适用于机器对机器的身份验证。例如，你可以在执行 API 上的维护任务的定时作业中使用此授权。

在你的应用程序可以通过客户端凭据授权发放令牌之前，你需要创建一个客户端凭据授权客户端。你可以使用 `passport:client` Artisan 命令的 `--client` 选项来执行此操作：

```shell
php artisan passport:client --client
```

接下来，为 `CheckClientCredentials` 中间件注册一个中间件别名以使用此授权类型。你可以在应用程序的 `bootstrap/app.php` 文件中定义中间件别名：

```php
use Laravel\Passport\Http\Middleware\CheckClientCredentials;

->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'client' => CheckClientCredentials::class
    ]);
})
```

然后，将中间件附加到一个路由上：

```php
Route::get('/orders', function (Request $request) {
    ...
})->middleware('client');
```

如果要限制对路由的访问权限以特定作用域，你可以在将 `client` 中间件附加到路由时提供所需作用域的逗号分隔列表：

```php
Route::get('/orders', function (Request $request) {
    ...
})->middleware('client:check-status,your-scope');
```

### 检索令牌

要使用此授权类型检索令牌，请向 `oauth/token` 端点发出请求：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('http://passport-app.test/oauth/token', [
    'grant_type' => 'client_credentials',
    'client_id' => 'client-id',
    'client_secret' => 'client-secret',
    'scope' => 'your-scope',
]);

return $response->json()['access_token'];
```

## 个人访问令牌

有时，你的用户可能希望向自己发放访问令牌，而无需经过典型的授权码重定向流程。允许用户通过你的应用程序的 UI 向自己发放令牌，可以让用户在实验你的 API 时更加方便，或者作为一种更简单的发放访问令牌的方法。

> **注意**  
> 如果你的应用程序主要使用 Passport 发放个人访问令牌，请考虑使用 [Laravel Sanctum](https://learnku.com/docs/laravel/11.x/sanctum)，这是 Laravel 的轻量级官方库，用于发放 API 访问令牌。

### 创建个人访问客户端

在你的应用程序可以发放个人访问令牌之前，你需要创建一个个人访问客户端。你可以通过执行带有 `--personal` 选项的 `passport:client` Artisan 命令来执行此操作。如果你已经运行过 `passport:install` 命令，则无需运行此命令：

```shell
php artisan passport:client --personal
```

创建完个人访问客户端后，将客户端的 ID 和明文密钥值放入你的应用程序的`.env` 文件中：

```ini
PASSPORT_PERSONAL_ACCESS_CLIENT_ID="client-id-value"
PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET="unhashed-client-secret-value"
```

### 管理个人访问令牌

创建完个人访问客户端后，你可以使用 `App\Models\User` 模型实例上的 `createToken` 方法为特定用户发放令牌。`createToken` 方法接受令牌名称作为第一个参数，以及可选的[作用域](#token-scopes)数组作为第二个参数：

```php
use App\Models\User;

$user = User::find(1);

// 创建不带作用域的令牌...
$token = $user->createToken('Token Name')->accessToken;

// 创建带作用域的令牌...
$token = $user->createToken('My Token', ['place-orders'])->accessToken;
```

#### JSON API

Passport 还包括一个用于管理个人访问令牌的 JSON API。你可以将其与自己的前端配对，为用户提供一个管理个人访问令牌的仪表板。接下来，我们将查看管理个人访问令牌的所有 API 端点。为了方便起见，我们将使用 [Axios](https://github.com/mzabriskie/axios) 来演示如何向这些端点发出 HTTP 请求。

JSON API 由 `web` 和 `auth` 中间件保护；因此，它只能从你自己的应用程序中调用。无法从外部来源调用。

#### `GET /oauth/scopes`

该路由返回为你的应用程序定义的所有[作用域](#token-scopes)。你可以使用这个路由列出用户可以分配给个人访问令牌的作用域：

```js
axios.get('/oauth/scopes')
    .then(response => {
        console.log(response.data);
    });
```

#### `GET /oauth/personal-access-tokens`

该路由返回认证用户创建的所有个人访问令牌。主要用于列出用户的所有令牌，以便他们可以编辑或撤销这些令牌：

```js
axios.get('/oauth/personal-access-tokens')
    .then(response => {
        console.log(response.data);
    });
```

#### `POST /oauth/personal-access-tokens`

该路由创建新的个人访问令牌。它需要两个数据：令牌的 `name` 和应分配给令牌的 `scopes`：

```js
const data = {
    name: '令牌名称',
    scopes: []
};

axios.post('/oauth/personal-access-tokens', data)
    .then(response => {
        console.log(response.data.accessToken);
    })
    .catch (response => {
        // 在响应中列出错误...
    });
```

#### `DELETE /oauth/personal-access-tokens/{token-id}`

该路由可用于撤销个人访问令牌：

```js
axios.delete('/oauth/personal-access-tokens/' + tokenId);
```

## 保护路由

### 通过中间件

Passport 包含一个[身份验证守卫](https://learnku.com/docs/laravel/11.x/authentication#adding-custom-guards)，用于验证传入请求的访问令牌。一旦你配置了 `api` 守卫使用 `passport` 驱动程序，你只需要在任何需要有效访问令牌的路由上指定 `auth:api` 中间件：

```php
Route::get('/user', function () {
    // ...
})->middleware('auth:api');
```

> **警告**  
> 如果你正在使用[客户端凭证授权](#client-credentials-grant-tokens)，你应该使用[客户端中间件](#client-credentials-grant-tokens)来保护你的路由，而不是 `auth:api` 中间件。

#### 多个身份验证守卫

如果你的应用程序对不同类型的用户进行身份验证，可能使用完全不同的 Eloquent 模型，你可能需要为应用程序中的每种用户提供程序类型定义一个守卫配置。这样可以保护特定用户提供程序的请求。例如，考虑以下在 `config/auth.php` 配置文件中的守卫配置：

```php
'api' => [
    'driver' => 'passport',
    'provider' => 'users',
],

'api-customers' => [
    'driver' => 'passport',
    'provider' => 'customers',
],
```

以下路由将使用 `api-customers` 守卫，该守卫使用 `customers` 用户提供程序来验证传入请求：

```php
Route::get('/customer', function () {
    // ...
})->middleware('auth:api-customers');
```

> **注意**  
> 关于如何在 Passport 中使用多个用户提供程序的更多信息，请参考[密码授权文档](#customizing-the-user-provider)。

### 传递访问令牌

当调用由 Passport 保护的路由时，你的应用程序的 API 消费者应该将他们的访问令牌作为请求的 `Authorization` 头部中的 `Bearer` 令牌指定。例如，当使用 Guzzle HTTP 库时：

```php
use Illuminate\Support\Facades\Http;

$response = Http::withHeaders([
    'Accept' => 'application/json',
    'Authorization' => 'Bearer '.$accessToken,
])->get('https://passport-app.test/api/user');

return $response->json();
```

## 令牌作用域

作用域允许你的 API 客户端在请求授权以访问账户时请求特定权限集。例如，如果你正在构建一个电子商务应用程序，不是所有的 API 消费者都需要具有下订单的能力。相反，你可以允许消费者仅请求授权以访问订单发货状态。换句话说，作用域允许你的应用程序用户限制第三方应用程序代表他们执行的操作。

### 定义作用域

你可以使用 `Passport::tokensCan` 方法在你的应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中定义 API 的作用域。`tokensCan` 方法接受一个作用域名称和作用域描述的数组。作用域描述可以是任何你希望的内容，并将显示在授权批准屏幕上供用户查看：

```php
/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Passport::tokensCan([
        'place-orders' => '下订单',
        'check-status' => '检查订单状态',
    ]);
}
```

### 默认作用域

如果客户端没有请求任何特定作用域，你可以配置你的 Passport 服务器使用 `setDefaultScope` 方法将默认作用域附加到令牌上。通常，你应该从你的应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中调用这个方法：

```php
use Laravel\Passport\Passport;

Passport::tokensCan([
    'place-orders' => '下订单',
    'check-status' => '检查订单状态',
]);

Passport::setDefaultScope([
    'check-status',
    'place-orders',
]);
```

> **注意**  
> Passport 的默认作用域不适用于用户生成的个人访问令牌。

### 将作用域分配给令牌

#### 请求授权码时

在使用授权码授权请求访问令牌时，消费者应将他们所需的作用域指定为 `scope` 查询字符串参数。`scope` 参数应该是一个以空格分隔的作用域列表：

```php
Route::get('/redirect', function () {
    $query = http_build_query([
        'client_id' => 'client-id',
        'redirect_uri' => 'http://example.com/callback',
        'response_type' => 'code',
        'scope' => 'place-orders check-status',
    ]);

    return redirect('http://passport-app.test/oauth/authorize?'.$query);
});
```

#### 发放个人访问令牌时

如果你正在使用 `App\Models\User` 模型的 `createToken` 方法发放个人访问令牌，你可以将所需作用域的数组作为方法的第二个参数传递：

```php
$token = $user->createToken('My Token', ['place-orders'])->accessToken;
```

### 检查作用域

Passport 包含两个中间件，可用于验证传入请求是否使用被授予特定作用域的令牌进行身份验证。要开始，请在你的应用程序的 `bootstrap/app.php` 文件中定义以下中间件别名：

```php
use Laravel\Passport\Http\Middleware\CheckForAnyScope;
use Laravel\Passport\Http\Middleware\CheckScopes;

->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'scopes' => CheckScopes::class,
        'scope' => CheckForAnyScope::class,
    ]);
})
```

#### 检查所有作用域

`scopes` 中间件可以分配给一个路由，以验证传入请求的访问令牌是否具有所有列出的作用域：

```php
Route::get('/orders', function () {
    // 访问令牌具有 "check-status" 和 "place-orders" 作用域...
})->middleware(['auth:api', 'scopes:check-status,place-orders']);
```

#### 检查任意作用域

`scope` 中间件可以分配给一个路由，以验证传入请求的访问令牌是否具有*至少一个*列出的作用域：

```php
Route::get('/orders', function () {
    // 访问令牌具有 "check-status" 或 "place-orders" 作用域之一...
})->middleware(['auth:api', 'scope:check-status,place-orders']);
```

#### 在令牌实例上检查作用域

一旦访问令牌认证请求进入你的应用程序，你仍然可以使用经过身份验证的 `App\Models\User` 实例上的 `tokenCan` 方法来检查令牌是否具有给定作用域：

```php
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    if ($request->user()->tokenCan('place-orders')) {
        // ...
    }
});
```

#### 其他作用域方法

`scopeIds` 方法将返回所有已定义的 ID / 名称的数组：

```php
use Laravel\Passport\Passport;

Passport::scopeIds();
```

`scopes` 方法将返回所有已定义作用域的数组，作用域是 `Laravel\Passport\Scope` 的实例：

`scopesFor` 方法将返回与给定 ID / 名称匹配的 `Laravel\Passport\Scope` 实例数组：

```php
Passport::scopesFor(['place-orders', 'check-status']);
```

你可以使用 `hasScope` 方法来确定给定作用域是否已定义：

```php
Passport::hasScope('place-orders');
```

## 使用 JavaScript 消费你的 API

在构建 API 时，能够从你的 JavaScript 应用程序中消费自己的 API 是非常有用的。这种 API 开发方法允许你的应用程序消费与你与世界共享的相同 API。同一个 API 可以被你的 Web 应用程序、移动应用程序、第三方应用程序以及你可能在各种包管理器上发布的任何 SDK 消费。

通常情况下，如果你想从你的 JavaScript 应用程序中消费你的 API，你需要手动向应用程序发送一个访问令牌，并在每个请求中传递它到你的应用程序。然而，Passport 包含一个可以为你处理这个过程的中间件。你只需要将 `CreateFreshApiToken` 中间件附加到应用程序的 `bootstrap/app.php` 文件中的 `web` 中间件组：

```php
use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

->withMiddleware(function (Middleware $middleware) {
    $middleware->web(append: [
        CreateFreshApiToken::class,
    ]);
})
```

> **警告**  
> 你应该确保 `CreateFreshApiToken` 中间件是中间件堆栈中列出的最后一个中间件。

这个中间件将在你的响应中附加一个 `laravel_token` cookie。该 cookie 包含一个加密的 JWT，Passport 将使用它来验证来自你的 JavaScript 应用程序的 API 请求。JWT 的生命周期等于你的 `session.lifetime` 配置值。现在，由于浏览器将自动发送该 cookie 到所有后续请求，你可以在不显式传递访问令牌的情况下向你的应用程序 API 发送请求：

```javascript
axios.get('/api/user')
    .then(response => {
        console.log(response.data);
    });
```

#### 自定义 Cookie 名称

如果需要，你可以使用 `Passport::cookie` 方法来自定义 `laravel_token`cookie 的名称。通常，这个方法应该从你的应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中调用：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::cookie('custom_name');
}
```

#### CSRF 保护

在使用这种身份验证方法时，你需要确保请求中包含有效的 CSRF 令牌头。默认的 Laravel JavaScript 支架包含一个 Axios 实例，它将自动使用加密的 `XSRF-TOKEN` cookie 值在同源请求中发送 `X-XSRF-TOKEN` 头。

> **注意**  
> 如果选择发送 `X-CSRF-TOKEN` 头而不是 `X-XSRF-TOKEN`，你需要使用 `csrf_token()` 提供的未加密令牌。

## 事件

在发放访问令牌和刷新令牌时，Passport 会触发事件。你可以[监听这些事件](https://learnku.com/docs/laravel/11.x/events)来清理或撤销数据库中的其他访问令牌：

| 事件名称 |
| --- |
| `Laravel\Passport\Events\AccessTokenCreated` |
| `Laravel\Passport\Events\RefreshTokenCreated` |

## 测试

Passport 的 `actingAs` 方法可用于指定当前认证的用户及其作用域。`actingAs` 方法的第一个参数是用户实例，第二个参数是应授予用户令牌的作用域数组：

```php
use App\Models\User;
use Laravel\Passport\Passport;

test('servers can be created', function () {
    Passport::actingAs(
        User::factory()->create(),
        ['create-servers']
    );

    $response = $this->post('/api/create-server');

    $response->assertStatus(201);
});
```

```php
use App\Models\User;
use Laravel\Passport\Passport;

public function test_servers_can_be_created(): void
{
    Passport::actingAs(
        User::factory()->create(),
        ['create-servers']
    );

    $response = $this->post('/api/create-server');

    $response->assertStatus(201);
}
```

Passport 的 `actingAsClient` 方法可用于指定当前认证的客户端及其作用域。`actingAsClient` 方法的第一个参数是客户端实例，第二个参数是应授予客户端令牌的作用域数组：

```php
use Laravel\Passport\Client;
use Laravel\Passport\Passport;

test('orders can be retrieved', function () {
    Passport::actingAsClient(
        Client::factory()->create(),
        ['check-status']
    );

    $response = $this->get('/api/orders');

    $response->assertStatus(200);
});
```

```php
use Laravel\Passport\Client;
use Laravel\Passport\Passport;

public function test_orders_can_be_retrieved(): void
{
    Passport::actingAsClient(
        Client::factory()->create(),
        ['check-status']
    );

    $response = $this->get('/api/orders');

    $response->assertStatus(200);
}
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/pa...](https://learnku.com/docs/laravel/11.x/passportmd/16724)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/pa...](https://learnku.com/docs/laravel/11.x/passportmd/16724)