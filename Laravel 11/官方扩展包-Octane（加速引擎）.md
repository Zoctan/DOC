本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Octane

+   [介绍](#introduction)
+   [安装](#installation)
+   [服务器先决条件](#server-prerequisites)
    +   [FrankenPHP](#frankenphp)
    +   [RoadRunner](#roadrunner)
    +   [Swoole](#swoole)
+   [为你的应用提供服务](#serving-your-application)
    +   [通过 HTTPS 为你的应用提供服务](#serving-your-application-via-https)
    +   [通过 Nginx 为你的应用提供服务](#serving-your-application-via-nginx)
    +   [监视文件更改](#watching-for-file-changes)
    +   [指定工作进程数](#specifying-the-worker-count)
    +   [指定最大请求次数](#specifying-the-max-request-count)
    +   [重新加载工作进程](#reloading-the-workers)
    +   [停止服务器](#stopping-the-server)
+   [依赖注入和 Octane](#dependency-injection-and-octane)
    +   [容器注入](#container-injection)
    +   [请求注入](#request-injection)
    +   [配置存储库注入](#configuration-repository-injection)
+   [管理内存泄漏](#managing-memory-leaks)
+   [并发任务](#concurrent-tasks)
+   [时钟周期和间隔](#ticks-and-intervals)
+   [Octane 缓存](#the-octane-cache)
+   [Tables](#tables)

## 介绍

[Laravel Octane](https://github.com/laravel/octane) 通过使用高性能的应用服务器（包括 [FrankenPHP](https://frankenphp.dev/)、[Open Swoole](https://openswoole.com/)、[Swoole](https://github.com/swoole/swoole-src) 和 [RoadRunner](https://roadrunner.dev/)）来提升你的应用性能。Octane 会在内存中启动你的应用程序，并以超音速速度处理请求。

## 安装

可以通过 Composer 包管理器安装 Octane：

```shell
composer require laravel/octane
```

安装 Octane 后，可以执行 `octane:install` Artisan 命令，该命令会将 Octane 的配置文件安装到你的应用程序中：

```shell
php artisan octane:install
```

## 服务器先决条件

> **警告**  
> Laravel Octane 需要 [PHP 8.1+](https://php.net/releases/)。

### FrankenPHP

[FrankenPHP](https://frankenphp.dev/) 是一个用 Go 编写的 PHP 应用服务器，支持现代 Web 功能，如提前提示、Brotli 和 Zstandard 压缩。当你安装 Octane 并选择 FrankenPHP 作为你的服务器时，Octane 将自动为你下载并安装 FrankenPHP 二进制文件。

#### 通过 Laravel Sail 使用 FrankenPHP

如果你计划使用 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sail) 开发你的应用程序，你应该运行以下命令来安装 Octane 和 FrankenPHP：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane
```

接下来，你应该使用 `octane:install` Artisan 命令来安装 FrankenPHP 二进制文件：

```shell
./vendor/bin/sail artisan octane:install --server=frankenphp
```

最后，在你的应用程序的 `docker-compose.yml` 文件中的 `laravel.test` 服务定义中添加一个 `SUPERVISOR_PHP_COMMAND` 环境变量。这个环境变量将包含 Sail 用来使用 Octane 为你的应用提供服务的命令，而不是使用 PHP 开发服务器：

```ini
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=frankenphp --host=0.0.0.0 --admin-port=2019 --port=80" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

要启用 HTTPS、HTTP/2 和 HTTP/3，应用以下修改：

```ini
services:
  laravel.test:
    ports:
        - '${APP_PORT:-80}:80'
        - '${VITE_PORT:-5173}:${VITE_PORT:-5173}'
        - '443:443' # [tl! add]
        - '443:443/udp' # [tl! add]
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --host=localhost --port=443 --admin-port=2019 --https" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

通常情况下，你应该通过 `https://localhost` 访问你的 FrankenPHP Sail 应用程序，因为使用 `https://127.0.0.1` 需要额外配置，并且是[不推荐的](https://frankenphp.dev/docs/known-issues/#using-https127001-with-docker)。

#### 通过 Docker 使用 FrankenPHP

使用 FrankenPHP 的官方 Docker 镜像可以提供更好的性能，并使用附带的额外扩展，这些扩展在静态安装的 FrankenPHP 中不包含。此外，官方 Docker 镜像支持在它本身不支持的平台上运行 FrankenPHP，比如 Windows。FrankenPHP 的官方 Docker 镜像适用于本地开发和生产用途。

你可以使用以下 Dockerfile 作为容器化你的 FrankenPHP 驱动的 Laravel 应用程序的起点：

```dockerfile
FROM dunglas/frankenphp

RUN install-php-extensions \
    pcntl
    # 在这里添加其他 PHP 扩展...

COPY . /app

ENTRYPOINT ["php", "artisan", "octane:frankenphp"]
```

然后，在开发过程中，你可以使用以下 Docker Compose 文件来运行你的应用程序：

```ini
# compose.yaml
services:
  frankenphp:
    build:
      context: .
    entrypoint: php artisan octane:frankenphp --max-requests=1
    ports:
      - "8000:8000"
    volumes:
      - .:/app
```

你可以查阅[官方 FrankenPHP 文档](https://frankenphp.dev/docs/docker/)，了解更多关于使用 Docker 运行 FrankenPHP 的信息。

### RoadRunner

[RoadRunner](https://roadrunner.dev/) 使用由 Go 构建的 RoadRunner 二进制文件。第一次启动基于 RoadRunner 的 Octane 服务器时，Octane 会提供下载和安装 RoadRunner 二进制文件的选项。

#### 通过 Laravel Sail 使用 RoadRunner

如果你计划使用 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sail) 开发你的应用程序，你应该运行以下命令来安装 Octane 和 RoadRunner：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane spiral/roadrunner-cli spiral/roadrunner-http 
```

接下来，你应该启动一个 Sail shell，并使用 `rr` 可执行文件来获取最新的基于 Linux 构建的 RoadRunner 二进制文件：

```shell
./vendor/bin/sail shell

# 在 Sail shell 中...
./vendor/bin/rr get-binary
```

然后，在你的应用程序的 `docker-compose.yml` 文件中的 `laravel.test` 服务定义中添加一个 `SUPERVISOR_PHP_COMMAND` 环境变量。这个环境变量将包含 Sail 用来使用 Octane 为你的应用提供服务的命令，而不是使用 PHP 开发服务器：

```ini
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=roadrunner --host=0.0.0.0 --rpc-port=6001 --port=80" # [tl! add]
```

最后，确保 `rr` 可执行并构建你的 Sail 镜像：

```shell
chmod +x ./rr

./vendor/bin/sail build --no-cache
```

### Swoole

如果你计划使用 Swoole 应用服务器为你的 Laravel Octane 应用提供服务，你必须安装 Swoole PHP 扩展。通常情况下，可以通过 PECL 完成：

#### Open Swoole

如果你想使用 Open Swoole 应用服务器为你的 Laravel Octane 应用提供服务，你必须安装 Open Swoole PHP 扩展。通常情况下，可以通过 PECL 完成：

使用 Laravel Octane 与 Open Swoole 提供了与 Swoole 相同的功能，如并发任务、时钟周期和间隔。

#### 通过 Laravel Sail 使用 Swoole

> **警告**  
> 在通过 Sail 提供 Octane 应用服务之前，请确保你有最新版本的 Laravel Sail，并在应用程序的根目录中执行 `./vendor/bin/sail build --no-cache`。

或者，你可以使用 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sail) 来开发基于 Swoole 的 Octane 应用程序，这是 Laravel 的官方基于 Docker 的开发环境。Laravel Sail 默认包含 Swoole 扩展。然而，你仍需要调整 Sail 使用的 `docker-compose.yml` 文件。

要开始，向你应用程序的 `docker-compose.yml` 文件中的 `laravel.test` 服务定义中添加一个 `SUPERVISOR_PHP_COMMAND` 环境变量。这个环境变量将包含 Sail 用来使用 Octane 为你的应用提供服务的命令，而不是使用 PHP 开发服务器：

```ini
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=swoole --host=0.0.0.0 --port=80" # [tl! add]
```

最后，构建你的 Sail 镜像：

```shell
./vendor/bin/sail build --no-cache
```

#### Swoole 配置

Swoole 支持一些额外的配置选项，如果需要，你可以将它们添加到你的 `octane` 配置文件中。由于它们很少需要被修改，这些选项没有包含在默认配置文件中：

```php
'swoole' => [
    'options' => [
        'log_file' => storage_path('logs/swoole_http.log'),
        'package_max_length' => 10 * 1024 * 1024,
    ],
],
```

## 为你的应用提供服务

Octane 服务器可以通过 `octane:start` Artisan 命令启动。默认情况下，此命令将使用你应用程序的 `octane` 配置文件中指定的服务器：

默认情况下，Octane 将在端口 8000 上启动服务器，所以你可以通过 `http://localhost:8000` 在 Web 浏览器中访问你的应用程序。

### 通过 HTTPS 提供你的应用程序

默认情况下，通过 Octane 运行的应用程序生成以 `http://` 为前缀的链接。在通过 HTTPS 提供你的应用程序时，可以在你应用程序的 `config/octane.php` 配置文件中使用 `OCTANE_HTTPS` 环境变量，设置为 `true`。当这个配置值设置为 `true` 时，Octane 将指示 Laravel 使用 `https://` 为所有生成的链接添加前缀：

```php
'https' => env('OCTANE_HTTPS', false),
```

### 通过 Nginx 为你的应用提供服务

> **注意**  
> 如果你还没有准备好管理自己的服务器配置，或者不熟悉配置运行强大的 Laravel Octane 应用所需的各种服务，请查看 [Laravel Forge](https://forge.laravel.com/)。

在生产环境中，你应该将你的 Octane 应用程序放在传统的 Web 服务器（如 Nginx 或 Apache）后面。这样做将允许 Web 服务器提供你的静态资源，例如图片和样式表，以及管理你的 SSL 证书终止。

在下面的 Nginx 配置示例中，Nginx 将提供站点的静态资源并代理请求到运行在端口 8000 上的 Octane 服务器：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    listen [::]:80;
    server_name domain.com;
    server_tokens off;
    root /home/forge/domain.com/public;

    index index.php;

    charset utf-8;

    location /index.php {
        try_files /not_exists @octane;
    }

    location / {
        try_files $uri $uri/ @octane;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/domain.com-error.log error;

    error_page 404 /index.php;

    location @octane {
        set $suffix "";

        if ($uri = /index.php) {
            set $suffix ?$query_string;
        }

        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header Scheme $scheme;
        proxy_set_header SERVER_PORT $server_port;
        proxy_set_header REMOTE_ADDR $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_pass http://127.0.0.1:8000$suffix;
    }
}
```

### 监视文件更改

由于在 Octane 服务器启动时一次性加载你的应用程序到内存中，所以当你刷新浏览器时，对应用程序文件的任何更改都不会反映出来。例如，添加到你的 `routes/web.php` 文件的路由定义在服务器重新启动之前不会反映出来。为了方便起见，你可以使用 `--watch` 标志指示 Octane 在应用程序内部的任何文件更改时自动重新启动服务器：

```shell
php artisan octane:start --watch
```

在使用此功能之前，你应该确保 [Node](https://nodejs.org/) 已安装在你的本地开发环境中。此外，你应该在你的项目中安装 [Chokidar](https://github.com/paulmillr/chokidar) 文件监视库：

```shell
npm install --save-dev chokidar
```

你可以使用应用程序的 `config/octane.php` 配置文件中的 `watch` 配置选项来配置应该监视的目录和文件。

### 指定工作进程数量

默认情况下，Octane 将为你的机器提供的每个 CPU 核心启动一个应用程序请求工作进程。然后，这些工作进程将用于在进入应用程序时为传入的 HTTP 请求提供服务。当调用 `octane:start` 命令时，你可以手动指定要启动多少个工作进程，使用 `--workers` 选项：

```shell
php artisan octane:start --workers=4
```

如果你正在使用 Swoole 应用服务器，还可以指定你希望启动多少个 [任务工作进程](#concurrent-tasks)：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

### 指定最大请求计数

为了帮助防止内存泄漏，Octane 在处理 500 个请求后会优雅地重新启动任何工作进程。要调整此数字，你可以使用 `--max-requests` 选项：

```shell
php artisan octane:start --max-requests=250
```

### 重新加载工作进程

你可以使用 `octane:reload` 命令优雅地重新启动 Octane 服务器的应用程序工作进程。通常，在部署后应该执行此操作，以便你的新部署代码被加载到内存中，并用于为后续请求提供服务：

```shell
php artisan octane:reload
```

### 停止服务器

你可以使用 `octane:stop` Artisan 命令停止 Octane 服务器：

#### 检查服务器状态

你可以使用 `octane:status` Artisan 命令检查 Octane 服务器的当前状态：

```shell
php artisan octane:status
```

## 依赖注入与 Octane

由于 Octane 一次启动你的应用程序并将其保存在内存中，同时为请求提供服务，因此在构建应用程序时应考虑一些注意事项。例如，当请求工作进程初始启动时，你的应用程序服务提供者的 `register` 和 `boot` 方法只会执行一次。在后续请求中，将重用相同的应用程序实例。

因此，在将应用程序服务容器或请求注入到任何对象的构造函数时，你应特别小心。这样做可能会导致在后续请求中，该对象使用的容器或请求版本过时。

Octane 将自动处理在请求之间重置任何第一方框架状态。但是，Octane 并不总是知道如何重置你的应用程序创建的全局状态。因此，你应该知道如何以符合 Octane 的方式构建应用程序。接下来，我们将讨论在使用 Octane 时可能会导致问题的最常见情况。

### 容器注入

一般来说，你应该避免将应用程序服务容器或 HTTP 请求实例注入到其他对象的构造函数中。例如，以下绑定将整个应用程序服务容器注入到一个被绑定为单例的对象中：

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

/**
 * 注册任何应用程序服务。
 */
public function register(): void
{
    $this->app->singleton(Service::class, function (Application $app) {
        return new Service($app);
    });
}
```

在这个例子中，如果在应用程序启动过程中解析 `Service` 实例，容器将被注入到服务中，并且该容器将在后续请求中由 `Service` 实例持有。对于你的特定应用程序，这**可能**不是问题；但是，它可能导致容器意外地缺少后续请求中添加的绑定或在启动周期后由后续请求添加的绑定。

作为解决方法，你可以停止将绑定注册为单例，或者你可以将一个容器解析器闭包注入到服务中，始终解析当前容器实例：

```php
use App\Service;
use Illuminate\Container\Container;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Service::class, function (Application $app) {
    return new Service($app);
});

$this->app->singleton(Service::class, function () {
    return new Service(fn () => Container::getInstance());
});
```

全局的 `app` 助手和 `Container::getInstance()` 方法将始终返回应用程序容器的最新版本。

### 请求注入

一般来说，你应该避免将应用程序服务容器或 HTTP 请求实例注入到其他对象的构造函数中。例如，以下绑定将整个请求实例注入到一个被绑定为单例的对象中：

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

/**
 * 注册任何应用程序服务。
 */
public function register(): void
{
    $this->app->singleton(Service::class, function (Application $app) {
        return new Service($app['request']);
    });
}
```

在这个例子中，如果在应用程序启动过程中解析 `Service` 实例，HTTP 请求将被注入到服务中，并且该请求将在后续请求中由 `Service` 实例持有。因此，所有标头、输入和查询字符串数据都将不正确，以及所有其他请求数据。

作为解决方法，你可以停止将绑定注册为单例，或者你可以将一个请求解析器闭包注入到服务中，始终解析当前请求实例。或者，最推荐的方法是在运行时将对象需要的特定请求信息传递给对象的某个方法：

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Service::class, function (Application $app) {
    return new Service($app['request']);
});

$this->app->singleton(Service::class, function (Application $app) {
    return new Service(fn () => $app['request']);
});

// 或者...

$service->method($request->input('name'));
```

全局的 `request` 助手将始终返回应用程序当前处理的请求，因此在应用程序中使用它是安全的。

> **警告**  
> 在控制器方法和路由闭包中对 `Illuminate\Http\Request` 实例进行类型提示是可以接受的。

### 配置存储库注入

一般来说，你应该避免将配置存储库实例注入到其他对象的构造函数中。例如，以下绑定将配置存储库注入到一个被绑定为单例的对象中：

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

/**
 * 注册任何应用程序服务。
 */
public function register(): void
{
    $this->app->singleton(Service::class, function (Application $app) {
        return new Service($app->make('config'));
    });
}
```

在这个例子中，如果配置数值在请求之间发生变化，那么该服务将无法访问新值，因为它依赖于原始存储库实例。

作为解决方法，你可以停止将绑定注册为单例，或者你可以向类注入一个配置存储库解析器闭包：

```php
use App\Service;
use Illuminate\Container\Container;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Service::class, function (Application $app) {
    return new Service($app->make('config'));
});

$this->app->singleton(Service::class, function () {
    return new Service(fn () => Container::getInstance()->make('config'));
});
```

全局的 `config` 将始终返回配置存储库的最新版本，因此在应用程序中使用它是安全的。

### 管理内存泄漏

请记住，Octane 在请求之间保持应用程序在内存中；因此，向静态维护的数组添加数据将导致内存泄漏。例如，以下控制器存在内存泄漏，因为对应用程序的每个请求将继续向静态的 `$data` 数组添加数据：

```php
use App\Service;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

/**
 * 处理传入请求。
 */
public function index(Request $request): array
{
    Service::$data[] = Str::random(10);

    return [
        // ...
    ];
}
```

在构建应用程序时，你应特别注意避免创建这类内存泄漏。建议在本地开发过程中监控应用程序的内存使用情况，以确保不会引入新的内存泄漏到应用程序中。

## 并发任务

> **警告**  
> 此功能需要 [Swoole](#swoole)。

在使用 Swoole 时，你可以通过轻量级的后台任务并发执行操作。你可以使用 Octane 的 `concurrently` 方法实现这一点。你可以将此方法与 PHP 数组解构结合使用，以检索每个操作的结果：

```php
use App\Models\User;
use App\Models\Server;
use Laravel\Octane\Facades\Octane;

[$users, $servers] = Octane::concurrently([
    fn () => User::all(),
    fn () => Server::all(),
]);
```

由 Octane 处理的并发任务利用了 Swoole 的 "task workers"，并在与传入请求完全不同的进程中执行。用于处理并发任务的工作进程数量由 `octane:start` 命令中的 `--task-workers` 指令确定：

```shell
php artisan octane:start --workers=4 --task-workers=6
```

在调用 `concurrently` 方法时，由于 Swoole 任务系统施加的限制，不应提供超过 1024 个任务。

## Ticks 和 Intervals

> **警告**  
> 此功能需要 [Swoole](#swoole)。

在使用 Swoole 时，你可以注册每隔指定秒数执行的 "tick" 操作。你可以通过 `tick` 方法注册 "tick" 回调。提供给 `tick` 方法的第一个参数应该是表示 tick 名称的字符串。第二个参数应该是在指定间隔调用的可调用函数。

在这个例子中，我们将注册一个闭包，每 10 秒调用一次。通常，`tick` 方法应该在应用程序的某个服务提供者的 `boot` 方法中调用：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
        ->seconds(10);
```

使用 `immediate` 方法，你可以指示 Octane 在 Octane 服务器初始启动时立即调用 tick 回调，并在此后每 N 秒调用一次：

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
        ->seconds(10)
        ->immediate();
```

## Octane 缓存

> **警告**  
> 此功能需要 [Swoole](#swoole)。

在使用 Swoole 时，你可以利用 Octane 缓存驱动程序，它提供每秒高达 200 万次的读写速度。因此，对于需要从缓存层获得极致读 / 写速度的应用程序来说，这个缓存驱动是一个绝佳选择。

这个缓存驱动由 [Swoole 表](https://www.swoole.co.uk/docs/modules/swoole-table) 提供支持。缓存中存储的所有数据对服务器上的所有工作进程都是可用的。然而，在服务器重新启动时，缓存数据将被清空：

```php
Cache::store('octane')->put('framework', 'Laravel', 30);
```

> **警告**  
> 可在应用程序的 `octane` 配置文件中定义 Octane 缓存允许的最大条目数。

### 缓存间隔

除了 Laravel 缓存系统提供的典型方法之外，Octane 缓存驱动还提供基于间隔的缓存。这些缓存会在指定的间隔自动刷新，应该在应用程序的某个服务提供者的 `boot` 方法中注册。例如，以下缓存将每五秒刷新一次：

```php
use Illuminate\Support\Str;

Cache::store('octane')->interval('random', function () {
    return Str::random(10);
}, seconds: 5);
```

## 表

> **警告**  
> 此功能需要 [Swoole](#swoole)。

在使用 Swoole 时，你可以定义并与自己的任意 [Swoole 表](https://www.swoole.co.uk/docs/modules/swoole-table) 交互。Swoole 表提供极高的性能吞吐量，这些表中的数据可以被服务器上的所有工作进程访问。然而，在服务器重新启动时，其中的数据将会丢失。

表应该在应用程序的 `octane` 配置文件的 `tables` 配置数组中定义。已经为你配置了一个示例表，允许最多有 1000 行。可以通过在列类型后指定列大小来配置字符串列的最大大小，如下所示：

```php
'tables' => [
    'example:1000' => [
        'name' => 'string:1000',
        'votes' => 'int',
    ],
],
```

要访问表，你可以使用 `Octane::table` 方法：

```php
use Laravel\Octane\Facades\Octane;

Octane::table('example')->set('uuid', [
    'name' => 'Nuno Maduro',
    'votes' => 1000,
]);

return Octane::table('example')->get('uuid');
```

> **警告**  
> Swoole 表支持的列类型有：`string`、`int` 和 `float`。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/oc...](https://learnku.com/docs/laravel/11.x/octanemd/16723)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/oc...](https://learnku.com/docs/laravel/11.x/octanemd/16723)