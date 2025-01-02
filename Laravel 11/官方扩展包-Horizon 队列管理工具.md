本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Horizon

+   [介绍](#introduction)
+   [安装](#installation)
    +   [配置](#configuration)
    +   [均衡策略](#balancing-strategies)
    +   [仪表盘授权](#dashboard-authorization)
    +   [静默任务](#silenced-jobs)
+   [升级 Horizon](#upgrading-horizon)
+   [运行 Horizon](#running-horizon)
    +   [部署 Horizon](#deploying-horizon)
+   [标签](#tags)
+   [通知](#notifications)
+   [指标](#metrics)
+   [删除失败的任务](#deleting-failed-jobs)
+   [从队列中清除任务](#clearing-jobs-from-queues)

## 介绍

> **提示**  
> 在深入了解 Laravel Horizon 之前，你应该熟悉 Laravel 的基础 [队列服务](https://learnku.com/docs/laravel/11.x/queues)。 Horizon 为 Laravel 的队列增加了额外的功能，如果你还不熟悉 Laravel 提供的基本队列功能，这些功能可能会让你感到困惑。

[Laravel Horizon](https://github.com/laravel/horizon) 为你的 Laravel [Redis queues](https://learnku.com/docs/laravel/11.x/queues) 提供了一个美观的仪表盘和代码驱动的配置。它可以方便地监控队列系统的关键指标：任务吞吐量、运行时间、任务失败情况。

在使用 Horizon 时，所有队列的 worker 配置都存储在一个简单的配置文件中。通过在受版本控制的文件中定义应用程序的 worker 配置，你可以在部署应用程序时轻松扩展或修改应用程序的队列 worker。

![horizon-example.png](https://laravel.com/img/docs/horizon-example.png)

## 安装

> **警告**  
> Laravel Horizon 要求你使用 [Redis](https://redis.io/) 来加强你的队列。因此，你应该确保在应用程序的 `config/queue.php` 配置文件中将队列连接设置为 `redis`。

你可以使用 Composer 包管理工具将 Horizon 安装到你的项目里：

```shell
composer require laravel/horizon
```

Horizon 安装之后，使用 `horizon:install` Artisan 命令发布资源：

```shell
php artisan horizon:install
```

### 配置

Horizon 资源发布之后，其主要配置文件将位于 `config/horizon.php` 文件中。此配置文件允许你为应用程序配置队列 worker 选项。每个配置选项包含其用途的描述，所以请务必仔细研究这个文件。

> **警告**  
> Horizon 在内部使用名为 `horizon` 的 Redis 连接。此 Redis 连接名称是保留的，不应再分配给 `database.php` 配置文件中的另一个 Redis 连接，也不可作为 `horizon.php` 配置文件中的 `use` 选项的值。

#### 环境

安装完成后，你应该熟悉的主要 Horizon 配置选项是 `environments` 配置选项。此配置选项是一个数组，包含了你的应用程序运行的环境，并为每个环境定义了 worker 进程选项。默认情况下，此条目包含一个 `production` 和 一个 `local` 环境。不过，你可以根据自己需要自由添加更多环境：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            'maxProcesses' => 10,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
        ],
    ],

    'local' => [
        'supervisor-1' => [
            'maxProcesses' => 3,
        ],
    ],
],
```

你也可以定义一个通配符环境（`*`），当找不到其他匹配的环境时将使用该环境：

```php
'environments' => [
    // ...

    '*' => [
        'supervisor-1' => [
            'maxProcesses' => 3,
        ],
    ],
],
```

当你启动 Horizon 时，它将使用应用程序运行环境所配置的 worker 进程选项。通常情况下，环境由 `APP_ENV` [环境变量](https://learnku.com/docs/laravel/11.x/configuration#determining-the-current-environment) 的值决定。例如，默认的 `local` Horizon 环境配置为启动三个 worker 进程，并自动平衡分配给每个队列的 worker 进程数量。默认的 `production` 环境配置为最多启动 10 个 worker 进程，并自动平衡分配给每个队列的 worker 进程数量。

> **警告**  
> 你应该确保你的 `horizon` 配置文件的 `environments` 部分包含计划在其上运行 Horizon 的每个 [环境](https://learnku.com/docs/laravel/11.x/configuration#environment-configuration) 的配置。

#### Supervisors

正如你在 Horizon 的默认配置文件中看到的那样，每个环境可以包含一个或多个 Supervisor 配置。默认情况下，配置文件将这个 Supervisor 定义为 `supervisor-1`；但是，你可以随意命名你的 Supervisor。每个 Supervisor 负责「监督」一组 worker 进程，并负责平衡队列之间的 worker 进程。

如果你想定义一组在指定环境中运行的新 worker 进程，可以向相应的环境添加额外的 Supervisor。如果你想为应用程序使用的特定队列定义不同的平衡策略或 worker 数量，也可以选择这样做。

#### 维护模式

当你的应用进入维护模式[维护模式](https://learnku.com/docs/laravel/11.x/configurationmd#maintenance-mode)，除非 Horizon 配置文件中对应 supervisor 的 `force` 配置项被定义为 `true`， 否则 Horizion 将无法执行队列任务：

```php
    'environments' => [
        'production' => [
            'supervisor-1' => [
                // ...
                'force' => true,
            ],
        ],
    ],
```

#### 默认值

在 Horizon 的默认配置文件中，你会注意到一个 `defaults` 配置选项。  
这个配置选项指定应用程序的 [supervisors](#supervisors) 的默认值。Supervisor 的默认配置值将合并到每个环境的 Supervisor 配置中，让你在定义 Supervisor 时避免不必要的重复工作。

### 均衡策略

与 Laravel 的默认队列系统不同，Horizon 允许你从三个平衡策略中进行选择：`simple`， `auto`， 和 `false`。`simple` 策略会在进程之间均匀分配进入的任务：

配置文件默认的 `auto` 策略根据队列的当前工作负载来调整每个队列的工作进程数量。例如，如果你的 `notifications` 队列有 1000 个等待种的任务，而你的 `render` 队列是空的，那么 Horizon 将为 `notifications` 队列分配更多的 Worker 进程，直到队列为空。

当使用 `auto` 策略时，你可以定义 `minProcesses` 和 `maxProcesses` 的配置选项来控制 Horizon 的 Woker 进程最小和最大数量：

```php
    'environments' => [
        'production' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'auto',
                'autoScalingStrategy' => 'time',
                'minProcesses' => 1,
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
            ],
        ],
    ],
```

`autoScalingStrategy` 配置值决定了 Horizon 分配更多 Worker 进程到队列是基于清空队列的总时长（`time` 策略）还是队列中任务的总数量（`size` 策略）。

`balanceMaxShift` 和 `balanceCooldown` 配置项可以决定 Horizon 将以多快的速度扩展进程。在上面的示例中，每秒钟最多创建一个新的进程，或每 3 秒钟最多销毁一个进程，你可以根据应用程序的需要随意调整这些值。

当 `balance` 选项设置为 `false` 时，将使用默认的 Laravel 行为，它按照队列在配置中列出的顺序处理队列。

### 控制面板授权

Horizon 在 `/horizon` 上显示了一个控制面板。默认情况下，你只能在 `local` 环境中访问这个面板。在你的 `app/Providers/HorizonServiceProvider.php` 文件中，有一个 [授权拦截器（Gates）](https://learnku.com/docs/laravel/11.x/authorizationmd#gates) 的方法定义。该拦截器用于控制在**非本地**环境中对 Horizon 的访问。你可以根据需要修改此方法，来限制对 Horizon 的访问：

```php
    /**
     * 注册 Horizon 守门逻辑。
     *
     * 该守门逻辑决定谁可以在非本地环境下访问 Horizon。
     */
    protected function gate(): void
    {
        Gate::define('viewHorizon', function (User $user) {
            return in_array($user->email, [
                'taylor@laravel.com',
            ]);
        });
    }
```

#### 可替代的身份验证策略

需要留意的是，Laravel 会自动将经过认证的用户注入到拦截器（Gate）闭包中。如果你的应用程序通过其他方法提供 Horizon 安全性保障，例如 IP 限制，那么你访问 Horizon 用户可能不需要实现这个「登录」动作。因此，你需要将上面的 `function (User $user)` 更改为 `function (User $user = null)` 以强制 Laravel 不必进行身份验证。

### 静默任务

有时，你可能对查看某些由你的应用程序或第三方软件包发出的工作不感兴趣。与其让这些任务在你的「已完成的任务」列表中占用空间，你可以让它们静默。要开始的话，在你的应用程序的 `horizon` 配置文件中的 `silenced` 配置选项中添加任务的类名：

```php
    'silenced' => [
        App\Jobs\ProcessPodcast::class,
    ],
```

或者，你希望静默的任务可以实现 `Laravel\Horizon\Contracts\Silenced` 接口。如果一个任务实现了这个接口，即使它不在 `silenced` 配置数组中，它也将自动被静默：

```php
    use Laravel\Horizon\Contracts\Silenced;

    class ProcessPodcast implements ShouldQueue, Silenced
    {
        use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

        // ...
    }
```

## 升级 Horizon

当你升级到 Horizon 的一个新的主要版本时，你需要仔细阅读[升级指南](https://github.com/laravel/horizon/blob/master/UPGRADE.md)。

## 运行 Horizon

当在你的应用的 `config/horizon.php` 配置文件中配置了你的 supervisors 和 workers 之后，你可以使用 Artisan 命令 `horizon` 启动 Horizon。只需这一个命令你就可以启动为当前环境配置的所有 workers 进程：

你可以使用 Artisan 命令 `horizon:pause` 和 `horizon:continue` 来暂停和指示任务继续运行：

```shell
php artisan horizon:pause

php artisan horizon:continue
```

你还可以使用 `horizon:pause-supervisor` 和 `horizon:continue-supervisor` Artisan 命令暂停和继续指定的 Horizon [supervisors](#supervisors)：

```shell
php artisan horizon:pause-supervisor supervisor-1

php artisan horizon:continue-supervisor supervisor-1
```

你可以使用 Artisan 命令 `horizon:status` 检查 Horizon 进程的当前状态：

```shell
php artisan horizon:status
```

你可以使用 Artisan 命令 `horizon:terminate` 优雅地终止机器上的主 Horizon 进程。Horizon 会等当前正在处理的所有任务都完成后退出：

```shell
php artisan horizon:terminate
```

### 部署 Horizon

如果要将 Horizon 部署到一个正在运行的服务器上，应该配置一个进程监视器来监视 `php artisan horizon` 命令，并在它意外退出时重新启动它。别担心，我们接下来将讨论如何安装进程监视器。

在将新代码部署到服务器时，你需要指示 Horizon 终止主进程，以便进程监视器重新启动它并接收代码的更改：

```shell
php artisan horizon:terminate
```

#### 安装 Supervisor

Supervisor 是一个用于 Linux 操作系统的进程监视器，如果 `Horizon` 进程被退出或终止，Supervisor 将自动重启你的 `Horizon` 进程。如果要在 Ubuntu 上安装 Supervisor，你可以使用以下命令。如果你不使用 Ubuntu，也可以使用操作系统的包管理器安装 Supervisor：

```shell
sudo apt-get install supervisor
```

> **技巧**  
> 如果自己配置 Supervisor 听起来很麻烦，可以考虑使用 [Laravel Forge](https://forge.laravel.com/)，它将自动为你的 Laravel 项目安装和配置 Supervisor。

#### Supervisor 配置

Supervisor 配置文件通常存储在 `/etc/supervisor/conf.d` 目录下。在此目录中，你可以创建任意数量的配置文件，这些配置文件会告诉 supervisor 如何监视你的进程。例如，让我们创建一个 `horizon.conf` 文件，它启动并监视一个 `horizon` 进程：

```ini
[program:horizon]
process_name=%(program_name)s
command=php /home/forge/example.com/artisan horizon
autostart=true
autorestart=true
user=forge
redirect_stderr=true
stdout_logfile=/home/forge/example.com/horizon.log
stopwaitsecs=3600
```

在定义 Supervisor 配置时，你应该确保 `stopwaitsecs` 的值大于最长运行任务所消耗的秒数。否则，Supervisor 可能会在任务处理完之前就将其杀死。

> **注意**  
> 虽然上面的例子对基于 Ubuntu 的服务器有效，但其他服务器操作系统对 Supervisor 配置文件的位置和文件扩展名可能有所不同。请查阅你的服务器的文档以了解更多信息。

#### 启动 Supervisor

创建了配置文件后，可以使用以下命令更新 Supervisor 配置并启动进程：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start horizon
```

> **技巧**  
> 关于 Supervisor 的更多信息，可以查阅 [Supervisor 文档](http://supervisord.org/index.html)。

## 标签

Horizon 允许你将「tags」分配给任务，包括邮件、事件广播、事件、通知和排队的事件监听器。实际上，Horizon 会智能且自动地标记大多数依赖 Eloquent 模型的任务。例如，看看下面的任务：

```php
    <?php

    namespace App\Jobs;

    use App\Models\Video;
    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Bus\Dispatchable;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Queue\SerializesModels;

    class RenderVideo implements ShouldQueue
    {
        use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

        /**
         * 创建一个新的任务实例。
         */
        public function __construct(
            public Video $video,
        ) {}

        /**
         * 执行任务。
         */
        public function handle(): void
        {
            // ...
        }
    }
```

如果此任务与 `App\Models\Video` 实例一起排队，且该实例的 `id` 为 `1`，则该任务将自动接收 `App\Models\Video:1` 标记。这是因为 Horizon 将为任何有 Eloquent 的模型检查任务的属性。如果找到了有 Eloquent 的模型，Horizon 将智能地使用模型的类名和主键标记任务：

```php
    use App\Jobs\RenderVideo;
    use App\Models\Video;

    $video = Video::find(1);

    RenderVideo::dispatch($video);
```

#### 手动标记任务

如果你想手动定义你的一个队列对象的标签，你可以在类上定义一个 `tags` 方法：

```php
    class RenderVideo implements ShouldQueue
    {
        /**
         * 获取应该分配给任务的标签。
         *
         * @return array<int, string>
         */
        public function tags(): array
        {
            return ['render', 'video:'.$this->video->id];
        }
    }
```

#### 手动标记事件监听器

当为时间监听器检索标签时，Horizon 将自动传递事件实例给 `tags` 方法，并允许你添加事件数据到标签：

```php
    class SendRenderNotifications implements ShouldQueue
    {
        /**
         * 获取要分配给事件监听器的标签。
         *
         * @return array<int, string>
         */
        public function tags(VideoRendered $event): array
        {
            return ['video:'.$event->video->id];
        }
    }
```

## 通知

> **注意**  
> 注意： 当配置 Horizon 发送 Slack 或 SMS 通知时，你应该查看[相关通知驱动程序的先决条件](https://learnku.com/docs/laravel/11.x/notificationsmd)。

如果你希望在一个队列有较长的等待时间时得到通知，你可以使用 `Horizon::routeMailNotificationsTo`, `Horizon::routeSlackNotificationsTo`, 和 `Horizon::routeSmsNotificationsTo` 方法。你可以通过应用程序的 `App\Providers\HorizonServiceProvider` 中的 `boot` 方法来调用这些方法：

```php
    /**
     * 引导应用的一些服务。
     */
    public function boot(): void
    {
        parent::boot();

        Horizon::routeSmsNotificationsTo('15556667777');
        Horizon::routeMailNotificationsTo('example@example.com');
        Horizon::routeSlackNotificationsTo('slack-webhook-url', '#channel');
    }
```

#### 配置通知等待时间阈值

你可以在 `config/horizon.php` 的配置文件中配置多少秒算是「长等待」。你可以用该文件中的 `waits` 配置选项控制每个 连接 / 队列 组合的长等待阈值。任何未定义 连接 / 队列 组合将默认 60 秒长等待阈值：

```php
    'waits' => [
        'redis:critical' => 30,
        'redis:default' => 60,
        'redis:batch' => 120,
    ],
```

## 指标

Horizon 有一个指标控制面板，它提供了任务和队列的等待时间和吞吐量等信息。要让这些信息显示在这个控制面板上，你应该在 `routes/console.php` 文件配置应用程序每五分钟运行一次 Horizon 的 Artisan 命令 `snapshot`：

```php
    use Illuminate\Support\Facades\Schedule;

    Schedule::command('horizon:snapshot')->everyFiveMinutes();
```

## 删除失败的任务

如果你想删除失败的任务，可以使用 `horizon:forget` 命令。 `horizon:forget` 命令接受失败任务的 ID 或 UUID 作为其唯一参数：

```shell
php artisan horizon:forget 5
```

## 从队列中清除任务

如果你想从应用程序的默认队列中删除所有任务，你可以使用 Artisan 命令 `horizon:clear` 执行此操作：

```shell
php artisan horizon:clear
```

你可以设置 `queue` 选项来从特定队列中删除任务：

```shell
php artisan horizon:clear --queue=emails
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/ho...](https://learnku.com/docs/laravel/11.x/horizonmd/16721)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/ho...](https://learnku.com/docs/laravel/11.x/horizonmd/16721)