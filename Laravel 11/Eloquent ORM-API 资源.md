## Eloquent: API 资源

+   [简介](#introduction)
+   [生成资源](#generating-resources)
+   [概念综述](#concept-overview)
    +   [资源集合](#resource-collections)
+   [编写资源](#writing-resources)
    +   [数据包裹](#data-wrapping)
    +   [分页](#pagination)
    +   [条件属性](#conditional-attributes)
    +   [条件关系](#conditional-relationships)
    +   [添加元数据](#adding-meta-data)
+   [资源响应](#resource-responses)

## 简介

在构建 API 时，你可能需要一个位于 Eloquent 模型和实际返回给应用程序用户的 JSON 响应之间的转换层。例如，你可能希望为一部分用户显示某些属性，而对于其他用户则不显示，或者你可能希望始终在模型的 JSON 表示中包含某些关联关系。Eloquent 的资源类允许你以直观简便的方式将模型和模型集合转换为 JSON。

当然，你始终可以使用它们的 `toJson` 方法将 Eloquent 模型或集合转换为 JSON；然而，Eloquent 资源提供了对模型及其关系的 JSON 序列化更细粒度和更强大的控制。

## 生成资源

要生成一个资源类，你可以使用 `make:resource` Artisan 命令。默认情况下，资源将放置在应用程序的 `app/Http/Resources` 目录中。资源继承了 `Illuminate\Http\Resources\Json\JsonResource` 类：

```shell
php artisan make:resource UserResource
```

#### 资源集合

除了生成转换单个模型的资源外，你还可以生成负责转换模型集合的资源。这使得你的 JSON 响应能够包含与给定资源整个集合相关的链接和其他元信息。

要创建一个资源集合，你应该在创建资源时使用 `--collection` 标志。或者，在资源名称中包含单词 `Collection` 以指示 Laravel 应该创建一个集合资源。集合资源继承了 `Illuminate\Http\Resources\Json\ResourceCollection` 类：

```shell
php artisan make:resource User --collection

php artisan make:resource UserCollection
```

## 概念综述

> 注意
> 这是对资源和资源集合的高层次概述。强烈建议你阅读本文档的其他部分，以更深入地了解资源所提供的定制化和强大功能。

在深入了解编写资源时所有可用的选项之前，让我们先从高层次看一下资源在 Laravel 中的使用方式。一个资源类表示一个需要转换为 JSON 结构的单个模型。例如，这里是一个简单的 `UserResource` 资源类：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * 将资源转换为数组。
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}
```

每个资源类都定义了一个 `toArray` 方法，该方法返回一个属性数组，这些属性应在资源作为路由或控制器方法的响应返回时转换为 JSON。

请注意，我们可以直接从 `$this` 变量访问模型属性。这是因为资源类会自动将属性和方法访问代理到底层模型，以便于访问。一旦定义了资源，就可以从路由或控制器返回它。资源通过其构造函数接受底层模型实例：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return new UserResource(User::findOrFail($id));
});
```

### 资源集合

如果你返回的是资源集合或分页响应，你应该在路由或控制器中创建资源实例时使用资源类提供的 `collection` 方法：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all());
});
```

请注意，这种方式不允许在你的集合中添加任何可能需要返回的自定义元数据。如果你想要自定义资源集合响应，你可以创建一个专用的资源来表示集合：

```shell
php artisan make:resource UserCollection
```

资源集合类一旦生成，你就可以轻松定义应包含在响应中的任何元数据：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    /**
     * 将资源集合转换为数组。
     *
     * @return array<int|string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'links' => [
                'self' => 'link-value',
            ],
        ];
    }
}
```

资源集合定义好后，可以从路由或控制器返回它：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

#### 保留集合键

当从路由返回资源集合时，Laravel 会重置集合的键，使它们按数字顺序排列。然而，你可以在资源类中添加一个 `preserveKeys` 属性，指示是否应保留集合的原始键：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * 指示是否应保留资源的集合键。
     *
     * @var bool
     */
    public $preserveKeys = true;
}
```

当 `preserveKeys` 属性设置为 `true` 时，集合键将在从路由或控制器返回集合时被保留：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all()->keyBy->id);
});
```

#### 自定义底层资源类

通常，资源集合的 `$this->collection` 属性会自动填充为集合中每个项映射到其单个资源类的结果。单个资源类假定为集合类名去掉末尾的 `Collection` 部分。此外，根据你的个人偏好，单个资源类可能有也可能没有 `Resource` 后缀。

例如，`UserCollection` 会尝试将给定的用户实例映射到 `UserResource` 资源。要自定义此行为，你可以覆盖资源集合的 `$collects` 属性：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    /**
     * 此资源集合收集的资源。
     *
     * @var string
     */
    public $collects = Member::class;
}
```

## 编写资源

> \[! 注意\]  
> 如果你还没有阅读[概念综述](#concept-overview)，强烈建议你在继续阅读本文档之前先阅读该部分。

资源只需要将给定模型转换为数组。因此，每个资源都包含一个 `toArray` 方法，该方法将模型的属性转换为 API 友好的数组，可以从应用程序的路由或控制器返回：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * 将资源转换为数组。
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}
```

资源一定义就可以直接从路由或控制器返回它：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return new UserResource(User::findOrFail($id));
});
```

#### 关联关系

如果你想要在响应中包含关联资源，可以将它们添加到资源 `toArray` 方法返回的数组中。在这个例子中，我们将使用 `PostResource` 资源的 `collection` 方法将用户的博客文章添加到资源响应中：

```php
use App\Http\Resources\PostResource;
use Illuminate\Http\Request;

/**
 * 将资源转换为数组。
 *
 * @return array<string, mixed>
 */
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'posts' => PostResource::collection($this->posts),
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

> \[! 注意\]  
> 如果你只想要包含已经加载的关联关系，请查看关于[条件关系](#conditional-relationships)的文档。

#### 资源集合

资源将单个模型转换为数组，而资源集合将模型集合转换为数组。但是，没必要为每个模型定义一个资源集合类，因为所有资源都提供了一个 `collection` 方法，用于随时生成「临时」资源集合：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/users', function () {
    return UserResource::collection(User::all());
});
```

不过，如果你需要自定义与集合一起返回的元数据，则需要定义自己的资源集合：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    /**
     * 将资源集合转换为数组。
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'links' => [
                'self' => 'link-value',
            ],
        ];
    }
}
```

和单个资源一样，资源集合也可以直接从路由或控制器返回:

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::all());
});
```

### 数据包裹

默认情况下，当资源响应转换为 JSON 时，最外层的资源会被包裹在一个 `data` 键中。因此，例如，典型的资源集合响应看起来像以下内容：

```json
{
    "data": [
        {
            "id": 1,
            "name": "Eladio Schroeder Sr.",
            "email": "therese28@example.com"
        },
        {
            "id": 2,
            "name": "Liliana Mayert",
            "email": "evandervort@example.com"
        }
    ]
}
```

如果你想禁用最外层资源的包裹，你应该在基础 `Illuminate\Http\Resources\Json\JsonResource` 类上调用 `withoutWrapping` 方法。通常，你应该在 `AppServiceProvider` 或其他在应用程序的每个请求中加载的[服务提供者](https://learnku.com/docs/laravel/11.x/providers)中调用此方法：

```php
<?php

