本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Sail

+   [介绍](#introduction)
+   [安装和设置](#installation)
    +   [将 Sail 安装到现有应用程序中](#installing-sail-into-existing-applications)
    +   [重建 Sail 镜像](#rebuilding-sail-images)
    +   [配置 Shell 别名](#configuring-a-shell-alias)
+   [启动和停止 Sail](#starting-and-stopping-sail)
+   [执行命令](#executing-sail-commands)
    +   [执行 PHP 命令](#executing-php-commands)
    +   [执行 Composer 命令](#executing-composer-commands)
    +   [执行 Artisan 命令](#executing-artisan-commands)
    +   [执行 Node / NPM 命令](#executing-node-npm-commands)
+   [与数据库交互](#interacting-with-sail-databases)
    +   [MySQL](#mysql)
    +   [Redis](#redis)
    +   [Meilisearch](#meilisearch)
    +   [Typesense](#typesense)
+   [文件存储](#file-storage)
+   [运行测试](#running-tests)
    +   [Laravel Dusk](#laravel-dusk)
+   [预览电子邮件](#previewing-emails)
+   [容器 CLI](#sail-container-cli)
+   [PHP 版本](#sail-php-versions)
+   [Node 版本](#sail-node-versions)
+   [共享您的网站](#sharing-your-site)
+   [使用 Xdebug 进行调试](#debugging-with-xdebug)
    +   [通过命令行使用 Xdebug 进行调试](#xdebug-cli-usage)
    +   [通过浏览器使用 Xdebug 进行调试](#xdebug-browser-usage)
+   [定制化](#sail-customization)

## 介绍

[Laravel Sail](https://github.com/laravel/sail) 是一个轻量级的命令行界面，用于与 Laravel 的默认 Docker 开发环境进行交互。Sail 为使用 PHP, MySQL, 和 Redis 构建 Laravel 应用提供了一个很好的起点，而不需要事先有 Docker 经验。

从本质上讲，Sail 是存储在项目根目录中的 `docker-compose.yml` 文件和 `sail` 脚本。该 `sail` 脚本提供了一个 CLI，其中包含与 `docker-compose.yml` 文件定义的 Docker 容器进行交互的便捷方法。

Laravel Sail 支持 macOS、Linux 和 Windows （通过 [WSL2](https://docs.microsoft.com/en-us/windows/wsl/about)）。

## 安装和设置

Laravel Sail 会随着所有全新的 Laravel 应用程序一起自动安装，因此你可以立即的开始使用它。要了解如何创建一个新的 Laravel 应用程序，请查阅适合您目前操作系统的 [安装文档](https://learnku.com/docs/laravel/11.x/installation#docker-installation-using-sail)。在安装过程中，你将被要求选择你的应用程序将与哪些 Sail 支持的服务进行交互。

### 将 Sail 安装到现有应用程序中

如果你有兴趣在现有的 Laravel 应用程序中使用 Sail，你可以简单地使用 Composer 包管理器安装 Sail。当然，这些步骤假设你的现有本地开发环境允许你安装 Composer 依赖项：

```shell
composer require laravel/sail --dev
```

安装了 Sail 后，你可以运行 `sail:install` Artisan 命令。这个命令将会将 Sail 的 `docker-compose.yml` 文件发布到你的应用程序的根目录，并通过修改你的 `.env` 文件，添加所需的环境变量以连接到 Docker 服务：

最后，你可以启动 Sail。要继续学习如何使用 Sail，请继续阅读本文档的其余部分：

> **注意**  
> 如果你使用 Docker Desktop for Linux，你应该通过执行以下命令使用 `default` Docker 上下文：`docker context use default`。

#### 添加额外的服务

如果你想要向现有的 Sail 安装中添加额外的服务，你可以运行 `sail:add` Artisan 命令：

#### 使用 Devcontainers

如果你想要在 [Devcontainer](https://code.visualstudio.com/docs/remote/containers) 中进行开发，你可以向 `sail:install` 命令提供 `--devcontainer` 选项。`--devcontainer` 选项将指示 `sail:install` 命令将默认的 `.devcontainer/devcontainer.json` 文件发布到你的应用程序的根目录：

```shell
php artisan sail:install --devcontainer
```

### 重建 Sail 镜像

有时候你可能想要完全重建你的 Sail 镜像，以确保所有镜像的软件包和软件都是最新的。你可以使用 `build` 命令来实现这一点：

```shell
docker compose down -v

sail build --no-cache

sail up
```

### 配置 Shell 别名

默认情况下，Sail 命令是通过所有新的 Laravel 应用程序中包含的 `vendor/bin/sail` 脚本来调用的：

然而，与其反复输入 `vendor/bin/sail` 来执行 Sail 命令，你可能希望配置一个 shell 别名，这样可以更轻松地执行 Sail 的命令：

```shell
alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'
```

为了确保这个别名始终可用，你可以将其添加到你的家目录中的 shell 配置文件中，比如 `~/.zshrc` 或 `~/.bashrc`，然后重新启动你的 shell。

一旦配置了 shell 别名，你可以通过简单地输入 `sail` 来执行 Sail 命令。本文档中其余的示例将假定你已经配置了这个别名：

## 启动和停止 Sail

Laravel Sail 的 `docker-compose.yml` 文件定义了各种 Docker 容器，这些容器共同协作，帮助你构建 Laravel 应用程序。这些容器中的每一个都是你的 `docker-compose.yml` 文件中 `services` 配置的一个条目。`laravel.test` 容器是主要的应用程序容器，将为你提供应用程序服务。

在启动 Sail 之前，你应该确保你的本地计算机上没有其他的 web 服务器或数据库正在运行。要启动应用程序 `docker-compose.yml` 文件中定义的所有 Docker 容器，你应该执行 `up` 命令：

要在后台启动所有 Docker 容器，你可以以 「detached」 模式启动 Sail：

一旦应用程序的容器已经启动，你可以在 web 浏览器中访问项目：[localhost](http://localhost/)。

要停止所有容器，你可以简单地按下 Control + C 来停止容器的执行。或者，如果容器在后台运行，你可以使用 `stop` 命令：

## 执行命令

在使用 Laravel Sail 时，你的应用程序是在一个 Docker 容器中执行的，并且与你的本地计算机隔离。然而，Sail 提供了一种方便的方式来运行各种命令，比如任意的 PHP 命令、Artisan 命令、Composer 命令以及 Node / NPM 命令。

**在阅读 Laravel 文档时，你经常会看到关于 Composer、Artisan 和 Node / NPM 命令的引用，而没有提到 Sail。** 这些示例假设这些工具已经安装在你的本地计算机上。如果你正在使用 Sail 来搭建本地的 Laravel 开发环境，你应该使用 Sail 来执行这些命令：

```shell
# 在本地运行 Artisan 命令...
php artisan queue:work

# 在 Laravel Sail 中运行 Artisan 命令...
sail artisan queue:work
```

### 执行 PHP 命令

可以使用 `php` 命令来执行 PHP 命令。当然，这些命令将使用为你的应用程序配置的 PHP 版本来执行。要了解更多关于 Laravel Sail 可用的 PHP 版本，请参考 [PHP 版本文档](#sail-php-versions)：

```shell
sail php --version

sail php script.php
```

### 执行 Composer 命令

可以使用 `composer` 命令来执行 Composer 命令。Laravel Sail 的应用程序容器包含了 Composer 的安装：

```nothing
sail composer require laravel/sanctum
```

#### 为现有应用程序安装 Composer 依赖

如果你是与团队一起开发应用程序，可能不是最初创建 Laravel 应用程序的人。因此，在将应用程序的存储库克隆到你的本地计算机后，应用程序的 Composer 依赖关系，包括 Sail，在你的计算机上都不会安装。

你可以通过进入应用程序目录并执行以下命令来安装应用程序的依赖关系。该命令使用一个包含 PHP 和 Composer 的小型 Docker 容器来安装应用程序的依赖项：

```shell
docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v "$(pwd):/var/www/html" \
    -w /var/www/html \
    laravelsail/php83-composer:latest \
    composer install --ignore-platform-reqs
```

在使用 `laravelsail/phpXX-composer` 镜像时，你应该使用与你计划在应用程序中使用的 PHP 版本相同的版本（`80`、`81`、`82` 或 `83`）。

### 执行 Artisan 命令

可以使用 `artisan` 命令来执行 Laravel Artisan 命令：

### 执行 Node / NPM 命令

可以使用 `node` 命令来执行 Node 命令，而使用 `npm` 命令来执行 NPM 命令：

```shell
sail node --version

sail npm run dev
```

如果你希望，也可以使用 Yarn 而不是 NPM：

## 与数据库交互

### MySQL

你可能已经注意到，你的应用程序的 `docker-compose.yml` 文件中包含了一个 MySQL 容器的条目。这个容器使用了一个 [Docker 卷](https://docs.docker.com/storage/volumes/)，这样即使停止和重新启动容器，数据库中存储的数据也会被保留。

此外，当 MySQL 容器第一次启动时，它会为你创建两个数据库。第一个数据库的名称使用你的 `DB_DATABASE` 环境变量的值，并用于你的本地开发。第二个是一个专用的测试数据库，名称为 `testing`，将确保你的测试不会干扰你的开发数据。

一旦启动了你的容器，你可以通过将应用程序的 `.env` 文件中的 `DB_HOST` 环境变量设置为 `mysql` 来连接到应用程序内部的 MySQL 实例。

要从本地计算机连接到应用程序的 MySQL 数据库，你可以使用诸如 [TablePlus](https://tableplus.com/) 之类的图形化数据库管理应用程序。默认情况下，MySQL 数据库可以在 `localhost` 的 3306 端口访问，访问凭据对应于你的 `DB_USERNAME` 和 `DB_PASSWORD` 环境变量的值。或者，你可以作为 `root` 用户连接，这也会使用你的 `DB_PASSWORD` 环境变量的值作为密码。

### Redis

你的应用程序的 `docker-compose.yml` 文件还包含了一个 [Redis](https://redis.io/) 容器的条目。这个容器使用了一个 [Docker 卷](https://docs.docker.com/storage/volumes/)，这样即使停止和重新启动容器，Redis 数据中存储的数据也会被保留。一旦启动了你的容器，你可以通过将应用程序的 `.env` 文件中的 `REDIS_HOST` 环境变量设置为 `redis` 来连接到应用程序内部的 Redis 实例。

要从本地计算机连接到应用程序的 Redis 数据库，你可以使用诸如 [TablePlus](https://tableplus.com/) 之类的图形化数据库管理应用程序。默认情况下，Redis 数据库可以在 `localhost` 的 6379 端口访问。

### Meilisearch

如果在安装 Sail 时选择安装 [Meilisearch](https://www.meilisearch.com/) 服务，你的应用程序的 `docker-compose.yml` 文件将包含一个条目，用于这个与 [Laravel Scout](https://learnku.com/docs/laravel/11.x/scout) 集成的强大搜索引擎。一旦启动了你的容器，你可以通过将应用程序的 `MEILISEARCH_HOST` 环境变量设置为 `http://meilisearch:7700` 来连接到应用程序内部的 Meilisearch 实例。

从你的本地计算机，你可以通过在 Web 浏览器中导航到 `http://localhost:7700` 来访问 Meilisearch 的基于 Web 的管理面板。

### Typesense

如果在安装 Sail 时选择安装 [Typesense](https://typesense.org/) 服务，你的应用程序的 `docker-compose.yml` 文件将包含一个条目，用于这个与 [Laravel Scout](https://learnku.com/docs/laravel/11.x/scout#typesense) 原生集成的高速开源搜索引擎。一旦启动了你的容器，你可以通过设置以下环境变量来连接到应用程序内部的 Typesense 实例：

```ini
TYPESENSE_HOST=typesense
TYPESENSE_PORT=8108
TYPESENSE_PROTOCOL=http
TYPESENSE_API_KEY=xyz
```

从你的本地计算机，你可以通过 `http://localhost:8108` 访问 Typesense 的 API。

## 文件存储

如果在生产环境中运行应用程序时计划使用 Amazon S3 存储文件，你可能希望在安装 Sail 时安装 [MinIO](https://min.io/) 服务。MinIO 提供了一个与 S3 兼容的 API，你可以在本地开发中使用 Laravel 的 `s3` 文件存储驱动，而无需在生产 S3 环境中创建 “测试” 存储桶。如果选择在安装 Sail 时安装 MinIO，将会在你的应用程序的 `docker-compose.yml` 文件中添加一个 MinIO 配置部分。

默认情况下，你的应用程序的 `filesystems` 配置文件已经包含了一个用于 `s3` 磁盘的配置。除了用于与 Amazon S3 交互外，你还可以使用它与任何兼容 S3 的文件存储服务进行交互，比如 MinIO，只需简单地修改控制其配置的相关环境变量。例如，当使用 MinIO 时，你的文件系统环境变量配置应定义如下：

```ini
FILESYSTEM_DISK=s3
AWS_ACCESS_KEY_ID=sail
AWS_SECRET_ACCESS_KEY=password
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=local
AWS_ENDPOINT=http://minio:9000
AWS_USE_PATH_STYLE_ENDPOINT=true
```

为了让 Laravel 的 Flysystem 集成在使用 MinIO 时生成正确的 URL，你应该定义 `AWS_URL` 环境变量，使其与应用程序的本地 URL 匹配，并在 URL 路径中包含存储桶名称：

```ini
AWS_URL=http://localhost:9000/local
```

你可以通过 MinIO 控制台创建存储桶，控制台地址为 `http://localhost:8900`。MinIO 控制台的默认用户名为 `sail`，默认密码为 `password`。

> **注意**  
> 当使用 MinIO 时，通过 `temporaryUrl` 方法生成临时存储 URL 不受支持。

## 运行测试

Laravel 提供了出色的测试支持，并且你可以使用 Sail 的 `test` 命令来运行你的应用程序的 [功能和单元测试](https://learnku.com/docs/laravel/11.x/testing)。任何 Pest / PHPUnit 接受的 CLI 选项也可以传递给 `test` 命令：

```shell
sail test

sail test --group orders
```

Sail 的 `test` 命令等同于运行 `test` Artisan 命令：

默认情况下，Sail 将创建一个专用的 `testing` 数据库，以确保你的测试不会影响到数据库的当前状态。在默认的 Laravel 安装中，Sail 还会配置你的 `phpunit.xml` 文件，以在执行测试时使用这个数据库：

```xml
<env name="DB_DATABASE" value="testing"/>
```

### Laravel Dusk

[Laravel Dusk](https://learnku.com/docs/laravel/11.x/dusk) 提供了一套表达力强、易于使用的浏览器自动化和测试 API。借助 Sail，你可以在本地计算机上运行这些测试，而无需安装 Selenium 或其他工具。要开始使用，取消注释应用程序的 `docker-compose.yml` 文件中的 Selenium 服务：

```ini
selenium:
    image: 'selenium/standalone-chrome'
    extra_hosts:
      - 'host.docker.internal:host-gateway'
    volumes:
        - '/dev/shm:/dev/shm'
    networks:
        - sail
```

接下来，确保应用程序的 `docker-compose.yml` 文件中的 `laravel.test` 服务有一个 `depends_on` 条目指向 `selenium`：

```ini
depends_on:
    - mysql
    - redis
    - selenium
```

最后，你可以通过启动 Sail 并运行 `dusk` 命令来运行你的 Dusk 测试套件：

#### Apple Silicon 上的 Selenium

如果你的本地计算机使用的是 Apple Silicon 芯片，你的 `selenium` 服务必须使用 `seleniarm/standalone-chromium` 镜像：

```ini
selenium:
    image: 'seleniarm/standalone-chromium'
    extra_hosts:
        - 'host.docker.internal:host-gateway'
    volumes:
        - '/dev/shm:/dev/shm'
    networks:
        - sail
```

## 预览电子邮件

Laravel Sail 的默认 `docker-compose.yml` 文件包含一个用于 [Mailpit](https://github.com/axllent/mailpit) 的服务条目。Mailpit 在本地开发期间拦截应用程序发送的电子邮件，并提供一个方便的 Web 界面，让你可以在浏览器中预览你的电子邮件消息。在使用 Sail 时，Mailpit 的默认主机是 `mailpit`，可通过端口 1025 访问：

```ini
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_ENCRYPTION=null
```

当 Sail 在运行时，你可以通过以下链接访问 Mailpit 网页界面：[localhost:8025](http://localhost:8025/)

## 容器 CLI

有时候你可能希望在应用程序的容器内启动一个 Bash 会话。你可以使用 `shell` 命令连接到应用程序的容器，这样你就可以检查其文件和安装的服务，以及在容器内执行任意的 shell 命令：

```shell
sail shell

sail root-shell
```

要启动一个新的 [Laravel Tinker](https://github.com/laravel/tinker) 会话，你可以执行 `tinker` 命令：

## PHP 版本

Sail 目前支持通过 PHP 8.3、8.2、8.1 或 PHP 8.0 提供应用程序服务。Sail 当前使用的默认 PHP 版本是 PHP 8.3。要更改用于提供应用程序服务的 PHP 版本，你应该更新应用程序的 `docker-compose.yml` 文件中 `laravel.test` 容器的 `build` 定义：

```ini
# PHP 8.3
context: ./vendor/laravel/sail/runtimes/8.3

# PHP 8.2
context: ./vendor/laravel/sail/runtimes/8.2

# PHP 8.1
context: ./vendor/laravel/sail/runtimes/8.1

# PHP 8.0
context: ./vendor/laravel/sail/runtimes/8.0
```

此外，你可能希望更新你的 `image` 名称以反映你的应用程序使用的 PHP 版本。此选项也在应用程序的 `docker-compose.yml` 文件中定义：

更新应用程序的 `docker-compose.yml` 文件后，你应该重建容器镜像：

```shell
sail build --no-cache

sail up
```

## Node 版本

Sail 默认安装 Node 20。要更改构建镜像时安装的 Node 版本，你可以更新应用程序的 `docker-compose.yml` 文件中 `laravel.test` 服务的 `build.args` 定义：

```ini
build:
    args:
        WWWGROUP: '${WWWGROUP}'
        NODE_VERSION: '18'
```

更新应用程序的 `docker-compose.yml` 文件后，你应该重建容器镜像：

```shell
sail build --no-cache

sail up
```

## 分享你的网站

有时候你可能需要公开分享你的网站，以便为同事预览你的网站或测试应用程序的 Webhook 集成。要分享你的网站，你可以使用 `share` 命令。执行此命令后，你将获得一个随机的 `laravel-sail.site` URL，你可以用它来访问你的应用程序：

在通过 `share` 命令分享你的网站时，你应该在应用程序的 `bootstrap/app.php` 文件中使用 `trustProxies` 中间件方法配置应用程序的受信任代理。否则，诸如 `url` 和 `route` 这样的 URL 生成辅助程序将无法确定在 URL 生成期间应该使用的正确 HTTP 主机：

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->trustProxies(at: [
        '*',
    ]);
})
```

如果你想为你的共享网站选择子域名，你可以在执行 `share` 命令时提供 `subdomain` 选项：

```shell
sail share --subdomain=my-sail-site
```

> **注意**  
> `share` 命令由 [BeyondCode](https://beyondco.de/) 开发的开源隧道服务 [Expose](https://github.com/beyondcode/expose) 提供支持。

## 使用 Xdebug 进行调试

Laravel Sail 的 Docker 配置包含对 [Xdebug](https://xdebug.org/) 的支持，这是 PHP 的一款流行且强大的调试器。为了启用 Xdebug，你需要在应用程序的 `.env` 文件中添加一些变量来[配置 Xdebug](https://xdebug.org/docs/step_debug#mode)。在启动 Sail 之前，你必须设置适当的模式来启用 Xdebug：

```ini
SAIL_XDEBUG_MODE=develop,debug,coverage
```

#### Linux 主机 IP 配置

在内部，`XDEBUG_CONFIG` 环境变量被定义为 `client_host=host.docker.internal`，以便为 Mac 和 Windows (WSL2) 正确配置 Xdebug。如果你的本地计算机运行的是 Linux，你应该确保你正在运行 Docker Engine 17.06.0+ 和 Compose 1.16.0+。否则，你将需要手动定义这个环境变量，如下所示。

首先，你应该通过运行以下命令确定要添加到环境变量的正确主机 IP 地址。通常，`<container-name>` 应该是为你的应用程序提供服务的容器的名称，通常以 `_laravel.test_1` 结尾：

```shell
docker inspect -f {{range.NetworkSettings.Networks}}{{.Gateway}}{{end}} <container-name>
```

获得正确的主机 IP 地址后，你应该在应用程序的 `.env` 文件中定义 `SAIL_XDEBUG_CONFIG` 变量：

```ini
SAIL_XDEBUG_CONFIG="client_host=<host-ip-address>"
```

### Xdebug CLI 使用

可以使用 `sail debug` 命令在运行 Artisan 命令时启动调试会话：

```shell
# 不使用 Xdebug 运行 Artisan 命令...
sail artisan migrate

# 使用 Xdebug 运行 Artisan 命令...
sail debug migrate
```

### Xdebug 浏览器使用

在通过 Web 浏览器与应用程序交互时调试应用程序，请按照 Xdebug 提供的 [说明](https://xdebug.org/docs/step_debug#web-application) 来启动一个来自 Web 浏览器的 Xdebug 会话。

如果你使用 PhpStorm，请查阅 JetBrains 的关于 [零配置调试](https://www.jetbrains.com/help/phpstorm/zero-configuration-debugging.html) 的文档。

> **警告**  
> Laravel Sail 依赖于 `artisan serve` 来提供应用程序服务。`artisan serve` 命令仅在 Laravel 版本 8.53.0 中接受 `XDEBUG_CONFIG` 和 `XDEBUG_MODE` 变量。较旧版本的 Laravel（8.52.0 及以下）不支持这些变量，并且不会接受调试连接。

## 自定义

由于 Sail 只是 Docker，你可以自由地定制几乎关于它的一切。要发布 Sail 自己的 Dockerfile，你可以执行 `sail:publish` 命令：

```shell
sail artisan sail:publish
```

运行此命令后，Laravel Sail 使用的 Dockerfile 和其他配置文件将被放置在应用程序根目录下的 `docker` 目录中。在自定义 Sail 安装后，你可能希望在应用程序的 `docker-compose.yml` 文件中为应用程序容器更改镜像名称。这样做后，使用 `build` 命令重新构建应用程序的容器。如果你在单台计算机上使用 Sail 开发多个 Laravel 应用程序，为应用程序镜像分配一个唯一的名称尤为重要：

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/sa...](https://learnku.com/docs/laravel/11.x/sailmd/16731)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/sa...](https://learnku.com/docs/laravel/11.x/sailmd/16731)