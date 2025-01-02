本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Scout

+   [介绍](#introduction)
+   [安装](#installation)
    +   [队列化](#queueing)
+   [驱动程序先决条件](#driver-prerequisites)
    +   [Algolia](#algolia)
    +   [Meilisearch](#meilisearch)
    +   [Typesense](#typesense)
+   [配置](#configuration)
    +   [配置模型索引](#configuring-model-indexes)
    +   [配置可搜索数据](#configuring-searchable-data)
    +   [配置模型 ID](#configuring-the-model-id)
    +   [配置每个模型的搜索引擎](#configuring-search-engines-per-model)
    +   [识别用户](#identifying-users)
+   [数据库 / 集合引擎](#database-and-collection-engines)
    +   [数据库引擎](#database-engine)
    +   [集合引擎](#collection-engine)
+   [索引](#indexing)
    +   [批量导入](#batch-import)
    +   [添加记录](#adding-records)
    +   [更新记录](#updating-records)
    +   [移除记录](#removing-records)
    +   [暂停索引](#pausing-indexing)
    +   [有条件地搜索模型实例](#conditionally-searchable-model-instances)
+   [搜索](#searching)
    +   [Where 子句](#where-clauses)
    +   [分页](#pagination)
    +   [软删除](#soft-deleting)
    +   [自定义引擎搜索](#customizing-engine-searches)
+   [自定义引擎](#custom-engines)

## 介绍

[Laravel Scout](https://github.com/laravel/scout) 提供了一个简单的、基于驱动程序的解决方案，用于为你的 [Eloquent 模型](https://learnku.com/docs/laravel/11.x/eloquent)添加全文搜索功能。使用模型观察者，Scout 将自动保持你的搜索索引与你的 Eloquent 记录同步。

目前，Scout 配备了 [Algolia](https://www.algolia.com/)、[Meilisearch](https://www.meilisearch.com/)、[Typesense](https://typesense.org/) 和 MySQL/PostgreSQL（`database`）驱动程序。此外，Scout 还包括一个 “集合” 驱动程序，专为本地开发使用而设计，不需要任何外部依赖或第三方服务。此外，编写自定义驱动程序很简单，你可以自由地使用你自己的搜索实现扩展 Scout。

## 安装

首先，通过 Composer 包管理器安装 Scout：

```shell
composer require laravel/scout
```

安装完 Scout 后，你应该使用 `vendor:publish` Artisan 命令发布 Scout 配置文件。此命令将会把 `scout.php` 配置文件发布到你的应用程序的 `config` 目录中：

```shell
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

最后，将 `Laravel\Scout\Searchable` trait 添加到你想要进行搜索的模型中。这个 trait 将注册一个模型观察者，它将自动保持模型与你的搜索驱动程序同步：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class Post extends Model
{
    use Searchable;
}
```

### 队列化

虽然使用 Scout 并不严格要求配置队列驱动，但在使用这个库之前，强烈建议你配置一个[队列驱动程序](https://learnku.com/docs/laravel/11.x/queues)。运行队列工作者将允许 Scout 将所有同步模型信息到搜索索引的操作排入队列，为你的应用程序的 Web 接口提供更好的响应时间。

一旦配置了队列驱动程序，请将 `config/scout.php` 配置文件中的 `queue` 选项的值设置为 `true`：

即使 `queue` 选项设置为 `false`，也要记住，一些 Scout 驱动程序（如 Algolia 和 Meilisearch）总是异步索引记录。这意味着，即使在 Laravel 应用程序中完成了索引操作，搜索引擎本身可能不会立即反映新的和更新的记录。

要指定 Scout 作业使用的连接和队列，你可以将 `queue` 配置选项定义为一个数组：

```php
'queue' => [
    'connection' => 'redis',
    'queue' => 'scout'
],
```

当然，如果你自定义了 Scout 作业使用的连接和队列，你应该运行一个队列工作者来处理该连接和队列上的作业：

```php
php artisan queue:work redis --queue=scout
```

## 驱动程序先决条件

### Algolia

当使用 Algolia 驱动程序时，你应该在 `config/scout.php` 配置文件中配置你的 Algolia `id` 和 `secret` 凭据。一旦配置了你的凭据，你还需要通过 Composer 包管理器安装 Algolia PHP SDK：

```shell
composer require algolia/algoliasearch-client-php
```

### Meilisearch

[Meilisearch](https://www.meilisearch.com/) 是一个极快的开源搜索引擎。如果你不确定如何在本地安装 Meilisearch，你可以使用 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sail#meilisearch)，这是 Laravel 官方支持的 Docker 开发环境。

当使用 Meilisearch 驱动程序时，你需要通过 Composer 包管理器安装 Meilisearch PHP SDK：

```shell
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle
```

然后，在应用程序的 `.env` 文件中设置 `SCOUT_DRIVER` 环境变量以及你的 Meilisearch `host` 和 `key` 凭据：

```ini
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

有关 Meilisearch 的更多信息，请参考 [Meilisearch 文档](https://docs.meilisearch.com/learn/getting_started/quick_start.html)。

此外，你应确保安装一个与你的 Meilisearch 二进制版本兼容的 `meilisearch/meilisearch-php` 版本，通过查看 [Meilisearch 关于二进制兼容性的文档](https://github.com/meilisearch/meilisearch-php#-compatibility-with-meilisearch)。

> **注意**  
> 在升级使用 Meilisearch 的应用程序的 Scout 时，你应始终[查看 Meilisearch 服务本身的任何其他重大变更](https://github.com/meilisearch/Meilisearch/releases)。

### Typesense

[Typesense](https://typesense.org/) 是一个极快的开源搜索引擎，支持关键字搜索、语义搜索、地理位置搜索和向量搜索。

你可以选择 [自托管](https://typesense.org/docs/guide/install-typesense.html#option-2-local-machine-self-hosting) Typesense，或使用 [Typesense Cloud](https://cloud.typesense.org/)。

要开始使用 Typesense 和 Scout，通过 Composer 包管理器安装 Typesense PHP SDK：

```shell
composer require typesense/typesense-php
```

然后，在应用程序的 `.env` 文件中设置 `SCOUT_DRIVER` 环境变量以及你的 Typesense 主机和 API 密钥凭据：

```env
SCOUT_DRIVER=typesense
TYPESENSE_API_KEY=masterKey
TYPESENSE_HOST=localhost
```

如果需要，你还可以指定安装的端口、路径和协议：

```env
TYPESENSE_PORT=8108
TYPESENSE_PATH=
TYPESENSE_PROTOCOL=http
```

你可以在应用程序的 `config/scout.php` 配置文件中找到有关 Typesense 集合的额外设置和模式定义。有关 Typesense 的更多信息，请参考 [Typesense 文档](https://typesense.org/docs/guide/#quick-start)。

#### 准备数据以在 Typesense 中存储

在使用 Typesense 时，你的可搜索模型必须定义一个 `toSearchableArray` 方法，将你的模型的主键转换为字符串，并将创建日期转换为 UNIX 时间戳：

```php
/**
 * 获取模型的可索引数据数组。
 *
 * @return array<string, mixed>
 */
public function toSearchableArray()
{
    return array_merge($this->toArray(),[
        'id' => (string) $this->id,
        'created_at' => $this->created_at->timestamp,
    ]);
}
```

你还应该在应用程序的 `config/scout.php` 文件中定义你的 Typesense 集合模式。集合模式描述了通过 Typesense 可搜索的每个字段的数据类型。有关所有可用模式选项的更多信息，请参考 [Typesense 文档](https://typesense.org/docs/latest/api/collections.html#schema-parameters)。

如果需要在定义后更改 Typesense 集合的模式，你可以运行 `scout:flush` 和 `scout:import`，这将删除所有现有的索引数据并重新创建模式。或者，你可以使用 Typesense 的 API 在不删除任何索引数据的情况下修改集合的模式。

如果你的可搜索模型支持软删除，你应该在应用程序的 `config/scout.php` 配置文件中为模型对应的 Typesense 模式定义一个 `__soft_deleted` 字段：

```php
User::class => [
    'collection-schema' => [
        'fields' => [
            // ...
            [
                'name' => '__soft_deleted',
                'type' => 'int32',
                'optional' => true,
            ],
        ],
    ],
],
```

#### 动态搜索参数

Typesense 允许你在执行搜索操作时通过 `options` 方法动态修改你的[搜索参数](https://typesense.org/docs/latest/api/search.html#search-parameters)：

```php
use App\Models\Todo;

Todo::search('Groceries')->options([
    'query_by' => 'title, description'
])->get();
```

## 配置

### 配置模型索引

每个 Eloquent 模型都与一个特定的搜索 “索引” 同步，其中包含该模型的所有可搜索记录。换句话说，你可以将每个索引视为一个 MySQL 表。默认情况下，每个模型将持久化到与模型典型 “表” 名称匹配的索引中。通常，这是模型名称的复数形式；但是，你可以通过在模型上重写 `searchableAs` 方法来自定义模型的索引：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class Post extends Model
{
    use Searchable;

    /**
     * 获取与模型关联的索引名称。
     */
    public function searchableAs(): string
    {
        return 'posts_index';
    }
}
```

### 配置可搜索数据

默认情况下，给定模型的整个 `toArray` 表单将持久化到其搜索索引中。如果你想要自定义同步到搜索索引的数据，可以在模型上重写 `toSearchableArray` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class Post extends Model
{
    use Searchable;

    /**
     * 获取模型的可索引数据数组。
     *
     * @return array<string, mixed>
     */
    public function toSearchableArray(): array
    {
        $array = $this->toArray();

        // 自定义数据数组...

        return $array;
    }
}
```

一些搜索引擎（如 Meilisearch）仅会在正确类型的数据上执行过滤操作（`>`, `<` 等）。因此，在使用这些搜索引擎并自定义可搜索数据时，应确保将数字值转换为其正确的类型：

```php
public function toSearchableArray()
{
    return [
        'id' => (int) $this->id,
        'name' => $this->name,
        'price' => (float) $this->price,
    ];
}
```

#### 配置可过滤数据和索引设置（Meilisearch）

与 Scout 的其他驱动程序不同，Meilisearch 需要预定义索引搜索设置，例如可过滤属性、可排序属性和[其他支持的设置字段](https://docs.meilisearch.com/reference/api/settings.html)。

可过滤属性是在调用 Scout 的 `where` 方法时计划进行过滤的任何属性，而可排序属性是在调用 Scout 的 `orderBy` 方法时计划进行排序的任何属性。要定义索引设置，请调整应用程序的 `scout` 配置文件中 `meilisearch` 配置条目的 `index-settings` 部分：

```php
use App\Models\User;
use App\Models\Flight;

'meilisearch' => [
    'host' => env('MEILISEARCH_HOST', 'http://localhost:7700'),
    'key' => env('MEILISEARCH_KEY', null),
    'index-settings' => [
        User::class => [
            'filterableAttributes'=> ['id', 'name', 'email'],
            'sortableAttributes' => ['created_at'],
            // 其他设置字段...
        ],
        Flight::class => [
            'filterableAttributes'=> ['id', 'destination'],
            'sortableAttributes' => ['updated_at'],
        ],
    ],
],
```

如果给定索引下的模型支持软删除，并包含在 `index-settings` 数组中，Scout 将自动在该索引上包含对软删除模型的过滤支持。如果对于支持软删除的模型索引没有其他可过滤或可排序属性需要定义，你可以简单地为该模型在 `index-settings` 数组中添加一个空条目：

```php
'index-settings' => [
    Flight::class => []
],
```

在配置应用程序的索引设置后，你必须调用 `scout:sync-index-settings` Artisan 命令。该命令将通知 Meilisearch 您当前配置的索引设置。为方便起见，你可能希望将此命令作为部署过程的一部分：

```shell
php artisan scout:sync-index-settings
```

### 配置模型 ID

默认情况下，Scout 将使用模型的主键作为存储在搜索索引中的模型唯一 ID / 键。如果需要自定义此行为，可以在模型上重写 `getScoutKey` 和 `getScoutKeyName` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class User extends Model
{
    use Searchable;

    /**
     * 获取用于索引模型的值。
     */
    public function getScoutKey(): mixed
    {
        return $this->email;
    }

    /**
     * 获取用于索引模型的键名。
     */
    public function getScoutKeyName(): mixed
    {
        return 'email';
    }
}
```

### 配置每个模型的搜索引擎

在搜索时，Scout 通常会使用应用程序 `scout` 配置文件中指定的默认搜索引擎。但是，可以通过在模型上重写 `searchableUsing` 方法来更改特定模型的搜索引擎：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Engines\Engine;
use Laravel\Scout\EngineManager;
use Laravel\Scout\Searchable;

class User extends Model
{
    use Searchable;

    /**
     * 获取用于索引模型的引擎。
     */
    public function searchableUsing(): Engine
    {
        return app(EngineManager::class)->engine('meilisearch');
    }
}
```

### 识别用户

Scout 还允许你在使用 [Algolia](https://algolia.com/) 时自动识别用户。将经过身份验证的用户与搜索操作关联起来，在查看 Algolia 仪表板中的搜索分析时可能会很有帮助。你可以通过在应用的 `.env` 文件中将 `SCOUT_IDENTIFY` 环境变量定义为 `true` 来启用用户识别：

启用此功能还将传递请求的 IP 地址和经过身份验证的用户的主要标识符到 Algolia，以便将这些数据与用户发出的任何搜索请求关联起来。

## 数据库 / 集合引擎

### 数据库引擎

> **注意**  
> 当前数据库引擎支持 MySQL 和 PostgreSQL。

如果你的应用与中小型数据库交互或工作负载较轻，你可能会发现使用 Scout 的 “database” 引擎更方便。数据库引擎将在从现有数据库筛选结果以确定查询的适用搜索结果时使用 “where like” 子句和全文索引。

要使用数据库引擎，你可以简单地将 `SCOUT_DRIVER` 环境变量的值设置为 `database`，或者直接在应用的 `scout` 配置文件中指定 `database` 驱动程序：

一旦你将数据库引擎指定为首选驱动程序，你必须[配置可搜索的数据](#%E9%85%8D%E7%BD%AE%E5%8F%AF%E6%90%9C%E7%B4%A2%E7%9A%84%E6%95%B0%E6%8D%AE)。然后，你可以开始对你的模型执行[搜索查询](#%E6%90%9C%E7%B4%A2)。使用数据库引擎时，不需要进行搜索引擎索引，例如需要为 Algolia、Meilisearch 或 Typesense 索引播种的索引。

#### 自定义数据库搜索策略

默认情况下，数据库引擎将针对你已经[配置为可搜索的](#%E9%85%8D%E7%BD%AE%E5%8F%AF%E6%90%9C%E7%B4%A2%E7%9A%84%E6%95%B0%E6%8D%AE)每个模型属性执行 「where like」 查询。然而，在某些情况下，这可能导致性能不佳。因此，可以配置数据库引擎的搜索策略，以便一些指定的列利用全文搜索查询，或者仅使用 “where like” 约束来搜索字符串的前缀（`example%`），而不是在整个字符串内进行搜索（`%example%`）。

要定义这种行为，你可以为模型的 `toSearchableArray` 方法分配 PHP 属性。未分配额外搜索策略行为的任何列将继续使用默认的 「where like」 策略：

```php
use Laravel\Scout\Attributes\SearchUsingFullText;
use Laravel\Scout\Attributes\SearchUsingPrefix;

/**
 * 获取模型的可索引数据数组。
 *
 * @return array<string, mixed>
 */
#[SearchUsingPrefix(['id', 'email'])]
#[SearchUsingFullText(['bio'])]
public function toSearchableArray(): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'bio' => $this->bio,
    ];
}
```

> **注意**  
> 在指定某个列应使用全文搜索查询约束之前，请确保该列已被分配了[全文索引](https://learnku.com/docs/laravel/11.x/migrations#available-index-types)。

### 集合引擎

虽然你可以在本地开发过程中自由使用 Algolia、Meilisearch 或 Typesense 搜索引擎，但你可能会发现使用 「collection」 引擎更为方便。集合引擎将在从现有数据库筛选结果以确定查询的适用搜索结果时使用 「where」 子句和集合过滤。使用此引擎时，不需要对可搜索的模型进行 「索引」，因为它们将直接从你的本地数据库中检索。

要使用集合引擎，你可以简单地将 `SCOUT_DRIVER` 环境变量的值设置为 `collection`，或者直接在应用的 `scout` 配置文件中指定 `collection` 驱动程序：

一旦你将集合驱动程序指定为首选驱动程序，你可以开始对你的模型执行[搜索查询](#%E6%90%9C%E7%B4%A2)。使用集合引擎时，不需要进行搜索引擎索引，例如需要为 Algolia、Meilisearch 或 Typesense 索引播种的索引。

#### 与数据库引擎的区别

乍一看，“database” 和 “collections” 引擎非常相似。它们都直接与你的数据库交互以检索搜索结果。然而，集合引擎不使用全文索引或 `LIKE` 子句来查找匹配的记录。相反，它获取所有可能的记录，并使用 Laravel 的 `Str::is` 辅助函数来确定搜索字符串是否存在于模型属性值中。

集合引擎是最通用的搜索引擎，因为它适用于 Laravel 支持的所有关系数据库（包括 SQLite 和 SQL Server）；然而，它不如 Scout 的数据库引擎高效。

## 索引

### 批量导入

如果你将 Scout 安装到现有项目中，可能已经有需要导入到你的索引中的数据库记录。Scout 提供了一个 `scout:import` Artisan 命令，你可以使用它将所有现有记录导入到你的搜索索引中：

```shell
php artisan scout:import "App\Models\Post"
```

`flush` 命令可用于从你的搜索索引中删除模型的所有记录：

```shell
php artisan scout:flush "App\Models\Post"
```

#### 修改导入查询

如果你想修改用于检索所有模型以进行批量导入的查询，你可以在你的模型上定义一个 `makeAllSearchableUsing` 方法。这是一个很好的地方，可以在导入模型之前添加任何可能需要的急切关联加载：

```php
use Illuminate\Database\Eloquent\Builder;

/**
 * 修改用于在使所有模型可搜索时检索模型的查询。
 */
protected function makeAllSearchableUsing(Builder $query): Builder
{
    return $query->with('author');
}
```

> **注意**  
> 当使用队列批量导入模型时，`makeAllSearchableUsing` 方法可能不适用。当模型集合由作业处理时，关系[不会被恢复](https://learnku.com/docs/laravel/11.x/queues#handling-relationships)。

### 添加记录

一旦你向模型添加了 `Laravel\Scout\Searchable` 特性，你只需对模型实例执行 `save` 或 `create` 操作，它将自动添加到你的搜索索引中。如果你已经配置 Scout [使用队列](#%E9%98%9F%E5%88%97%E5%8C%96)，此操作将由你的队列工作程序在后台执行：

```php
use App\Models\Order;

$order = new Order;

// ...

$order->save();
```

#### 通过查询添加记录

如果你想通过 Eloquent 查询将一组模型添加到你的搜索索引中，你可以在 Eloquent 查询上链式调用 `searchable` 方法。`searchable` 方法将[分块处理查询结果](https://learnku.com/docs/laravel/11.x/eloquent#chunking-results)，并将记录添加到你的搜索索引中。同样，如果你已经配置 Scout 使用队列，所有的分块将由你的队列工作者在后台导入：

```php
use App\Models\Order;

Order::where('price', '>', 100)->searchable();
```

你也可以在 Eloquent 关联实例上调用 `searchable` 方法：

```php
$user->orders()->searchable();
```

或者，如果你已经在内存中有一组 Eloquent 模型实例，你可以在集合实例上调用 `searchable` 方法，将模型实例添加到它们对应的索引中：

> **注意**  
> `searchable` 方法可以被视为一种 “upsert” 操作。换句话说，如果模型记录已经存在于你的索引中，它将被更新。如果它不存在于搜索索引中，它将被添加到索引中。

### 更新记录

要更新可搜索的模型，你只需更新模型实例的属性并将模型 `save` 到你的数据库中。Scout 将自动将更改持久化到你的搜索索引中：

```php
use App\Models\Order;

$order = Order::find(1);

// 更新订单...

$order->save();
```

你也可以在 Eloquent 查询实例上调用 `searchable` 方法来更新一组模型。如果这些模型不存在于你的搜索索引中，它们将被创建：

```php
Order::where('price', '>', 100)->searchable();
```

如果你想更新关系中所有模型的搜索索引记录，你可以在关联实例上调用 `searchable`：

```php
$user->orders()->searchable();
```

或者，如果你已经在内存中有一组 Eloquent 模型实例，你可以在集合实例上调用 `searchable` 方法，来更新它们对应的索引中的模型实例：

#### 在导入前修改记录

有时候，在使模型可搜索之前，你可能需要准备好模型的集合。例如，你可能希望预加载一个关联，以便将关联数据高效地添加到你的搜索索引中。为实现这一目的，在相应的模型上定义一个 `makeSearchableUsing` 方法：

```php
use Illuminate\Database\Eloquent\Collection;

/**
 * 修改正在使其可搜索的模型集合。
 */
public function makeSearchableUsing(Collection $models): Collection
{
    return $models->load('author');
}
```

### 移除记录

要从索引中移除记录，你可以简单地从数据库中 `delete` 模型。即使你使用了[软删除](https://learnku.com/docs/laravel/11.x/eloquent#soft-deleting)模型，也可以这样做：

```php
use App\Models\Order;

$order = Order::find(1);

$order->delete();
```

如果你不想在删除记录之前检索模型，你可以在 Eloquent 查询实例上使用 `unsearchable` 方法：

```php
Order::where('price', '>', 100)->unsearchable();
```

如果你想移除关系中所有模型的搜索索引记录，你可以在关联实例上调用 `unsearchable`：

```php
$user->orders()->unsearchable();
```

或者，如果你已经在内存中有一组 Eloquent 模型实例，你可以在集合实例上调用 `unsearchable` 方法，从它们对应的索引中移除模型实例：

要从它们对应的索引中移除所有模型记录，你可以调用 `removeAllFromSearch` 方法：

```php
Order::removeAllFromSearch();
```

### 暂停索引

有时候，你可能需要对模型执行一批 Eloquent 操作，而不将模型数据同步到你的搜索索引中。你可以使用 `withoutSyncingToSearch` 方法来实现这一点。该方法接受一个闭包，该闭包将立即执行。闭包内发生的任何模型操作都不会被同步到模型的索引中：

```php
use App\Models\Order;

Order::withoutSyncingToSearch(function () {
    // 执行模型操作...
});
```

### 有条件地使模型实例可搜索

有时候，你可能只想在特定条件下使模型可搜索。例如，假设你有一个 `App\Models\Post` 模型，该模型可能处于 「草稿」 和 「已发布」 两种状态之一。你可能只想允许 「已发布」 的帖子可搜索。为实现这一目的，你可以在你的模型上定义一个 `shouldBeSearchable` 方法：

```php
/**
 * 确定模型是否应该可搜索。
 */
public function shouldBeSearchable(): bool
{
    return $this->isPublished();
}
```

`shouldBeSearchable` 方法仅在通过 `save` 和 `create` 方法、查询或关联操作模型时应用。直接使用 `searchable` 方法使模型或集合可搜索将覆盖 `shouldBeSearchable` 方法的结果。

> **注意**  
> 当使用 Scout 的 “database” 引擎时，`shouldBeSearchable` 方法不适用，因为所有可搜索数据始终存储在数据库中。当使用数据库引擎时，为了实现类似的行为，你应该使用 [where 子句](#where-clauses)。

## 搜索

你可以使用 `search` 方法开始搜索模型。`search` 方法接受一个字符串，该字符串将用于搜索你的模型。然后，你应该在搜索查询上链接 `get` 方法，以检索与给定搜索查询匹配的 Eloquent 模型：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->get();
```

由于 Scout 搜索返回一个 Eloquent 模型集合，你甚至可以直接从路由或控制器返回结果，它们将自动转换为 JSON：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/search', function (Request $request) {
    return Order::search($request->search)->get();
});
```

如果你想在将搜索结果转换为 Eloquent 模型之前获取原始搜索结果，你可以使用 `raw` 方法：

```php
$orders = Order::search('Star Trek')->raw();
```

#### 自定义索引

搜索查询通常将在模型的 [`searchableAs`](#configuring-model-indexes) 方法指定的索引上执行。但是，你可以使用 `within` 方法指定应该搜索的自定义索引：

```php
$orders = Order::search('Star Trek')
    ->within('tv_shows_popularity_desc')
    ->get();
```

### Where 子句

Scout 允许你向搜索查询添加简单的 「where」 子句。目前，这些子句仅支持基本的数值相等性检查，并且主要用于通过所有者 ID 对搜索查询进行范围限定：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->where('user_id', 1)->get();
```

此外，`whereIn` 方法可用于验证给定列的值是否包含在给定数组中：

```php
$orders = Order::search('Star Trek')->whereIn(
    'status', ['open', 'paid']
)->get();
```

`whereNotIn` 方法验证给定列的值是否不包含在给定数组中：

```php
$orders = Order::search('Star Trek')->whereNotIn(
    'status', ['closed']
)->get();
```

由于搜索索引不是关系数据库，目前不支持更高级的 「where」 子句。

> **注意**  
> 如果你的应用程序使用 Meilisearch，则必须在使用 Scout 的 「where」 子句之前配置应用程序的[可过滤属性](#configuring-filterable-data-for-meilisearch)。

### 分页

除了检索模型集合之外，你还可以使用 `paginate` 方法对搜索结果进行分页。该方法将返回一个 `Illuminate\Pagination\LengthAwarePaginator` 实例，就像你对传统的 Eloquent 查询进行分页一样：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->paginate();
```

你可以通过将数量作为 `paginate` 方法的第一个参数来指定每页检索多少个模型：

```php
$orders = Order::search('Star Trek')->paginate(15);
```

一旦你获取了结果，你可以使用 [Blade](https://learnku.com/docs/laravel/11.x/blade) 显示结果并渲染页面链接，就像对传统的 Eloquent 查询进行分页一样：

```html
<div class="container">
    @foreach ($orders as $order)
        {{ $order->price }}
    @endforeach
</div>

{{ $orders->links() }}
```

当然，如果你想将分页结果作为 JSON 返回，你可以直接从路由或控制器返回分页器实例：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    return Order::search($request->input('query'))->paginate(15);
});
```

> **注意**  
> 由于搜索引擎不了解你的 Eloquent 模型的全局作用域定义，因此在使用 Scout 分页的应用程序中不应该使用全局作用域。或者，在通过 Scout 进行搜索时，你应该重新创建全局作用域的约束。

### 软删除

如果你的索引模型正在进行[软删除](https://learnku.com/docs/laravel/11.x/eloquent#soft-deleting)，并且你需要搜索软删除的模型，将 `config/scout.php` 配置文件中的 `soft_delete` 选项设置为 `true`：

当此配置选项为 `true` 时，Scout 不会从搜索索引中删除软删除的模型。相反，它会在索引记录上设置一个隐藏的 `__soft_deleted` 属性。然后，当搜索时，你可以使用 `withTrashed` 或 `onlyTrashed` 方法检索软删除的记录：

```php
use App\Models\Order;

// 在检索结果时包括已删除的记录...
$orders = Order::search('Star Trek')->withTrashed()->get();

// 在检索结果时仅包括已删除的记录...
$orders = Order::search('Star Trek')->onlyTrashed()->get();
```

> **注意**  
> 当使用 `forceDelete` 永久删除软删除的模型时，Scout 将自动从搜索索引中删除它。

### 自定义引擎搜索

如果你需要对引擎的搜索行为进行高级定制，你可以将闭包作为 `search` 方法的第二个参数传递。例如，你可以使用此回调在将搜索查询传递给 Algolia 之前向搜索选项添加地理位置数据：

```php
use Algolia\AlgoliaSearch\SearchIndex;
use App\Models\Order;

Order::search(
    'Star Trek',
    function (SearchIndex $algolia, string $query, array $options) {
        $options['body']['query']['bool']['filter']['geo_distance'] = [
            'distance' => '1000km',
            'location' => ['lat' => 36, 'lon' => 111],
        ];

        return $algolia->search($query, $options);
    }
)->get();
```

#### 自定义 Eloquent 结果查询

在 Scout 从你的应用程序搜索引擎中检索到一组匹配的 Eloquent 模型之后，Eloquent 会通过它们的主键检索所有匹配的模型。你可以通过调用 `query` 方法来自定义此查询。`query` 方法接受一个闭包作为参数，该闭包将接收 Eloquent 查询构建器实例：

```php
use App\Models\Order;
use Illuminate\Database\Eloquent\Builder;

$orders = Order::search('Star Trek')
    ->query(fn (Builder $query) => $query->with('invoices'))
    ->get();
```

由于此回调在相关模型已经从你的应用程序搜索引擎中检索到后被调用，因此 `query` 方法不应该用于 “过滤” 结果。相反，你应该使用 [Scout where 子句](#where%E5%AD%90%E5%8F%A5)。

## 自定义引擎

#### 编写引擎

如果内置的 Scout 搜索引擎之一不符合你的需求，你可以编写自己的自定义引擎并将其注册到 Scout 中。你的引擎应该扩展 `Laravel\Scout\Engines\Engine` 抽象类。这个抽象类包含你的自定义引擎必须实现的八个方法：

```php
use Laravel\Scout\Builder;

abstract public function update($models);
abstract public function delete($models);
abstract public function search(Builder $builder);
abstract public function paginate(Builder $builder, $perPage, $page);
abstract public function mapIds($results);
abstract public function map(Builder $builder, $results, $model);
abstract public function getTotalCount($results);
abstract public function flush($model);
```

你可能会发现审查 `Laravel\Scout\Engines\AlgoliaEngine` 类中这些方法的实现对你有所帮助。这个类将为你提供一个很好的起点，帮助你学习如何在自己的引擎中实现每个方法。

#### 注册引擎

一旦你编写了自定义引擎，你可以使用 Scout 引擎管理器的 `extend` 方法将其注册到 Scout 中。Scout 的引擎管理器可以从 Laravel 服务容器中解析。你应该在 `App\Providers\AppServiceProvider` 类的 `boot` 方法或应用程序使用的任何其他服务提供者中调用 `extend` 方法：

```php
use App\ScoutExtensions\MySqlSearchEngine;
use Laravel\Scout\EngineManager;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    resolve(EngineManager::class)->extend('mysql', function () {
        return new MySqlSearchEngine;
    });
}
```

一旦你的引擎被注册，你可以在应用程序的 `config/scout.php` 配置文件中将其指定为默认的 Scout `driver`：

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/sc...](https://learnku.com/docs/laravel/11.x/scoutmd/16733)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/sc...](https://learnku.com/docs/laravel/11.x/scoutmd/16733)