namespace App\Providers;

use Illuminate\Http\Resources\Json\JsonResource;
use Illuminate\Support\ServiceProvider;

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
        JsonResource::withoutWrapping();
    }
}
```

> \[! 警告\]  
> `withoutWrapping` 方法仅影响最外层的响应，不会移除你手动添加到自己的资源集合中的 `data` 键。

#### 包裹嵌套资源

你可以完全自由地决定如何包裹资源的关联关系。如果你想要所有资源集合都包裹在一个 `data` 键中，无论它们的嵌套如何，你应该为每个资源定义一个资源集合类，并在 `data` 键中返回集合。

你可能会想知道这是否会导致最外层的资源被包裹在两个 `data` 键中。不用担心，Laravel 永远不会让你的资源被意外地双重包裹，因此你不必担心正在转换的资源集合的嵌套级别：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class CommentsCollection extends ResourceCollection
{
    /**
     * 将资源集合转换为数组。
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return ['data' => $this->collection];
    }
}
```

#### 数据包裹和分页

通过资源响应返回分页集合时，即使调用了 `withoutWrapping` 方法，Laravel 也会将你的资源数据包裹在一个 `data` 键中。这是因为分页响应总是包含 `meta` 和 `links` 键，其中包含有关分页器状态的信息：

```json
{
    "data": [
        {
            "id": 1,
            "name": "Eladio Schroeder Sr.",
            "email": "therese28@example.com"
        },
        {
            "id": 2,
            "name": "Liliana Mayert",
            "email": "evandervort@example.com"
        }
    ],
    "links":{
        "first": "http://example.com/users?page=1",
        "last": "http://example.com/users?page=1",
        "prev": null,
        "next": null
    },
    "meta":{
        "current_page": 1,
        "from": 1,
        "last_page": 1,
        "path": "http://example.com/users",
        "per_page": 15,
        "to": 10,
        "total": 10
    }
}
```

### 分页

你可以将 Laravel 分页器实例传递给资源的 `collection` 方法或自定义的资源集合：

```php
use App\Http\Resources\UserCollection;
use App\Models\User;

Route::get('/users', function () {
    return new UserCollection(User::paginate());
});
```

