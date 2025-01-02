本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Pulse

+   [介绍](#introduction)
+   [安装](#installation)
    +   [配置](#configuration)
+   [仪表板](#dashboard)
    +   [授权](#dashboard-authorization)
    +   [自定义](#dashboard-customization)
    +   [解决用户](#dashboard-resolving-users)
    +   [卡片](#dashboard-cards)
+   [记录条目](#capturing-entries)
    +   [记录器](#recorders)
    +   [过滤](#filtering)
+   [性能](#performance)
    +   [使用不同数据库](#using-a-different-database)
    +   [Redis 导入](#ingest)
    +   [抽样](#sampling)
    +   [修剪](#trimming)
    +   [处理 Pulse 异常](#pulse-exceptions)
+   [自定义卡片](#custom-cards)
    +   [卡片组件](#custom-card-components)
    +   [样式](#custom-card-styling)
    +   [数据捕获和聚合](#custom-card-data)

## 介绍

[Laravel Pulse](https://github.com/laravel/pulse) 提供了应用程序性能和使用情况的一览洞察。通过 Pulse，你可以追踪诸如慢作业和端点、找到最活跃的用户等瓶颈问题。

要深入调试单个事件，请查看 [Laravel Telescope](https://learnku.com/docs/laravel/11.x/telescope)。

## 安装

> **注意**  
> Pulse 的官方存储实现目前需要 MySQL、MariaDB 或 PostgreSQL 数据库。如果你使用不同的数据库引擎，你将需要一个单独的 MySQL、MariaDB 或 PostgreSQL 数据库来存储 Pulse 数据。

你可以使用 Composer 包管理器安装 Pulse：

```sh
composer require laravel/pulse
```

接下来，你应该使用 `vendor:publish` Artisan 命令发布 Pulse 配置和迁移文件：

```shell
php artisan vendor:publish --provider="Laravel\Pulse\PulseServiceProvider"
```

最后，你应该运行 `migrate` 命令来创建存储 Pulse 数据所需的表：

一旦运行了 Pulse 的数据库迁移，你可以通过 `/pulse` 路由访问 Pulse 仪表板。

> **注意**  
> 如果你不想将 Pulse 数据存储在应用程序的主数据库中，你可以[指定一个专用的数据库连接](#using-a-different-database)。

### 配置

许多 Pulse 的配置选项可以使用环境变量进行控制。要查看可用选项、注册新的记录器或配置高级选项，你可以发布 `config/pulse.php` 配置文件：

```sh
php artisan vendor:publish --tag=pulse-config
```

## 仪表板

### 授权

可以通过 `/pulse` 路由访问 Pulse 仪表板。默认情况下，你只能在 `local` 环境中访问此仪表板，因此你需要为生产环境配置授权，通过自定义 `'viewPulse'` 授权门来实现。你可以在应用程序的 `app/Providers/AppServiceProvider.php` 文件中完成这个操作：

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Gate::define('viewPulse', function (User $user) {
        return $user->isAdmin();
    });

    // ...
}
```

### 自定义

Pulse 仪表板的卡片和布局可以通过发布仪表板视图进行配置。仪表板视图将发布到 `resources/views/vendor/pulse/dashboard.blade.php`：

```sh
php artisan vendor:publish --tag=pulse-dashboard
```

该仪表板由 [Livewire](https://livewire.laravel.com/) 提供支持，允许你自定义卡片和布局，而无需重新构建任何 JavaScript 资源。

在这个文件中，`<x-pulse>` 组件负责渲染仪表板，并为卡片提供网格布局。如果你希望仪表板跨越屏幕的整个宽度，你可以向组件提供 `full-width` 属性：

```blade
<x-pulse full-width>
    ...
</x-pulse>
```

默认情况下，`<x-pulse>` 组件将创建一个 12 列网格，但你可以使用 `cols` 属性进行自定义：

```blade
<x-pulse cols="16">
    ...
</x-pulse>
```

每个卡片都接受 `cols` 和 `rows` 属性来控制空间和定位：

```blade
<livewire:pulse.usage cols="4" rows="2" />
```

大多数卡片还接受一个 `expand` 属性，以显示完整卡片而不是滚动：

```blade
<livewire:pulse.slow-queries expand />
```

### 解决用户

对于显示关于用户信息的卡片，例如应用程序使用情况卡片，Pulse 只会记录用户的 ID。在渲染仪表板时，Pulse 将从你的默认 `Authenticatable` 模型中解析 `name` 和 `email` 字段，并使用 Gravatar 网络服务显示头像。

你可以通过在应用程序的 `App\Providers\AppServiceProvider` 类中调用 `Pulse::user` 方法来自定义字段和头像。

`user` 方法接受一个闭包，该闭包将接收要显示的 `Authenticatable` 模型，并应返回一个包含用户的 `name`、`extra` 和 `avatar` 信息的数组：

```php
use Laravel\Pulse\Facades\Pulse;

/**
 * 引导任何应用服务。
 */
public function boot(): void
{
    Pulse::user(fn ($user) => [
        'name' => $user->name,
        'extra' => $user->email,
        'avatar' => $user->avatar_url,
    ]);

    // ...
}
```

> **注意**  
> 通过实现 `Laravel\Pulse\Contracts\ResolvesUsers` 合同并在 Laravel 的[服务容器](https://learnku.com/docs/laravel/11.x/container#binding-a-singleton)中绑定它，你可以完全自定义如何捕获和检索认证用户。

### 卡片

#### 服务器

`<livewire:pulse.servers />` 卡片显示运行 `pulse:check` 命令的所有服务器的系统资源使用情况。有关系统资源报告的更多信息，请参考有关 [服务器记录器](#servers-recorder) 的文档。

如果你替换了基础架构中的服务器，你可能希望在给定的持续时间后停止在 Pulse 仪表板中显示非活动服务器。你可以使用 `ignore-after` 属性来实现这一点，该属性接受一段时间（秒）后应从 Pulse 仪表板中移除非活动服务器。另外，你也可以提供一个相对时间格式化字符串，比如 `1 小时` 或 `3 天 1 小时`：

```blade
<livewire:pulse.servers ignore-after="3 小时" />
```

#### 应用程序使用情况

`<livewire:pulse.usage />` 卡片显示了向您的应用程序发出请求、调度作业以及遇到慢请求的前 10 位用户。

如果你希望同时在屏幕上查看所有使用情况指标，你可以多次包含该卡片并指定 `type` 属性：

```blade
<livewire:pulse.usage type="requests" />
<livewire:pulse.usage type="slow_requests" />
<livewire:pulse.usage type="jobs" />
```

要了解如何自定义 Pulse 检索和显示用户信息的方式，请参阅我们关于[解析用户](#dashboard-resolving-users)的文档。

> **注意**  
> 如果你的应用程序接收大量请求或调度大量作业，你可能希望启用[抽样](#sampling)。有关更多信息，请参阅关于[用户请求记录器](#user-requests-recorder)、[用户作业记录器](#user-jobs-recorder)和[慢作业记录器](#slow-jobs-recorder)的文档。

#### 异常

`<livewire:pulse.exceptions />` 卡片显示了应用程序中异常发生的频率和最近情况。默认情况下，异常根据异常类和发生位置进行分组。有关更多信息，请参阅[异常记录器](#exceptions-recorder)文档。

#### 队列

`<livewire:pulse.queues />` 卡片显示了应用程序中队列的吞吐量，包括排队、处理、已处理、释放和失败的作业数量。有关更多信息，请参阅[队列记录器](#queues-recorder)文档。

#### 慢请求

`<livewire:pulse.slow-requests />` 卡片显示了超过配置阈值的传入请求，该阈值默认为 1,000 毫秒。有关更多信息，请参阅[慢请求记录器](#slow-requests-recorder)文档。

#### 慢作业

`<livewire:pulse.slow-jobs />` 卡片显示了超过配置阈值的应用程序中排队的作业，该阈值默认为 1,000 毫秒。有关更多信息，请参阅[慢作业记录器](#slow-jobs-recorder)文档。

#### 慢查询

`<livewire:pulse.slow-queries />` 卡片显示了应用程序中超过配置阈值的数据库查询，该阈值默认为 1,000 毫秒。

默认情况下，慢查询根据 SQL 查询（不包括绑定）和发生位置进行分组，但如果你希望仅根据 SQL 查询进行分组，也可以选择不捕获位置。

如果由于接收到非常大的 SQL 查询而遇到渲染性能问题，导致语法高亮显示，你可以通过添加 `without-highlighting` 属性来禁用高亮显示：

```blade
<livewire:pulse.slow-queries without-highlighting />
```

有关更多信息，请参阅[慢查询记录器](#slow-queries-recorder)文档。

#### 慢传出请求

`<livewire:pulse.slow-outgoing-requests />` 卡片显示了使用 Laravel 的 [HTTP 客户端](https://learnku.com/docs/laravel/11.x/http-client)发出的传出请求，这些请求超过了配置的阈值，默认为 1,000 毫秒。

默认情况下，条目将按照完整 URL 进行分组。但是，你可能希望使用正则表达式对相似的传出请求进行归一化或分组。有关更多信息，请参阅[慢传出请求记录器](#slow-outgoing-requests-recorder)文档。

#### 缓存

`<livewire:pulse.cache />` 卡片显示了应用程序的缓存命中和未命中统计信息，包括全局统计和单个键的统计信息。

默认情况下，条目将按键进行分组。但是，你可能希望使用正则表达式对相似的键进行归一化或分组。有关更多信息，请参阅[缓存交互记录器](#cache-interactions-recorder)文档。

## 记录条目

大多数 Pulse 记录器将根据 Laravel 发布的框架事件自动捕获条目。然而，[服务器记录器](#servers-recorder)和一些第三方卡片必须定期轮询信息。要使用这些卡片，你必须在所有单独的应用程序服务器上运行 `pulse:check` 守护进程：

> \[!NOTE\]  
> 为了持续在后台运行 `pulse:check` 进程，你应该使用诸如 Supervisor 等进程监视器，以确保命令不会停止运行。

由于 `pulse:check` 命令是一个长期运行的进程，它不会在没有重新启动的情况下查看你的代码库更改。你应该在应用程序部署过程中通过调用 `pulse:restart` 命令来优雅地重新启动命令：

```sh
php artisan pulse:restart
```

> **注意**  
> Pulse 使用[缓存](https://learnku.com/docs/laravel/11.x/cache)来存储重新启动信号，因此在使用此功能之前，你应该验证为应用程序正确配置了缓存驱动程序。

### 记录器

记录器负责捕获从你的应用程序中记录到 Pulse 数据库中的条目。记录器在 [Pulse 配置文件](#configuration)的 `recorders` 部分中注册和配置。

#### 缓存交互

`CacheInteractions` 记录器捕获有关应用程序中[缓存](https://learnku.com/docs/laravel/11.x/cache)命中和未命中的信息，以便在[缓存](#cache-card)卡片上显示。

你可以选择调整[抽样率](#sampling)和忽略的键模式。

你还可以配置键分组，以便将相似的键作为单个条目分组。例如，你可能希望从缓存相同类型信息的键中删除唯一 ID。组使用正则表达式进行配置，以 “查找和替换” 键的部分。配置文件中包含一个示例：

```php
Recorders\CacheInteractions::class => [
    // ...
    'groups' => [
        // '/:\d+/' => ':*',
    ],
],
```

第一个匹配的模式将被使用。如果没有模式匹配，则将按原样捕获键。

#### 异常

`Exceptions` 记录器捕获应用程序中发生的可报告异常的信息，以在[异常](#exceptions-card)卡片上显示。

你可以选择调整[采样率](#sampling)和忽略的异常模式。你还可以配置是否捕获异常的来源位置。捕获的位置将显示在 Pulse 仪表板上，有助于追踪异常的来源；但是，如果在多个位置发生相同的异常，则对于每个唯一位置，它将多次出现。

#### 队列

`Queues` 记录器捕获有关你的应用程序队列的信息，以在[队列](#queues-card)卡片上显示。

你可以选择调整[采样率](#sampling)和忽略的作业模式。

#### 慢作业

`SlowJobs` 记录器捕获应用程序中发生的慢作业的信息，以在[慢作业](#slow-jobs-recorder)卡片上显示。

你可以选择调整慢作业阈值、[采样率](#sampling)和忽略的作业模式。

你可能有一些作业预计会比其他作业花费更长时间。在这种情况下，你可以配置每个作业的阈值：

```php
Recorders\SlowJobs::class => [
    // ...
    'threshold' => [
        '#^App\\Jobs\\GenerateYearlyReports$#' => 5000,
        'default' => env('PULSE_SLOW_JOBS_THRESHOLD', 1000),
    ],
],
```

如果没有正则表达式模式与作业的类名匹配，则将使用 `'default'` 值。

#### 慢出站请求

`SlowOutgoingRequests` 记录器捕获使用 Laravel 的 [HTTP 客户端](https://learnku.com/docs/laravel/11.x/http-client) 发出的出站 HTTP 请求的信息，如果超过配置的阈值，则在[慢出站请求](#slow-outgoing-requests-card)卡片上显示。

你可以选择调整慢出站请求阈值、[采样率](#sampling)和忽略的 URL 模式。

你可能有一些出站请求预计会比其他请求花费更长时间。在这种情况下，你可以配置每个请求的阈值：

```php
Recorders\SlowOutgoingRequests::class => [
    // ...
    'threshold' => [
        '#backup.zip$#' => 5000,
        'default' => env('PULSE_SLOW_OUTGOING_REQUESTS_THRESHOLD', 1000),
    ],
],
```

如果没有正则表达式模式与请求的 URL 匹配，则将使用 `'default'` 值。

你还可以配置 URL 分组，以便将相似的 URL 作为单个条目进行分组。例如，你可能希望从 URL 路径中删除唯一标识符或仅按域分组。使用正则表达式来配置组，以 “查找和替换” URL 的部分。配置文件中包含一些示例：

```php
Recorders\SlowOutgoingRequests::class => [
    // ...
    'groups' => [
        // '#^https://api\.github\.com/repos/.*$#' => 'api.github.com/repos/*',
        // '#^https?://([^/]*).*$#' => '\1',
        // '#/\d+#' => '/*',
    ],
],
```

将使用第一个匹配的模式。如果没有模式匹配，则将按原样捕获 URL。

#### 慢查询

`SlowQueries` 记录器捕获在你的应用程序中超过配置阈值的任何数据库查询，以在[慢查询](#slow-queries-card)卡片上显示。

你可以选择调整慢查询阈值、[采样率](#sampling)和忽略的查询模式。你还可以配置是否捕获查询位置。捕获的位置将显示在 Pulse 仪表板上，有助于追踪查询的来源；但是，如果在多个位置进行相同的查询，则对于每个唯一位置，它将多次出现。

你可能有一些查询预计会比其他查询花费更长时间。在这种情况下，你可以配置每个查询的阈值：

```php
Recorders\SlowQueries::class => [
    // ...
    'threshold' => [
        '#^insert into `yearly_reports`#' => 5000,
        'default' => env('PULSE_SLOW_QUERIES_THRESHOLD', 1000),
    ],
],
```

如果没有正则表达式模式与查询的 SQL 匹配，则将使用 `'default'` 值。

#### 慢请求

`Requests` 记录器捕获有关发送到你的应用程序的请求的信息，以在[慢请求](#slow-requests-card)和[应用程序使用情况](#application-usage-card)卡片上显示。

你可以选择调整慢路由阈值、[采样率](#sampling)和忽略的路径。

你可能有一些请求预计会比其他请求花费更长时间。在这种情况下，你可以配置每个请求的阈值：

```php
Recorders\SlowRequests::class => [
    // ...
    'threshold' => [
        '#^/admin/#' => 5000,
        'default' => env('PULSE_SLOW_REQUESTS_THRESHOLD', 1000),
    ],
],
```

如果没有正则表达式模式与请求的 URL 匹配，则将使用 `'default'` 值。

#### 服务器

`Servers` 记录器捕获用于运行你的应用程序的服务器的 CPU、内存和存储使用情况，以在[服务器](#servers-card)卡片上显示。此记录器需要在你希望监视的每台服务器上运行 [`pulse:check` 命令](#capturing-entries)。

每个报告服务器必须具有唯一名称。默认情况下，Pulse 将使用 PHP 的 `gethostname` 函数返回的值。如果你想自定义此内容，可以设置 `PULSE_SERVER_NAME` 环境变量：

```env
PULSE_SERVER_NAME=load-balancer
```

Pulse 配置文件还允许你自定义要监视的目录。

#### 用户作业

`UserJobs` 记录器捕获关于在你的应用程序中调度作业的用户的信息，以在[应用程序使用情况](#application-usage-card)卡片上显示。

你可以选择调整[采样率](#sampling)和忽略的作业模式。

#### 用户请求

`UserRequests` 记录器捕获关于向你的应用程序发出请求的用户的信息，以在[应用程序使用情况](#application-usage-card)卡片上显示。

你可以选择调整[采样率](#sampling)和忽略的作业模式。

### 过滤

正如我们所见，许多[记录器](#recorders)通过配置提供了根据值（如请求的 URL）“忽略” 传入条目的能力。但有时根据其他因素过滤记录可能很有用，比如当前已认证的用户。为了过滤这些记录，你可以将闭包传递给 Pulse 的 `filter` 方法。通常，`filter` 方法应该在应用程序的 `AppServiceProvider` 的 `boot` 方法中调用：

```php
use Illuminate\Support\Facades\Auth;
use Laravel\Pulse\Entry;
use Laravel\Pulse\Facades\Pulse;
use Laravel\Pulse\Value;

/**
 * 启动任何应用程序服务。
 */
public function boot(): void
{
    Pulse::filter(function (Entry|Value $entry) {
        return Auth::user()->isNotAdmin();
    });

    // ...
}
```

## 性能

Pulse 被设计为可以轻松集成到现有应用程序中，无需额外的基础设施。然而，对于高流量应用程序，有几种方法可以消除 Pulse 对应用程序性能可能产生的影响。

### 使用不同的数据库

对于高流量应用程序，你可能希望为 Pulse 使用一个专用的数据库连接，以避免影响应用程序数据库。

你可以通过设置 `PULSE_DB_CONNECTION` 环境变量来自定义 Pulse 使用的[数据库连接](https://learnku.com/docs/laravel/11.x/database#configuration)。

```env
PULSE_DB_CONNECTION=pulse
```

### Redis 摄取

> **警告**  
> Redis 摄取需要 Redis 6.2 或更高版本以及 `phpredis` 或 `predis` 作为应用程序配置的 Redis 客户端驱动程序。

默认情况下，Pulse 将在 HTTP 响应发送给客户端后或作业处理完成后，直接将条目存储到[配置的数据库连接](#using-a-different-database)中；然而，你可以使用 Pulse 的 Redis 摄取驱动程序将条目发送到 Redis 流中。这可以通过配置 `PULSE_INGEST_DRIVER` 环境变量来启用：

```php
PULSE_INGEST_DRIVER=redis
```

Pulse 默认情况下将使用你的默认 [Redis 连接](https://learnku.com/docs/laravel/11.x/redis#configuration)，但你可以通过 `PULSE_REDIS_CONNECTION` 环境变量进行自定义：

```php
PULSE_REDIS_CONNECTION=pulse
```

在使用 Redis 摄取时，你需要运行 `pulse:work` 命令来监视流并将条目从 Redis 移动到 Pulse 的数据库表中。

> **注意**  
> 为了在后台持续运行 `pulse:work` 进程，你应该使用进程监控器，例如 Supervisor，以确保 Pulse 工作进程不会停止运行。

由于 `pulse:work` 命令是一个长期运行的进程，如果不重新启动，它将无法看到代码库的更改。在应用程序部署过程中，你应该通过调用 `pulse:restart` 命令来优雅地重新启动命令：

```sh
php artisan pulse:restart
```

> **注意**  
> Pulse 使用[缓存](https://learnku.com/docs/laravel/11.x/cache)来存储重新启动信号，因此在使用此功能之前，你应该验证为应用程序正确配置了缓存驱动程序。

### 采样

默认情况下，Pulse 将捕获应用程序中发生的每个相关事件。对于高流量应用程序，这可能导致需要在仪表板中聚合数百万条数据库行，特别是对于较长时间段。

相反，你可以选择在某些 Pulse 数据记录器上启用「采样」。例如，在[`用户请求`](#user-requests-recorder)记录器上将采样率设置为 `0.1`，这意味着你只记录大约应用程序请求的 10%。在仪表板中，这些值将被缩放并以 `~` 前缀，表示它们是一个近似值。

一般来说，对于特定指标，你拥有的条目越多，你就可以将采样率设置得越低，而不会牺牲太多准确性。

### 裁剪

一旦超出仪表板窗口范围，Pulse 将自动裁剪其存储的条目。裁剪发生在使用抽奖系统进行数据摄取时，这可以在 Pulse 的[配置文件](#configuration)中进行自定义。

### 处理 Pulse 异常

如果在捕获 Pulse 数据时发生异常，例如无法连接到存储数据库，Pulse 将会静默失败，以避免影响你的应用程序。

如果你希望自定义如何处理这些异常，你可以为 `handleExceptionsUsing` 方法提供一个闭包：

```php
use Laravel\Pulse\Facades\Pulse;
use Illuminate\Support\Facades\Log;

Pulse::handleExceptionsUsing(function ($e) {
    Log::debug('Pulse 中发生了异常', [
        'message' => $e->getMessage(),
        'stack' => $e->getTraceAsString(),
    ]);
});
```

## 自定义卡片

Pulse 允许你构建自定义卡片，以显示与你的应用程序特定需求相关的数据。Pulse 使用 [Livewire](https://livewire.laravel.com/)，因此在构建第一个自定义卡片之前，你可能需要[查阅其文档](https://livewire.laravel.com/docs)。

### 卡片组件

在 Laravel Pulse 中创建自定义卡片始于扩展基本的 `Card` Livewire 组件并定义相应的视图：

```php
namespace App\Livewire\Pulse;

use Laravel\Pulse\Livewire\Card;
use Livewire\Attributes\Lazy;

#[Lazy]
class TopSellers extends Card
{
    public function render()
    {
        return view('livewire.pulse.top-sellers');
    }
}
```

当使用 Livewire 的[延迟加载](https://livewire.laravel.com/docs/lazy)功能时，`Card` 组件将自动提供一个占位符，以符合传递给你的组件的 `cols` 和 `rows` 属性。

在编写 Pulse 卡片的相应视图时，你可以利用 Pulse 的 Blade 组件以获得一致的外观和感觉：

```blade
<x-pulse::card :cols="$cols" :rows="$rows" :class="$class" wire:poll.5s="">
    <x-pulse::card-header name="Top Sellers">
        <x-slot:icon>
            ...
        </x-slot:icon>
    </x-pulse::card-header>

    <x-pulse::scroll :expand="$expand">
        ...
    </x-pulse::scroll>
</x-pulse::card>
```

应将 `$cols`、`$rows`、`$class` 和 `$expand` 变量传递给各自的 Blade 组件，以便从仪表板视图自定义卡片布局。你还可以在视图中包含 `wire:poll.5s=""` 属性，以使卡片自动更新。

一旦你定义了 Livewire 组件和模板，该卡片就可以包含在你的[仪表板视图](#dashboard-customization)中：

```blade
<x-pulse>
    ...

    <livewire:pulse.top-sellers cols="4" />
</x-pulse>
```

> **注意**  
> 如果你的卡片包含在一个软件包中，你需要使用 `Livewire::component` 方法向 Livewire 注册组件。

### 样式

如果你的卡片需要超出 Pulse 包含的类和组件的额外样式，有几种选项可用于为你的卡片包含自定义 CSS。

#### Laravel Vite 集成

如果你的自定义卡片位于应用程序的代码库中，并且你正在使用 Laravel 的 [Vite 集成](https://learnku.com/docs/laravel/11.x/vite)，你可以更新你的 `vite.config.js` 文件，包含一个专门的 CSS 入口点用于你的卡片：

```js
laravel({
    input: [
        'resources/css/pulse/top-sellers.css',
        // ...
    ],
}),
```

然后，在[仪表板视图](#dashboard-customization)中使用 `@vite` Blade 指令，指定你的卡片的 CSS 入口点：

```blade
<x-pulse>
    @vite('resources/css/pulse/top-sellers.css')

    ...
</x-pulse>
```

#### CSS 文件

对于其他用例，包括包含在软件包中的 Pulse 卡片，你可以通过在 Livewire 组件上定义一个返回 CSS 文件路径的 `css` 方法，指示 Pulse 加载额外的样式表：

```php
class TopSellers extends Card
{
    // ...

    protected function css()
    {
        return __DIR__.'/../../dist/top-sellers.css';
    }
}
```

当这个卡片包含在仪表板上时，Pulse 将自动在 `<style>` 标签中包含此文件的内容，因此无需将其发布到 `public` 目录中。

#### Tailwind CSS

使用 Tailwind CSS 时，你应该创建一个专门的 Tailwind 配置文件，以避免加载不必要的 CSS 或与 Pulse 的 Tailwind 类冲突：

```js
export default {
    darkMode: 'class',
    important: '#top-sellers',
    content: [
        './resources/views/livewire/pulse/top-sellers.blade.php',
    ],
    corePlugins: {
        preflight: false,
    },
};
```

然后，在你的 CSS 入口点中指定配置文件：

```css
@config "../../tailwind.top-sellers.config.js";
@tailwind base;
@tailwind components;
@tailwind utilities;
```

你还需要在卡片的视图中包含一个与传递给 Tailwind 的 [`important` 选择器策略](https://tailwindcss.com/docs/configuration#selector-strategy)匹配的 `id` 或 `class` 属性：

```blade
<x-pulse::card id="top-sellers" :cols="$cols" :rows="$rows" class="$class">
    ...
</x-pulse::card>
```

### 数据捕获和聚合

自定义卡片可以从任何地方获取和显示数据；然而，你可能希望利用 Pulse 强大而高效的数据记录和聚合系统。

#### 捕获条目

Pulse 允许你使用 `Pulse::record` 方法记录 "条目"：

```php
use Laravel\Pulse\Facades\Pulse;

Pulse::record('user_sale', $user->id, $sale->amount)
    ->sum()
    ->count();
```

提供给 `record` 方法的第一个参数是你要记录的条目的 `type`，而第二个参数是确定如何分组聚合数据的 `key`。对于大多数聚合方法，你还需要指定要聚合的 `value`。在上面的示例中，要聚合的值是 `$sale->amount`。然后，你可以调用一个或多个聚合方法（例如 `sum`），以便 Pulse 可以捕获预先聚合的值到 "存储桶" 中，以便稍后高效地检索。

可用的聚合方法有：

+   `avg`
+   `count`
+   `max`
+   `min`
+   `sum`

> **注意**  
> 当构建一个捕获当前认证用户 ID 的卡片包时，你应该使用 `Pulse::resolveAuthenticatedUserId()` 方法，该方法尊重应用程序中进行的任何[用户解析器自定义](#dashboard-resolving-users)。

#### 检索聚合数据

当扩展 Pulse 的 `Card` Livewire 组件时，你可以使用 `aggregate` 方法来检索在仪表板中查看的期间的聚合数据：

```php
class TopSellers extends Card
{
    public function render()
    {
        return view('livewire.pulse.top-sellers', [
            'topSellers' => $this->aggregate('user_sale', ['sum', 'count']);
        ]);
    }
}
```

`aggregate` 方法返回一个 PHP `stdClass` 对象的集合。每个对象将包含之前捕获的 `key` 属性，以及每个请求的聚合的键：

```php
@foreach ($topSellers as $seller)
    {{ $seller->key }}
    {{ $seller->sum }}
    {{ $seller->count }}
@endforeach
```

Pulse 主要会从预先聚合的存储桶中检索数据；因此，指定的聚合必须事先使用 `Pulse::record` 方法捕获。最老的存储桶通常会部分地超出该期间，因此 Pulse 将聚合最老的条目以填补间隙，并为整个期间提供准确的值，而无需在每次轮询请求中聚合整个期间。

你也可以通过使用 `aggregateTotal` 方法来检索给定类型的总值。例如，以下方法将检索所有用户销售的总和，而不是按用户分组。

```php
$total = $this->aggregateTotal('user_sale', 'sum');
```

#### 显示用户

当使用记录用户 ID 作为键的聚合时，你可以使用 `Pulse::resolveUsers` 方法将键解析为用户记录：

```php
$aggregates = $this->aggregate('user_sale', ['sum', 'count']);

$users = Pulse::resolveUsers($aggregates->pluck('key'));

return view('livewire.pulse.top-sellers', [
    'sellers' => $aggregates->map(fn ($aggregate) => (object) [
        'user' => $users->find($aggregate->key),
        'sum' => $aggregate->sum,
        'count' => $aggregate->count,
    ])
]);
```

`find` 方法返回一个包含 `name`、`extra` 和 `avatar` 键的对象，你可以选择直接将其传递给 `<x-pulse::user-card>` Blade 组件：

```blade
<x-pulse::user-card :user="{{ $seller->user }}" :stats="{{ $seller->sum }}" />
```

#### 自定义记录器

包作者可能希望提供记录器类，以允许用户配置数据的捕获。

记录器在应用程序的 `config/pulse.php` 配置文件的 `recorders` 部分中注册：

```php
[
    // ...
    'recorders' => [
        Acme\Recorders\Deployments::class => [
            // ...
        ],

        // ...
    ],
]
```

记录器可以通过指定 `$listen` 属性来监听事件。Pulse 将自动注册监听器并调用记录器的 `record` 方法：

```php
<?php

namespace Acme\Recorders;

use Acme\Events\Deployment;
use Illuminate\Support\Facades\Config;
use Laravel\Pulse\Facades\Pulse;

class Deployments
{
    /**
     * 要监听的事件。
     *
     * @var array<int, class-string>
     */
    public array $listen = [
        Deployment::class,
    ];

    /**
     * 记录部署。
     */
    public function record(Deployment $event): void
    {
        $config = Config::get('pulse.recorders.'.static::class);

        Pulse::record(
            // ...
        );
    }
}
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/pu...](https://learnku.com/docs/laravel/11.x/pulsemd/16729)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/pu...](https://learnku.com/docs/laravel/11.x/pulsemd/16729)