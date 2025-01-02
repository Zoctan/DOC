本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Sanctum

+   [简介](#introduction)
    +   [工作原理](#how-it-works)
+   [安装](#installation)
+   [配置](#configuration)
    +   [覆盖默认模型](#overriding-default-models)
+   [API 令牌认证](#api-token-authentication)
    +   [发放 API 令牌](#issuing-api-tokens)
    +   [令牌权限](#token-abilities)
    +   [保护路由](#protecting-routes)
    +   [撤销令牌](#revoking-tokens)
    +   [令牌过期](#token-expiration)
+   [单页面应用认证](#spa-authentication)
    +   [配置](#spa-configuration)
    +   [认证](#spa-authenticating)
    +   [保护路由](#protecting-spa-routes)
    +   [授权私有广播频道](#authorizing-private-broadcast-channels)
+   [移动应用认证](#mobile-application-authentication)
    +   [发放 API 令牌](#issuing-mobile-api-tokens)
    +   [保护路由](#protecting-mobile-api-routes)
    +   [撤销令牌](#revoking-mobile-api-tokens)
+   [测试](#testing)

## 简介

[Laravel Sanctum](https://github.com/laravel/sanctum) 为单页应用（SPA）、移动应用以及基于令牌的简易 API 提供了一个轻量级的认证系统。Sanctum 允许应用的每个用户为他们的账户生成多个 API 令牌。这些令牌可以被授予能力 / 范围，指定令牌允许执行的操作。

### 工作原理

Laravel Sanctum 旨在解决两个独立的问题。在深入探讨该库之前，让我们先讨论每个问题。

#### API 令牌

首先，Sanctum 是一个简单的包，您可以使用它为用户发放 API 令牌，无需复杂的 OAuth。这个功能的灵感来自 GitHub 和其他发放 “个人访问令牌” 的应用。例如，想象一下您的应用中的 “账户设置” 页面，用户可以在此页面为他们的账户生成一个 API 令牌。您可以使用 Sanctum 来生成和管理这些令牌。这些令牌通常具有非常长的过期时间（数年），但用户可以随时手动撤销。

## Laravel Sanctum

Laravel Sanctum 通过在单一数据库表中存储用户 API 令牌，并通过应包含有效 API 令牌的 `Authorization` 头部来认证传入的 HTTP 请求，提供了这一功能。

#### 单页面应用认证

其次，Sanctum 提供了一种简单的方法来认证需要与 Laravel 驱动的 API 通信的单页应用程序（SPA）。这些 SPA 可能与你的 Laravel 应用存在于同一个仓库中，或者可能是一个完全独立的仓库，例如使用 Vue CLI 或 Next.js 应用程序创建的 SPA。

对于这一功能，Sanctum 不使用任何类型的令牌。相反，Sanctum 使用 Laravel 内置的基于 cookie 的会话认证服务。通常，Sanctum 利用 Laravel 的 `web` 认证守卫来实现这一点。这提供了 CSRF 保护、会话认证的好处，同时还防止了通过 XSS 泄露认证凭据。

当传入请求来自你自己的 SPA 前端时，Sanctum 仅尝试使用 cookie 进行认证。当 Sanctum 检查一个传入的 HTTP 请求时，它首先检查认证 cookie，如果不存在，Sanctum 然后会检查 `Authorization` 头部以寻找有效的 API 令牌。

> \[!NOTE\]  
> 仅使用 Sanctum 进行 API 令牌认证或仅用于 SPA 认证都是完全可以的。使用 Sanctum 并不意味着你必须使用它提供的两个功能。

## 安装

你可以通过 `install:api` Artisan 命令安装 Laravel Sanctum：

接下来，如果你计划使用 Sanctum 来认证一个 SPA，请参阅本文档的[单页面应用认证](#spa-authentication)部分。

## 配置

### 重写默认模型

虽然通常不需要，但你可以自由扩展 Sanctum 内部使用的 `PersonalAccessToken` 模型：

```php
use Laravel\Sanctum\PersonalAccessToken as SanctumPersonalAccessToken;

class PersonalAccessToken extends SanctumPersonalAccessToken
{
    // ...
}
```

然后，你可以通过 Sanctum 提供的 `usePersonalAccessTokenModel` 方法指定使用你的自定义模型。通常，你应该在应用程序的 `AppServiceProvider` 文件的 `boot` 方法中调用此方法：

```php
use App\Models\Sanctum\PersonalAccessToken;
use Laravel\Sanctum\Sanctum;

/**
 * 启动任何应用服务。
 */
public function boot(): void
{
    Sanctum::usePersonalAccessTokenModel(PersonalAccessToken::class);
}
```

## API 令牌认证

> \[!NOTE\]  
> 你不应使用 API 令牌来认证你自己的第一方 SPA。相反，应使用 Sanctum 的内置[单页面应用认证功能](#spa-authentication)。

### 发放 API 令牌

Sanctum 允许你发放可用于认证应用程序 API 请求的 API 令牌 / 个人访问令牌。在使用 API 令牌发出请求时，令牌应包含在 `Authorization` 头部作为一个 `Bearer` 令牌。

要开始为用户发放令牌，你的用户模型应使用 `Laravel\Sanctum\HasApiTokens` 特性：

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

要发放一个令牌，你可以使用 `createToken` 方法。`createToken` 方法返回一个 `Laravel\Sanctum\NewAccessToken` 实例。API 令牌在存储到你的数据库之前会使用 SHA-256 哈希加密，但你可以使用 `NewAccessToken` 实例的 `plainTextToken` 属性访问令牌的明文值。令牌创建后，你应立即向用户显示此值：

```php
use Illuminate\Http\Request;

Route::post('/tokens/create', function (Request $request) {
    $token = $request->user()->createToken($request->token_name);

    return ['token' => $token->plainTextToken];
});
```

你可以通过 `HasApiTokens` 特性提供的 `tokens` Eloquent 关系访问用户的所有令牌：

```php
foreach ($user->tokens as $token) {
    // ...
}
```

### 令牌权限

Sanctum 允许你为令牌分配 “权限”。权限的作用与 OAuth 的 “范围” 类似。你可以将字符串权限数组作为第二个参数传递给 `createToken` 方法：

```php
return $user->createToken('token-name', ['server:update'])->plainTextToken;
```

在处理由 Sanctum 认证的传入请求时，你可以使用 `tokenCan` 方法确定令牌是否具有给定的权限：

```php
if ($user->tokenCan('server:update')) {
    // ...
}
```

#### 令牌权限中间件

Sanctum 还包括两个中间件，可用于验证传入请求是否使用了被授予特定权限的令牌进行认证。要开始，请在应用程序的 `bootstrap/app.php` 文件中定义以下中间件别名：

```php
use Laravel\Sanctum\Http\Middleware\CheckAbilities;
use Laravel\Sanctum\Http\Middleware\CheckForAnyAbility;

->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'abilities' => CheckAbilities::class,
        'ability' => CheckForAnyAbility::class,
    ]);
})
```

`abilities` 中间件可以分配给一个路由，以验证传入请求的令牌是否具有所有列出的权限：

```php
Route::get('/orders', function () {
    // 令牌具有“检查状态”和“下订单”权限...
})->middleware(['auth:sanctum', 'abilities:check-status,place-orders']);
```

`ability` 中间件可以分配给一个路由，以验证传入请求的令牌是否至少具有列出的某个权限：

```php
Route::get('/orders', function () {
    // 令牌具有“检查状态”或“下订单”权限...
})->middleware(['auth:sanctum', 'ability:check-status,place-orders']);
```

#### 首方 UI 发起的请求

为了方便起见，如果传入的认证请求来自你的首方单页面应用（SPA），并且你使用的是 Sanctum 的内置[单页面应用认证](#spa-authentication)，那么 `tokenCan` 方法将始终返回 `true`。

然而，这并不意味着你的应用必须允许用户执行该操作。通常，你的应用的[授权策略](https://learnku.com/docs/laravel/11.x/authorization#creating-policies)将决定令牌是否被授予权限执行相关操作，并检查用户实例本身是否应被允许执行该操作。

例如，假设有一个管理服务器的应用，这可能意味着需要检查令牌是否有权更新服务器**并且**服务器属于该用户：

```php
return $request->user()->id === $server->user_id &&
       $request->user()->tokenCan('server:update')
```

起初，允许 `tokenCan` 方法被调用并始终对首方 UI 发起的请求返回 `true` 可能看起来很奇怪；然而，能够始终假设一个 API 令牌是可用的，并可以通过 `tokenCan` 方法检查，这是非常方便的。采取这种方法，你可以始终在应用的授权策略中调用 `tokenCan` 方法，而不用担心请求是由应用的 UI 触发的，还是由你的 API 的第三方消费者发起的。

### 保护路由

为了保护路由，确保所有传入请求都必须经过认证，你应该在你的 `routes/web.php` 和 `routes/api.php` 路由文件中，将 `sanctum` 认证守卫附加到你的受保护路由上。这个守卫将确保传入请求要么是有状态的、使用 Cookie 进行认证的请求，要么如果请求来自第三方，则包含有效的 API 令牌头。

你可能想知道为什么我们建议你在应用程序的 `routes/web.php` 文件中使用 `sanctum` 守卫来认证路由。请记住，Sanctum 首先会尝试使用 Laravel 的标准会话认证 Cookie 来认证传入的请求。如果没有该 Cookie，Sanctum 则会尝试使用请求的 `Authorization` 头部中的令牌来认证请求。此外，使用 Sanctum 认证所有请求确保我们始终可以在当前认证的用户实例上调用 `tokenCan` 方法：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

### 撤销令牌

你可以通过使用 `Laravel\Sanctum\HasApiTokens` 特性提供的 `tokens` 关系从数据库中删除令牌来 “撤销” 令牌：

```php
// 撤销所有令牌...
$user->tokens()->delete();

// 撤销用于认证当前请求的令牌...
$request->user()->currentAccessToken()->delete();

// 撤销特定令牌...
$user->tokens()->where('id', $tokenId)->delete();
```

### 令牌过期

默认情况下，Sanctum 令牌永不过期，只能通过[撤销令牌](#revoking-tokens)来使其失效。然而，如果你希望为你的应用程序的 API 令牌配置一个过期时间，你可以通过在应用程序的 `sanctum` 配置文件中定义的 `expiration` 配置选项来实现。此配置选项定义了发行令牌被视为过期的分钟数：

如果你希望独立指定每个令牌的过期时间，你可以通过将过期时间作为 `createToken` 方法的第三个参数来提供：

```php
return $user->createToken(
    'token-name', ['*'], now()->addWeek()
)->plainTextToken;
```

如果你为你的应用程序配置了令牌过期时间，你可能还希望[安排一个任务](https://learnku.com/docs/laravel/11.x/scheduling)来清理应用程序中已过期的令牌。幸运的是，Sanctum 提供了一个 `sanctum:prune-expired` Artisan 命令，你可以用它来完成这项工作。例如，你可以配置一个定时任务，删除至少已过期 24 小时的所有过期令牌数据库记录：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('sanctum:prune-expired --hours=24')->daily();
```

## 单页面应用认证

Sanctum 也旨在提供一种简单的方法来认证需要与 Laravel 驱动的 API 通信的单页面应用（SPA）。这些 SPA 可能与你的 Laravel 应用存在于同一个仓库中，或者可能是一个完全独立的仓库。

对于这个功能，Sanctum 不使用任何形式的令牌。相反，Sanctum 使用 Laravel 内置的基于 Cookie 的会话认证服务。这种认证方式提供了 CSRF 保护、会话认证的好处，同时也防止了通过 XSS 泄露认证凭证。

> \[!WARNING\]  
> 为了进行认证，你的 SPA 和 API 必须共享相同的顶级域名。然而，它们可以放在不同的子域上。此外，你应确保发送 `Accept: application/json` 头和 `Referer` 或 `Origin` 头中的任意一个与你的请求一起。

### 配置

#### 配置你的首方域名

首先，你应该配置你的 SPA 将从哪些域名发出请求。你可以在你的 `sanctum` 配置文件中使用 `stateful` 配置选项来配置这些域名。此配置设置决定了哪些域名在向你的 API 发出请求时将保持 “有状态” 的认证，使用 Laravel 会话 Cookie。

> \[!WARNING\]  
> 如果你是通过包含端口的 URL（如 `127.0.0.1:8000`）访问你的应用程序，你应确保你在域名中包含端口号。

#### Sanctum 中间件

接下来，你应该指导 Laravel 让来自你的 SPA 的请求可以使用 Laravel 的会话 Cookie 进行认证，同时允许来自第三方或移动应用的请求使用 API 令牌进行认证。这可以通过在你的应用程序的 `bootstrap/app.php` 文件中调用 `statefulApi` 中间件方法轻松完成：

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->statefulApi();
})
```

#### CORS 和 Cookies

如果你在尝试从执行在不同子域上的 SPA 认证你的应用程序时遇到问题，你可能配置错误了你的 CORS（跨源资源共享）或会话 Cookie 设置。

`config/cors.php` 配置文件默认情况下不会发布。如果你需要自定义 Laravel 的 CORS 选项，你应该使用 `config:publish` Artisan 命令发布完整的 `cors` 配置文件：

```bash
php artisan config:publish cors
```

接下来，你应确保你的应用程序的 CORS 配置返回 `Access-Control-Allow-Credentials` 头，并且其值为 `True`。这可以通过在你的应用程序的 `config/cors.php` 配置文件中设置 `supports_credentials` 选项为 `true` 来实现。

此外，你应该在你的应用程序的全局 `axios` 实例上启用 `withCredentials` 和 `withXSRFToken` 选项。通常，这应该在你的 `resources/js/bootstrap.js` 文件中执行。如果你没有使用 Axios 从前端发起 HTTP 请求，你应该在你自己的 HTTP 客户端上执行等效配置：

```js
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;
```

最后，你应该确保你的应用程序的会话 Cookie 域配置支持你根域名的任何子域。你可以通过在你的应用程序的 `config/session.php` 配置文件中为域名前加上一个点 `.` 来实现这一点：

```php
'domain' => '.domain.com',
```

### 认证流程

#### CSRF 保护

为了认证你的单页面应用（SPA），你的 SPA 的 “登录” 页面首先应该向 `/sanctum/csrf-cookie` 端点发起请求，以初始化应用程序的 CSRF 保护：

```js
axios.get('/sanctum/csrf-cookie').then(response => {
    // 登录...
});
```

在此请求期间，Laravel 会设置一个包含当前 CSRF 令牌的 `XSRF-TOKEN` Cookie。然后，这个令牌应该在后续请求中通过 `X-XSRF-TOKEN` 头传递，一些 HTTP 客户端库如 Axios 和 Angular HttpClient 会自动为你做这件事。如果你的 JavaScript HTTP 库没有为你设置这个值，你需要手动设置 `X-XSRF-TOKEN` 头，以匹配由此路由设置的 `XSRF-TOKEN` Cookie 的值。

#### 登录

一旦初始化了 CSRF 保护，你应该向你的 Laravel 应用的 `/login` 路径发送一个 `POST` 请求。这个 `/login` 路径可以[手动实现](https://learnku.com/docs/laravel/11.x/authentication#authenticating-users)或使用无头认证包如 [Laravel Fortify](https://learnku.com/docs/laravel/11.x/fortify)。

如果登录请求成功，你将被认证，随后对你的应用程序路由的请求将通过 Laravel 应用颁发给你的客户端的会话 Cookie 自动认证。此外，由于你的应用已经向 `/sanctum/csrf-cookie` 路径发起了请求，只要你的 JavaScript HTTP 客户端在 `X-XSRF-TOKEN` 头中发送 `XSRF-TOKEN` Cookie 的值，后续请求应自动接收 CSRF 保护。

当然，如果你的用户会话因活动不足而过期，随后对 Laravel 应用的请求可能会收到 401 或 419 HTTP 错误响应。在这种情况下，你应该将用户重定向到你的 SPA 的登录页面。

> \[!WARNING\]  
> 你可以自行编写你的 `/login` 端点；然而，你应该确保它使用 Laravel 提供的标准[基于会话的认证服务](https://learnku.com/docs/laravel/11.x/authentication#authenticating-users)来认证用户。通常，这意味着使用 `web` 认证守卫。

### 保护路由

为了保护路由，确保所有进来的请求都必须经过认证，你应该在你的 `routes/api.php` 文件中为你的 API 路由附加 `sanctum` 认证守卫。这个守卫将确保进来的请求要么是来自你的 SPA 的有状态认证请求，要么如果是来自第三方的请求，包含一个有效的 API 令牌头：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

### 授权私有广播频道

如果你的 SPA 需要与[私有 / 存在广播频道](https://learnku.com/docs/laravel/11.x/broadcasting#authorizing-channels)进行认证，你应该从你的应用程序的 `bootstrap/app.php` 文件中的 `withRouting` 方法中移除 `channels` 条目。相反，你应该调用 `withBroadcasting` 方法，以便为你的应用程序的广播路由指定正确的中间件：

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        // ...
    )
    ->withBroadcasting(
        __DIR__.'/../routes/channels.php',
        ['prefix' => 'api', 'middleware' => ['api', 'auth:sanctum']],
    )
```

接下来，为了使 Pusher 的授权请求成功，你需要在初始化 [Laravel Echo](https://learnku.com/docs/laravel/11.x/broadcasting#client-side-installation) 时提供一个自定义的 Pusher `authorizer`。这允许你的应用配置 Pusher 使用[正确配置了跨域请求的](#cors-and-cookies) `axios` 实例：

```js
window.Echo = new Echo({
    broadcaster: "pusher",
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    encrypted: true,
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    authorizer: (channel, options) => {
        return {
            authorize: (socketId, callback) => {
                axios.post('/api/broadcasting/auth', {
                    socket_id: socketId,
                    channel_name: channel.name
                })
                .then(response => {
                    callback(false, response.data);
                })
                .catch(error => {
                    callback(true, error);
                });
            }
        };
    },
})
```

## 移动应用认证

你也可以使用 Sanctum 令牌来认证你的移动应用对 API 的请求。认证移动应用请求的过程与认证第三方 API 请求类似；然而，在发放 API 令牌的方式上有些许差异。

### 发放 API 令牌

首先，创建一个路由，接受用户的电子邮件 / 用户名、密码和设备名称，然后用这些凭证交换一个新的 Sanctum 令牌。此端点所需的 “设备名称” 仅为信息目的，可以是你希望的任何值。通常，设备名称应该是用户能认出的名称，比如 “Nuno 的 iPhone 12”。

通常，你会从移动应用的 “登录” 屏幕向令牌端点发出请求。该端点将返回纯文本 API 令牌，该令牌随后可存储在移动设备上并用于发起额外的 API 请求：

```php
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

Route::post('/sanctum/token', function (Request $request) {
    $request->validate([
        'email' => 'required|email',
        'password' => 'required',
        'device_name' => 'required',
    ]);

    $user = User::where('email', $request->email)->first();

    if (!$user || !Hash::check($request->password, $user->password)) {
        throw ValidationException::withMessages([
            'email' => ['提供的凭证不正确。'],
        ]);
    }

    return $user->createToken($request->device_name)->plainTextToken;
});
```

当移动应用使用令牌向你的应用发起 API 请求时，应该在 `Authorization` 头中作为 `Bearer` 令牌传递该令牌。

> \[!NOTE\]  
> 在为移动应用发放令牌时，你也可以自由指定[令牌权限](#token-abilities)。

### 保护路由

如前文所述，你可以通过在路由上附加 `sanctum` 认证守卫来保护路由，确保所有进入的请求都必须经过认证：

```php
Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

### 撤销令牌

为了允许用户撤销发放给移动设备的 API 令牌，你可以在你的网页应用的 “账户设置” 部分列出这些令牌，并提供一个 “撤销” 按钮。当用户点击 “撤销” 按钮时，你可以从数据库中删除该令牌。记住，你可以通过 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 关系访问用户的 API 令牌：

```php
// 撤销所有令牌...
$user->tokens()->delete();

// 撤销特定令牌...
$user->tokens()->where('id', $tokenId)->delete();
```

## 测试

在测试时，可以使用 `Sanctum::actingAs` 方法来认证用户，并指定应授予其令牌的权限：

```php
use App\Models\User;
use Laravel\Sanctum\Sanctum;

test('task list can be retrieved', function () {
    Sanctum::actingAs(
        User::factory()->create(),
        ['view-tasks']
    );

    $response = $this->get('/api/task');

    $response->assertOk();
});
```

```php
use App\Models\User;
use Laravel\Sanctum\Sanctum;

public function test_task_list_can_be_retrieved(): void
{
    Sanctum::actingAs(
        User::factory()->create(),
        ['view-tasks']
    );

    $response = $this->get('/api/task');

    $response->assertOk();
}
```

如果你想授予令牌所有权限，应在 `actingAs` 方法提供的权限列表中包含 `*`：

```php
Sanctum::actingAs(
    User::factory()->create(),
    ['*']
);
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/sa...](https://learnku.com/docs/laravel/11.x/sanctummd/16732)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/sa...](https://learnku.com/docs/laravel/11.x/sanctummd/16732)