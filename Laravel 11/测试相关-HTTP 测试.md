本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## HTTP 测试

+   [简介](#introduction)
+   [创建请求](#making-requests)
    +   [自定义请求头](#customizing-request-headers)
    +   [Cookies](#cookies)
    +   [会话 / 认证](#session-and-authentication)
    +   [调试响应](#debugging-responses)
    +   [异常处理](#exception-handling)
+   [测试 JSON APIs](#testing-json-apis)
    +   [流畅 JSON 测试](#fluent-json-testing)
+   [测试文件上传](#testing-file-uploads)
+   [测试视图](#testing-views)
    +   [渲染 Blade & 组件](#rendering-blade-and-components)
+   [可用断言](#available-assertions)
    +   [响应断言](#response-assertions)
    +   [身份验证断言](#authentication-assertions)
    +   [验证断言](#validation-assertions)

## 简介

Laravel 提供了一个非常流畅的 API，用于向应用程序发出 HTTP 请求并检查响应。例如，看看下面定义的特性测试：

```php
<?php
# 译者注：Pest 例子
test('the application returns a successful response', function () {
    $response = $this->get('/');

    $response->assertStatus(200);
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 一个基础测试例子。
     */
    public function test_the_application_returns_a_successful_response(): void
    {
        $response = $this->get('/');

        $response->assertStatus(200);
    }
}
```

`get` 方法向应用程序发出 `Get` 请求，而 `assertStatus` 方法则断言返回的响应应该具有给定的 HTTP 状态代码。除了这个简单的断言之外，Laravel 还包含各种用于检查响应头、内容、JSON 结构等的断言。

## 创建请求

要向应用程序发出请求，可以在测试中调用 `get`、`post`、`put`、`patch` 或 `delete` 方法。这些方法实际上不会向应用程序发出「真正的」HTTP 请求。相反，整个网络请求是在内部模拟的。

测试请求方法不会返回 `Illuminate\Http\Response` 实例，而是返回 `Illuminate\Testing\TestResponse` 实例，该实例提供[各种有用的断言](#available-assertions) , 允许你检查应用程序的响应：

```php
<?php
# 译者注：Pest 例子
test('基础的 请求', function () {
    $response = $this->get('/');

    $response->assertStatus(200);
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 一个基础的测试例子。
     */
    public function test_a_basic_request(): void
    {
        $response = $this->get('/');

        $response->assertStatus(200);
    }
}
```

一般情况下，你的每个测试应该只向你的应用发出一次请求。如果在单个测试方法中执行多个请求，则可能会出现意料外的行为。

> **技巧**  
> 为了方便起见，运行测试时会自动禁用 CSRF 中间件。

### 自定义请求头

你可以使用此 `withHeaders` 方法自定义请求的标头，然后再将其发送到应用程序。这个方法允许你添加任何想要的自定义标头添加到请求中：

```php
<?php
# 译者注：Pest 例子
test('与请求头 交互', function () {
    $response = $this->withHeaders([
        'X-Header' => 'Value',
    ])->post('/user', ['name' => 'Sally']);

    $response->assertStatus(201);
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 一个基础的功能测试例子。
     */
    public function test_interacting_with_headers(): void
    {
        $response = $this->withHeaders([
            'X-Header' => 'Value',
        ])->post('/user', ['name' => 'Sally']);

        $response->assertStatus(201);
    }
}
```

### Cookies

在发送请求前你可能会用到 `withCookie` 或 `withCookies` 方法设置 cookie 。`withCookie` 方法接受 cookie 名称和值这两个参数，而 `withCookies` 方法接受一个名称 / 值对的数组：

```php
<?php
# 译者注：Pest 例子
test('与 cookie 交互', function () {
    $response = $this->withCookie('color', 'blue')->get('/');

    $response = $this->withCookies([
        'color' => 'blue',
        'name' => 'Taylor',
    ])->get('/');

    //
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_interacting_with_cookies(): void
    {
        $response = $this->withCookie('color', 'blue')->get('/');

        $response = $this->withCookies([
            'color' => 'blue',
            'name' => 'Taylor',
        ])->get('/');

        //
    }
}
```

### 会话 / 认证

Laravel 提供了几个可在 HTTP 测试种与 Session 交互的辅助函数。首先，你也许会用到 `withSession` 方法通过一个数组设置 session 数据。这对你的程序发送请求前加载数据到 session 非常有用：

```php
<?php
# 译者注：Pest 例子
test('与 session 交互', function () {
    $response = $this->withSession(['banned' => false])->get('/');

    //
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_interacting_with_the_session(): void
    {
        $response = $this->withSession(['banned' => false])->get('/');

        //
    }
}
```

Laravel 的 session 通常用于维护当前已验证用户的状态。因此，`actingAs` 方法提供了一种将给定用户作为当前用户进行身份验证的便捷方法。例如，我们可以使用一个[模型工厂](https://learnku.com/docs/laravel/11.x/eloquent-factoriesmd)来生成和认证一个用户：

```php
<?php
# 译者注：Pest 例子
use App\Models\User;

test('一个 需要用户认证的 行为', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
                     ->withSession(['banned' => false])
                     ->get('/');

    //
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use App\Models\User;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_an_action_that_requires_authentication(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
                         ->withSession(['banned' => false])
                         ->get('/');

        //
    }
}
```

你也可以通过传递看守器名称作为 `actingAs` 方法的第二参数以指定用户通过哪种看守器来认证。提供给 `actingAs` 方法的看守器也将成为测试期间的默认看守器。

```php
    $this->actingAs($user, 'web')
```

### 调试响应

在向你的应用程序发出测试请求之后，可以使用 `dump`、`dumpHeaders` 和 `dumpSession` 方法来检查和调试响应内容：

```php
<?php
# 译者注：Pest 例子
test('基础 测试', function () {
    $response = $this->get('/');

    $response->dumpHeaders();

    $response->dumpSession();

    $response->dump();
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 一个基础测试例子。
     */
    public function test_basic_test(): void
    {
        $response = $this->get('/');

        $response->dumpHeaders();

        $response->dumpSession();

        $response->dump();
    }
}
```

或者，你可以使用 `dd`、`ddHeaders` 和 `ddSession` 方法转储有关响应的信息，然后停止执行：

```php
<?php
# 译者注：Pest 例子
test('基础 测试', function () {
    $response = $this->get('/');

    $response->ddHeaders();

    $response->ddSession();

    $response->dd();
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 一个基础的测试例子。
     */
    public function test_basic_test(): void
    {
        $response = $this->get('/');

        $response->ddHeaders();

        $response->ddSession();

        $response->dd();
    }
}
```

### 异常处理

有时你可能想要测试你的应用程序是否引发了特定异常。为此，你需要通过 `Exceptions` facade 「伪造」异常处理逻辑。一旦异常处理逻辑被伪造，你就可以利用 `assertReported` 和 `assertNotReported` 方法对请求期间抛出的异常进行断言：

```php
<?php
# 译者注：Pest 例子
use App\Exceptions\InvalidOrderException;
use Illuminate\Support\Facades\Exceptions;

test('异常 被抛出', function () {
    Exceptions::fake();

    $response = $this->get('/order/1');

    // 断言一个异常被抛出过...
    Exceptions::assertReported(InvalidOrderException::class);

    // 断言异常...
    Exceptions::assertReported(function (InvalidOrderException $e) {
        return $e->getMessage() === '订单已经失效。';
    });
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use App\Exceptions\InvalidOrderException;
use Illuminate\Support\Facades\Exceptions;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 一个基础的测试例子。
     */
    public function test_exception_is_thrown(): void
    {
        Exceptions::fake();

        $response = $this->get('/');

        // 断言一个异常被抛出过...
        Exceptions::assertReported(InvalidOrderException::class);

        // 断言异常...
        Exceptions::assertReported(function (InvalidOrderException $e) {
            return $e->getMessage() === '订单已经失效。';
        });
    }
}
```

`assertNotReported` 和 `assertNothingReported` 方法可用于断言请求期间未抛出给定异常或没有抛出任何异常：

```php
Exceptions::assertNotReported(InvalidOrderException::class);

Exceptions::assertNothingReported();
```

你可以通过在发出请求之前调用 `withoutExceptionHandling` 方法来完全禁用给定请求的异常处理：

```php
    $response = $this->withoutExceptionHandling()->get('/');
```

此外，如果你想确保应用程序未使用 PHP 语言或应用程序正在使用的库已弃用的功能，你可以在发出请求之前调用 `withoutDeprecationHandling` 方法。当弃用处理被禁用时，弃用警告将转换为异常，从而导致您的测试失败：

```php
    $response = $this->withoutDeprecationHandling()->get('/');
```

`assertThrows` 方法可用于断言给定闭包内的代码抛出指定类型的异常：

```php
$this->assertThrows(
    fn () => (new ProcessOrder)->execute(),
    OrderInvalid::class
);
```

## 测试 JSON APIs

Laravel 也提供了几个辅助函数来测试 JSON APIs 和其响应。例如，`json`、`getJson`、`postJson`、`putJson`、`patchJson`、`deleteJson` 以及 `optionsJson` 方法可以被用于通过各种 HTTP 动作来发送 JSON 请求。你也可以轻松地将数据和请求头传递到这些方法中。首先，让我们实现一个测试示例，发送 `POST` 请求到 `/api/user`，并断言返回的期望的 JSON 数据：

```php
<?php
# 译者注：Pest 例子
test('发送一个 API 请求', function () {
    $response = $this->postJson('/api/user', ['name' => 'Sally']);

    $response
        ->assertStatus(201)
        ->assertJson([
            'created' => true,
         ]);
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 一个基础的功能测试例子。
     */
    public function test_making_an_api_request(): void
    {
        $response = $this->postJson('/api/user', ['name' => 'Sally']);

        $response
            ->assertStatus(201)
            ->assertJson([
                'created' => true,
            ]);
    }
}
```

此外，可以像数组一样访问 JSON 响应的数据，从而使你可以方便地检查 JSON 响应中返回的各个值：

```php
# 译者注：Pest 例子
expect($response['created'])->toBeTrue();
```

```php
# 译者注：PHPUnit 例子
$this->assertTrue($response['created']);
```

> **技巧**  
> `assertJson` 方法将响应转换为数组，并利用 `PHPUnit::assertArraySubset` 验证给定数组是否存在于应用程序返回的 JSON 响应中。因此，如果 JSON 响应中还有其他属性，则只要存在给定的片段，此测试仍将通过。

#### 断言 JSON 完全匹配

如前所述，`assertJson` 方法可用于断言 JSON 响应中存在 JSON 片段。如果你想验证给定数组是否与应用程序返回的 JSON **完全匹配**，则应使用 `assertExactJson` 方法：

```php
<?php
# 译者注：Pest 例子
test('断言一个 JSON 完全匹配', function () {
    $response = $this->postJson('/user', ['name' => 'Sally']);

    $response
        ->assertStatus(201)
        ->assertExactJson([
            'created' => true,
        ]);
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 一个基础功能测试例子。
     */
    public function test_asserting_an_exact_json_match(): void
    {
        $response = $this->postJson('/user', ['name' => 'Sally']);

        $response
            ->assertStatus(201)
            ->assertExactJson([
                'created' => true,
            ]);
    }
}
```

#### 断言 JSON 路径

如果你想验证 JSON 响应是否包含指定路径上的某些给定数据，可以使用 `assertJsonPath` 方法：

```php
<?php
# 译者注：Pest 例子
test('断言一个 JSON 路径的值', function () {
    $response = $this->postJson('/user', ['name' => 'Sally']);

    $response
        ->assertStatus(201)
        ->assertJsonPath('team.owner.name', 'Darian');
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 一个基础的功能测试例子。
     */
    public function test_asserting_a_json_paths_value(): void
    {
        $response = $this->postJson('/user', ['name' => 'Sally']);

        $response
            ->assertStatus(201)
            ->assertJsonPath('team.owner.name', 'Darian');
    }
}
```

`assertJsonPath` 方法也接受一个闭包，可以用来动态地确定断言是否应该通过：

```php
    $response->assertJsonPath('team.owner.name', fn (string $name) => strlen($name) >= 3);
```

### 流畅 JSON 测试

Laravel 还提供了一种漂亮的方式来流畅地测试应用程序的 JSON 响应。首先，将闭包传递给 `assertJson` 方法。这个闭包将使用 `Illuminate\Testing\Fluent\AssertableJson` 的实例调用，该实例可用于对应用程序返回的 JSON 进行断言。 `where` 方法可用于对 JSON 的特定属性进行断言，而 `missing` 方法可用于断言 JSON 中缺少特定属性：

```php
# 译者注：Pest 例子
use Illuminate\Testing\Fluent\AssertableJson;

test('流畅 JSON', function () {
    $response = $this->getJson('/users/1');

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->where('id', 1)
                 ->where('name', 'Victoria Faith')
                 ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                 ->whereNot('status', 'pending')
                 ->missing('password')
                 ->etc()
        );
});
```

```php
# 译者注：PHPUnit 例子
use Illuminate\Testing\Fluent\AssertableJson;

/**
 * 一个基础功能测试例子。
 */
public function test_fluent_json(): void
{
    $response = $this->getJson('/users/1');

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->where('id', 1)
                 ->where('name', 'Victoria Faith')
                 ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                 ->whereNot('status', 'pending')
                 ->missing('password')
                 ->etc()
        );
}
```

#### 了解 `etc` 方法

在上面的例子中，你可能已经注意到我们在断言链的末端调用了 `etc` 方法。这个方法通知 Laravel，在 JSON 对象上可能还有其他的属性存在。如果没有使用 `etc` 方法，且你没有对 JSON 对象的其他属性进行断言，测试将失败。

这种行为背后的意图是保护你不会在你的 JSON 响应中无意地暴露敏感信息，因为它迫使你明确地对该属性进行断言或通过 `etc` 方法明确地允许额外的属性。

然而，你应该意识到，在你的断言链中不包括 `etc` 方法并不能确保额外的属性不会被添加到嵌套在 JSON 对象中的数组。`etc` 方法只能确保在调用 `etc` 方法的嵌套的级别中不存在额外的属性。

#### 断言属性存在 / 不存在

要断言属性存在或不存在，可以使用 `has` 和 `missing` 方法：

```php
    $response->assertJson(fn (AssertableJson $json) =>
        $json->has('data')
             ->missing('message')
    );
```

此外，`hasAll` 和 `missingAll` 方法允许同时断言多个属性的存在或不存在：

```php
    $response->assertJson(fn (AssertableJson $json) =>
        $json->hasAll(['status', 'data'])
             ->missingAll(['message', 'code'])
    );
```

你可以使用 hasAny 方法来确定是否存在给定属性列表中的至少一个：

```php
    $response->assertJson(fn (AssertableJson $json) =>
        $json->has('status')
             ->hasAny('data', 'message', 'code')
    );
```

#### 针对 JSON 集合的断言

通常，你的路由将返回一个 JSON 响应，其中包含多个项目，例如多个用户：

```php
    Route::get('/users', function () {
        return User::all();
    });
```

在这些情况下，我们可以使用流畅 JSON 对象的 `has` 方法对响应中包含的用户进行断言。例如，让我们断言 JSON 响应包含三个用户。接下来，我们将使用 `first` 方法对集合中的第一个用户进行一些断言。 `first` 方法接受一个闭包，该闭包接收另一个可断言的 JSON 字符串，我们可以使用它来对 JSON 集合中的第一个对象进行断言：

```php
    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->has(3)
                 ->first(fn (AssertableJson $json) =>
                    $json->where('id', 1)
                         ->where('name', 'Victoria Faith')
                         ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                         ->missing('password')
                         ->etc()
                 )
        );
```

#### JSON 集合范围断言

有时，你的应用程序的路由将返回分配有命名键的 JSON 集合：

```php
    Route::get('/users', function () {
        return [
            'meta' => [...],
            'users' => User::all(),
        ];
    })
```

在测试这些路由时，你可以使用 `has` 方法来断言集合中的项目数。此外，你可以使用 `has` 方法来确定断言链的范围：

```php
    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->has('meta')
                 ->has('users', 3)
                 ->has('users.0', fn (AssertableJson $json) =>
                    $json->where('id', 1)
                         ->where('name', 'Victoria Faith')
                         ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                         ->missing('password')
                         ->etc()
                 )
        );
```

但是，你可以进行一次调用，提供一个闭包作为其第三个参数，而不是对 `has` 方法进行两次单独调用来断言 `users` 集合。这样做时，将自动调用闭包并将其范围限定为集合中的第一项：

```php
    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->has('meta')
                 ->has('users', 3, fn (AssertableJson $json) =>
                    $json->where('id', 1)
                         ->where('name', 'Victoria Faith')
                         ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                         ->missing('password')
                         ->etc()
                 )
        );
```

#### 断言 JSON 类型

你可能只想断言 JSON 响应中的属性属于某种类型。 `Illuminate\Testing\Fluent\AssertableJson` 类提供了 `whereType` 和 `whereAllType` 方法来做到这一点：

```php
    $response->assertJson(fn (AssertableJson $json) =>
        $json->whereType('id', 'integer')
             ->whereAllType([
                'users.0.name' => 'string',
                'meta' => 'array'
            ])
    );
```

你可以使用 `|` 字符指定多种类型，或者将类型数组作为第二个参数传递给 `whereType` 方法。如果响应值为任何列出的类型，则断言将成功：

```php
    $response->assertJson(fn (AssertableJson $json) =>
        $json->whereType('name', 'string|null')
             ->whereType('id', ['string', 'integer'])
    );
```

`whereType` 和 `whereAllType` 方法识别以下类型：`string`、`integer`、`double`、`boolean`、`array` 和 `null`。

## 测试文件上传

`Illuminate\Http\UploadedFile` 提供了一个 `fake` 方法用于生成虚拟的文件或者图像以供测试之用。它可以和 `Storage` facade 的 `fake` 方法相结合，大幅度简化了文件上传测试。举个例子，你可以结合这两者的功能非常方便地进行头像上传表单测试：

```php
<?php
# 译者注：Pest 例子
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

test('头像 可被上传', function () {
    Storage::fake('avatars');

    $file = UploadedFile::fake()->image('avatar.jpg');

    $response = $this->post('/avatar', [
        'avatar' => $file,
    ]);

    Storage::disk('avatars')->assertExists($file->hashName());
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_avatars_can_be_uploaded(): void
    {
        Storage::fake('avatars');

        $file = UploadedFile::fake()->image('avatar.jpg');

        $response = $this->post('/avatar', [
            'avatar' => $file,
        ]);

        Storage::disk('avatars')->assertExists($file->hashName());
    }
}
```

如果你想断言一个给定的文件不存在，则可以使用由 `Storage` facade 提供的 `AssertMissing` 方法：

```php
    Storage::fake('avatars');

    // ...

    Storage::disk('avatars')->assertMissing('missing.jpg');
```

#### 虚拟文件定制

当使用 `UploadedFile` 类提供的 `fake` 方法创建文件时，你可以指定图片的宽度、高度和大小（以千字节为单位），以便更好地测试你的应用程序的验证规则。

```php
    UploadedFile::fake()->image('avatar.jpg', $width, $height)->size(100);
```

除创建图像外，你也可以用 `create` 方法创建其他类型的文件：

```php
    UploadedFile::fake()->create('document.pdf', $sizeInKilobytes);
```

如果需要，可以向该方法传递一个 `$mimeType` 参数，以显式定义文件应返回的 MIME 类型：

```php
    UploadedFile::fake()->create(
        'document.pdf', $sizeInKilobytes, 'application/pdf'
    );
```

## 测试视图

Laravel 同样允许你在不向应用程序发出模拟 HTTP 请求的情况下独立呈现视图。为此，可以在测试中使用 `view` 方法。`view` 方法接受视图名称和一个可选的数据数组。这个方法返回一个 `Illuminate\Testing\TestView` 的实例，它提供了几个方法来方便地断言视图的内容：

```php
<?php
# 译者注：Pest 例子
test('a welcome view can be rendered', function () {
    $view = $this->view('welcome', ['name' => 'Taylor']);

    $view->assertSee('Taylor');
});
```

```php
<?php
# 译者注：PHPUnit 例子
namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_a_welcome_view_can_be_rendered(): void
    {
        $view = $this->view('welcome', ['name' => 'Taylor']);

        $view->assertSee('Taylor');
    }
}
```

`TestView` 对象提供了以下断言方法：`assertSee`、`assertSeeInOrder`、`assertSeeText`、`assertSeeTextInOrder`、`assertDontSee` 和 `assertDontSeeText`。

如果需要，你可以通过将 `TestView` 实例转换为一个由原始的视图内容组成的字符串：

```php
    $contents = (string) $this->view('welcome');
```

#### 共享错误

一些视图可能依赖于 Laravel 提供的 [全局错误包](https://learnku.com/docs/laravel/11.x/validationmd#quick-displaying-the-validation-errors) 中共享的错误。要在错误包中生成错误消息，可以使用 `withViewErrors` 方法：

```php
    $view = $this->withViewErrors([
        'name' => ['请提供一个有效的名字。']
    ])->view('form');

    $view->assertSee('请提供一个有效的名字。');
```

### 渲染 Blade & 组件

必要的话，你可以使用 `blade` 方法来运算和渲染原始的 [Blade](https://learnku.com/docs/laravel/11.x/blademd) 字符串。与 `view` 方法一样，`blade` 方法返回的是 `Illuminate\Testing\TestView` 的实例：

```php
    $view = $this->blade(
        '<x-component :name="$name" />',
        ['name' => 'Taylor']
    );

    $view->assertSee('Taylor');
```

你可以使用 `component` 方法来运算和渲染 [Blade 组件](https://learnku.com/docs/laravel/11.x/blademd#components)。`component` 方法返回一个 `Illuminate\Testing\TestComponent` 的实例：

```php
    $view = $this->component(Profile::class, ['name' => 'Taylor']);

    $view->assertSee('Taylor');
```

## 可用断言

### 响应断言

Laravel 的 `Illuminate\Testing\TestResponse` 类提供了各种自定义断言方法，你可以在测试应用程序时使用它们。可以在由 `json`、`get`、`post`、`put` 和 `delete` 方法返回的响应上访问这些断言：

#### assertBadRequest

断言响应有 错误请求（400）HTTP 状态码：

```php
    $response->assertBadRequest();
```

#### assertAccepted

断言响应有 已收处理中（202）HTTP 状态码：

```php
    $response->assertAccepted();
```

#### assertConflict

断言响应有 冲突（409）HTTP 状态码：

```php
    $response->assertConflict();
```

#### assertCookie

断言响应中包含给定的 cookie：

```php
    $response->assertCookie($cookieName, $value = null);
```

#### assertCookieExpired

断言响应包含给定的过期的 cookie：

```php
    $response->assertCookieExpired($cookieName);
```

#### assertCookieNotExpired

断言响应包含给定的未过期的 cookie：

```php
    $response->assertCookieNotExpired($cookieName);
```

#### assertCookieMissing

断言响应不包含给定的 cookie：

```php
    $response->assertCookieMissing($cookieName);
```

#### assertCreated

断言响应有（201）HTTP 状态码：

```php
    $response->assertCreated();
```

#### assertDontSee

断言给定的字符串不包含在响应中。除非传递第二个参数 `false`，否则此断言将给定字符串进行转义后匹配：

```php
    $response->assertDontSee($value, $escaped = true);
```

#### assertDontSeeText

断言给定的字符串不包含在响应文本中。除非你传递第二个参数 `false`，否则该断言将自动转义给定的字符串。该方法将在做出断言之前将响应内容传递给 PHP 的 `strip_tags` 函数：

```php
    $response->assertDontSeeText($value, $escaped = true);
```

#### assertDownload

断言是「下载」响应。通常，这意味着返回响应的调用路由返回了 `Response::download` 响应，`BinaryFileResponse` 或 `Storage::download` 响应：

```php
    $response->assertDownload();
```

如果你愿意，你可以断言可下载的文件被分配了一个给定的文件名：

```php
    $response->assertDownload('image.jpg');
```

#### assertExactJson

断言响应包含与给定 JSON 数据的完全匹配：

```php
    $response->assertExactJson(array $data);
```

#### assertForbidden

断言响应中有 禁止访问 (403) HTTP 状态码：

```php
    $response->assertForbidden();
```

#### assertFound

断言响应中有 对象已移动 (302) HTTP 状态码：

```php
    $response->assertFound();
```

#### assertGone

断言响应中有 资源已永久删除 (410) HTTP 状态码：

#### assertHeader

断言给定的标头在响应中存在：

```php
    $response->assertHeader($headerName, $value = null);
```

#### assertHeaderMissing

断言给定的标头在响应中不存在：

```php
    $response->assertHeaderMissing($headerName);
```

#### assertInternalServerError

断言响应中有「服务器内部错误」 (500) HTTP 状态码：

```php
    $response->assertInternalServerError();
```

#### assertJson

断言响应包含给定的 JSON 数据：

```php
    $response->assertJson(array $data, $strict = false);
```

`AssertJson` 方法将响应转换为数组，并利用 `PHPUnit::assertArraySubset` 验证给定数组是否存在于应用程序返回的 JSON 响应中。因此，如果 JSON 响应中还有其他属性，则只要存在给定的片段，此测试仍将通过。

#### assertJsonCount

断言响应 JSON 中有一个数组，其中包含给定键的预期元素数量：

```php
    $response->assertJsonCount($count, $key = null);
```

#### assertJsonFragment

断言响应包含给定 JSON 片段：

```php
    Route::get('/users', function () {
        return [
            'users' => [
                [
                    'name' => 'Taylor Otwell',
                ],
            ],
        ];
    });

    $response->assertJsonFragment(['name' => 'Taylor Otwell']);
```

#### assertJsonIsArray

断言响应的 JSON 是一个数组：

```php
    $response->assertJsonIsArray();
```

#### assertJsonIsObject

断言响应的 JSON 是一个对象：

```php
    $response->assertJsonIsObject();
```

#### assertJsonMissing

断言响应未包含给定的 JSON 片段：

```php
    $response->assertJsonMissing(array $data);
```

#### assertJsonMissingExact

断言响应不包含确切的 JSON 片段：

```php
    $response->assertJsonMissingExact(array $data);
```

#### assertJsonMissingValidationErrors

断言响应响应对于给定的键没有 JSON 验证错误：

```php
    $response->assertJsonMissingValidationErrors($keys);
```

> **技巧**  
> 更通用的 [assertValid](#assert-valid) 方法可用于断言响应没有以 JSON 形式返回的验证错误**并且**没有错误被闪现到会话存储中。

#### assertJsonPath

断言响应包含指定路径上的给定数据：

```php
    $response->assertJsonPath($path, $expectedValue);
```

例如，如果你的应用程序返回的 JSON 响应包含以下数据：

```json
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

你可以断言 `user` 对象的 `name` 属性匹配给定值，如下所示：

```php
    $response->assertJsonPath('user.name', 'Steve Schoger');
```

#### assertJsonMissingPath

断言响应不包含给定的 JSON 路径：

```php
    $response->assertJsonMissingPath($path);
```

例如，如果你的应用程序返回的 JSON 响应以下数据：

```json
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

你可以断言它不包含 `user` 对象的 `email` 属性：

```php
    $response->assertJsonMissingPath('user.email');
```

#### assertJsonStructure

断言响应具有给定的 JSON 结构：

```php
    $response->assertJsonStructure(array $structure);
```

例如，如果你的应用程序返回的 JSON 响应包含以下数据：

```json
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

你可以断言 JSON 结构符合你的期望，如下所示：

```php
    $response->assertJsonStructure([
        'user' => [
            'name',
        ]
    ]);
```

有时，你的应用程序返回的 JSON 响应可能包含对象数组：

```json
{
    "user": [
        {
            "name": "Steve Schoger",
            "age": 55,
            "location": "Earth"
        },
        {
            "name": "Mary Schoger",
            "age": 60,
            "location": "Earth"
        }
    ]
}
```

在这种情况下，你可以使用 `*` 字符来断言数组中所有对象的结构：

```php
    $response->assertJsonStructure([
        'user' => [
            '*' => [
                 'name',
                 'age',
                 'location'
            ]
        ]
    ]);
```

#### assertJsonValidationErrors

断言响应给定键具有给定 JSON 验证错误。在断言验证错误作为 JSON 结构返回而不是闪现到会话的响应时，应使用此方法：

```php
    $response->assertJsonValidationErrors(array $data, $responseKey = 'errors');
```

> **技巧**  
> 更通用的 [assertInvalid](#assert-invalid) 方法可用于断言响应具有以 JSON 形式返回的验证错误或错误已闪存到会话存储。

#### assertJsonValidationErrorFor

断言响应对给定键有任何 JSON 验证错误：

```php
    $response->assertJsonValidationErrorFor(string $key, $responseKey = 'errors');
```

#### assertMethodNotAllowed

断言响应为 方法被禁止 （405）HTTP 状态码：

```php
    $response->assertMethodNotAllowed();
```

#### assertMovedPermanently

断言响应 永久移动（301）HTTP 状态码：

```php
    $response->assertMovedPermanently();
```

#### assertLocation

断言响应在 `Location` 头部中具有给定的 URI 值：

```php
    $response->assertLocation($uri);
```

#### assertContent

断言给定的字符串与响应内容匹配：

```php
    $response->assertContent($value);
```

#### assertNoContent

断言响应具有给定的 HTTP 状态码且没有内容：

```php
    $response->assertNoContent($status = 204);
```

#### assertStreamedContent

断言给定的字符串与流式响应的内容相匹配：

```php
    $response->assertStreamedContent($value);
```

#### assertNotFound

断言响应 未找到（404）HTTP 状态码：

```php
    $response->assertNotFound();
```

#### assertOk

断言响应 200 HTTP 状态码：

#### assertPaymentRequired

断言响应 需付款 （402）HTTP 状态码：

```php
    $response->assertPaymentRequired();
```

#### assertPlainCookie

断言响应包含给定未加密的 cookie：

```php
    $response->assertPlainCookie($cookieName, $value = null);
```

#### assertRedirect

断言响应会重定向到给定的 URI：

```php
    $response->assertRedirect($uri = null);
```

#### assertRedirectContains

断言响应是否重定向到包含给定字符串的 URI：

```php
    $response->assertRedirectContains($string);
```

#### assertRedirectToRoute

断言响应是对给定的[命名路由](https://learnku.com/docs/laravel/11.x/routingmd#named-routes)的重定向：

```php
    $response->assertRedirectToRoute($name, $parameters = []);
```

#### assertRedirectToSignedRoute

断言响应是对给定[签名 URL](https://learnku.com/docs/laravel/11.x/urlsmd#signed-urls) 的重定向：

```php
    $response->assertRedirectToSignedRoute($name = null, $parameters = []);
```

#### assertRequestTimeout

断言响应 请求超时 （408）HTTP 状态码：

```php
    $response->assertRequestTimeout();
```

#### assertSee

断言给定的字符串包含在响应中。除非传递第二个参数 `false`，否则此断言将给定字符串自动进行转义后匹配：

```php
    $response->assertSee($value, $escaped = true);
```

#### assertSeeInOrder

断言给定的字符串按顺序包含在响应中。除非传递第二个参数 `false`，否则此断言将给定字符串自动进行转义后匹配：

```php
    $response->assertSeeInOrder(array $values, $escaped = true);
```

#### assertSeeText

断言给定字符串包含在响应文本中。除非传递第二个参数 `false`，否则此断言将给定字符串自动进行转义后匹配。在做出断言之前，响应内容将被传递到 PHP 的 `strip_tags` 函数：

```php
    $response->assertSeeText($value, $escaped = true);
```

#### assertSeeTextInOrder

断言给定的字符串按顺序包含在响应的文本中。除非传递第二个参数 `false`，否则此断言将给定字符串自动进行转义后匹配。在做出断言之前，响应内容将被传递到 PHP 的 `strip_tags` 函数：

```php
    $response->assertSeeTextInOrder(array $values, $escaped = true);
```

#### assertServerError

断言响应 服务器错误（>= 500 , < 600）HTTP 状态码：

```php
    $response->assertServerError();
```

#### assertServiceUnavailable

断言响应「服务不可用」（503）HTTP 状态码：

```php
    $response->assertServiceUnavailable();
```

#### assertSessionHas

断言 Session 包含给定的数据段：

```php
    $response->assertSessionHas($key, $value = null);
```

如果需要，可以提供一个闭包作为 `assertSessionHas` 方法的第二个参数。如果闭包返回 `true`，则断言将通过：

```php
    $response->assertSessionHas($key, function (User $value) {
        return $value->name === 'Taylor Otwell';
    });
```

#### assertSessionHasInput

session 在 [闪存输入数组](https://learnku.com/docs/laravel/11.x/responsesmd#redirecting-with-flashed-session-data) 中断言具有给定值：

```php
    $response->assertSessionHasInput($key, $value = null);
```

如果需要，可以提供一个闭包作为 `assertSessionHasInput` 方法的第二个参数。如果闭包返回 `true`，则断言将通过：

```php
    use Illuminate\Support\Facades\Crypt;

    $response->assertSessionHasInput($key, function (string $value) {
        return Crypt::decryptString($value) === 'secret';
    });
```

#### assertSessionHasAll

断言 Session 中具有给定的键 / 值对列表：

```php
    $response->assertSessionHasAll(array $data);
```

例如，如果你的应用程序会话包含 `name` 和 `status` 键，则可以断言它们存在并且具有指定的值，如下所示：

```php
    $response->assertSessionHasAll([
        'name' => 'Taylor Otwell',
        'status' => 'active',
    ]);
```

#### assertSessionHasErrors

断言 session 包含给定 `$keys` 的错误。如果 `$keys` 是关联数组，则断言 session 包含每个字段（key）的特定错误消息（value）。测试将闪存验证错误到 session 的路由时，应使用此方法，而不是将其作为 JSON 结构返回：

```php
    $response->assertSessionHasErrors(
        array $keys = [], $format = null, $errorBag = 'default'
    );
```

例如，要断言 `name` 和 `email` 字段具有已闪存到 session 的验证错误消息，可以调用 `assertSessionHasErrors` 方法，如下所示：

```php
    $response->assertSessionHasErrors(['name', 'email']);
```

或者，你可以断言给定字段具有特定的验证错误消息：

```php
    $response->assertSessionHasErrors([
        'name' => 'The given name was invalid.'
    ]);
```

> **技巧**  
> 更加通用的 [assertInvalid](#assert-invalid) 方法可以用来断言一个响应有验证错误，以 JSON 形式返回，**或**将错误被闪存到会话存储中。

#### assertSessionHasErrorsIn

断言会话在特定的[错误包](https://learnku.com/docs/laravel/11.x/validationmd#named-error-bags)中包含给定 `$keys` 的错误。如果 `$keys` 是一个关联数组，则断言该 session 在错误包内包含每个字段（键）的特定错误消息（值）：

```php
    $response->assertSessionHasErrorsIn($errorBag, $keys = [], $format = null);
```

#### assertSessionHasNoErrors

断言 session 没有验证错误：

```php
    $response->assertSessionHasNoErrors();
```

#### assertSessionDoesntHaveErrors

断言会话对给定键没有验证错误：

```php
    $response->assertSessionDoesntHaveErrors($keys = [], $format = null, $errorBag = 'default');
```

> **技巧**  
> 更加通用的 [assertValid](#assert-valid) 方法可以用来断言一个响应没有以 JSON 形式返回的验证错误，**同时**不会将错误被闪存到会话存储中。

#### assertSessionMissing

断言 session 中缺少指定的 $key：

```php
    $response->assertSessionMissing($key);
```

#### assertStatus

断言响应指定的 http 状态码：

```php
    $response->assertStatus($code);
```

#### assertSuccessful

断言响应 成功 (>= 200 且 < 300) HTTP 状态码：

```php
    $response->assertSuccessful();
```

#### assertTooManyRequests

断言响应 请求过多（429）HTTP 状态码：

```php
    $response->assertTooManyRequests();
```

#### assertUnauthorized

断言一个未认证的状态码 (401)：

```php
$response->assertUnauthorized();
```

#### assertUnprocessable

断言响应具有不可处理的实体 (422) HTTP 状态代码：

```php
$response->assertUnprocessable();
```

#### assertUnsupportedMediaType

断言响应具有不支持的媒体类型 (415) HTTP 状态码：

```php
$response->assertUnsupportedMediaType();
```

#### assertValid

断言响应对给定键没有验证错误。此方法可用于断言验证错误作为 JSON 结构返回或验证错误已闪存到会话的响应：

```php
// 断言不存在验证错误...
$response->assertValid();

// 断言给定的键没有验证错误...
$response->assertValid(['name', 'email']);
```

#### assertInvalid

断言响应对给定键有验证错误。此方法可用于断言验证错误作为 JSON 结构返回或验证错误已闪存到会话的响应：

```php
$response->assertInvalid(['name', 'email']);
```

你还可以断言给定键具有特定的验证错误消息。这样做时，你可以提供整条消息或仅提供一部分消息：

```php
$response->assertInvalid([
    'name' => 'The name field is required.',
    'email' => 'valid email address',
]);
```

#### assertViewHas

断言为响应视图提供了一个键值对数据：

```php
$response->assertViewHas($key, $value = null);
```

将闭包作为第二个参数传递给 `assertViewHas` 方法将允许你检查并针对特定的视图数据做出断言：

```php
$response->assertViewHas('user', function (User $user) {
    return $user->name === 'Taylor';
});
```

此外，视图数据可以通过响应上的数组变量进行访问，这使得你可以方便地检查它：

```php
expect($response['name'])->toBe('Taylor');
```

```php
$this->assertEquals('Taylor', $response['name']);
```

#### assertViewHasAll

断言响应视图具有给定的数据列表：

```php
$response->assertViewHasAll(array $data);
```

该方法可用于断言该视图仅包含与给定键匹配的数据：

```php
$response->assertViewHasAll([
    'name',
    'email',
]);
```

或者，你可以断言该视图数据存在并且具有特定值：

```php
$response->assertViewHasAll([
    'name' => 'Taylor Otwell',
    'email' => 'taylor@example.com,',
]);
```

#### assertViewIs

断言当前路由返回的的视图是给定的视图：

```php
$response->assertViewIs($value);
```

#### assertViewMissing

断言给定的数据键不可用于应用程序响应中返回的视图：

```php
$response->assertViewMissing($key);
```

### Authentication Assertions

Laravel 还提供了各种与身份验证相关的断言，你可以在应用程序的功能测试中使用它们。请注意，这些方法是在测试类本身上调用的，而不是由诸如 `get` 和 `post` 等方法返回的 `Illuminate\Testing\TestResponse` 实例。

#### assertAuthenticated

断言用户已通过身份验证：

```php
$this->assertAuthenticated($guard = null);
```

#### assertGuest

断言用户未通过身份验证：

```php
$this->assertGuest($guard = null);
```

#### assertAuthenticatedAs

断言特定用户已通过身份验证：

```php
$this->assertAuthenticatedAs($user, $guard = null);
```

## 验证断言

Laravel 提供了两个主要的验证相关的断言，你可以用它来确保在你的请求中提供的数据是有效或无效的。

#### assertValid

断言响应对于给定的键没有验证错误。该方法可用于断言响应中的验证错误是以 JSON 结构返回的，或者验证错误已经闪存到会话中：

```php
// 断言没有验证错误存在...
$response->assertValid();

// 断言给定的键没有验证错误...
$response->assertValid(['name', 'email']);
```

#### assertInvalid

断言响应对给定的键有验证错误。这个方法可用于断言响应中的验证错误是以 JSON 结构返回的，或者验证错误已经被闪存到会话中：

```php
$response->assertInvalid(['name', 'email']);
```

你也可以断言一个给定的键有一个特定的验证错误信息。当这样做时，你可以提供整个消息或只提供消息的一小部分：

```php
$response->assertInvalid([
    'name' => 'The name field is required.',
    'email' => 'valid email address',
]);
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/ht...](https://learnku.com/docs/laravel/11.x/http-testsmd/16710)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/ht...](https://learnku.com/docs/laravel/11.x/http-testsmd/16710)