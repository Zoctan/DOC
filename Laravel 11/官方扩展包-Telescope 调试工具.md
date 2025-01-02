本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Telescope

+   [介绍](#introduction)
+   [安装](#installation)
    +   [仅本地安装](#local-only-installation)
    +   [配置](#configuration)
    +   [数据修剪](#data-pruning)
    +   [仪表板授权](#dashboard-authorization)
+   [升级 Telescope](#upgrading-telescope)
+   [过滤](#filtering)
    +   [条目](#filtering-entries)
    +   [批次](#filtering-batches)
+   [标记](#tagging)
+   [可用观察者](#available-watchers)
    +   [批次观察者](#batch-watcher)
    +   [缓存观察者](#cache-watcher)
    +   [命令观察者](#command-watcher)
    +   [转储观察者](#dump-watcher)
    +   [事件观察者](#event-watcher)
    +   [异常观察者](#exception-watcher)
    +   [门禁观察者](#gate-watcher)
    +   [HTTP 客户端观察者](#http-client-watcher)
    +   [作业观察者](#job-watcher)
    +   [日志观察者](#log-watcher)
    +   [邮件观察者](#mail-watcher)
    +   [模型观察者](#model-watcher)
    +   [通知观察者](#notification-watcher)
    +   [查询观察者](#query-watcher)
    +   [Redis 观察者](#redis-watcher)
    +   [请求观察者](#request-watcher)
    +   [计划观察者](#schedule-watcher)
    +   [视图观察者](#view-watcher)
+   [显示用户头像](#displaying-user-avatars)

## 介绍

[Laravel Telescope](https://github.com/laravel/telescope) 是你本地 Laravel 开发环境的绝佳伴侣。Telescope 可以为你的应用程序提供请求、异常、日志条目、数据库查询、队列作业、邮件、通知、缓存操作、定时任务、变量转储等方面的洞察。

![telescope-example.png](https://laravel.com/img/docs/telescope-example.png)

## 安装

你可以使用 Composer 包管理器将 Telescope 安装到你的 Laravel 项目中：

```shell
composer require laravel/telescope
```

安装了 Telescope 后，使用 `telescope:install` Artisan 命令发布其资产和迁移。安装了 Telescope 后，你还应该运行 `migrate` 命令以创建存储 Telescope 数据所需的表：

```shell
php artisan telescope:install

php artisan migrate
```

最后，你可以通过 `/telescope` 路由访问 Telescope 仪表板。

### 仅本地安装

如果你计划仅在本地开发中使用 Telescope，可以使用 `--dev` 标志安装 Telescope：

```shell
composer require laravel/telescope --dev

php artisan telescope:install

php artisan migrate
```

运行完 `telescope:install` 后，应该从你的应用程序的 `bootstrap/providers.php` 配置文件中移除 `TelescopeServiceProvider` 服务提供程序的注册。相反，在 `App\Providers\AppServiceProvider` 类的 `register` 方法中手动注册 Telescope 的服务提供程序。我们将确保当前环境为 `local`，然后再注册这些提供程序：

```php
/**
 * Register any application services.
 */
public function register(): void
{
    if ($this->app->environment('local')) {
        $this->app->register(\Laravel\Telescope\TelescopeServiceProvider::class);
        $this->app->register(TelescopeServiceProvider::class);
    }
}
```

最后，你还应该防止 Telescope 包被 [自动发现](https://learnku.com/docs/laravel/11.x/packages#package-discovery) ，方法是在你的 `composer.json` 文件中添加以下内容：

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "laravel/telescope"
        ]
    }
},
```

### 配置

在发布 Telescope 的资产后，其主要配置文件将位于 `config/telescope.php`。这个配置文件允许你配置你的[观察者选项](#available-watchers)。每个配置选项都包括其用途的描述，所以一定要彻底探索这个文件。

如果需要，你可以使用 `enabled` 配置选项完全禁用 Telescope 的数据收集：

```php
'enabled' => env('TELESCOPE_ENABLED', true),
```

### 数据修剪

如果不进行修剪，`telescope_entries` 表会很快积累记录。为了缓解这个问题，你应该 [调度](https://learnku.com/docs/laravel/11.x/scheduling) `telescope:prune` Artisan 命令以每天运行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune')->daily();
```

默认情况下，所有超过 24 小时的条目将被修剪。在调用命令时，你可以使用 `hours` 选项来确定保留 Telescope 数据的时间长度。例如，以下命令将删除 48 小时前创建的所有记录：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('telescope:prune --hours=48')->daily();
```

### 仪表板授权

Telescope 仪表板可以通过 `/telescope` 路由访问。默认情况下，你只能在 `local` 环境中访问此仪表板。在你的 `app/Providers/TelescopeServiceProvider.php` 文件中，有一个[授权门](https://learnku.com/docs/laravel/11.x/authorization#gates)定义。这个授权门控制在**非本地**环境中访问 Telescope。你可以根据需要修改此门以限制对 Telescope 的访问：

```php
use App\Models\User;

/**
 * Register the Telescope gate.
 *
 * This gate determines who can access Telescope in non-local environments.
 */
protected function gate(): void
{
    Gate::define('viewTelescope', function (User $user) {
        return in_array($user->email, [
            'taylor@laravel.com',
        ]);
    });
}
```

> \[!WARNING\]  
> 你应确保在生产环境中将你的 `APP_ENV` 环境变量更改为 `production`。否则，你的 Telescope 安装将公开可访问。

## 升级 Telescope

当升级到 Telescope 的新主要版本时，重要的是仔细阅读[升级指南](https://github.com/laravel/telescope/blob/master/UPGRADE.md)。

此外，当升级到任何新的 Telescope 版本时，你应该重新发布 Telescope 的资产：

```shell
php artisan telescope:publish
```

为了保持资产的最新状态并避免未来更新中的问题，你可以将 `vendor:publish --tag=laravel-assets` 命令添加到你的应用程序的 `composer.json` 文件中的 `post-update-cmd` 脚本中：

```json
{
    "scripts": {
        "post-update-cmd": [
            "@php artisan vendor:publish --tag=laravel-assets --ansi --force"
        ]
    }
}
```

## 过滤

### 条目

你可以通过在 `App\Providers\TelescopeServiceProvider` 类中定义的 `filter` 闭包来过滤 Telescope 记录的数据。默认情况下，此闭包在 `local` 环境中记录所有数据以及异常、失败的作业、计划任务和在所有其他环境中具有监视标记的数据：

```php
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->hideSensitiveRequestDetails();

    Telescope::filter(function (IncomingEntry $entry) {
        if ($this->app->environment('local')) {
            return true;
        }

        return $entry->isReportableException() ||
            $entry->isFailedJob() ||
            $entry->isScheduledTask() ||
            $entry->isSlowQuery() ||
            $entry->hasMonitoredTag();
    });
}
```

### 批处理

虽然 `filter` 闭包用于过滤单个条目的数据，但你可以使用 `filterBatch` 方法来注册一个闭包，用于过滤给定请求或控制台命令的所有数据。如果闭包返回 `true`，则所有条目都将被 Telescope 记录：

```php
use Illuminate\Support\Collection;
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->hideSensitiveRequestDetails();

    Telescope::filterBatch(function (Collection $entries) {
        if ($this->app->environment('local')) {
            return true;
        }

        return $entries->contains(function (IncomingEntry $entry) {
            return $entry->isReportableException() ||
                $entry->isFailedJob() ||
                $entry->isScheduledTask() ||
                $entry->isSlowQuery() ||
                $entry->hasMonitoredTag();
        });
    });
}
```

## 标记

Telescope 允许你通过「标记」搜索条目。通常，标记是 Eloquent 模型类名或经过身份验证的用户 ID，Telescope 会自动将其添加到条目中。偶尔，你可能希望将自定义标记附加到条目。为此，你可以使用 `Telescope::tag` 方法。`tag` 方法接受一个闭包，该闭包应返回一个标记数组。闭包返回的标记将与 Telescope 自动附加到条目的任何标记合并。通常情况下，你应该在 `App\Providers\TelescopeServiceProvider` 类的 `register` 方法中调用 `tag` 方法：

```php
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->hideSensitiveRequestDetails();

    Telescope::tag(function (IncomingEntry $entry) {
        return $entry->type === 'request'
                    ? ['status:'.$entry->content['response_status']]
                    : [];
    });
 }
```

## 可用观察者

Telescope 的「观察者」在执行请求或控制台命令时收集应用程序数据。你可以在 `config/telescope.php` 配置文件中自定义要启用的观察者列表：

```php
'watchers' => [
    Watchers\CacheWatcher::class => true,
    Watchers\CommandWatcher::class => true,
    ...
],
```

某些观察者还允许你提供额外的定制选项：

```php
'watchers' => [
    Watchers\QueryWatcher::class => [
        'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
        'slow' => 100,
    ],
    ...
],
```

### 批处理观察者

批处理观察者记录有关排队的[批处理](https://learnku.com/docs/laravel/11.x/queues#job-batching)的信息，包括作业和连接信息。

### 缓存观察者

缓存观察者在缓存键命中、未命中、更新和遗忘时记录数据。

### 命令观察者

命令观察者在执行 Artisan 命令时记录参数、选项、退出代码和输出。如果你希望排除某些命令被观察者记录，你可以在 `config/telescope.php` 文件中的 `ignore` 选项中指定这些命令：

```php
'watchers' => [
    Watchers\CommandWatcher::class => [
        'enabled' => env('TELESCOPE_COMMAND_WATCHER', true),
        'ignore' => ['key:generate'],
    ],
    ...
],
```

### Dump 观察者

Dump 观察者记录并在 Telescope 中显示你的变量转储。在使用 Laravel 时，可以使用全局的 `dump` 函数转储变量。要记录转储，浏览器中必须打开 Dump 观察者选项卡，否则观察者将忽略转储。

### 事件观察者

事件观察者记录应用程序分派的任何[事件](https://learnku.com/docs/laravel/11.x/events)的负载、侦听器和广播数据。 Laravel 框架内部事件将被事件观察者忽略。

### 异常观察者

异常观察者记录应用程序抛出的任何可报告异常的数据和堆栈跟踪。

### Gate 观察者

Gate 观察者记录应用程序进行的[门禁和策略](https://learnku.com/docs/laravel/11.x/authorization)检查的数据和结果。如果你希望排除观察者记录的某些权限，则可以在 `config/telescope.php` 文件中的 `ignore_abilities` 选项中指定这些权限：

```php
'watchers' => [
    Watchers\GateWatcher::class => [
        'enabled' => env('TELESCOPE_GATE_WATCHER', true),
        'ignore_abilities' => ['viewNova'],
    ],
    ...
],
```

### HTTP 客户端观察者

HTTP 客户端观察者记录应用程序发出的 [HTTP 客户端请求](https://learnku.com/docs/laravel/11.x/http-client)。

### 作业观察者

作业观察者记录应用程序分派的任何[作业](https://learnku.com/docs/laravel/11.x/queues)的数据和状态。

### 日志观察者

日志观察者记录应用程序写入的[日志数据](https://learnku.com/docs/laravel/11.x/logging)。默认情况下，Telescope 仅记录 `error` 级别及以上的日志。但是，你可以修改应用程序的 `config/telescope.php` 配置文件中的 `level` 选项来修改此行为：

```php
'watchers' => [
    Watchers\LogWatcher::class => [
        'enabled' => env('TELESCOPE_LOG_WATCHER', true),
        'level' => 'debug',
    ],

    // ...
],
```

### 邮件观察者

邮件观察者允许你在浏览器中预览应用程序发送的[邮件](https://learnku.com/docs/laravel/11.x/mail)，以及它们的相关数据。你还可以将邮件下载为 `.eml` 文件。

### 模型观察者

### 模型观察者

模型观察者在调度 Eloquent [模型事件](https://learnku.com/docs/laravel/11.x/eloquent#events) 时记录模型更改。你可以通过观察者的 `events` 选项指定应记录哪些模型事件：

```php
'watchers' => [
    Watchers\ModelWatcher::class => [
        'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
        'events' => ['eloquent.created*', 'eloquent.updated*'],
    ],
    ...
],
```

如果你想记录在给定请求期间加载的模型数量，请启用 `hydrations` 选项：

```php
'watchers' => [
    Watchers\ModelWatcher::class => [
        'enabled' => env('TELESCOPE_MODEL_WATCHER', true),
        'events' => ['eloquent.created*', 'eloquent.updated*'],
        'hydrations' => true,
    ],
    ...
],
```

### 通知观察者

通知观察者记录应用程序发送的所有[通知](https://learnku.com/docs/laravel/11.x/notifications)。如果通知触发了邮件发送，并且你已启用邮件观察者，则邮件也可以在邮件观察者屏幕上预览。

### 查询观察者

查询观察者记录应用程序执行的所有查询的原始 SQL、绑定和执行时间。观察者还将任何执行时间超过 100 毫秒的查询标记为 `slow`。你可以使用观察者的 `slow` 选项自定义慢查询阈值：

```php
'watchers' => [
    Watchers\QueryWatcher::class => [
        'enabled' => env('TELESCOPE_QUERY_WATCHER', true),
        'slow' => 50,
    ],
    ...
],
```

### Redis 观察者

Redis 观察者记录应用程序执行的所有 [Redis](https://learnku.com/docs/laravel/11.x/redis) 命令。如果你在使用 Redis 进行缓存，缓存命令也将被 Redis 观察者记录。

### 请求观察者

请求观察者记录应用程序处理的所有请求的请求、标头、会话和响应数据。你可以通过 `size_limit`（以千字节为单位）选项限制记录的响应数据大小：

```php
'watchers' => [
    Watchers\RequestWatcher::class => [
        'enabled' => env('TELESCOPE_REQUEST_WATCHER', true),
        'size_limit' => env('TELESCOPE_RESPONSE_SIZE_LIMIT', 64),
    ],
    ...
],
```

### 计划任务观察者

计划任务观察者记录应用程序运行的任何[定时任务](https://learnku.com/docs/laravel/11.x/scheduling)的命令和输出。

### 视图观察者

视图观察者记录渲染视图时使用的[视图](https://learnku.com/docs/laravel/11.x/views)名称、路径、数据和 "组合器"。

## 显示用户头像

Telescope 仪表板显示了在保存特定条目时进行身份验证的用户的头像。默认情况下，Telescope 将使用 Gravatar 网络服务来获取头像。但是，你可以通过在 `App\Providers\TelescopeServiceProvider` 类中注册回调来自定义头像 URL。回调函数将接收用户的 ID 和电子邮件地址，并应返回用户的头像图片 URL：

```php
use App\Models\User;
use Laravel\Telescope\Telescope;

/**
 * Register any application services.
 */
public function register(): void
{
    // ...

    Telescope::avatar(function (string $id, string $email) {
        return '/avatars/'.User::find($id)->avatar_path;
    });
}
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/te...](https://learnku.com/docs/laravel/11.x/telescopemd/16735)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/te...](https://learnku.com/docs/laravel/11.x/telescopemd/16735)