分页响应总是包含 `meta` 和 `links` 键，其中包含有关分页器状态的信息：

```json
{
    "data": [
        {
            "id": 1,
            "name": "Eladio Schroeder Sr.",
            "email": "therese28@example.com"
        },
        {
            "id": 2,
            "name": "Liliana Mayert",
            "email": "evandervort@example.com"
        }
    ],
    "links":{
        "first": "http://example.com/users?page=1",
        "last": "http://example.com/users?page=1",
        "prev": null,
        "next": null
    },
    "meta":{
        "current_page": 1,
        "from": 1,
        "last_page": 1,
        "path": "http://example.com/users",
        "per_page": 15,
        "to": 10,
        "total": 10
    }
}
```

#### 自定义分页信息

如果你想要自定义包含在分页响应中的 `links` 或 `meta` 键中的信息，可以在资源上定义一个 `paginationInformation` 方法。此方法将接收 `$paginated` 数据和 `$default` 信息数组，后者是一个包含 `links` 和 `meta` 键的数组：

```php
/**
 * 自定义资源的分页信息。
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  array $paginated
 * @param  array $default
 * @return array
 */
public function paginationInformation($request, $paginated, $default)
{
    $default['links']['custom'] = 'https://example.com';

    return $default;
}
```

### 条件属性

有时你可能想仅在满足给定条件时才在资源响应中包含一个属性。例如，你可能希望仅在当前用户是「管理员」时才包含一个值。Laravel 提供了各种辅助方法来帮助你应对这种情况。`when` 方法可用于有条件地向资源响应添加一个属性：

```php
/**
 * 将资源转换为数组。
 *
 * @return array<string, mixed>
 */
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'secret' => $this->when($request->user()->isAdmin(), 'secret-value'),
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

在这个例子中，`secret` 键仅在经过身份验证的用户的 `isAdmin` 方法返回 `true` 时才会返回在最终资源响应中。如果该方法返回 `false`，`secret` 键将在发送给客户端之前从资源响应中移除。`when` 方法允许你以表达性的方式定义资源，而不必在构建数组时借助条件语句。

`when` 方法还接受一个闭包作为其第二个参数，允许你仅在给定条件为 `true` 时计算结果值：

```php
'secret' => $this->when($request->user()->isAdmin(), function () {
    return 'secret-value';
}),
```

`whenHas` 方法可用于在底层模型中实际存在该属性时包含：

```php
'name' => $this->whenHas('name'),
```

此外，`whenNotNull` 方法可用于在属性不为 null 时包含该属性在资源响应中：

```php
'name' => $this->whenNotNull($this->name),
```

#### 合并条件属性

有时你可能有多个属性应仅在满足相同条件时包含在资源响应中。在这种情况下，你可以使用 `mergeWhen` 方法仅在给定条件为 `true` 时包含这些属性在响应中：

```php
/**
 * 将资源转换为数组。
 *
 * @return array<string, mixed>
 */
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        $this->mergeWhen($request->user()->isAdmin(), [
            'first-secret' => 'value',
            'second-secret' => 'value',
        ]),
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

同样，如果给定条件为 `false`，这些属性将在发送给客户端之前从资源响应中移除。

> \[! 警告\]  
> `mergeWhen` 方法不应在混合有字符串和数字键的数组中使用。此外，它不应在数字键未按顺序排列的数组中使用。

### 条件关联关系

除了按条件地加载属性外，还可以根据模型上是否已加载关联关系来有条件地包含资源响应中的关联关系。这允许你的控制器决定应在模型上加载哪些关联关系，而你的资源可以仅在它们实际被加载时轻松包含它们。最终，这使得在你的资源中更容易避免「N+1」查询问题。

`whenLoaded` 方法可用于有条件地加载关联关系。为了避免不必要地加载关联关系，此方法接受关联关系的名称而不是关联关系本身：

