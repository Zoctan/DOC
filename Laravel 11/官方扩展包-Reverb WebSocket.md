本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Reverb

+   [简介](#introduction)
+   [安装](#installation)
+   [配置](#configuration)
    +   [应用凭据](#application-credentials)
    +   [允许的来源](#allowed-origins)
    +   [多应用](#additional-applications)
    +   [SSL](#ssl)
+   [运行服务](#running-server)
    +   [调试](#debugging)
    +   [重启](#restarting)
+   [监控](#monitoring)
+   [生产环境下运行服务](#production)
    +   [打开的文件](#open-files)
    +   [事件循环](#event-loop)
    +   [Web 服务](#web-server)
    +   [端口](#ports)
    +   [进程管理](#process-management)
    +   [扩展](#scaling)

## 简介

[Laravel Reverb](https://github.com/laravel/reverb) 为你的 Laravel 应用带来了快速的、可扩展的实时 WebSocket 通信，并提供与 Laravel 现有[事件广播工具套件](https://learnku.com/docs/laravel/11.x/broadcasting)的无缝集成。

## 安装

你可以使用 `install:broadcasting` Artisan 命令安装 Reverb：

```php
php artisan install:broadcasting
```

## 配置

在底层，`install:broadcasting` Artisan 命令会运行 `reverb:install` 命令，它会使用一系列合理的默认配置选项来安装 Reverb。如果你想要更改任何配置选项，可以通过更新 Reverb 的环境变量或者更新 `config/reverb.php` 配置文件来进行。

### 应用凭据

为了建立与 Reverb 的连接，必须在客户端和服务器之间交换一组 Reverb 「应用程序」凭据。这些凭据在服务器上配置，用于验证来自客户端的请求。你可以使用以下环境变量定义这些凭据：

```ini
REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
```

### 允许的来源

您还可以通过更新 `config/reverb.php` 配置文件的 `apps` 部分中的 `allowed_origins` 配置值的值来定义客户端请求的来源。任何未列在允许来源中的请求都会被拒绝。 您可以使用 `*` 来允许所有来源:

```php
'apps' => [
    [
        'id' => 'my-app-id',
        'allowed_origins' => ['laravel.com'],
        // ...
    ]
]
```

### 多应用

通常情况下，Reverb 为安装它的应用提供 WebSocket 服务。然而，也有可能通过单个 Reverb 安装来为多个应用提供服务。

例如，您可能希望维护一个单一的 Laravel 应用，通过 Reverb 为多个应用提供 WebSocket 连接。这可以通过在应用的 `config/reverb.php` 配置文件中定义多个 apps 来实现：

```php
'apps' => [
    [
        'app_id' => 'my-app-one',
        // ...
    ],
    [
        'app_id' => 'my-app-two',
        // ...
    ],
],
```

### SSL

在大多数情况下，在将请求代理到 Reverb 服务器之前，安全的 WebSocket 连接由上游 Web 服务器（Nginx 等）处理。

但是在本地开发，直接使用 Reverb 服务处理连接是比较有用的。 如果您正在使用 [Laravel Herd’s](https://herd.laravel.com/) 安全站点亦或是 [Laravel Valet](https://learnku.com/docs/laravel/11.x/valet) 并且您的应用已经运行 [secure command](https://learnku.com/docs/laravel/11.x/valet#securing-sites) 命令， 您可以使用为您的站点生成的 Herd / Valet 证书来保护您的 Reverb 连接。 为此，设置 `REVERB_HOST` 环境变量为您站点的主机名或者在启动 Reverb 服务器时明确传递主机名选项：:

```sh
php artisan reverb:start --host="0.0.0.0" --port=8080 --hostname="laravel.test"
```

因为 Herd 和 Valet 域指向 `localhost`，执行上述命令可以使你的 Reverb 服务可以通过安全的 WebSocket 协议 (`wss`) 在 `wss://laravel.test:8080` 上进行访问。

你还可以通过在应用的 `config/reverb.php` 配置文件中定义 `tls` 选项来手动选择证书。 在 `tls` 选项数组中，你可以提供 [PHP 的 SSL 上下文选项](https://www.php.net/manual/en/context.ssl.php) 支持的任意选项:

```php
'options' => [
    'tls' => [
        'local_cert' => '/path/to/cert.pem'
    ],
],
```

## 运行服务器

Reverb 服务可以使用 `reverb:start` Artisan 命令来启动：

默认情况下，Reverb 服务会在 `0.0.0.0:8080` 启动，使其能够从所有网络接口进行访问。

如果你需要指定自定义主机或端口，你可以在启动服务时通过 `--host` 和 `--port` 选项来这么做：

```sh
php artisan reverb:start --host=127.0.0.1 --port=9000
```

或者，你可以在应用的 `.env` 配置文件中定义 `REVERB_SERVER_HOST` 和 `REVERB_SERVER_PORT` 环境变量。

不要把 `REVERB_SERVER_HOST` 和 `REVERB_SERVER_PORT` 环境变量与 `REVERB_HOST` 和 `REVERB_PORT` 弄混了。前者指定运行 Reverb 服务本身的主机和端口，而后一对则指示 Laravel 在何处发送广播消息。例如，在生产环境中，你可以将请求从端口 `443` 上的公共 Reverb 主机名路由到在 `0.0.0.0:8080` 上运行的 Reverb 服务器。在此方案中，环境变量的定义如下：

```ini
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080

REVERB_HOST=ws.laravel.com
REVERB_PORT=443
```

### 调试

为了提升性能，Reverb 默认不会输出任何调试信息。如果你想要看到通过 Reverb 服务传播的数据流，你可以在 `reverb:start` 命令中提供 `--debug` 选项：

```sh
php artisan reverb:start --debug
```

### 重启

由于 Reverb 是一个长期运行的过程，不重启服务器的话，你的代码变更将不会生效。可以通过 `reverb:restart` Artisan 命令来重启服务。

`reverb:restart` 命令确保在停止服务器之前所有连接都优雅地终止。如果你使用如 Supervisor 这样的进程管理器运行 Reverb，服务器将在所有连接被终止后由进程管理器自动重启：

```sh
php artisan reverb:restart
```

## 监控

Reverb 的监控可以集成在 [Laravel Pulse](https://learnku.com/docs/laravel/11.x/pulse)。 启用 Reverb 的 Pulse 集成后，你可以追踪服务器处理的连接数和消息数。

为了启用集成，你首先应该确保你已经 [安装了 Pulse](https://learnku.com/docs/laravel/11.x/pulse#installation)。 然后，将 Reverb 的任何记录器添加到应用的 `config/pulse.php` 配置文件中：

```php
use Laravel\Reverb\Pulse\Recorders\ReverbConnections;
use Laravel\Reverb\Pulse\Recorders\ReverbMessages;

'recorders' => [
    ReverbConnections::class => [
        'sample_rate' => 1,
    ],

    ReverbMessages::class => [
        'sample_rate' => 1,
    ],

    ...
],
```

接下来，将每个记录器的 Pulse 卡片添加到你的 [Pulse 面板](https://learnku.com/docs/laravel/11.x/pulse#dashboard-customization):

```blade
<x-pulse>
    <livewire:reverb.connections cols="full" />
    <livewire:reverb.messages cols="full" />
    ...
</x-pulse>
```

## 在生产环境中运行 Reverb

由于 WebSocket 服务器的长期运行特性，您可能需要对服务器和托管环境进行一些优化，以确保 Reverb 服务器能够有效地处理服务器上可用资源的最佳连接数。

> \[! 注意\]  
> 如果您的网站由 [Laravel Forge](https://forge.laravel.com/) 管理，您可以直接从” 应用程序” 面板自动优化您的服务器以进行混响。通过启用混响集成，Forge 将确保您的服务器已准备好生产，包括安装任何所需的扩展和增加允许的连接数量。

### 打开的文件

每个 WebSocket 连接都保存在内存中，直到客户端或服务器断开连接。在 Unix 和类 Unix 环境中，每个连接都由一个文件表示。但是，在操作系统和应用程序级别，通常对允许打开的文件数有限制。

#### 操作系统

在基于 Unix 的操作系统上，您可以使用以下 `ulimit` 命令确定允许的打开文件数：

此命令将显示允许不同用户打开的文件限制。您可以通过编辑 `/etc/security/limits.conf` 文件来更新这些值。例如，将 `forge` 用户打开的文件的最大数更新为 10,000 个，如下所示：

```ini
# /etc/security/limits.conf
forge        soft  nofile  10000
forge        hard  nofile  10000
```

### 事件循环

Reverb 在底层使用了 ReactPHP 事件循环来管理服务器上的 WebSocket 连接。默认情况下，这个事件循环由 `stream_select` 驱动，它不需要任何额外的扩展。然而，`stream_select` 通常限制为最多 1024 个打开的文件。因此，如果您计划处理超过 1000 个并发连接，您将需要使用其他不受同样限制的事件循环。

Reverb 将在可用时自动切换到 `ext-uv` 有源循环。此 PHP 扩展可通过 PECL 安装：

### Web 服务

在大多数情况下，Reverb 在服务器上不面向 Web 的端口上运行。因此，为了将流量路由到 Reverb，您应该配置反向代理。假设 Reverb 在主机 `0.0.0.0` 和端口 `8080` 上运行，并且您的服务器使用 Nginx Web 服务器，则可以使用以下 Nginx 站点配置为您的 Reverb 服务器定义反向代理：

```nginx
server {
    ...

    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header Scheme $scheme;
        proxy_set_header SERVER_PORT $server_port;
        proxy_set_header REMOTE_ADDR $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";

        proxy_pass http://0.0.0.0:8080;
    }

    ...
}
```

通常，Web 服务器配置为限制允许的连接数，以防止服务器过载。要将 Nginx Web 服务器上允许的连接数增加到 10,000 个，应更新 `nginx.conf` 文件 `worker_rlimit_nofile` 的 和 `worker_connections` 值：

```nginx
user forge;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;
worker_rlimit_nofile 10000;

events {
  worker_connections 10000;
  multi_accept on;
}
```

上述配置将允许每个进程生成多达 10,000 个 Nginx 工作线程。此外，此配置将 Nginx 的打开文件限制设置为 10,000。

### 端口

基于 Unix 的操作系统通常会限制服务器上可以打开的端口数。您可以通过以下命令查看当前允许的范围：

```sh
 cat /proc/sys/net/ipv4/ip_local_port_range
# 32768    60999
```

上面的输出显示服务器最多可以处理 28,231 （60,999 - 32,768） 个连接，因为每个连接都需要一个空闲端口。尽管我们建议使用 [水平扩展](#scaling) 来增加允许的连接数，但您可以通过更新服务器 `/etc/sysctl.conf` 配置文件中允许的端口范围来增加可用开放端口的数量。

### 进程管理

在大多数情况下，您应该使用进程管理器（如 Supervisor）来确保 Reverb 服务器持续运行。如果使用 Supervisor 运行 Reverb， `supervisor.conf` 则应更新服务器文件 `minfds` 的设置，以确保 Supervisor 能够打开处理与 Reverb 服务器连接所需的文件：

```ini
[supervisord]
...
minfds=10000
```

### 扩展

如果需要处理的连接数超过单个服务器允许的连接数，则可以水平扩展 Reverb 服务。利用 Redis 的发布 / 订阅功能，Reverb 能够管理跨多个服务的连接。当应用程序的某个 Reverb 服务收到消息时，该服务将使用 Redis 将传入消息发布到所有其他服务器。

若要启用水平扩展，应在应用程序的 `.env` 配置文件中将 `REVERB_SCALING_ENABLED` 环境变量设置为 `true` ：

```env
REVERB_SCALING_ENABLED=true
```

接下来，您应该有一个专门的 Redis 服务器，与所有的 Reverb 服务通信。Reverb 将使用[应用程序配置的默认 Redis 连接](https://learnku.com/docs/laravel/11.x/redis#configuration)将消息发布到所有 Reverb 服务器。

一旦您启用 Reverb 的拓展选项并配置 Redis 服务器后，只需在能够与 Redis 服务器通信的多个服务器上调用该 `reverb:start` 命令即可。这些 Reverb 服务应放置在负载均衡器后面，该负载均衡器在服务之间均匀分配传入请求。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/re...](https://learnku.com/docs/laravel/11.x/reverbmd/16730)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/re...](https://learnku.com/docs/laravel/11.x/reverbmd/16730)