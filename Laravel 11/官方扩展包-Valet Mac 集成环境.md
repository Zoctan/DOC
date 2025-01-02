本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Valet

+   [介绍](#introduction)
+   [安装](#installation)
    +   [升级 Valet](#upgrading-valet)
+   [服务站点](#serving-sites)
    +   [Park 命令](#the-park-command)
    +   [Link 命令](#the-link-command)
    +   [通过 TLS 保护站点](#securing-sites)
    +   [服务默认站点](#serving-a-default-site)
    +   [站点特定 PHP 版本](#per-site-php-versions)
+   [共享站点](#sharing-sites)
    +   [在本地网络上共享站点](#sharing-sites-on-your-local-network)
+   [站点特定环境变量](#site-specific-environment-variables)
+   [代理服务](#proxying-services)
+   [自定义 Valet 驱动程序](#custom-valet-drivers)
    +   [本地驱动程序](#local-drivers)
+   [其他 Valet 命令](#other-valet-commands)
+   [Valet 目录和文件](#valet-directories-and-files)
    +   [磁盘访问](#disk-access)

## 介绍

> \[!NOTE\]  
> 想要在 macOS 或 Windows 上更轻松地开发 Laravel 应用程序吗？请查看 [Laravel Herd](https://herd.laravel.com/)。Herd 包括您开始使用 Laravel 开发所需的一切，包括 Valet、PHP 和 Composer。

[Laravel Valet](https://github.com/laravel/valet) 是为 macOS 极简主义者设计的开发环境。Laravel Valet 配置您的 Mac，在您的机器启动时始终在后台运行 [Nginx](https://www.nginx.com/)。然后，使用 [DnsMasq](https://en.wikipedia.org/wiki/Dnsmasq)，Valet 会将 `*.test` 域上的所有请求代理到安装在本地机器上的站点。

换句话说，Valet 是一个极速的 Laravel 开发环境，大约使用 7 MB 的 RAM。Valet 不是 [Sail](https://learnku.com/docs/laravel/11.x/sail) 或 [Homestead](https://learnku.com/docs/laravel/11.x/homestead) 的完全替代品，但如果您想要灵活的基础设施、极速体验，或者在内存有限的机器上工作，Valet 提供了一个很好的选择。

Valet 默认支持，但不限于以下内容：

+   [Laravel](https://laravel.com/)
+   [Bedrock](https://roots.io/bedrock/)
+   [CakePHP 3](https://cakephp.org/)
+   [ConcreteCMS](https://www.concretecms.com/)
+   [Contao](https://contao.org/en/)
+   [Craft](https://craftcms.com/)
+   [Drupal](https://www.drupal.org/)
+   [ExpressionEngine](https://www.expressionengine.com/)
+   [Jigsaw](https://jigsaw.tighten.co/)
+   [Joomla](https://www.joomla.org/)
+   [Katana](https://github.com/themsaid/katana)
+   [Kirby](https://getkirby.com/)
+   [Magento](https://magento.com/)
+   [OctoberCMS](https://octobercms.com/)
+   [Sculpin](https://sculpin.io/)
+   [Slim](https://www.slimframework.com/)
+   [Statamic](https://statamic.com/)
+   静态 HTML
+   [Symfony](https://symfony.com/)
+   [WordPress](https://wordpress.org/)
+   [Zend](https://framework.zend.com/)

然而，你可以通过自己的[自定义驱动程序](#custom-valet-drivers)来扩展 Valet。

## 安装

> \[!WARNING\]  
> Valet 需要 macOS 和 [Homebrew](https://brew.sh/)。在安装之前，你应确保没有其他程序（如 Apache 或 Nginx）绑定到您本地机器的端口 80。

要开始之前，首先需要确保使用 `update` 命令使 Homebrew 处于最新状态：

接下来，你应该使用 Homebrew 安装 PHP：

安装完 PHP 后，就可以安装 [Composer 包管理器](https://getcomposer.org/)。此外，你应确保 `$HOME/.composer/vendor/bin` 目录在你系统的 「PATH」 中。安装完 Composer 后，你可以将 Laravel Valet 安装为全局 Composer 包：

```shell
composer global require laravel/valet
```

最后，你可以执行 Valet 的 `install` 命令。这将配置和安装 Valet 和 DnsMasq。此外，Valet 依赖的守护程序将被配置为在系统启动时启动：

安装完 Valet 后，你可以尝试在终端上使用类似 `ping foobar.test` 的命令来 ping 任何 `*.test` 域。如果 Valet 安装正确，应该看到该域正在 `127.0.0.1` 上响应。

Valet 将在每次启动您的机器时自动启动其所需的服务。

#### PHP 版本

> \[!NOTE\]  
> 您可以通过 `isolate` [命令](#per-site-php-versions)指示 Valet 使用每个站点的 PHP 版本，而不是修改全局 PHP 版本。

Valet 允许使用 `valet use php@version` 命令切换 PHP 版本。如果指定的 PHP 版本尚未安装，Valet 将通过 Homebrew 安装它：

```shell
valet use php@8.2

valet use php
```

你也可以在项目的根目录中创建一个 `.valetrc` 文件。`.valetrc` 文件应包含站点应该使用的 PHP 版本：

创建完这个文件后，你可以简单地执行 `valet use` 命令，该命令将通过读取文件来确定站点的首选 PHP 版本。

> **警告**  
> Valet 一次只能提供一个 PHP 版本，即使你安装了多个 PHP 版本。

#### 数据库

如果你的应用程序需要数据库，请查看 [DBngin](https://dbngin.com/)，它提供了一个免费的一体化数据库管理工具，包括 MySQL、PostgreSQL 和 Redis。安装完 DBngin 后，你可以使用 `root` 用户名和空字符串作为密码在 `127.0.0.1` 上连接到你的数据库。

#### 重置你的安装

如果你在让 Valet 安装正常运行方面遇到问题，执行 `composer global require laravel/valet` 命令，然后执行 `valet install` 将重置你的安装并可以解决各种问题。在罕见情况下，可能需要通过执行 `valet uninstall --force`，然后执行 `valet install` 来「硬重置」Valet。

### 升级 Valet

你可以通过在终端中执行 `composer global require laravel/valet` 命令来更新你的 Valet 安装。升级后，最好运行 `valet install` 命令，这样 Valet 可以根据需要对你的配置文件进行额外的升级。

#### 升级到 Valet 4

如果你从 Valet 3 升级到 Valet 4，请按照以下步骤正确升级你的 Valet 安装：

+   如果你添加了 `.valetphprc` 文件来自定义站点的 PHP 版本，请将每个`.valetphprc` 文件重命名为 `.valetrc`。然后，在 `.valetrc` 文件的现有内容前加上 `php=`。
+   更新任何自定义驱动程序以匹配新驱动程序系统的命名空间、扩展、类型提示和返回类型提示。你可以参考 Valet 的 [SampleValetDriver](https://github.com/laravel/valet/blob/d7787c025e60abc24a5195dc7d4c5c6f2d984339/cli/stubs/SampleValetDriver.php) 作为示例。
+   如果你使用 PHP 7.1 - 7.4 来提供你的站点，请确保你仍然使用 Homebrew 安装一个 PHP 版本为 8.0 或更高版本，因为 Valet 将使用这个版本，即使它不是你的主要链接版本，来运行一些脚本。

### 提供站点

安装好 Valet 后，你就可以开始提供你的 Laravel 应用程序了。Valet 提供了两个命令来帮助你提供应用程序：`park` 和 `link`。

### `park` 命令

`park` 命令会在你的机器上注册一个包含应用程序的目录。一旦目录被 Valet 「停放」，该目录中的所有目录将可以在你的 Web 浏览器中访问，地址为 `http://<目录名称>.test`：

就是这样。现在，你在「停放」目录中创建的任何应用程序都将自动使用 `http://<目录名称>.test` 约定进行提供。因此，如果你的停放目录包含一个名为 「laravel」 的目录，该目录中的应用程序将可以在 `http://laravel.test` 上访问。此外，Valet 还允许你使用通配符子域名访问该站点（`http://foo.laravel.test`）。

### `link` 命令

`link` 命令也可以用来提供你的 Laravel 应用程序。如果你只想在一个目录中提供单个站点而不是整个目录，这个命令就很有用：

```shell
cd ~/Sites/laravel

valet link
```

一旦使用 `link` 命令将应用程序链接到 Valet，你可以使用其目录名称访问该应用程序。因此，在上面的示例中链接的站点可以在 `http://laravel.test` 上访问。此外，Valet 还允许你使用通配符子域名访问该站点（`http://foo.laravel.test`）。

如果你想要在不同的主机名下提供应用程序，你可以将主机名传递给 `link` 命令。例如，你可以运行以下命令使一个应用程序在 `http://application.test` 上可用：

```shell
cd ~/Sites/laravel

valet link application
```

当然，你也可以使用 `link` 命令在子域上提供应用程序：

```shell
valet link api.application
```

你可以执行 `links` 命令来显示所有已链接目录的列表：

`unlink` 命令可用于销毁站点的符号链接：

```shell
cd ~/Sites/laravel

valet unlink
```

### 使用 TLS 保护站点

默认情况下，Valet 通过 HTTP 提供站点服务。但是，如果你想要通过加密的 TLS 使用 HTTP/2 提供站点服务，你可以使用 `secure` 命令。例如，如果你的站点由 Valet 在 `laravel.test` 域上提供服务，你应该运行以下命令来保护它：

要 “取消安全” 一个站点，并恢复将其流量通过普通 HTTP 提供服务，使用 `unsecure` 命令。与 `secure` 命令类似，这个命令接受你希望取消安全的主机名：

### 提供默认站点

有时，你可能希望配置 Valet 在访问未知的 `test` 域时提供一个 “默认” 站点，而不是 `404` 页面。为了实现这一点，你可以在你的 `~/.config/valet/config.json` 配置文件中添加一个 `default` 选项，其中包含应该作为默认站点的站点路径：

```php
"default": "/Users/Sally/Sites/example-site",
```

### 每个站点的 PHP 版本

默认情况下，Valet 使用全局 PHP 安装来提供站点服务。然而，如果你需要在各个站点上支持多个 PHP 版本，你可以使用 `isolate` 命令来指定特定站点应该使用哪个 PHP 版本。`isolate` 命令配置 Valet 在你当前工作目录中的站点使用指定的 PHP 版本：

```shell
cd ~/Sites/example-site

valet isolate php@8.0
```

如果你的站点名称与包含它的目录名称不匹配，你可以使用 `--site` 选项指定站点名称：

```shell
valet isolate php@8.0 --site="site-name"
```

为了方便起见，你可以使用 `valet php`、`composer` 和 `which-php` 命令来代理调用适当的 PHP CLI 或工具，根据站点配置的 PHP 版本：

```shell
valet php
valet composer
valet which-php
```

你可以执行 `isolated` 命令来显示所有被隔离的站点及其 PHP 版本的列表：

要将一个站点恢复到 Valet 全局安装的 PHP 版本，你可以从站点的根目录调用 `unisolate` 命令：

## 分享站点

Valet 包含一个命令，可以让你将本地站点与世界分享，为你在移动设备上测试站点或与团队成员和客户分享提供了一种简单的方式。

Valet 默认支持通过 ngrok 或 Expose 分享你的站点。在分享站点之前，你应该使用 `share-tool` 命令更新你的 Valet 配置，指定 `ngrok` 或 `expose`：

如果你选择了一个工具，但没有通过 Homebrew（对于 ngrok）或 Composer（对于 Expose）安装它，Valet 会自动提示你安装它。当然，你需要在开始分享站点之前先认证你的 ngrok 或 Expose 账户。

要分享一个站点，进入终端中站点的目录并运行 Valet 的 `share` 命令。一个可公开访问的 URL 将被放入你的剪贴板，准备直接粘贴到你的浏览器中或与你的团队分享：

```shell
cd ~/Sites/laravel

valet share
```

停止分享你的站点，你可以按下 `Control + C`。

> \[! 警告\]  
> 如果你正在使用自定义 DNS 服务器（如 `1.1.1.1`），ngrok 分享可能无法正常工作。如果在你的设备上是这种情况，请打开你的 Mac 系统设置，进入网络设置，打开高级设置，然后转到 DNS 选项卡，并将 `127.0.0.1` 添加为你的第一个 DNS 服务器。

#### 通过 Ngrok 分享站点

使用 ngrok 分享你的站点需要你[创建一个 ngrok 账户](https://dashboard.ngrok.com/signup)并[设置一个认证令牌](https://dashboard.ngrok.com/get-started/your-authtoken)。一旦你有了认证令牌，你可以使用该令牌更新你的 Valet 配置：

```shell
valet set-ngrok-token YOUR_TOKEN_HERE
```

> \[! 注意\]  
> 你可以向分享命令传递额外的 ngrok 参数，比如 `valet share --region=eu`。欲了解更多信息，请参阅 [ngrok 文档](https://ngrok.com/docs)。

#### 通过 Expose 分享站点

使用 Expose 分享你的站点需要你[创建一个 Expose 账户](https://expose.dev/register)并[通过你的认证令牌与 Expose 进行认证](https://expose.dev/docs/getting-started/getting-your-token)。

你可以查阅 [Expose 文档](https://expose.dev/docs)以获取关于它支持的额外命令行参数的信息。

### 在本地网络上分享站点

Valet 默认将传入流量限制在内部的 `127.0.0.1` 接口，以防止你的开发机器受到来自互联网的安全风险。

如果你希望允许本地网络上的其他设备通过你的机器的 IP 地址（例如：`192.168.1.10/application.test`）访问你机器上的 Valet 站点，你需要手动编辑相应站点的 Nginx 配置文件，以移除 `listen` 指令上的限制。你应该移除端口 80 和 443 上 `listen` 指令的 `127.0.0.1:` 前缀。

### 如果未在项目上运行 `valet secure`

如果你尚未在项目上运行 `valet secure`，可以通过编辑 `/usr/local/etc/nginx/valet/valet.conf` 文件为所有非 HTTPS 站点打开网络访问。但是，如果你正在通过 HTTPS 提供项目站点（已为站点运行 `valet secure`），那么应该编辑 `~/.config/valet/Nginx/app-name.test` 文件。

更新 Nginx 配置后，运行 `valet restart` 命令以应用配置更改。

## 站点特定环境变量

一些使用其他框架的应用程序可能依赖于服务器环境变量，但没有提供在项目内配置这些变量的方式。Valet 允许在项目根目录中添加一个 `.valet-env.php` 文件来配置站点特定的环境变量。该文件应返回一个「站点 / 环境变量」对的数组，这些对将被添加到全局 `$_SERVER` 数组中，用于数组中指定的每个站点：

```php
<?php

return [
    // 为 laravel.test 站点设置 $_SERVER['key'] 为 "value"...
    'laravel' => [
        'key' => 'value',
    ],

    // 为所有站点设置 $_SERVER['key'] 为 "value"...
    '*' => [
        'key' => 'value',
    ],
];
```

## 代理服务

有时你可能希望将 Valet 域代理到本地计算机上的另一个服务。例如可能偶尔需要在运行 Docker 的同时运行 Valet 的独立站点；但是，Valet 和 Docker 不能同时绑定到端口 80。

为解决这个问题，可以使用 `proxy` 命令生成代理。例如，可以将来自 `http://elasticsearch.test` 的所有流量代理到 `http://127.0.0.1:9200`：

```shell
# 通过 HTTP 进行代理...
valet proxy elasticsearch http://127.0.0.1:9200

# 通过 TLS + HTTP/2 进行代理...
valet proxy elasticsearch http://127.0.0.1:9200 --secure
```

### 移除代理和列出代理配置

可以使用 `unproxy` 命令来移除代理：

```shell
valet unproxy elasticsearch
```

可以使用 `proxies` 命令列出所有被代理的站点配置：

## 自定义 Valet 驱动程序

你可以编写自己的 Valet 「驱动程序」，用于为 Valet 原生不支持的框架或 CMS 上运行的 PHP 应用程序提供服务。安装 Valet 时，会创建一个 `~/.config/valet/Drivers` 目录，其中包含一个 `SampleValetDriver.php` 文件。该文件包含一个演示如何编写自定义驱动程序的示例实现。编写驱动程序只需要实现三个方法：`serves`、`isStaticFile` 和 `frontControllerPath`。

这三个方法都以 `$sitePath`、`$siteName` 和 `$uri` 作为参数。`$sitePath` 是您机器上提供服务的站点的完全限定路径，例如 `/Users/Lisa/Sites/my-project`。`$siteName` 是域名的 「host」 / 「site name」 部分（`my-project`）。`$uri` 是传入的请求 URI（`/foo/bar`）。

完成你自定义的 Valet 驱动程序后，请将其放置在 `~/.config/valet/Drivers` 目录中，并使用 `FrameworkValetDriver.php` 命名约定。例如，如果你正在为 WordPress 编写自定义 Valet 驱动程序，则文件名应为 `WordPressValetDriver.php`。

让我们看一下自定义的 Valet 驱动程序应实现的每个方法的示例实现。

#### `serves` 方法

如果驱动程序应处理传入请求，则 `serves` 方法应返回 `true`。否则，该方法应返回 `false`。因此，在此方法中，你应尝试确定给定的 `$sitePath` 是否包含要提供服务的类型的项目。

例如，让我们想象我们正在编写一个 `WordPressValetDriver`。我们的 `serves` 方法可能如下所示：

```php
/**
 * Determine if the driver serves the request.
 */
public function serves(string $sitePath, string $siteName, string $uri): bool
{
    return is_dir($sitePath.'/wp-admin');
}
```

#### `isStaticFile` 方法

`isStaticFile` 方法应确定传入请求是否是针对「静态」文件，例如图像或样式表。如果文件是静态的，该方法应返回磁盘上静态文件的完全限定路径。如果传入请求不是针对静态文件，则该方法应返回 `false`：

```php
/**
 * Determine if the incoming request is for a static file.
 *
 * @return string|false
 */
public function isStaticFile(string $sitePath, string $siteName, string $uri)
{
    if (file_exists($staticFilePath = $sitePath.'/public/'.$uri)) {
        return $staticFilePath;
    }

    return false;
}
```

> \[!WARNING\]  
> `isStaticFile` 方法仅在 `serves` 方法对传入请求返回 `true` 且请求 URI 不是 `/` 时才会被调用。

#### `frontControllerPath` 方法

`frontControllerPath` 方法应返回应用程序的 “前端控制器” 的完全限定路径，通常是一个 “index.php” 文件或等效文件：

```php
/**
 * Get the fully resolved path to the application's front controller.
 */
public function frontControllerPath(string $sitePath, string $siteName, string $uri): string
{
    return $sitePath.'/public/index.php';
}
```

### 本地驱动程序

如果你想为单个应用程序定义自定义 Valet 驱动程序，请在应用程序的根目录中创建一个 `LocalValetDriver.php` 文件。您的自定义驱动程序可以扩展基本的 `ValetDriver` 类或扩展现有的特定应用程序驱动程序，如 `LaravelValetDriver`：

```php
use Valet\Drivers\LaravelValetDriver;

class LocalValetDriver extends LaravelValetDriver
{
    /**
     * Determine if the driver serves the request.
     */
    public function serves(string $sitePath, string $siteName, string $uri): bool
    {
        return true;
    }

    /**
     * Get the fully resolved path to the application's front controller.
     */
    public function frontControllerPath(string $sitePath, string $siteName, string $uri): string
    {
        return $sitePath.'/public_html/index.php';
    }
}
```

### 其他 Valet 命令

| 命令 | 描述 |
| --- | --- |
| `valet list` | 显示所有 Valet 命令的列表。 |
| `valet diagnose` | 输出诊断信息以帮助调试 Valet。 |
| `valet directory-listing` | 确定目录列表行为。默认为 「off」，对目录呈现 404 页面。 |
| `valet log` | 查看由 Valet 服务编写的日志列表。 |
| `valet paths` | 查看所有您的 「parked」 路径。 |
| `valet restart` | 重新启动 Valet 守护进程。 |
| `valet start` | 启动 Valet 守护进程。 |
| `valet stop` | 停止 Valet 守护进程。 |
| `valet trust` | 为 Brew 和 Valet 添加 sudoers 文件，以允许无需提示输入密码运行 Valet 命令。 |
| `valet uninstall` | 卸载 Valet：显示手动卸载的说明。使用 `--force` 选项来强制删除所有 Valet 资源。 |

### Valet 目录和文件

在排除 Valet 环境问题时，以下目录和文件信息有所帮助：

#### `~/.config/valet`

包含所有 Valet 的配置。你可能希望保留此目录的备份。

#### `~/.config/valet/dnsmasq.d/`

目录包含 DNSMasq 的配置。

#### `~/.config/valet/Drivers/`

目录包含 Valet 的驱动程序。驱动程序确定特定框架 / CMS 的提供方式。

#### `~/.config/valet/Nginx/`

目录包含所有 Valet 的 Nginx 站点配置。运行 `install` 和 `secure` 命令时，这些文件会重新构建。

#### `~/.config/valet/Sites/`

这个目录包含所有你的[已链接项目](#the-link-command)的符号链接。

### `~/.config/valet/config.json`

Valet 的主配置文件。

### `~/.config/valet/valet.sock`

Valet 的 Nginx 安装所使用的 PHP-FPM 套接字。只有在 PHP 正常运行时才会存在。

### `~/.config/valet/Log/fpm-php.www.log`

PHP 错误的用户日志。

### `~/.config/valet/Log/nginx-error.log`

Nginx 错误的用户日志。

### `/usr/local/var/log/php-fpm.log`

PHP-FPM 错误的系统日志。

### `/usr/local/var/log/nginx`

目录包含 Nginx 访问和错误日志。

### `/usr/local/etc/php/X.X/conf.d`

目录包含各种 PHP 配置设置的 `*.ini` 文件。

### `/usr/local/etc/php/X.X/php-fpm.d/valet-fpm.conf`

PHP-FPM 池配置文件。

### `~/.composer/vendor/laravel/valet/cli/stubs/secure.valet.conf`

用于站点构建的 SSL 证书默认 Nginx 配置文件。

### 磁盘访问

从 macOS 10.14 开始，默认情况下[限制了对某些文件和目录的访问](https://manuals.info.apple.com/MANUALS/1000/MA1902/en_US/apple-platform-security-guide.pdf)。这些限制包括桌面、文稿和下载目录。此外，网络卷和可移动卷的访问也受限。因此，Valet 建议您的站点文件夹位于这些受保护位置之外。

然而，如果你希望从其中一个位置提供站点服务，你需要授予 Nginx 「完全磁盘访问权限」。否则，你可能会遇到来自 Nginx 的服务器错误或其他不可预测的行为，特别是在提供静态资源时。通常情况下，macOS 将自动提示你授予 Nginx 对这些位置的完全访问权限。或者，也可以通过 `系统偏好设置` > `安全性与隐私` > `隐私` 并选择 `完全磁盘访问` 来手动执行此操作。接下来，在主窗格中启用任何 `nginx` 条目。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/va...](https://learnku.com/docs/laravel/11.x/valetmd/16736)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/va...](https://learnku.com/docs/laravel/11.x/valetmd/16736)