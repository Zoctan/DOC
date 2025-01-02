本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Folio

+   [介绍](#introduction)
+   [安装](#installation)
    +   [页面路径 / URI](#page-paths-uris)
    +   [子域名路由](#subdomain-routing)
+   [创建路由](#creating-routes)
    +   [嵌套路由](#nested-routes)
    +   [Index 路由](#index-routes)
+   [路由参数](#route-parameters)
+   [路由模型绑定](#route-model-binding)
    +   [软删除模型](#soft-deleted-models)
+   [渲染钩子](#render-hooks)
+   [命名路由](#named-routes)
+   [中间件](#middleware)
+   [路由缓存](#route-caching)

## 介绍

[Laravel Folio](https://github.com/laravel/folio) 是一款功能强大的基于页面的路由，旨在简化 Laravel 应用中的路由。使用 Laravel Folio，生成路由就像在应用的 `resources/views/pages` 目录中创建 Blade 模板一样轻松。

比如，要在 `/greeting` URL 上创建一个可访问的页面，只需在应用的 `resources/views/pages` 目录下创建一个 `greeting.blade.php` 文件：

## 安装

要开始使用 Folio，请使用 Composer 包管理器将 Folio 安装到你的项目中：

```bash
composer require laravel/folio
```

在安装完 Folio 后，你可用执行 `folio:install` Artisan 命令，该命令将在应用中安装 Folio 的服务提供者。此服务提供者注册了目录，Folio 将在该目录中搜索路由 / 页面：

```bash
php artisan folio:install
```

### 页面路径 / URIs

默认情况下，Folio 从应用的 `resources/views/pages` 目录中提供页面，但你可以在 Folio 服务提供者的 `boot` 方法中自定义这些目录。

比如，有时在同一个 Laravel 应用中指定多个 Folio 路径可能很方便。你可能希望为应用的「admin」区域提供一个单独的 Folio 页面目录，同时让应用其余页面使用另一个目录。

你可以使用 `Folio::path` 和 `Folio::uri` 方法实现。`path` 方法注册了一个目录，Folio 将在路由传入的 HTTP 请求时扫描该目录中的页面，而 `uri` 方法则为该页面目录指定「基本 URI」：

```php
use Laravel\Folio\Folio;

Folio::path(resource_path('views/pages/guest'))->uri('/');

Folio::path(resource_path('views/pages/admin'))
    ->uri('/admin')
    ->middleware([
        '*' => [
            'auth',
            'verified',

            // ...
        ],
    ]);
```

### 子域名路由

你也可以基于传入请求的子域名路由到页面。比如，你可能希望将请求从 `admin.example.com` 路由到与 Folio 页面其他部分不同的页面目录。你可以通过在调用 `Folio::path` 方法之后调用 `domain` 方法来实现这一点：

```php
use Laravel\Folio\Folio;

Folio::domain('admin.example.com')
    ->path(resource_path('views/pages/admin'));
```

`domain` 方法也允许你捕获域名或子域名的部分作为参数。这些参数将被注入到页面模板中：

```php
use Laravel\Folio\Folio;

Folio::domain('{account}.example.com')
    ->path(resource_path('views/pages/admin'));
```

## 创建路由

你可以通过在任何 Folio 安装的目录中放置 Blade 模板来创建 Folio 路由。默认情况下，Folio 将挂载 `resources/views/pages` 目录，但你可以在 Folio 服务提供者的 `boot` 方法中自定义这些目录。

将 Blade 模板放入 Folio 安装的目录后，你可以立即通过浏览器访问它。比如，位于 `pages/schedule.blade.php` 中的页面可以在浏览器的 `http://example.com/schedule` 中访问。

要快速查看所有的 Folio 页面 / 路由列表，你可以调用 `folio:list` Artisan 命令：

### 嵌套路由

你可以在 Folio 的其中一个目录中通过创建一个或多个目录，创建嵌套路由。比如，要创建一个通过 `user/profile` 访问的页面，请在 `pages/user` 目录中创建一个 `profile.blade.php` 模板：

```bash
php artisan folio:page user/profile

# pages/user/profile.blade.php → /user/profile
```

### Index 路由

有时，你希望让某个给定页面成为目录中的「index」页。在 Folio 目录中放置一个 `index.blade.php` 模板，任何指向该路由的根目录将路由到此页面中：

```bash
php artisan folio:page index
# pages/index.blade.php → /

php artisan folio:page users/index
# pages/users/index.blade.php → /users
```

## 路由参数

通常，你需要将请求的 URL 片段插入到页面中，以便与它们进行交互。比如，你可能需要访问用户简介页面的用户 `ID`。为此，你可以将页面文件名的一段封装在方括号中：

```bash
php artisan folio:page "users/[id]"

# pages/users/[id].blade.php → /users/1
```

捕获的 URL 分段可以作为 Blade 模板中的变量进行访问：

```html
<div>
    User {{ $id }}
</div>
```

要捕获多个 URL 段，你可以使用三个号 `...` 作为 URL 分段的前缀；

```bash
php artisan folio:page "users/[...ids]"

# pages/users/[...ids].blade.php → /users/1/2/3
```

当捕获多个 URL 分段时，捕获的分段将作为数组注入到页面中：

```html
<ul>
    @foreach ($ids as $id)
        <li>User {{ $id }}</li>
    @endforeach
</ul>
```

## 路由模型绑定

如果页面模板文件名的通配符分段对应于应用的 Eloquent 模型之一，Folio 将自动利用 Laravel 的路由模型绑定功能，并尝试将解析的模型实例注入页面：

```bash
php artisan folio:page "users/[User]"

# pages/users/[User].blade.php → /users/1
```

捕获的模型可以作为 Blade 模板中的变量进行访问。模型的变量名称将转换为「驼峰式」：

```html
<div>
    User {{ $user->id }}
</div>
```

#### 自定义键

有时候你需要使用非 `id` 的字段解析绑定的 Eloquent 模型。要实现此功能，你可以指定页面文件名中的字段。比如，使用 `[Post:slug].blade.php` 文件名的页面将会尝试通过 `slug` 字段而非 `id` 字段来解析绑定的模型。

在 Windows 上，请使用 `-` 来分隔模型名和键名：`[Post-slug].blade.php`。

#### Model 位置

默认情况下，Folio 将在应用的 `app/Models` 目录中搜索模型。但是，如果需要，可以在模板的文件名中指定完全限定的模型类名：

```bash
php artisan folio:page "users/[.App.Models.User]"

# pages/users/[.App.Models.User].blade.php → /users/1
```

### 软删除模型

默认情况下，解析隐式模型绑定时不会检索已软删除的模型。但是，如果愿意，你可以通过调用页面模板中的 `withTrashed` 函数来指示 Folio 检索软删除的模型：

```php
<?php

use function Laravel\Folio\{withTrashed};

withTrashed();

?>

<div>
    User {{ $user->id }}
</div>
```

## 渲染钩子

Folio 将返回页面 Blade 模板的内容，作为对传入请求的响应。但是，你可以通过调用页面模板中的 `render` 函数来自定义响应。

`render` 函数接受一个闭包，该闭包将接收 Folio 正在渲染的 `View` 实例，允许你向视图添加其他数据或自定义整个响应。除了接收 `View` 实例之外，还将向 `render` 闭包提供所有额外的路由参数或模型绑定：

```php
<?php

use App\Models\Post;
use Illuminate\Support\Facades\Auth;
use Illuminate\View\View;

use function Laravel\Folio\render;

render(function (View $view, Post $post) {
    if (! Auth::user()->can('view', $post)) {
        return response('Unauthorized', 403);
    }

    return $view->with('photos', $post->author->photos);
}); ?>

<div>
    {{ $post->content }}
</div>

<div>
    This author has also taken {{ count($photos) }} photos.
</div>
```

## 命名路由

你可以使用 `name` 函数指定给定页面的路由名：

```php
<?php

use function Laravel\Folio\name;

name('users.index');
```

就如 Laravel 的命名路由，你可以使用 `route` 函数生成到已分配名称的 Folio 页面的 URL:

```php
<a href="{{ route('users.index') }}">
    All Users
</a>
```

如果页面中有参数，你可以将其传给 `route` 函数：

```php
route('users.show', ['user' => $user]);
```

## 中间件

通过调用页面模板中的 `middleware` 函数，你可以将中间件应用到指定页面：

```php
<?php

use function Laravel\Folio\{middleware};

middleware(['auth', 'verified']);

?>

<div>
    Dashboard
</div>
```

或者，要将中间件分配给一组页面，你可以在调用 `Folio::path` 方法后链式调用 `middleware` 方法。

要指定那些页面使用中间件，中间件数组的键请对想适用的页面使用对应的 URL 模式。字符 `*` 可以作为通配符使用：

```php
use Laravel\Folio\Folio;

Folio::path(resource_path('views/pages'))->middleware([
    'admin/*' => [
        'auth',
        'verified',

        // ...
    ],
]);
```

你可以在数组中间站中使用闭包来定义内联的匿名中间件：

```php
use Closure;
use Illuminate\Http\Request;
use Laravel\Folio\Folio;

Folio::path(resource_path('views/pages'))->middleware([
    'admin/*' => [
        'auth',
        'verified',

        function (Request $request, Closure $next) {
            // ...

            return $next($request);
        },
    ],
]);
```

## 路由缓存

使用 Folio 时，你应该始终使用 [Laravel 的路由缓存功能](https://learnku.com/docs/laravel/11.x/routing#route-caching)。Folio 监听 `route:cache` 的 Artisan 命令，以确保 Folio 的页面定义及路由名正确地缓存以获得最大性能。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/fo...](https://learnku.com/docs/laravel/11.x/foliomd/16719)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/fo...](https://learnku.com/docs/laravel/11.x/foliomd/16719)