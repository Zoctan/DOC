本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Redis

+   [简介](#introduction)
+   [配置](#configuration)
    +   [集群](#clusters)
    +   [Predis](#predis)
    +   [PhpRedis](#phpredis)
+   [与 Redis 交互](#interacting-with-redis)
    +   [事物](#transactions)
    +   [管道命令](#pipelining-commands)
+   [发布 / 订阅](#pubsub)

## 简介

[Redis](https://redis.io/) 是一个开源的高级键值存储，它通常被称为数据结构服务器，因为键可以包含 [字符串【string】](https://redis.io/docs/data-types/strings/), [哈希【hashes】](https://redis.io/docs/data-types/hashes/), [列表【list】](https://redis.io/docs/data-types/lists/), [集合【sets】](https://redis.io/docs/data-types/sets/), [有序集合【sorted sets】](https://redis.io/docs/data-types/sorted-sets/).

在将 Redis 与 Laravel 结合使用之前，我们建议您通过 PECL 安装并使用 [PhpRedis](https://github.com/phpredis/phpredis) PHP 扩展。与 “用户级” PHP 软件包相比，此扩展的安装更为复杂，但对于大量使用 Redis 的应用程序而言，可能会带来更好的性能。如果您使用的是 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sail)，则此扩展已安装在应用程序的 Docker 容器中。

如果您无法安装 PhpRedis 扩展，您可以通过 Composer 安装 `predis/predis` 包。Predis 是一个完全用 PHP 编写的 Redis 客户端，不需要任何其他扩展：

```shell
composer require predis/predis:^2.0
```

## 配置

在你的应用中配置 Redis 信息，你要在 config/database.php 文件中进行配置。在该文件中，你将看到一个 Redis 数组包含了你的 Redis 配置信息。

```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'phpredis'),
    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
    ],
    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'username' => env('REDIS_USERNAME'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],
    'cache' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'username' => env('REDIS_USERNAME'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_CACHE_DB', '1'),
    ],

],
```

配置文件中定义的每个 Redis 服务器都需要具有名称、主机和端口，除非您定义单个 URL 来表示 Redis 连接：

```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'phpredis'),
    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
    ],
    'default' => [
        'url' => 'tcp://127.0.0.1:6379?database=0',
    ],
    'cache' => [
        'url' => 'tls://user:password@127.0.0.1:6380?database=1',
    ],

],
```

#### 配置连接方案

默认情况下，Redis 客户端使用 tcp 方案连接 Redis 服务器。另外，你也可以在你的 Redis 服务配置数组中指定一个 scheme 配置项，来使用 TLS/SSL 加密：

```php
'default' => [
    'scheme' => 'tls',
    'url' => env('REDIS_URL'),
    'host' => env('REDIS_HOST', '127.0.0.1'),
    'username' => env('REDIS_USERNAME'),
    'password' => env('REDIS_PASSWORD'),
    'port' => env('REDIS_PORT', '6379'),
    'database' => env('REDIS_DB', '0'),
],
```

### 集群

如果你的应用使用 Redis 集群，你应该在 Redis 配置文件中用 clusters 键来定义集群。这个配置键默认没有，所以你需要在 config/database.php 配置文件中手动创建：

```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'phpredis'),
    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
    ],
    'clusters' => [
        'default' => [
            [
                'url' => env('REDIS_URL'),
                'host' => env('REDIS_HOST', '127.0.0.1'),
                'username' => env('REDIS_USERNAME'),
                'password' => env('REDIS_PASSWORD'),
                'port' => env('REDIS_PORT', '6379'),
                'database' => env('REDIS_DB', '0'),
            ],
        ],
    ],

    // ...
],
```

默认情况下，Laravel 将使用本机 Redis 集群，`options.cluster` 配置值设置为 `redis`。Redis 集群是个默认选项，处理故障转移。

Laravel 还支持客户端分片。但是，客户端分片不处理故障转移；因此，它主要适用于可从另一个主要数据存储中获取的临时缓存数据。

如果您想使用客户端分片而不是本机 Redis 集群，删除应用程序的 `config/database.php` 配置文件中的 `options.cluster` 配置值：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    'clusters' => [
        // ...
    ],

    // ...
],
```

### Predis

要使用 Predis 扩展去连接 Redis， 请确保环境变量 REDIS\_CLIENT 的值为 predis ：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'predis'),

    // ...
],
```

除了默认配置选项外，Predis 还支持可为每个 Redis 服务器定义的其他 [连接参数](https://github.com/nrk/predis/wiki/Connection-Parameters)。要使用这些其他配置选项，请将它们添加到应用程序的 `config/database.php` 配置文件中的 Redis 服务器配置中：

```php
'default' => [
    'url' => env('REDIS_URL'),
    'host' => env('REDIS_HOST', '127.0.0.1'),
    'username' => env('REDIS_USERNAME'),
    'password' => env('REDIS_PASSWORD'),
    'port' => env('REDIS_PORT', '6379'),
    'database' => env('REDIS_DB', '0'),
    'read_write_timeout' => 60,
],
```

### PhpRedis

Laravel 默认使用 phpredis 扩展与 Redis 通信。Laravel 用于与 Redis 通信的客户端由 redis.client 配置项决定，这个配置通常为环境变量 REDIS\_CLIENT 的值：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    // ...
],
```

除了默认配置选项外，PhpRedis 支持以下附加连接参数：`name`、`persistent`、`persistent_id`、`prefix`、`read_timeout`、`retry_interval`、`timeout`、`context`。在 `config/database.php` 配置文件中修改：

```php
'default' => [
    'url' => env('REDIS_URL'),
    'host' => env('REDIS_HOST', '127.0.0.1'),
    'username' => env('REDIS_USERNAME'),
    'password' => env('REDIS_PASSWORD'),
    'port' => env('REDIS_PORT', '6379'),
    'database' => env('REDIS_DB', '0'),
    'read_timeout' => 60,
    'context' => [
        // 'auth' => ['username', 'secret'],
        // 'stream' => ['verify_peer' => false],
    ],
],
```

#### phpredis 序列化和压缩

PhpRedis 扩展还可以配置为使用各种序列化器和压缩算法。这些算法可以通过 Redis 配置的 “options” 数组进行配置：

```php
'redis' => [

    'client' => env('REDIS_CLIENT', 'phpredis'),

    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
        'serializer' => Redis::SERIALIZER_MSGPACK,
        'compression' => Redis::COMPRESSION_LZ4,
    ],

    // ...
],
```

当前支持的序列化算法包括： Redis::SERIALIZER\_NONE （默认）, Redis::SERIALIZER\_PHP , Redis::SERIALIZER\_JSON , Redis::SERIALIZER\_IGBINARY 和 Redis::SERIALIZER\_MSGPACK 。

支持的压缩算法包括： Redis::COMPRESSION\_NONE （默认）, Redis::COMPRESSION\_LZF , Redis::COMPRESSION\_ZSTD 和 Redis::COMPRESSION\_LZ4 。

## 与 Redis 交互

你可以通过调用 `Redis` [facade](https://learnku.com/docs/laravel/11.x/facades) 上的各种方法与 Redis 进行交互。`Redis` 外观支持动态方法，这意味着你可以在外观上调用任何 [Redis 命令](https://redis.io/commands)，并且该命令将直接传递给 Redis。在此示例中，我们将通过调用 `Redis` 外观上的 `get` 方法来调用 Redis `GET` 命令：

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use Illuminate\Support\Facades\Redis;
use Illuminate\View\View;

class UserController extends Controller
{
    /**
     * Show the profile for the given user.
     */
    public function show(string $id): View
    {
        return view('user.profile', [
            'user' => Redis::get('user:profile:'.$id)
        ]);
    }
}
```

你可以通过调用 `Redis` [facade](https://learnku.com/docs/laravel/11.x/facades) 上的各种方法与 Redis 进行交互。`Redis` 外观支持动态方法，这意味着您可以在外观上调用任何 [Redis 命令](https://redis.io/commands)。在此示例中，我们将通过调用 `Redis` 的 `get` 方法来调用 Redis `GET` 命令：

```php
use Illuminate\Support\Facades\Redis;

Redis::set('name', 'Taylor');

$values = Redis::lrange('names', 5, 10);
```

或者，你可以使用 `Redis` 的 `command` 方法将命令传递给服务器，该方法接受命令的名称作为其第一个参数，并接受值数组作为其第二个参数：

```php
$values = Redis::command('lrange', ['name', 5, 10]);
```

#### 使用多个 Redis 连接

应用程序的 `config/database.php` 配置文件允许您定义多个 Redis 连接 / 服务器。你可以使用 `Redis` 的 `connection` 方法获取到特定 Redis 连接的连接：

```php
$redis = Redis::connection('connection-name');
```

要获取默认 Redis 连接的实例，你可以调用 `connection` 方法而不使用任何其他参数：

```php
$redis = Redis::connection();
```

### 事务

`Redis` 的 `transaction` 方法为 Redis 的原生 `MULTI` 和 `EXEC` 命令提供了一个方便的包装器。`transaction` 方法接受一个闭包作为其唯一参数。此闭包将接收一个 Redis 连接实例，并可以向此实例发出任何命令。闭包内发出的所有 Redis 命令都将在单个原子事务中执行：

```php
use Redis;
use Illuminate\Support\Facades;

Facades\Redis::transaction(function (Redis $redis) {
    $redis->incr('user_visits', 1);
    $redis->incr('total_visits', 1);
});
```

> \[! 警告\]  
> 定义 Redis 事务时，您可能无法从 Redis 连接中检索任何值。请记住，您的事务将作为单个原子操作执行，并且该操作直到整个闭包完成其命令的执行后才会执行。

#### Lua 脚本

eval 方法提供了另外一种原子性执行多条 Redis 命令的方式。但是，eval 方法的好处是能够在操作期间与 Redis 键值交互并检查它们。 Redis 脚本是用 Lua 编程语言 编写的。

eval 方法一开始可能有点令人劝退，所以我们将用一个基本示例来明确它的使用方法。 eval 方法需要几个参数。第一，在方法中传递一个 Lua 脚本（作为一个字符串）。第二，在方法中传递脚本交互中用到的键的数量（作为一个整数）。第三，在方法中传递所有键名。最后，你可以传递一些脚本中用到的其他参数。

在本例中，我们要对第一个计数器进行递增，检查它的新值，如果该计数器的值大于 5，那么递增第二个计数器。最终，我们将返回第一个计数器的值：

```php
$value = Redis::eval(<<<'LUA'
    local counter = redis.call("incr", KEYS[1])

    if counter > 5 then
        redis.call("incr", KEYS[2])
    end

    return counter
LUA, 2, 'first-counter', 'second-counter');
```

> \[! 警告\]  
> 请参考 [Redis 文档](https://redis.io/commands/eval) 更多关于 Redis 脚本的信息。

### 管道命令

当你需要执行很多个 Redis 命令时，你可以使用 pipeline 方法一次性提交所有命令，而不需要每条命令都与 Redis 服务器建立一次网络连接。 pipeline 方法只接受一个参数：接收一个 Redis 实例的闭包。你可以将所有命令发给这个 Redis 实例，它们将同时发送到 Redis 服务器，以减少到服务器的网络访问。这些命令仍然会按照发出的顺序执行：

```php
use Redis;
use Illuminate\Support\Facades;

Facades\Redis::pipeline(function (Redis $pipe) {
    for ($i = 0; $i < 1000; $i++) {
        $pipe->set("key:$i", $i);
    }
});
```

## 发布 / 订阅

Laravel 为 Redis `publish` 和 `subscribe` 命令提供了一个方便的接口。这些 Redis 命令允许您监听给定 “频道” 上的消息。您可以从另一个应用程序向频道发布消息，甚至可以使用另一种编程语言，从而实现应用程序和进程之间的轻松通信。

首先，让我们使用 `subscribe` 方法设置一个频道监听器。我们将此方法调用放在 [Artisan 命令](https://learnku.com/docs/laravel/11.x/artisan) 中，因为调用 `subscribe` 方法会启动一个长时间运行的进程：

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Redis;

class RedisSubscribe extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'redis:subscribe';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Subscribe to a Redis channel';

    /**
     * Execute the console command.
     */
    public function handle(): void
    {
        Redis::subscribe(['test-channel'], function (string $message) {
            echo $message;
        });
    }
}
```

现在我们可以使用 `publish` 方法将消息发布到频道:

```php
use Illuminate\Support\Facades\Redis;

Route::get('/publish', function () {
    // ...

    Redis::publish('test-channel', json_encode([
        'name' => 'Adam Wathan'
    ]));
});
```

#### 通配符订阅

使用 psubscribe 方法，你可以订阅一个通配符频道，用来获取所有频道中的所有消息，频道名称将作为第二个参数传递给提供的回调闭包：

```php
Redis::psubscribe(['*'], function (string $message, string $channel) {
    echo $message;
});

Redis::psubscribe(['users.*'], function (string $message, string $channel) {
    echo $message;
});
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/re...](https://learnku.com/docs/laravel/11.x/redismd/16701)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/re...](https://learnku.com/docs/laravel/11.x/redismd/16701)