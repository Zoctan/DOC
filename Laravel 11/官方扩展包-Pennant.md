本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Pennant

+   [介绍](#introduction)
+   [安装](#installation)
+   [配置](#configuration)
+   [定义特性](#defining-features)
    +   [基于类的特性](#class-based-features)
+   [检查特性](#checking-features)
    +   [条件执行](#conditional-execution)
    +   [`HasFeatures` 特性](#the-has-features-trait)
    +   [Blade 指令](#blade-directive)
    +   [中间件](#middleware)
    +   [内存缓存](#in-memory-cache)
+   [范围](#scope)
    +   [指定范围](#specifying-the-scope)
    +   [默认范围](#default-scope)
    +   [可空范围](#nullable-scope)
    +   [识别范围](#identifying-scope)
    +   [序列化范围](#serializing-scope)
+   [丰富的特性数值](#rich-feature-values)
+   [检索多个特性](#retrieving-multiple-features)
+   [预加载](#eager-loading)
+   [更新数值](#updating-values)
    +   [批量更新](#bulk-updates)
    +   [清除特性](#purging-features)
+   [测试](#testing)
+   [添加自定义 Pennant 驱动](#adding-custom-pennant-drivers)
    +   [实现驱动](#implementing-the-driver)
    +   [注册驱动](#registering-the-driver)
+   [事件](#events)

## 介绍

[Laravel Pennant](https://github.com/laravel/pennant) 是一个简单且轻量级的特性标志包 - 没有多余的东西。特性标志使您能够有信心逐步推出新的应用程序特性、对新界面设计进行 A/B 测试、补充基于主干的开发策略等等。

## 安装

首先，使用 Composer 包管理器将 Pennant 安装到您的项目中：

```shell
composer require laravel/pennant
```

接下来，您应该使用 `vendor:publish` Artisan 命令发布 Pennant 的配置和迁移文件：

```shell
php artisan vendor:publish --provider="Laravel\Pennant\PennantServiceProvider"
```

最后，您应该运行应用程序的数据库迁移。这将创建一个 `features` 表，Pennant 使用它来支持其 `database` 驱动：

## 配置

在发布 Pennant 的资源之后，其配置文件将位于 `config/pennant.php`。这个配置文件允许您指定 Pennant 将用于存储已解析的特性标志值的默认存储机制。

Pennant 支持通过 `array` 驱动在内存数组中存储已解析的特性标志值。此外，Pennant 还可以通过 `database` 驱动将已解析的特性标志值持久存储在关系数据库中，这是 Pennant 默认使用的存储机制。

## 定义特性

要定义一个特性，你可以使用 `Feature` 门面提供的 `define` 方法。你需要为特性提供一个名称，以及一个将被调用以解析特性初始值的闭包。

通常，特性在服务提供者中使用 `Feature` 门面定义。闭包将接收特性检查的 “范围”。最常见的情况是，范围是当前经过身份验证的用户。在这个示例中，我们将为逐步向我们应用程序的用户推出一个新的 API 定义一个特性：

```php
<?php

namespace App\Providers;

use App\Models\User;
use Illuminate\Support\Lottery;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Feature::define('new-api', fn (User $user) => match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        });
    }
}
```

如你所见，我们对特性有以下规则：

+   所有内部团队成员应该使用新的 API。
+   任何高流量客户不应该使用新的 API。
+   否则，该特性应随机分配给用户，激活的几率为 1/100。

第一次针对给定用户检查 `new-api` 特性时，闭包的结果将由存储驱动存储。下次再针对同一用户检查特性时，值将从存储中检索，闭包将不会被调用。

为了方便起见，如果一个特性定义只返回一个抽奖结果，你可以完全省略闭包：

```php
Feature::define('site-redesign', Lottery::odds(1, 1000));
```

### 基于类的特性

Pennant 还允许你定义基于类的特性。与基于闭包的特性定义不同，无需在服务提供者中注册基于类的特性。要创建一个基于类的特性，你可以调用 `pennant:feature` Artisan 命令。默认情况下，特性类将放置在你的应用程序的 `app/Features` 目录中：

```shell
php artisan pennant:feature NewApi
```

在编写特性类时，你只需要定义一个 `resolve` 方法，该方法将被调用以解析给定范围的特性的初始值。同样，范围通常是当前经过身份验证的用户：

```php
<?php

namespace App\Features;

use Illuminate\Support\Lottery;

class NewApi
{
    /**
     * 解析特性的初始值。
     */
    public function resolve(User $user): mixed
    {
        return match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        };
    }
}
```

> \[!NOTE\] 特性类通过[容器](https://learnku.com/docs/laravel/11.x/container)解析，因此在需要时你可以将依赖项注入到特性类的构造函数中。

#### 自定义存储的特性名称

默认情况下，Pennant 将存储特性类的完全限定类名。如果你想要将存储的特性名称与应用程序的内部结构解耦，你可以在特性类上指定一个 `$name` 属性。该属性的值将代替类名存储：

```php
<?php

namespace App\Features;

class NewApi
{
    /**
     * 特性的存储名称。
     *
     * @var string
     */
    public $name = 'new-api';

    // ...
}
```

## 检查特性

要确定一个特性是否激活，你可以在 `Feature` 门面上使用 `active` 方法。默认情况下，特性将针对当前经过身份验证的用户进行检查：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * 显示资源列表。
     */
    public function index(Request $request): Response
    {
        return Feature::active('new-api')
                ? $this->resolveNewApiResponse($request)
                : $this->resolveLegacyApiResponse($request);
    }

    // ...
}
```

尽管默认情况下特性是针对当前经过身份验证的用户进行检查的，但你可以轻松地针对其他用户或[范围](#scope)检查特性。为此，使用 `Feature` 门面提供的 `for` 方法：

```php
return Feature::for($user)->active('new-api')
        ? $this->resolveNewApiResponse($request)
        : $this->resolveLegacyApiResponse($request);
```

Pennant 还提供了一些额外的便利方法，在确定特性是否激活时可能会很有用：

```php
// 确定所有给定的特性是否都激活...
Feature::allAreActive(['new-api', 'site-redesign']);

// 确定给定的任何特性是否有激活的...
Feature::someAreActive(['new-api', 'site-redesign']);

// 确定特性是否不激活...
Feature::inactive('new-api');

// 确定所有给定的特性是否都不激活...
Feature::allAreInactive(['new-api', 'site-redesign']);

// 确定给定的任何特性是否不激活...
Feature::someAreInactive(['new-api', 'site-redesign']);
```

> **注意**  
> 当在 HTTP 上下文之外使用 Pennant，比如在 Artisan 命令或队列作业中，你通常应该[明确指定特性的范围](#specifying-the-scope)。或者，你可以定义一个[默认范围](#default-scope)，既考虑到经过身份验证的 HTTP 上下文，也考虑到未经身份验证的上下文。

#### 检查基于类的特性

对于基于类的特性，在检查特性时，你应该提供类名：

```php
<?php

namespace App\Http\Controllers;

use App\Features\NewApi;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * 显示资源列表。
     */
    public function index(Request $request): Response
    {
        return Feature::active(NewApi::class)
                ? $this->resolveNewApiResponse($request)
                : $this->resolveLegacyApiResponse($request);
    }

    // ...
}
```

### 条件执行

`when` 方法可用于流畅地执行给定闭包，如果特性是激活的话。此外，可以提供第二个闭包，如果特性不激活，则执行该闭包：

```php
<?php

namespace App\Http\Controllers;

use App\Features\NewApi;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * 显示资源列表。
     */
    public function index(Request $request): Response
    {
        return Feature::when(NewApi::class,
            fn () => $this->resolveNewApiResponse($request),
            fn () => $this->resolveLegacyApiResponse($request),
        );
    }

    // ...
}
```

`unless` 方法作为 `when` 方法的反向，如果特性不激活，则执行第一个闭包：

```php
return Feature::unless(NewApi::class,
    fn () => $this->resolveLegacyApiResponse($request),
    fn () => $this->resolveNewApiResponse($request),
);
```

### `HasFeatures` 特性

Pennant 的 `HasFeatures` 特性可以添加到你的应用程序的 `User` 模型（或任何其他具有特性的模型）中，以提供一种流畅、方便的方式直接从模型中检查特性：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Pennant\Concerns\HasFeatures;

class User extends Authenticatable
{
    use HasFeatures;

    // ...
}
```

一旦将特性添加到你的模型中，你可以通过调用 `features` 方法轻松地检查特性：

```php
if ($user->features()->active('new-api')) {
    // ...
}
```

当然，`features` 方法还提供了许多其他方便的方法来与特性进行交互：

```php
// 值...
$value = $user->features()->value('purchase-button');
$values = $user->features()->values(['new-api', 'purchase-button']);

// 状态...
$user->features()->active('new-api');
$user->features()->allAreActive(['new-api', 'server-api']);
$user->features()->someAreActive(['new-api', 'server-api']);

$user->features()->inactive('new-api');
$user->features()->allAreInactive(['new-api', 'server-api']);
$user->features()->someAreInactive(['new-api', 'server-api']);

// 条件执行...
$user->features()->when('new-api',
    fn () => /* ... */,
    fn () => /* ... */,
);

$user->features()->unless('new-api',
    fn () => /* ... */,
    fn () => /* ... */,
);
```

### Blade 指令

为了使在 Blade 中检查特性变得更加无缝，Pennant 提供了一个 `@feature` 指令：

```blade
@feature('site-redesign')
    <!-- 'site-redesign' 是激活的 -->
@else
    <!-- 'site-redesign' 是不激活的 -->
@endfeature
```

### 中间件

Pennant 还包括一个[中间件](https://learnku.com/docs/laravel/11.x/middleware)，可用于在路由被调用之前验证当前经过身份验证的用户是否有权限访问某个特性。你可以将中间件分配给一个路由，并指定访问该路由所需的特性。如果当前经过身份验证的用户的任何指定特性未激活，则路由将返回一个 `400 Bad Request` HTTP 响应。多个特性可以传递给静态的 `using` 方法。

```php
use Illuminate\Support\Facades\Route;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

Route::get('/api/servers', function () {
    // ...
})->middleware(EnsureFeaturesAreActive::using('new-api', 'servers-api'));
```

#### 自定义响应

如果你想要自定义中间件在列出的特性之一未激活时返回的响应，你可以使用 `EnsureFeaturesAreActive` 中间件提供的 `whenInactive` 方法。通常，此方法应该在你的应用程序的某个服务提供者的 `boot` 方法中调用：

```php
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    EnsureFeaturesAreActive::whenInactive(
        function (Request $request, array $features) {
            return new Response(status: 403);
        }
    );

    // ...
}
```

### 内存缓存

在检查特性时，Pennant 将创建一个结果的内存缓存。如果你正在使用 `database` 驱动程序，这意味着在单个请求中重新检查相同的特性标志将不会触发额外的数据库查询。这也确保了特性在请求的整个持续时间内具有一致的结果。

如果你需要手动刷新内存缓存，可以使用 `Feature` 门面提供的 `flushCache` 方法：

## 作用域

### 指定作用域

正如讨论的那样，通常会针对当前经过身份验证的用户检查特性。然而，这可能并不总是符合你的需求。因此，可以通过 `Feature` 门面的 `for` 方法指定要针对的作用域以检查给定特性：

```php
return Feature::for($user)->active('new-api')
        ? $this->resolveNewApiResponse($request)
        : $this->resolveLegacyApiResponse($request);
```

当然，特性作用域不仅限于 “用户”。想象一下，你构建了一个新的计费体验，你要将其推出给整个团队，而不是单个用户。也许你希望最老的团队比较新的团队推出得慢一些。你的特性解析闭包可能如下所示：

```php
use App\Models\Team;
use Carbon\Carbon;
use Illuminate\Support\Lottery;
use Laravel\Pennant\Feature;

Feature::define('billing-v2', function (Team $team) {
    if ($team->created_at->isAfter(new Carbon('1st Jan, 2023'))) {
        return true;
    }

    if ($team->created_at->isAfter(new Carbon('1st Jan, 2019'))) {
        return Lottery::odds(1 / 100);
    }

    return Lottery::odds(1 / 1000);
});
```

你会注意到，我们定义的闭包并不期望一个 `User`，而是期望一个 `Team` 模型。为了确定这个特性对用户的团队是否激活，你应该将团队传递给 `Feature` 门面提供的 `for` 方法：

```php
if (Feature::for($user->team)->active('billing-v2')) {
    return redirect()->to('/billing/v2');
}

// ...
```

### 默认作用域

你也可以自定义 Pennant 用于检查特性的默认作用域。例如，也许你所有的特性都是针对当前经过身份验证的用户的团队而不是用户进行检查的。你可以将团队指定为默认作用域，而不是每次检查特性时都调用 `Feature::for($user->team)`。通常，这应该在你的应用程序的某个服务提供者中完成：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Auth;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * 引导任何应用服务。
     */
    public function boot(): void
    {
        Feature::resolveScopeUsing(fn ($driver) => Auth::user()?->team);

        // ...
    }
}
```

如果没有通过 `for` 方法显式提供作用域，特性检查现在将使用当前经过身份验证的用户的团队作为默认作用域：

```php
Feature::active('billing-v2');

// 现在等同于...

Feature::for($user->team)->active('billing-v2');
```

### 可空作用域

如果在检查特性时提供的作用域是 `null`，并且特性的定义不支持通过可空类型或在联合类型中包含 `null` 来支持 `null`，Pennant 将自动将特性的结果值返回为 `false`。

如果你将一个可能为 `null` 的作用域传递给一个特性，并且希望调用特性的值解析器，你应该在特性的定义中考虑到这一点。如果在 Artisan 命令、队列作业或未经身份验证的路由中检查特性，则可能会出现 `null` 作用域。由于在这些情境中通常没有经过身份验证的用户，因此默认作用域将为 `null`。

如果你并不总是[显式指定你的特性作用域](#specifying-the-scope)，那么你应该确保作用域的类型是 “可空的”，并在特性定义逻辑中处理 `null` 作用域值：

```php
use App\Models\User;
use Illuminate\Support\Lottery;
use Laravel\Pennant\Feature;

Feature::define('new-api', fn (User $user) => match (true) {// [tl! remove]
Feature::define('new-api', fn (User|null $user) => match (true) {// [tl! add]
    $user === null => true,// [tl! add]
    $user->isInternalTeamMember() => true,
    $user->isHighTrafficCustomer() => false,
    default => Lottery::odds(1 / 100),
});
```

### 识别作用域

Pennant 内置的 `array` 和 `database` 存储驱动程序知道如何正确存储所有 PHP 数据类型以及 Eloquent 模型的作用域标识符。然而，如果你的应用程序使用第三方 Pennant 驱动程序，该驱动程序可能不知道如何正确存储 Eloquent 模型或应用程序中的其他自定义类型的标识符。

基于此，Pennant 允许你通过在应用程序中用作 Pennant 作用域的对象上实现 `FeatureScopeable` 合同来为存储格式化作用域值。

例如，假设你在单个应用程序中使用两种不同的特性驱动程序：内置的 `database` 驱动程序和第三方的 “Flag Rocket” 驱动程序。"Flag Rocket" 驱动程序不知道如何正确存储 Eloquent 模型。相反，它需要一个 `FlagRocketUser` 实例。通过实现 `FeatureScopeable` 合同中定义的 `toFeatureIdentifier` 方法，我们可以自定义提供给应用程序使用的每个驱动程序的可存储作用域值：

```php
<?php

namespace App\Models;

use FlagRocket\FlagRocketUser;
use Illuminate\Database\Eloquent\Model;
use Laravel\Pennant\Contracts\FeatureScopeable;

class User extends Model implements FeatureScopeable
{
    /**
     * 将对象转换为给定驱动程序的特性作用域标识符。
     */
    public function toFeatureIdentifier(string $driver): mixed
    {
        return match($driver) {
            'database' => $this,
            'flag-rocket' => FlagRocketUser::fromId($this->flag_rocket_id),
        };
    }
}
```

### 序列化作用域

默认情况下，Pennant 在存储与 Eloquent 模型关联的特性时将使用完全限定的类名。如果你已经在使用 [Eloquent morph map](https://learnku.com/docs/laravel/11.x/eloquent-relationships#custom-polymorphic-types)，你可以选择让 Pennant 也使用 morph map 来将存储的特性与应用程序结构解耦。

为了实现这一点，在服务提供程序中定义了你的 Eloquent morph map 后，你可以调用 `Feature` 门面的 `useMorphMap` 方法：

```php
use Illuminate\Database\Eloquent\Relations\Relation;
use Laravel\Pennant\Feature;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);

Feature::useMorphMap();
```

## 丰富的特性值

到目前为止，我们主要展示特性是二进制状态，意味着它们要么是 “激活” 的，要么是 “未激活” 的，但 Pennant 也允许你存储丰富的值。

例如，假设你正在测试应用程序 “立即购买” 按钮的三种新颜色。在特性定义中，你可以返回一个字符串，而不是返回 `true` 或 `false`：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn (User $user) => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

你可以使用 `value` 方法检索 `purchase-button` 特性的值：

```php
$color = Feature::value('purchase-button');
```

Pennant 包含的 Blade 指令还可以根据特性的当前值轻松地有条件地呈现内容：

```blade
@feature('purchase-button', 'blue-sapphire')
    <!-- 'blue-sapphire' 激活 -->
@elsefeature('purchase-button', 'seafoam-green')
    <!-- 'seafoam-green' 激活 -->
@elsefeature('purchase-button', 'tart-orange')
    <!-- 'tart-orange' 激活 -->
@endfeature
```

> **注意**  
> 使用丰富的值时，重要的是要知道当特性具有除 `false` 之外的任何值时，它被视为「激活」。

在调用 [条件 `when`](#conditional-execution) 方法时，特性的丰富值将被提供给第一个闭包：

```php
Feature::when('purchase-button',
    fn ($color) => /* ... */,
    fn () => /* ... */,
);
```

同样，在调用条件 `unless` 方法时，特性的丰富值将被提供给可选的第二个闭包：

```php
Feature::unless('purchase-button',
    fn () => /* ... */,
    fn ($color) => /* ... */,
);
```

## 检索多个特性

`values` 方法允许检索给定作用域的多个特性：

```php
Feature::values(['billing-v2', 'purchase-button']);

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
// ]
```

或者，你可以使用 `all` 方法来检索给定作用域中所有已定义特性的值：

```php
Feature::all();

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

然而，基于类的特性是动态注册的，直到被显式检查之前 Pennant 是不知道它们的。这意味着如果在当前请求期间尚未检查过应用程序的基于类的特性，那么这些特性可能不会出现在 `all` 方法返回的结果中。

如果希望确保在使用 `all` 方法时始终包含特性类，可以使用 Pennant 的特性发现功能。要开始，请在你的应用程序的一个服务提供程序中调用 `discover` 方法：

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Feature::discover();

        // ...
    }
}
```

`discover` 方法将注册应用程序 `app/Features` 目录中的所有特性类。`all` 方法现在将包含这些类在其结果中，无论它们是否在当前请求期间已被检查：

```php
Feature::all();

// [
//     'App\Features\NewApi' => true,
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

## 预加载

尽管 Pennant 为单个请求保留了所有已解析特性的内存缓存，但仍有可能遇到性能问题。为了缓解这一点，Pennant 提供了预加载特性值的能力。

为了说明这一点，假设我们正在循环中检查特性是否处于活动状态：

```php
use Laravel\Pennant\Feature;

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

假设我们正在使用数据库驱动程序，这段代码将为循环中的每个用户执行一次数据库查询 - 可能执行数百次查询。然而，使用 Pennant 的 `load` 方法，我们可以通过为用户集合或作用域预加载特性值来消除这种潜在的性能瓶颈：

```php
Feature::for($users)->load(['notifications-beta']);

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

要仅在特性值尚未加载时加载特性值，可以使用 `loadMissing` 方法：

```php
Feature::for($users)->loadMissing([
    'new-api',
    'purchase-button',
    'notifications-beta',
]);
```

## 更新值

当特性的值首次解析时，底层驱动程序将结果存储在存储中。这通常是为了确保用户在请求之间获得一致的体验。然而，有时候，您可能希望手动更新特性的存储值。

为此，您可以使用 `activate` 和 `deactivate` 方法来切换特性的 “开” 或 “关” 状态：

```php
use Laravel\Pennant\Feature;

// 激活默认作用域的特性...
Feature::activate('new-api');

// 停用给定作用域的特性...
Feature::for($user->team)->deactivate('billing-v2');
```

也可以通过向 `activate` 方法提供第二个参数来手动为特性设置丰富值：

```php
Feature::activate('purchase-button', 'seafoam-green');
```

要指示 Pennant 忘记特性的存储值，可以使用 `forget` 方法。当再次检查特性时，Pennant 将从其特性定义中解析特性的值：

```php
Feature::forget('purchase-button');
```

### 批量更新

要批量更新存储的特性值，可以使用 `activateForEveryone` 和 `deactivateForEveryone` 方法。

例如，假设你现在对 `new-api` 特性的稳定性有信心，并已确定了结账流程中最佳的 `'purchase-button'` 颜色 - 你可以相应地更新所有用户的存储值：

```php
use Laravel\Pennant\Feature;

Feature::activateForEveryone('new-api');

Feature::activateForEveryone('purchase-button', 'seafoam-green');
```

或者，你可以为所有用户停用特性：

```php
Feature::deactivateForEveryone('new-api');
```

> **注意**  
> 这将仅更新已由 Pennant 存储驱动程序存储的已解析特性值。您还需要更新应用程序中的特性定义。

### 清除特性

有时，清除存储中的整个特性会很有用。这通常是必要的，如果您已从应用程序中移除了特性，或者您已对特性的定义进行了调整，并希望将其推广给所有用户。

您可以使用 `purge` 方法删除特性的所有存储值：

```php
// 清除单个特性...
Feature::purge('new-api');

// 清除多个特性...
Feature::purge(['new-api', 'purchase-button']);
```

如果你想要从存储中清除 *所有* 特性，可以调用 `purge` 方法而不带任何参数：

作为应用程序部署流水线的一部分清除特性可能很有用，Pennant 包括一个 `pennant:purge` Artisan 命令，该命令将清除存储中提供的特性：

```sh
php artisan pennant:purge new-api

php artisan pennant:purge new-api purchase-button
```

还可以清除除了给定特性列表中的特性之外的所有特性。例如，假设您想清除所有特性但保留存储中的「new-api」和「purchase-button」特性的值。为实现此目的，您可以将这些特性名称传递给 `--except` 选项：

```sh
php artisan pennant:purge --except=new-api --except=purchase-button
```

为方便起见，`pennant:purge` 命令还支持一个 `--except-registered` 标志。此标志表示应清除除了在服务提供程序中明确注册的特性之外的所有特性：

```sh
php artisan pennant:purge --except-registered
```

## 测试

在测试与特性标志交互的代码时，控制测试中特性标志的返回值最简单的方法是简单地重新定义特性。例如，假设您在应用程序的服务提供程序中定义了以下特性：

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn () => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

要在测试中修改特性的返回值，可以在测试开始时重新定义特性。以下测试将始终通过，即使 `Arr::random()` 的实现仍然存在于服务提供程序中：

```php
use Laravel\Pennant\Feature;

test('it can control feature values', function () {
    Feature::define('purchase-button', 'seafoam-green');

    expect(Feature::value('purchase-button'))->toBe('seafoam-green');
});
```

```php
use Laravel\Pennant\Feature;

public function test_it_can_control_feature_values()
{
    Feature::define('purchase-button', 'seafoam-green');

    $this->assertSame('seafoam-green', Feature::value('purchase-button'));
}
```

对于基于类的特性，可以使用相同的方法：

```php
use Laravel\Pennant\Feature;

test('it can control feature values', function () {
    Feature::define(NewApi::class, true);

    expect(Feature::value(NewApi::class))->toBeTrue();
});
```

```php
use App\Features\NewApi;
use Laravel\Pennant\Feature;

public function test_it_can_control_feature_values()
{
    Feature::define(NewApi::class, true);

    $this->assertTrue(Feature::value(NewApi::class));
}
```

如果你的特性返回一个 `Lottery` 实例，有一些有用的 [测试辅助函数可用](https://learnku.com/docs/laravel/11.x/helpers#testing-lotteries)。

#### 存储配置

你可以通过在应用程序的 `phpunit.xml` 文件中定义 `PENNANT_STORE` 环境变量来配置 Pennant 在测试期间将使用的存储：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit colors="true">
    <!-- ... -->
    <php>
        <env name="PENNANT_STORE" value="array"/>
        <!-- ... -->
    </php>
</phpunit>
```

## 添加自定义 Pennant 驱动程序

#### 实现驱动程序

如果 Pennant 的现有存储驱动程序都不符合你的应用程序需求，你可以编写自己的存储驱动程序。你的自定义驱动程序应该实现 `Laravel\Pennant\Contracts\Driver` 接口：

```php
<?php

namespace App\Extensions;

use Laravel\Pennant\Contracts\Driver;

class RedisFeatureDriver implements Driver
{
    public function define(string $feature, callable $resolver): void {}
    public function defined(): array {}
    public function getAll(array $features): array {}
    public function get(string $feature, mixed $scope): mixed {}
    public function set(string $feature, mixed $scope, mixed $value): void {}
    public function setForAllScopes(string $feature, mixed $value): void {}
    public function delete(string $feature, mixed $scope): void {}
    public function purge(array|null $features): void {}
}
```

现在，我们只需要使用 Redis 连接来实现每个方法。要查看如何实现每个方法的示例，请查看 [Pennant 源代码](https://github.com/laravel/pennant/blob/1.x/src/Drivers/DatabaseDriver.php) 中的 `Laravel\Pennant\Drivers\DatabaseDriver`。

> \[!NOTE\]  
> Laravel 不提供一个目录来包含你的扩展。你可以自由地将它们放在任何你喜欢的地方。在这个示例中，我们创建了一个 `Extensions` 目录来存放 `RedisFeatureDriver`。

#### 注册驱动程序

一旦你实现了驱动程序，就可以准备将其注册到 Laravel 中。要向 Pennant 添加其他驱动程序，可以使用 `Feature` 门面提供的 `extend` 方法。应该从应用程序的一个 [服务提供程序](https://learnku.com/docs/laravel/11.x/providers) 的 `boot` 方法中调用 `extend` 方法：

```php
<?php

namespace App\Providers;

use App\Extensions\RedisFeatureDriver;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * 注册任何应用程序服务。
     */
    public function register(): void
    {
        // ...
    }

    /**
     * 启动任何应用程序服务。
     */
    public function boot(): void
    {
        Feature::extend('redis', function (Application $app) {
            return new RedisFeatureDriver($app->make('redis'), $app->make('events'), []);
        });
    }
}
```

一旦驱动程序已注册，你可以在应用程序的 `config/pennant.php` 配置文件中使用 `redis` 驱动程序：

```php
'stores' => [

    'redis' => [
        'driver' => 'redis',
        'connection' => null,
    ],

    // ...

],
```

## 事件

Pennant 分发各种事件，可以在整个应用程序中跟踪特性标志时非常有用。

### `Laravel\Pennant\Events\FeatureRetrieved`

此事件在每次[检查特性](#checking-features)时触发。该事件可能对在整个应用程序中创建和跟踪度量标准对特性标志的使用情况很有用。

### `Laravel\Pennant\Events\FeatureResolved`

第一次为特定范围解析特性值时触发此事件。

### `Laravel\Pennant\Events\UnknownFeatureResolved`

第一次为特定范围解析未知特性时触发此事件。监听此事件可能很有用，如果您打算删除一个特性标志，但意外地在整个应用程序中留下了杂乱的引用：

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Laravel\Pennant\Events\UnknownFeatureResolved;

class AppServiceProvider extends ServiceProvider
{
    /**
     * 引导任何应用服务。
     */
    public function boot(): void
    {
        Event::listen(function (UnknownFeatureResolved $event) {
            Log::error("Resolving unknown feature [{$event->feature}].");
        });
    }
}
```

### `Laravel\Pennant\Events\DynamicallyRegisteringFeatureClass`

在请求期间首次动态检查[基于类的特性](#class-based-features)时触发此事件。

### `Laravel\Pennant\Events\UnexpectedNullScopeEncountered`

当传递给不支持 `null` 的特性定义的特性时，将触发此事件。这种情况会被优雅地处理，特性将返回 `false`。但是，如果您想要退出此特性的默认优雅行为，可以在应用程序的 `AppServiceProvider` 的 `boot` 方法中注册此事件的监听器：

```php
use Illuminate\Support\Facades\Log;
use Laravel\Pennant\Events\UnexpectedNullScopeEncountered;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Event::listen(UnexpectedNullScopeEncountered::class, fn () => abort(500));
}
```

### `Laravel\Pennant\Events\FeatureUpdated`

通过调用 `activate` 或 `deactivate` 更新范围内的特性时触发此事件。

### `Laravel\Pennant\Events\FeatureUpdatedForAllScopes`

通过调用 `activateForEveryone` 或 `deactivateForEveryone` 更新所有范围内的特性时触发此事件。

### `Laravel\Pennant\Events\FeatureDeleted`

通过调用 `forget` 删除范围内的特性时触发此事件。

### `Laravel\Pennant\Events\FeaturesPurged`

清除特定特性时触发此事件。

### `Laravel\Pennant\Events\AllFeaturesPurged`

清除所有特性时触发此事件。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/pe...](https://learnku.com/docs/laravel/11.x/pennantmd/16725)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/pe...](https://learnku.com/docs/laravel/11.x/pennantmd/16725)