```php
use App\Http\Resources\PostResource;

/**
 * 将资源转换为数组。
 *
 * @return array<string, mixed>
 */
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'posts' => PostResource::collection($this->whenLoaded('posts')),
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

在这个例子中，如果关联关系未被加载，`posts` 键将在发送给客户端之前从资源响应中移除。

#### 条件关联关系统计

除了有条件地包含关联关系外，还可以根据模型上是否加载了关联关系的统计来有条件地包含资源响应中的关联关系「数量」：

```php
new UserResource($user->loadCount('posts'));
```

`whenCounted` 方法可用于有条件地包含关联关系数量在资源响应中。如果关系的统计不存在，此方法避免了不必要地包含该属性：

```php
/**
 * 将资源转换为数组。
 *
 * @return array<string, mixed>
 */
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'posts_count' => $this->whenCounted('posts'),
        'created_at' => $this->created_at,
        'updated_at' => $this->updated_at,
    ];
}
```

在这个例子中，如果 `posts` 关系的统计未被加载，`posts_count` 键将在发送给客户端之前从资源响应中移除。

其他类型的聚合统计，如 `avg`、`sum`、`min` 和 `max`，也可以使用 `whenAggregated` 方法有条件地加载：

```php
'words_avg' => $this->whenAggregated('posts', 'words', 'avg'),
'words_sum' => $this->whenAggregated('posts', 'words', 'sum'),
'words_min' => $this->whenAggregated('posts', 'words', 'min'),
'words_max' => $this->whenAggregated('posts', 'words', 'max'),
```

#### 条件中间表信息

除了有条件地包含关联关系信息在你的资源响应中外，还可以使用 `whenPivotLoaded` 方法有条件地包含多对多关系中间表中的数据。`whenPivotLoaded` 方法接受中间表的名称作为其第一个参数，第二个参数应该是一个闭包，如果中间表信息在模型上可用，则返回要返回的值：

```php
/**
 * 将资源转换为数组。
 *
 * @return array<string, mixed>
 */
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'expires_at' => $this->whenPivotLoaded('role_user', function () {
            return $this->pivot->expires_at;
        }),
    ];
}
```

如果你的关联关系使用的是[自定义中间表模型](https://learnku.com/docs/laravel/11.x/eloquent-relationships#defining-custom-intermediate-table-models)，可以将中间表模型的实例作为第一个参数传递给 `whenPivotLoaded` 方法：

```php
'expires_at' => $this->whenPivotLoaded(new Membership, function () {
    return $this->pivot->expires_at;
}),
```

如果你的中间表使用不同于 `pivot` 的访问器，可以使用 `whenPivotLoadedAs` 方法：

```php
/**
 * 将资源转换为数组。
 *
 * @return array<string, mixed>
 */
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'expires_at' => $this->whenPivotLoadedAs('subscription', 'role_user', function () {
            return $this->subscription->expires_at;
        }),
    ];
}
```

### 添加元数据

一些 JSON API 标准要求在你的资源和资源集合响应中添加元数据。这通常包括指向资源或相关资源的 `links`，或关于资源本身的元数据。如果你需要返回关于资源的额外元数据，请在 `toArray` 方法中包含它。例如，你可以在转换资源集合时包含 `link` 信息：

```php
/**
 * 将资源转换为数组。
 *
 * @return array<string, mixed>
 */
public function toArray(Request $request): array
{
    return [
        'data' => $this->collection,
        'links' => [
            'self' => 'link-value',
        ],
    ];
}
```

当从你的资源返回额外元数据时，不必担心会意外覆盖 Laravel 在返回分页响应时自动添加的 `links` 或 `meta` 键。你定义的任何额外 `links` 将与分页器提供的链接合并。

#### 顶层元数据

有时你可能想仅在资源是最外层返回的资源时包含某些元数据。通常，这包括关于响应整体的元信息，要定义这些元数据，请在资源类中添加一个 `with` 方法。此方法应返回一个元数据数组，仅在资源是最外层转换的资源时包含在资源响应中：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    /**
     * 将资源集合转换为数组。
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return parent::toArray($request);
    }

    /**
     * 获取应该与资源数组一起返回的其他数据。
     *
     * @return array<string, mixed>
     */
    public function with(Request $request): array
    {
        return [
            'meta' => [
                'key' => 'value',
            ],
        ];
    }
}
```

#### 在构建资源时添加元数据

你还可以在路由或控制器中构造资源实例时添加顶级数据。`additional` 方法，在所有资源上都可用，接受一个应添加到资源响应的数据数组：

```php
return (new UserCollection(User::all()->load('roles')))
                ->additional(['meta' => [
                    'key' => 'value',
                ]]);
```

## 资源响应

如你所读，资源可以直接从路由和控制器返回：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user/{id}', function (string $id) {
    return new UserResource(User::findOrFail($id));
});
```

然而，有时你可能需要在发送给客户端之前自定义出站 HTTP 响应。有两种方法可以实现这一点。首先，你可以在资源上链式调用 `response` 方法。此方法将返回一个 `Illuminate\Http\JsonResponse` 实例，让你完全控制响应的头：

```php
use App\Http\Resources\UserResource;
use App\Models\User;

Route::get('/user', function () {
    return (new UserResource(User::find(1)))
                ->response()
                ->header('X-Value', 'True');
});
```

或者，你可以在资源本身中定义一个 `withResponse` 方法。当资源作为响应中的最外层资源返回时，将调用此方法：

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * 将资源转换为数组。
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
        ];
    }

    /**
     * 自定义资源的出站响应。
     */
    public function withResponse(Request $request, JsonResponse $response): void
    {
        $response->header('X-Value', 'True');
    }
}
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-resourcesmd/16706)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-resourcesmd/16706)