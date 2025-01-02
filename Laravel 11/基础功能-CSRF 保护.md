本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## CSRF 保护

+   [简介](#csrf-introduction)
+   [阻止 CSRF 请求](#preventing-csrf-requests)
    +   [排除 URIs](#csrf-excluding-uris)
+   [X-CSRF-TOKEN](#csrf-x-csrf-token)
+   [X-XSRF-TOKEN](#csrf-x-xsrf-token)

## 简介

跨站点请求伪造是一种恶意攻击类型，利用这种手段，代表经过身份验证的用户执行未经授权的命令。值得庆幸的是，Laravel 可以轻松保护你的应用程序免受[跨站点请求伪造](https://en.wikipedia.org/wiki/Cross-site_request_forgery)（CSRF）攻击。

#### 漏洞的解释

如果你不熟悉跨站点请求伪造，我们讨论一个利用此漏洞的示例。假设你的应用程序有一个 `/user/email` 路由，它接受 `POST` 请求来更改经过身份验证用户的电子邮件地址。最有可能的情况是，此路由希望 `email` 输入字段包含用户希望开始使用的电子邮件地址。

没有 CSRF 保护，恶意网站可能会创建一个 HTML 表单，指向你的应用程序 `/user/email` 路由，并提交恶意用户自己的电子邮件地址：

```blade
<form action="https://your-application.com/user/email" method="POST">
    <input type="email" value="malicious-email@example.com">
</form>

<script>
    document.forms[0].submit();
</script>
```

如果恶意网站在页面加载时自动提交了表单，则恶意用户只需要诱使你的应用程序的一个毫无戒心的用户访问他们的网站，他们的电子邮件地址就会在你的应用程序中更改。

为了防止这种漏洞，我们需要检查每一个传入的 `POST`，`PUT`，`PATCH` 或 `DELETE` 请求以获取恶意应用程序无法访问的秘密会话值。

## 阻止 CSRF 请求

Laravel 会自动为应用程序管理的每个活动 [用户会话](https://learnku.com/docs/laravel/11.x/sessionmd) 生成一个 CSRF 「令牌」。此令牌用于验证经过身份验证的用户是否是实际向应用程序发出请求的人。由于此令牌存储在用户的会话中，并且每次重新生成会话时都会发生变化，因此恶意应用程序无法访问它。

当前会话的 CSRF 令牌可以通过请求的会话或 `csrf_token` 辅助函数来访问：

```php
use Illuminate\Http\Request;

Route::get('/token', function (Request $request) {
    $token = $request->session()->token();

    $token = csrf_token();

    // ...
});
```

每当你在应用程序中定义 `POST` 、 `PUT` 、 `PATCH` 或 `DELETE` HTML 表单时，都应该在表单中包含一个隐藏的 CSRF `_token` 字段，以便 CSRF 保护中间件可以验证请求。为了方便起见，你可以使用 `@csrf` Blade 指令来生成隐藏令牌输入字段：

```blade
<form method="POST" action="/profile">
    @csrf

    <!-- 相当于... -->
    <input type="hidden" name="_token" value="{{ csrf_token() }}" />
</form>
```

默认情况下包含在 `web` 中间件组中的 `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` [中间件](https://learnku.com/docs/laravel/11.x/middlewaremd) 将自动验证请求输入中的令牌是否与会话中存储的令牌匹配。当这两个令牌匹配时，我们就知道经过身份验证的用户是发起请求的用户。

### CSRF Tokens & SPAs

如果你正在构建使用 Laravel 作为 API 后端的 SPA，则应查阅 [Laravel Sanctum](https://learnku.com/docs/laravel/11.x/sanctummd) 文档，了解有关使用 API 进行身份验证和防范 CSRF 漏洞的信息。

### 从 CSRF 保护中排除 URI

有时你可能希望从 CSRF 保护中排除一组 URIs。例如，如果你使用 [Stripe](https://stripe.com/) 处理付款并使用他们的 webhook 系统，则需要将你的 Stripe webhook 处理程序路由从 CSRF 保护中排除，因为 Stripe 不会知道要向你的路由发送什么 CSRF 令牌。

通常，你应该将这类路由放置在 Laravel 应用于 `routes/web.php` 文件中所有路由的 `web` 中间件组之外。但是，你也可以通过将特定路由的 URI 提供给应用程序的 `bootstrap/app.php` 文件中的 `validateCsrfTokens` 方法来排除这些路由：

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(except: [
        'stripe/*',
        'http://example.com/foo/bar',
        'http://example.com/foo/*',
    ]);
})
```

> **技巧**  
> 为方便起见，[运行测试](https://learnku.com/docs/laravel/11.x/testingmd)时自动禁用所有路由的 CSRF 中间件。

## X-CSRF-TOKEN

除了检查 POST 参数中的 CSRF 令牌外，`Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` 中间件默认包含在 `web` 中间件组中，它还会检查 `X-CSRF-TOKEN` 请求头。例如，你可以将令牌存储在 HTML 的 `meta` 标签中：

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

然后，你可以指示 jQuery 之类的库自动将令牌添加到所有请求标头。 这为使用传统 JavaScript 技术的基于 AJAX 的应用程序提供了简单、方便的 CSRF 保护：

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```

## X-XSRF-TOKEN

Laravel 将当前 CSRF 令牌存储在加密的 `XSRF-TOKEN` cookie 中，该 cookie 包含在框架生成的每个响应中。你可以使用 cookie 值设置 `X-XSRF-TOKEN` 请求标头。

由于一些 JavaScript 框架和库（如 Angular 和 Axios ）会自动将其值放置在同一源请求的 `X-XSRF-TOKEN` 标头中，因此发送此 cookie 主要是为了方便开发人员。

> **技巧**  
> 默认情况下，`resources/js/bootstrap.js` 文件包含 Axios HTTP 库，它会自动为你发送 `X-XSRF-TOKEN` 标头。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/cs...](https://learnku.com/docs/laravel/11.x/csrfmd/16659)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/cs...](https://learnku.com/docs/laravel/11.x/csrfmd/16659)