本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Facades

+   [介绍](#introduction)
+   [何时使用 Facades](#when-to-use-facades)
    +   [Facades vs 依赖注入](#facades-vs-dependency-injection)
    +   [Facades vs 辅助函数](#facades-vs-helper-functions)
+   [Facades 原理](#how-facades-work)
+   [实时 Facades](#real-time-facades)
+   [Facade 类引用](#facade-class-reference)

## 介绍

贯穿整个 Laravel 文档，可以看到许多通过 Facades 与 Laravel 功能交互的代码示例。Facades 为应用 [服务容器](https://learnku.com/docs/laravel/11.x/containermd) 中的类提供「静态」接口。 并且 Laravel 自带了许多 Facades, 它们几乎可以调用到 Laravel 中所有的功能。

Laravel Facades 充当服务容器中底层类的「静态代理」, 提供简洁、富有表现力的语法的优点，同时保持比传统静态方法更高的可测试性和灵活性。如果你不完全理解 Facades 是如何工作的，那也没关系 - 只需正常学习 Laravel 即可，后面自然而然就会明白了。

Laravel 的所有 Facades 都定义在 `Illuminate\Support\Facades` 命名空间。可以像这样访问 Facades：

```php
    use Illuminate\Support\Facades\Cache;
    use Illuminate\Support\Facades\Route;

    Route::get('/cache', function () {
        return Cache::get('key');
    });
```

#### 辅助函数

为了补充 Facades 能力，Laravel 提供了全局「辅助函数」, 和常见一些功能交互变得更加容易，常见辅助函数有 `view` 、`response` 、`url` 、`config` 等。 Laravel 提供的每个辅助函数都有其相应作用；[帮助文档](https://learnku.com/docs/laravel/11.x/helpersmd) 提供了完整的列表可供查看所有辅助函数。

例如，可以使用 `response` 函数，而不是使用 `Illuminate\Support\Facades\Response` Facade 来生成 JSON 响应。辅助函数是全局可用的，无需导入任何类即可使用：

```php
    use Illuminate\Support\Facades\Response;

    Route::get('/users', function () {
        return Response::json([
            // ...
        ]);
    });

    Route::get('/users', function () {
        return response()->json([
            // ...
        ]);
    });
```

## 何时使用 Facades

Facades 提供了简洁、易于记忆的语法，无需记住要手动注入或配置的长类名。此外，由于它是 PHP 动态调用的，很容易测试。

然而，在使用 Facades 时必须小心。它的主要危险是类的「作用范围扩张」。Facades 的易于使用且不需要注入，这就导致在类的持续开发过程中，很容易不知不觉用到许多的 Facades。使用依赖注入时，这种潜在问题通过构造函数变得更为明显，类慢慢变得越来越大。所以，在使用 Facades 时，需要特别关注类的大小，以便它保持责任范围单一，如果类变得太大，请考虑将它拆分成多个较小的类。

### Facades vs. 依赖注入

依赖注入的主要好处之一是能够交换注入类的实现。这在测试时非常有用，可以注入 mock 或 stub，并断言在 stub 上调用了各种方法。

通常，无法 mock 或 stub 一个真正的静态类方法。但是，由于 Laravel 的 Facades 使用动态方法来代理对从服务容器中解析出来的对象的方法调用，实际上可以像测试注入的类实例一样测试 Facades。例如，给定以下路由：

```php
    use Illuminate\Support\Facades\Cache;

    Route::get('/cache', function () {
        return Cache::get('key');
    });
```

使用 Laravel 的 Facade 测试方法，可以编写以下测试来验证 `Cache::get` 方法是否如期望的参数被调用：

```php
// Pest
use Illuminate\Support\Facades\Cache;

test('basic example', function () {
    Cache::shouldReceive('get')
         ->with('key')
         ->andReturn('value');

    $response = $this->get('/cache');

    $response->assertSee('value');
});
```

```php
// PHPUnit
use Illuminate\Support\Facades\Cache;

/**
 * 一个简单的功能测试示例
 */
public function test_basic_example(): void
{
    Cache::shouldReceive('get')
         ->with('key')
         ->andReturn('value');

    $response = $this->get('/cache');

    $response->assertSee('value');
}
```

### Facades vs. 辅助函数

除了 Facades，Laravel 还包含许多 「辅助」函数，这些函数可以执行常见的任务，如生成视图、触发事件、调度任务或发送 HTTP 响应。许多这些辅助函数与对应的 Facades 执行相同的功能。例如，以下 Facade 调用和辅助函数调用是等效的：

```php
    return Illuminate\Support\Facades\View::make('profile');

    return view('profile');
```

Facades 和辅助函数在实际上没有任何区别。当使用辅助函数时，仍然可以像测试对应的 Facade 那样来测试它们。例如，给定以下路由：

```php

    Route::get('/cache', function () {
        return cache('key');
    });
```

`cache` 辅助函数会调用 `Cache` Facade 底层类的 `get` 方法。因此，即使使用的是辅助函数，也可以用下面的测试来验证该方法是否如期望的参数被调用：

```php
    use Illuminate\Support\Facades\Cache;

    /**
     * 一个简单的功能测试示例。
     */
    public function test_basic_example(): void
    {
        Cache::shouldReceive('get')
             ->with('key')
             ->andReturn('value');

        $response = $this->get('/cache');

        $response->assertSee('value');
    }
```

## Facades 原理

在 Laravel 应用中，Facade 是一个类，它提供了对容器中对象的访问。使这一切工作的机制位于 `Facade` 类中。Laravel 的 Facades, 以及你创建的任何自定义 Facades, 都将扩展基础的 `Illuminate\Support\Facades\Facade` 类。

`Facade` 基础类使用 `__callStatic()` 魔术方法来将 Facade 上的调用委托给从容器中解析出来的对象。在下面的例子中，对 Laravel 缓存系统的调用被做出了。通过查看这段代码，你可能会认为 `Cache` 类上的静态 `get` 方法被调用了：

```php
    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Support\Facades\Cache;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * 显示给定用户的个人资料。
         */
        public function showProfile(string $id): View
        {
            $user = Cache::get('user:'.$id);

            return view('profile', ['user' => $user]);
        }
    }
```

注意观察上面的代码，在顶部 「导入」了 `Cache` Facade。这个 Facade 作为代理来访问 `Illuminate\Contracts\Cache\Factory` 接口的底层实现。使用 Facade 进行的任何调用都将传递给底层实例的相应方法。由于 Facade 使用了 Laravel 的服务容器来解析这些对象，因此可以轻松地在容器中模拟或替换这些对象以进行测试。

如果查看 `Illuminate\Support\Facades\Cache` 类，就会发现并没有静态方法 `get`：

```php
    class Cache extends Facade
    {
        /**
         * Get the registered name of the component.
         */
        protected static function getFacadeAccessor(): string
        {
            return 'cache';
        }
    }
```

相反，`Cache` Facade 继承自基础的 `Facade` 类，并定义了 `getFacadeAccessor()` 方法。这个方法的职责是返回服务容器绑定的名称。当用户引用 `Cache` Facade 上的任何静态方法时，Laravel 会从[服务容器](https://learnku.com/docs/laravel/11.x/containermd)中解析出 `cache` 绑定，并在那个对象上运行请求的方法（在本例中为 `get`）。

## 实时 Facades

使用实时 Facades, 可以将应用程序中的任何类当作 Facade 来处理。为了说明这如何被使用，首先查看一些没有使用实时 Facades 的代码。例如，假设 `Podcast` 模型有一个 `publish` 方法。但是，为了发布播客，需要注入一个 `Publisher` 实例：

```php
    <?php

    namespace App\Models;

    use App\Contracts\Publisher;
    use Illuminate\Database\Eloquent\Model;

    class Podcast extends Model
    {
        /**
         * 发布播客。
         */
        public function publish(Publisher $publisher): void
        {
            $this->update(['publishing' => now()]);

            $publisher->publish($this);
        }
    }
```

将发布者实现注入方法中可以让我们轻松地隔离测试该方法，因为我们可以模拟被注入的发布者。然而，它要求我们每次调用 `publish` 方法时总是传递一个发布者实例。使用实时 Facades，我们可以在不需要显式传递 Publisher 实例的情况下保持相同的可测试性。为了生成一个实时 facade，将导入类的命名空间前缀加上 Facades：

```php
    <?php

    namespace App\Models;

    use App\Contracts\Publisher; // [tl! remove]
    use Facades\App\Contracts\Publisher; // [tl! add]
    use Illuminate\Database\Eloquent\Model;

    class Podcast extends Model
    {
        /**
         * 发布播客。
         */
        public function publish(Publisher $publisher): void // [tl! remove]
        public function publish(): void // [tl! add]
        {
            $this->update(['publishing' => now()]);

            $publisher->publish($this); // [tl! remove]
            Publisher::publish($this); // [tl! add]
        }
    }
```

当使用实时 Facade 时，Publisher 的实现将从服务容器中解析出来，解析的依据是接口或类名中出现在 `Facades` 前缀之后的部分。在测试时，可以使用 Laravel 内置的 Facade 测试辅助函数来模拟这个方法调用：

```php
<?php
// Pest
use App\Models\Podcast;
use Facades\App\Contracts\Publisher;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('podcast can be published', function () {
    $podcast = Podcast::factory()->create();

    Publisher::shouldReceive('publish')->once()->with($podcast);

    $podcast->publish();
});
```

```php
<?php
// PHPUnit
namespace Tests\Feature;

use App\Models\Podcast;
use Facades\App\Contracts\Publisher;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PodcastTest extends TestCase
{
    use RefreshDatabase;

    /**
     * 测试例子
     */
    public function test_podcast_can_be_published(): void
    {
        $podcast = Podcast::factory()->create();

        Publisher::shouldReceive('publish')->once()->with($podcast);

        $podcast->publish();
    }
}
```

## Facade 类引用

下面表格列出每个 Facade 及其底层类，可以快速查阅给定 Facade 根目录的 API 文档，还包含了 `服务容器绑定` 的键名称 (key)。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/fa...](https://learnku.com/docs/laravel/11.x/facadesmd/16656)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/fa...](https://learnku.com/docs/laravel/11.x/facadesmd/16656)