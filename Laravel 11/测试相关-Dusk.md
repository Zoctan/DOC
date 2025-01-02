本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Dusk

+   [介绍](#introduction)
+   [安装](#installation)
    +   [管理 ChromeDriver 安装](#managing-chromedriver-installations)
    +   [使用其他浏览器](#using-other-browsers)
+   [入门指南](#getting-started)
    +   [生成测试](#generating-tests)
    +   [每次测试后重置数据库](#resetting-the-database-after-each-test)
    +   [运行测试](#running-tests)
    +   [环境处理](#environment-handling)
+   [浏览器基础](#browser-basics)
    +   [创建浏览器](#creating-browsers)
    +   [导航](#navigation)
    +   [调整浏览器窗口大小](#resizing-browser-windows)
    +   [浏览器宏](#browser-macros)
    +   [身份认证](#authentication)
    +   [Cookies](#cookies)
    +   [执行 JavaScript](#executing-javascript)
    +   [截图](#taking-a-screenshot)
    +   [将控制台输出存储到磁盘](#storing-console-output-to-disk)
    +   [将页面源代码存储到磁盘](#storing-page-source-to-disk)
+   [与元素交互](#interacting-with-elements)
    +   [Dusk 选择器](#dusk-selectors)
    +   [文本、值和属性](#text-values-and-attributes)
    +   [与表单交互](#interacting-with-forms)
    +   [附加文件](#attaching-files)
    +   [点击按钮](#pressing-buttons)
    +   [点击链接](#clicking-links)
    +   [使用键盘](#using-the-keyboard)
    +   [使用鼠标](#using-the-mouse)
    +   [JavaScript 弹框](#javascript-dialogs)
    +   [与内嵌窗口交互](#interacting-with-iframes)
    +   [范围选择器](#scoping-selectors)
    +   [等待元素](#waiting-for-elements)
    +   [滚动元素到视图中](#scrolling-an-element-into-view)
+   [可用的断言](#available-assertions)
+   [页面](#pages)
    +   [生成页面](#generating-pages)
    +   [配置页面](#configuring-pages)
    +   [导航到页面](#navigating-to-pages)
    +   [快捷选择器](#shorthand-selectors)
    +   [页面方法](#page-methods)
+   [组件](#components)
    +   [生成组件](#generating-components)
    +   [使用组件](#using-components)
+   [持续集成](#continuous-integration)
    +   [Heroku CI](#running-tests-on-heroku-ci)
    +   [Travis CI](#running-tests-on-travis-ci)
    +   [GitHub Actions](#running-tests-on-github-actions)
    +   [Chipper CI](#running-tests-on-chipper-ci)

## 介绍

[Laravel Dusk](https://github.com/laravel/dusk) 提供了一种表达性强、易于使用的浏览器自动化和测试 API。默认情况下，Dusk 不需要你在本地计算机上安装 JDK 或 Selenium。相反，Dusk 使用独立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安装。不过，你可以随便使用任何与 Selenium 兼容的驱动程序。

## 安装

要开始使用，你应该安装 [Google Chrome](https://www.google.com/chrome) 并将 `laravel/dusk` Composer 依赖项添加到你的项目中：

```shell
composer require laravel/dusk --dev
```

> \[! 警告\]  
> 如果你是手动注册 Dusk 的服务提供者，**永远不要** 在生产环境中注册它，因为这样做可能会导致任意用户能够通过你的应用程序进行认证。

安装 Dusk 包后，执行 `dusk:install` Artisan 命令。`dusk:install` 命令将创建一个 `tests/Browser` 目录和一个示例 Dusk 测试，并为你当前的操作系统安装 Chrome Driver 二进制文件：

接下来，在你的应用程序的 `.env` 文件中设置 `APP_URL` 环境变量。这个值应该与你通过浏览器访问应用程序时使用的 URL 一致。

> \[! 注意\]  
> 如果你使用 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sail) 来管理你的本地开发环境，请同时查阅 Sail 文档中关于 [配置和运行 Dusk 测试](https://learnku.com/docs/laravel/11.x/sail#laravel-dusk) 的部分。

### 管理 ChromeDriver 安装

如果你想安装一个与 Laravel Dusk 通过 `dusk:install` 命令安装的版本不同的 ChromeDriver，你可以使用 `dusk:chrome-driver` 命令：

```shell
# 为你的操作系统安装最新版本的 ChromeDriver ...
php artisan dusk:chrome-driver

# 为你的操作系统安装指定版本的 ChromeDriver ...
php artisan dusk:chrome-driver 86

# 为所有支持的操作系统安装指定版本的 ChromeDriver ...
php artisan dusk:chrome-driver --all

# 安装与你的操作系统检测到的 Chrome / Chromium 版本匹配的 ChromeDriver 版本...
php artisan dusk:chrome-driver --detect
```

> \[! 警告\]  
> Dusk 需要 `chromedriver` 二进制文件是可执行的。如果你在运行 Dusk 时遇到问题，你应该使用以下命令确保二进制文件是可执行的：`chmod -R 0755 vendor/laravel/dusk/bin/`。

### 使用其他浏览器

默认情况下，Dusk 使用 Google Chrome 和独立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安装来运行你的浏览器测试。不过，你可以启动自己的 Selenium 服务器，并针对任何你想要的浏览器运行测试。

要开始使用，请打开你的 `tests/DuskTestCase.php` 文件，这是你应用程序的基础 Dusk 测试用例。在这个文件中，你可以移除对 `startChromeDriver` 方法的调用。这将阻止 Dusk 自动启动 ChromeDriver：

```php
/**
 * 为 Dusk 测试执行做准备。
 *
 * @beforeClass
 */
public static function prepare(): void
{
    // static::startChromeDriver();
}
```

接下来，你可以修改 `driver` 方法以连接到你选择的 URL 和端口。此外，你还可以修改应该传递给 WebDriver 的「期望能力」：

```php
use Facebook\WebDriver\Remote\RemoteWebDriver;

/**
 * 创建 RemoteWebDriver 实例。
 */
protected function driver(): RemoteWebDriver
{
    return RemoteWebDriver::create(
        'http://localhost:4444/wd/hub', DesiredCapabilities::phantomjs()
    );
}
```

## 入门指南

### 生成测试

要生成一个 Dusk 测试，请使用 `dusk:make` Artisan 命令。生成的测试将放置在 `tests/Browser` 目录中：

```shell
php artisan dusk:make LoginTest
```

### 每次测试后重置数据库

你编写的大多数测试将与你带应用程序数据库检索数据的页面进行交互；但是，你的 Dusk 测试不应该使用 `RefreshDatabase` 特性。`RefreshDatabase` 特性利用了数据库事务，这些事务在跨 HTTP 请求时将不适用或不可用。然而，你有两个选择：`DatabaseMigrations` 特性和 `DatabaseTruncation` 特性。

#### 使用数据库迁移

`DatabaseMigrations` 特性将在每次测试之前运行你的数据库迁移。然而，对于每次测试，删除并重新创建数据库表通常比截断表要慢：

```php
<?php
// Pest
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;

uses(DatabaseMigrations::class);

//
```

```php
<?php
// PHPUnit
namespace Tests\Browser;

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    use DatabaseMigrations;

    //
}
```

> \[! 警告\]  
> 在执行 Dusk 测试时，SQLite 内存数据库可能不被使用。因为浏览器在其自己的进程中执行，它将无法访问其他进程的内存数据库。

#### 使用数据库截断

`DatabaseTruncation` 特性将在第一个测试中迁移你的数据库，以确保你的数据库表已正确创建。然而，在后续测试中，数据库的表将被简单地截断 —— 与重新运行所有数据库迁移相比，提供了速度提升：

```php
<?php
// Pest
use Illuminate\Foundation\Testing\DatabaseTruncation;
use Laravel\Dusk\Browser;

uses(DatabaseTruncation::class);

//
```

```php
<?php
// PHPUnit
namespace Tests\Browser;

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseTruncation;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    use DatabaseTruncation;

    //
}
```

默认情况下，此特性将截断除 `migrations` 表之外的所有表。如果你想自定义应该截断的表，可以在你的测试类上定义一个 `$tablesToTruncate` 属性：

> \[! 注意\]  
> 如果你使用 Pest，你应该在基础 `DuskTestCase` 类或你的测试文件扩展的任何类上定义属性或方法。

```php
/**
 * 指示应该截断哪些表
 *
 * @var array
 */
protected $tablesToTruncate = ['users'];
```

或者，你可以在你的测试类上定义一个 `$exceptTables` 属性，以指定哪些表不被截断：

```php
/**
 * 指示应该从截断中排除哪些表。
 *
 * @var array
 */
protected $exceptTables = ['users'];
```

要指定应该截断表所在的数据库连接，可以在你的测试类上定义一个 `$connectionsToTruncate` 属性：

```php
/**
 * 指示应该截断哪些数据库连接的表。
 *
 * @var array
 */
protected $connectionsToTruncate = ['mysql'];
```

如果你想在执行数据库截断之前或之后执行代码，可以在你的测试类上定义 `beforeTruncatingDatabase` 或 `afterTruncatingDatabase` 方法：

```php
/**
 * 在数据库开始截断之前执行任何应该完成的工作。
 */
protected function beforeTruncatingDatabase(): void
{
    //
}

/**
 * 在数据库完成截断之后执行任何应该完成的工作。
 */
protected function afterTruncatingDatabase(): void
{
    //
}
```

### 运行测试

要运行你的浏览器测试，执行 `dusk` Artisan 命令：

如果你上次运行 `dusk` 命令时出现了测试失败，你可以使用 `dusk:fails` 命令先重新运行失败的测试，以节省时间：

`dusk` 命令接受 Pest / PHPUnit 测试运行器正常接受的任何参数，例如允许你只为给定的 [组](https://docs.phpunit.de/en/10.5/annotations.html#group) 运行测试：

```shell
php artisan dusk --group=foo
```

> \[! 注意\]  
> 如果你使用 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sail) 来管理你的本地开发环境，请查阅 Sail 文档中关于 [配置和运行 Dusk 测试](https://learnku.com/docs/laravel/11.x/sail#laravel-dusk) 的部分。

#### 手动启动 ChromeDriver

默认情况下，Dusk 将自动尝试启动 ChromeDriver。如果这对你的特定系统不起作用，你可以在运行 `dusk` 命令之前手动启动 ChromeDriver。如果你选择手动启动 ChromeDriver，你应该注释掉 `tests/DuskTestCase.php` 文件中的以下行：

```php
/**
 * 为 Dusk 测试执行做准备。
 *
 * @beforeClass
 */
public static function prepare(): void
{
    // static::startChromeDriver();
}
```

此外，如果你在 9515 以外的端口上启动 ChromeDriver，你应该修改同一个类的 `driver` 方法来表明正确的端口：

```php
use Facebook\WebDriver\Remote\RemoteWebDriver;

/**
 * 创建 RemoteWebDriver 实例。
 */
protected function driver(): RemoteWebDriver
{
    return RemoteWebDriver::create(
        'http://localhost:9515', DesiredCapabilities::chrome()
    );
}
```

### 环境处理

要强制 Dusk 在运行测试时使用其自己的环境文件，请在你的项目根目录中创建一个 `.env.dusk.{environment}` 文件。例如，如果你将从 `local` 环境启动 `dusk` 命令，你应该创建一个 `.env.dusk.local` 文件。

运行测试时，Dusk 将备份你的 `.env` 文件，并将你的 Dusk 环境重命名为 `.env`。测试完成后，你的 `.env` 文件将被恢复。

## 浏览器基础

### 创建浏览器

要开始使用，让我们编写一个测试，验证我们可以登录到我们的应用程序。生成测试后，我们可以修改它导航到登录页面，输入一些凭证，并点击「登录」按钮。要创建一个浏览器实例，你可以从你的 Dusk 测试中调用 `browse` 方法：

```php
<?php
// Pest
use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;

uses(DatabaseMigrations::class);

test('basic example', function () {
    $user = User::factory()->create([
        'email' => 'taylor@laravel.com',
    ]);

    $this->browse(function (Browser $browser) use ($user) {
        $browser->visit('/login')
                ->type('email', $user->email)
                ->type('password', 'password')
                ->press('Login')
                ->assertPathIs('/home');
    });
});
```

```php
<?php
// PHPUnit
namespace Tests\Browser;

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    use DatabaseMigrations;

    /**
     * 一个基本的浏览器测试示例。
     */
    public function test_basic_example(): void
    {
        $user = User::factory()->create([
            'email' => 'taylor@laravel.com',
        ]);

        $this->browse(function (Browser $browser) use ($user) {
            $browser->visit('/login')
                    ->type('email', $user->email)
                    ->type('password', 'password')
                    ->press('Login')
                    ->assertPathIs('/home');
        });
    }
}
```

如上例所示，`browse` 方法接受一个闭包。由 Dusk 自动将浏览器实例传递给这个闭包，这是用于与你的应用程序交互和进行断言的主要对象。

#### 创建多个浏览器

有时你可能需要多个浏览器才能正确执行测试。例如，测试与 WebSocket 交互的聊天屏幕可能需要多个浏览器。要创建多个浏览器，只需在给 `browse` 方法的闭包签名中添加更多浏览器参数：

```php
$this->browse(function (Browser $first, Browser $second) {
    $first->loginAs(User::find(1))
          ->visit('/home')
          ->waitForText('Message');

    $second->loginAs(User::find(2))
           ->visit('/home')
           ->waitForText('Message')
           ->type('message', 'Hey Taylor')
           ->press('Send');

    $first->waitForText('Hey Taylor')
          ->assertSee('Jeffrey Way');
});
```

### 导航

`visit` 方法可用于在你的应用程序中导航到给定的 URI：

```php
$browser->visit('/login');
```

你可以使用 `visitRoute` 方法导航到 [命名路由](https://learnku.com/docs/laravel/11.x/routing#named-routes)：

```php
$browser->visitRoute('login');
```

你可以使用 `back` 和 `forward` 方法进行「后退」和「前进」导航：

```php
$browser->back();

$browser->forward();
```

你可以使用 `refresh` 方法刷新页面：

### 调整浏览器窗口大小

你可以使用 `resize` 方法调整浏览器窗口的大小：

```php
$browser->resize(1920, 1080);
```

`maximize` 方法可用于最大化浏览器窗口：

`fitContent` 方法将调整浏览器窗口的大小以适应其内容的大小：

当测试失败时，Dusk 会自动调整浏览器窗口的大小以适应内容，然后获取一个截图。你可以在测试中调用 `disableFitOnFailure` 方法来禁用此功能：

```php
$browser->disableFitOnFailure();
```

你可以使用 `move` 方法将浏览器窗口移动到屏幕上的不同位置：

```php
$browser->move($x = 100, $y = 100);
```

### 浏览器宏

如果你想定义一个可以在多个测试中重用的自定义浏览器方法，你可以在 `Browser` 类上使用 `macro` 方法。通常，你应该从 [服务提供者](https://learnku.com/docs/laravel/11.x/providers) 的 `boot` 方法中调用此方法：

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Laravel\Dusk\Browser;

class DuskServiceProvider extends ServiceProvider
{
    /**
     * 注册 Dusk 的浏览器宏。
     */
    public function boot(): void
    {
        Browser::macro('scrollToElement', function (string $element = null) {
            $this->script("$('html, body').animate({ scrollTop: $('$element').offset().top }, 0);");

            return $this;
        });
    }
}
```

`macro` 函数接受一个名称作为其第一个参数，一个闭包作为其第二个参数。宏的闭包将在 `Browser` 实例上调用宏方法时执行：

```php
$this->browse(function (Browser $browser) use ($user) {
    $browser->visit('/pay')
            ->scrollToElement('#credit-card-details')
            ->assertSee('Enter Credit Card Details');
});
```

### 身份认证

你经常会测试需要认证的页面。你可以使用 Dusk 的 `loginAs` 方法来避免在每次测试中与应用程序的登录界面交互。`loginAs` 方法接受与你的可认证模型关联的主键或一个可认证模型实例：

```php
use App\Models\User;
use Laravel\Dusk\Browser;

$this->browse(function (Browser $browser) {
    $browser->loginAs(User::find(1))
          ->visit('/home');
});
```

> \[! 警告\]  
> 使用 `loginAs` 方法后，用户会话（session）将在该文件中的所有测试中保持。

### Cookies

你可以使用 `cookie` 方法来获取或设置一个加密 cookie 的值。默认情况下，Laravel 创建的所有 cookie 都是加密的：

```php
$browser->cookie('name');

$browser->cookie('name', 'Taylor');
```

你可以使用 `plainCookie` 方法来获取或设置一个未加密 cookie 的值：

```php
$browser->plainCookie('name');

$browser->plainCookie('name', 'Taylor');
```

你可以使用 `deleteCookie` 方法来删除指定的 cookie：

```php
$browser->deleteCookie('name');
```

### 执行 JavaScript

你可以使用 `script` 方法在浏览器中执行任意的 JavaScript 语句：

```php
$browser->script('document.documentElement.scrollTop = 0');

$browser->script([
    'document.body.scrollTop = 0',
    'document.documentElement.scrollTop = 0',
]);

$output = $browser->script('return window.location.pathname');
```

### 截图

你可以使用 `screenshot` 方法来获取屏幕截图并使用指定的文件名存储。所有截图将存储在 `tests/Browser/screenshots` 目录中：

```php
$browser->screenshot('filename');
```

`responsiveScreenshots` 方法可用于在各种断点处截取一系列屏幕截图：

```php
$browser->responsiveScreenshots('filename');
```

`screenshotElement` 方法可用于获取页面上特定元素的屏幕截图：

```php
$browser->screenshotElement('#selector', 'filename');
```

### 将控制台输出存储到磁盘

你可以使用 `storeConsoleLog` 方法将当前浏览器的控制台输出以给定的文件名写入磁盘。控制台输出将存储在 `tests/Browser/console` 目录中：

```php
$browser->storeConsoleLog('filename');
```

### 将页面源代码存储到磁盘

你可以使用 `storeSource` 方法将当前页面的源代码以给定的文件名写入磁盘。页面源代码将存储在 `tests/Browser/source` 目录中：

```php
$browser->storeSource('filename');
```

## 与元素交互

### Dusk 选择器

选择好的 CSS 选择器来与元素交互是编写 Dusk 测试中最困难的部分之一。随着时间的推移，前端的变化可能导致以下这样的 CSS 选择器破坏你的测试：

```php
// HTML...

<button>Login</button>

// Test...

$browser->click('.login-page .container div > button');
```

Dusk 选择器允许你专注于编写有效的测试，而不是记住 CSS 选择器。要定义一个选择器，请在你的 HTML 元素中添加一个 `dusk` 属性。然后，在与 Dusk 浏览器交互时，在选择器前加上 `@` 来操作测试中的附加元素：

```php
// HTML...

<button dusk="login-button">Login</button>

// Test...

$browser->click('@login-button');
```

如果需要，你可以通过 `selectorHtmlAttribute` 方法自定义 Dusk 选择器使用的 HTML 属性。通常，这个方法应该从应用程序的 `AppServiceProvider` 的 `boot` 方法中调用：

```php
use Laravel\Dusk\Dusk;

Dusk::selectorHtmlAttribute('data-dusk');
```

### 文本、值和属性

#### 检索和设置值

Dusk 提供了多种方法来与页面上的元素的当前值、显示文本和属性进行交互。例如，要获取与给定 CSS 或 Dusk 选择器匹配的元素的「值」，请使用 `value` 方法：

```php
// 检索值...
$value = $browser->value('selector');

// 设置值...
$browser->value('selector', 'value');
```

你可以使用 `inputValue` 方法来获取具有指定字段名称的输入元素的「值」：

```php
$value = $browser->inputValue('field');
```

#### 检索文本

`text` 方法可用于检索与给定选择器匹配元素的显示文本：

```php
$text = $browser->text('selector');
```

#### 检索属性

最后，`attribute` 方法可用于检索与给定选择器匹配元素的属性的值：

```php
$attribute = $browser->attribute('selector', 'value');
```

### 与表单交互

#### 输入值

Dusk 提供了多种方法来与表单和输入元素进行交互。首先，让我们看一个在输入字段中输入文本的例子：

```php
$browser->type('email', 'taylor@laravel.com');
```

请注意，尽管该方法在必要时接受一个 CSS 选择器，但我们不需要在 `type` 方法中传递 CSS 选择器。如果没有提供 CSS 选择器，Dusk 将搜索具有给定 `name` 属性的 `input` 或 `textarea` 字段。

要在不清除其内容的情况下向字段追加文本，你可以使用 `append` 方法：

```php
$browser->type('tags', 'foo')
        ->append('tags', ', bar, baz');
```

你可以使用 `clear` 方法清除输入的值：

```php
$browser->clear('email');
```

你可以使用 `typeSlowly` 方法指示 Dusk 缓慢输入。默认情况下，Dusk 将在每次按键之间暂停 100 毫秒。要自定义按键之间的时间间隔，你可以将适当的毫秒数作为方法的第三个参数传递：

```php
$browser->typeSlowly('mobile', '+1 (202) 555-5555');

$browser->typeSlowly('mobile', '+1 (202) 555-5555', 300);
```

你可以使用 `appendSlowly` 方法缓慢追加文本：

```php
$browser->type('tags', 'foo')
        ->appendSlowly('tags', ', bar, baz');
```

#### 下拉选择

要选择 `select` 元素上可用的值，你可以使用 `select` 方法。与 `type` 方法类似，`select` 方法不需要完整的 CSS 选择器。当向 `select` 方法传递值时，你应该传递选项的底层值而不是显示文本：

```php
$browser->select('size', 'Large');
```

你可以通过省略第二个参数来选择一个随机选项：

```php
$browser->select('size');
```

通过向 `select` 方法的第二个参数提供一个数组，你可以指示该方法选择多个选项：

```php
$browser->select('categories', ['Art', 'Music']);
```

#### 复选框

要「勾选」一个复选框输入，你可以使用 `check` 方法。与其他许多与输入相关的方法类似，不需要完整的 CSS 选择器。如果找不到 CSS 选择器匹配，Dusk 将搜索具有匹配 `name` 属性的复选框：

```php
$browser->check('terms');
```

`uncheck` 方法可用于「取消勾选」一个复选框输入：

```php
$browser->uncheck('terms');
```

#### 单选按钮

要「选择」一个 `radio` 输入选项，你可以使用 `radio` 方法。与其他许多与输入相关的方法类似，不需要完整的 CSS 选择器。如果找不到 CSS 选择器匹配，Dusk 将搜索具有匹配 `name` 和 `value` 属性的 `radio` 输入：

```php
$browser->radio('size', 'large');
```

### 附加文件

`attach` 方法可用于将文件附加到 `file` 输入元素。与其他许多与输入相关的方法类似，不需要完整的 CSS 选择器。如果找不到 CSS 选择器匹配，Dusk 将搜索具有匹配 `name` 属性的 `file` 输入：

```php
$browser->attach('photo', __DIR__.'/photos/mountains.png');
```

> \[! 警告\]  
> `attach` 函数要求在你的服务器上安装并启用 `Zip` PHP 扩展。

### 点击按钮

`press` 方法可用于点击页面上的按钮元素。传递给 `press` 方法的参数可以是按钮的显示文本或 CSS / Dusk 选择器：

```php
$browser->press('Login');
```

在提交表单时，许多应用程序在按下提交按钮后会禁用该按钮，然后在表单提交的 HTTP 请求完成后重新启用该按钮。要按下按钮并等待按钮被重新启用，你可以使用 `pressAndWaitFor` 方法：

```php
// 按下按钮并等待最多 5 秒钟以重新启用它...
$browser->pressAndWaitFor('Save');

// 按下按钮并等待最多 1 秒钟以重新启用它...
$browser->pressAndWaitFor('Save', 1);
```

### 点击链接

要点击一个链接，你可以在浏览器实例上使用 `clickLink` 方法。`clickLink` 方法将点击具有指定显示文本的链接：

```php
$browser->clickLink($linkText);
```

你可以使用 `seeLink` 方法来确定页面上是否显示具有指定显示文本的链接：

```php
if ($browser->seeLink($linkText)) {
    // ...
}
```

> \[! 警告\]  
> 这些方法与 jQuery 交互。如果页面上没有 jQuery，Dusk 将自动将其注入页面，以便在测试期间可用。

### 使用键盘

`keys` 方法允许你向指定元素提供比 `type` 方法通常允许的更复杂的输入序列。例如，你可以指示 Dusk 在输入值时按住修饰键。在这个例子中，在输入与指定选择器匹配的元素时，将按住 `shift` 键输入 `taylor`。输入 `taylor` 后，将不带任何修饰键输入 `swift`：

```php
$browser->keys('selector', ['{shift}', 'taylor'], 'swift');
```

`keys` 方法的另一个有价值的用例是向应用程序的主 CSS 选择器发送「键盘快捷键」组合：

```php
$browser->keys('.app', ['{command}', 'j']);
```

> \[! 注意\]  
> 所有修饰键（如 `{command}`）都用 `{}` 字符包裹，并与 `Facebook\WebDriver\WebDriverKeys` 类中定义的常量匹配，可以在 [GitHub](https://github.com/php-webdriver/php-webdriver/blob/master/lib/WebDriverKeys.php) 上找到。

#### 流畅的键盘交互

Dusk 还提供了一个 `withKeyboard` 方法，允许你通过 `Laravel\Dusk\Keyboard` 类流畅地执行复杂的键盘交互。`Keyboard` 类提供了 `press`、`release`、`type` 和 `pause` 方法：

```php
use Laravel\Dusk\Keyboard;

$browser->withKeyboard(function (Keyboard $keyboard) {
    $keyboard->press('c')
        ->pause(1000)
        ->release('c')
        ->type(['c', 'e', 'o']);
});
```

#### 键盘宏

如果你想定义可以在整个测试套件中容易重用的自定义键盘交互，你可以使用 `Keyboard` 类提供的 `macro` 方法。通常，你应该从 [服务提供者](https://learnku.com/docs/laravel/11.x/providers) 的 `boot` 方法中调用此方法：

```php
<?php

namespace App\Providers;

use Facebook\WebDriver\WebDriverKeys;
use Illuminate\Support\ServiceProvider;
use Laravel\Dusk\Keyboard;
use Laravel\Dusk\OperatingSystem;

class DuskServiceProvider extends ServiceProvider
{
    /**
     * 注册 Dusk 的浏览器宏。
     */
    public function boot(): void
    {
        Keyboard::macro('copy', function (string $element = null) {
            $this->type([
                OperatingSystem::onMac() ? WebDriverKeys::META : WebDriverKeys::CONTROL, 'c',
            ]);

            return $this;
        });

        Keyboard::macro('paste', function (string $element = null) {
            $this->type([
                OperatingSystem::onMac() ? WebDriverKeys::META : WebDriverKeys::CONTROL, 'v',
            ]);

            return $this;
        });
    }
}
```

`macro` 函数接受一个名称作为其第一个参数，一个闭包作为其第二个参数。宏的闭包将在 `Keyboard` 实例上调用宏方法时执行：

```php
$browser->click('@textarea')
    ->withKeyboard(fn (Keyboard $keyboard) => $keyboard->copy())
    ->click('@another-textarea')
    ->withKeyboard(fn (Keyboard $keyboard) => $keyboard->paste());
```

### 使用鼠标

#### 点击元素

`click` 方法可用于点击与给定 CSS 或 Dusk 选择器匹配的元素：

```php
$browser->click('.selector');
```

`clickAtXPath` 方法可用于点击与指定 XPath 表达式匹配的元素：

```php
$browser->clickAtXPath('//div[@class = "selector"]');
```

`clickAtPoint` 方法可用于点击位于指定一对坐标处的最顶层元素，这些坐标相对于浏览器的可视区域：

```php
$browser->clickAtPoint($x = 0, $y = 0);
```

`doubleClick` 方法可用于模拟鼠标的双击：

```php
$browser->doubleClick();

$browser->doubleClick('.selector');
```

`rightClick` 方法可用于模拟鼠标的右键点击：

```php
$browser->rightClick();

$browser->rightClick('.selector');
```

`clickAndHold` 方法可用于模拟鼠标按钮被点击并按住。随后调用 `releaseMouse` 方法将取消此行为并释放鼠标按钮：

```php
$browser->clickAndHold('.selector');

$browser->clickAndHold()
        ->pause(1000)
        ->releaseMouse();
```

`controlClick` 方法可用于模拟浏览器中的 `ctrl+click` 事件：

```php
$browser->controlClick();

$browser->controlClick('.selector');
```

#### 鼠标悬停

`mouseover` 方法可用于当你需要将鼠标移动到与指定 CSS 或 Dusk 选择器匹配的元素上时：

```php
$browser->mouseover('.selector');
```

#### 拖放

`drag` 方法可用于将匹配给定选择器的元素拖动到另一个元素上：

```php
$browser->drag('.from-selector', '.to-selector');
```

或者，你可以将元素沿单一方向拖动：

```php
$browser->dragLeft('.selector', $pixels = 10);
$browser->dragRight('.selector', $pixels = 10);
$browser->dragUp('.selector', $pixels = 10);
$browser->dragDown('.selector', $pixels = 10);
```

最后，你可以通过给定的偏移量拖动元素：

```php
$browser->dragOffset('.selector', $x = 10, $y = 10);
```

### JavaScript 弹框

Dusk 提供了多种方法来与 JavaScript 对话框交互。例如，你可以使用 `waitForDialog` 方法等待 JavaScript 对话框出现。此方法接受一个可选参数，指示等待对话框出现的秒数：

```php
$browser->waitForDialog($seconds = null);
```

`assertDialogOpened` 方法可用于断言对话框已显示并包含指定的消息：

```php
$browser->assertDialogOpened('Dialog message');
```

如果 JavaScript 对话框包含一个提示，你可以使用 `typeInDialog` 方法在提示中输入一个值：

```php
$browser->typeInDialog('Hello World');
```

要通过点击「确定」按钮关闭打开的 JavaScript 对话框，你可以调用 `acceptDialog` 方法：

```php
$browser->acceptDialog();
```

要通过点击「取消」按钮关闭打开的 JavaScript 对话框，你可以调用 `dismissDialog` 方法：

```php
$browser->dismissDialog();
```

### 与内嵌窗口交互

如果你需要与 iframe 中的元素交互，你可以使用 `withinFrame` 方法。在提供给 `withinFrame` 方法的闭包中发生的所有元素交互都将限定在指定 iframe 的上下文中：

```php
$browser->withinFrame('#credit-card-details', function ($browser) {
    $browser->type('input[name="cardnumber"]', '4242424242424242')
        ->type('input[name="exp-date"]', '1224')
        ->type('input[name="cvc"]', '123')
        ->press('Pay');
});
```

### 范围选择器

有时你可能想在给定选择器的范围内执行多个操作。例如，你可能想要断言某些文本仅存在于表格中，然后在该表格中点击一个按钮。你可以使用 `with` 方法来实现这一点。在提供给 `with` 方法的闭包中执行的所有操作都将限定在原始选择器的范围内：

```php
$browser->with('.table', function (Browser $table) {
    $table->assertSee('Hello World')
          ->clickLink('Delete');
});
```

你可能偶尔需要在当前范围之外执行断言。你可以使用 `elsewhere` 和 `elsewhereWhenAvailable` 方法来实现这一点：

```php
 $browser->with('.table', function (Browser $table) {
    // 当前的作用域是 `body .table`...

    $browser->elsewhere('.page-title', function (Browser $title) {
        // 当前的作用域是 `body .page-title`...
        $title->assertSee('Hello World');
    });

    $browser->elsewhereWhenAvailable('.page-title', function (Browser $title) {
        // 当前的作用域是 `body .page-title`...
        $title->assertSee('Hello World');
    });
 });
```

### 等待元素

在测试大量使用 JavaScript 的应用程序时，通常需要在继续进行测试之前「等待」某些元素或数据变得可用。Dusk 使这变得非常简单。使用多种方法，你可以等待页面上的元素变得可见，甚至等待给定的 JavaScript 表达式评估为 `true`。

#### 等待

如果你只需要暂停测试一段时间（以毫秒为单位），请使用 `pause` 方法：

如果你只需要在给定条件为 `true` 时暂停测试，请使用 `pauseIf` 方法：

```php
$browser->pauseIf(App::environment('production'), 1000);
```

同样，如果你需要在给定条件不为 `true` 时暂停测试，可以使用 `pauseUnless` 方法：

```php
$browser->pauseUnless(App::environment('testing'), 1000);
```

#### 等待选择器

`waitFor` 方法可用于暂停测试的执行，直到与给定 CSS 或 Dusk 选择器匹配的元素显示在页面上。默认情况下，这将在抛出异常之前暂停测试最多五秒钟。如果需要，你可以将自定义超时阈值作为方法的第二个参数传递：

```php
// 等待选择器最多 5 秒...
$browser->waitFor('.selector');

// 等待选择器最多 1 秒...
$browser->waitFor('.selector', 1);
```

你也可以等待直到与指定选择器匹配的元素包含给定的文本：

```php
// 最多等待 5 秒，让选择器包含给定的文本...
$browser->waitForTextIn('.selector', 'Hello World');

// 最多等待 1 秒，让选择器包含给定的文本...
$browser->waitForTextIn('.selector', 'Hello World', 1);
```

你也可以等待直到与给定选择器匹配的元素从页面上消失：

```php
// 等待最多 5 秒，直到选择器消失...
$browser->waitUntilMissing('.selector');

// 等待最多 1 秒，直到选择器消失...
$browser->waitUntilMissing('.selector', 1);
```

或者，你可以等待与指定选择器匹配的元素被启用或禁用：

```php
// 最多等待 5 秒，直到选择器被启用...
$browser->waitUntilEnabled('.selector');

// 最多等待 1 秒，直到选择器被启用...
$browser->waitUntilEnabled('.selector', 1);

// 最多等待 5 秒，直到选择器被禁用...
$browser->waitUntilDisabled('.selector');

// 最多等待 1 秒，直到选择器被禁用...
$browser->waitUntilDisabled('.selector', 1);
```

#### 选择器可用时限定作用域

偶尔，你可能想等待与指定选择器匹配的元素出现，然后与该元素交互。例如，你可能想等待一个模态窗口变得可用，然后在模态窗口中点击「确定」按钮。`whenAvailable` 方法可用于实现这一点。在给定闭包中执行的所有元素操作都将限定在原始选择器的范围内：

```php
$browser->whenAvailable('.modal', function (Browser $modal) {
    $modal->assertSee('Hello World')
          ->press('OK');
});
```

#### 等待文本

`waitForText` 方法可用于等待指定文本显示在页面上：

```php
// 等待文本（显示）最多 5 秒...
$browser->waitForText('Hello World');

// 等待文本（显示）最多 1 秒...
$browser->waitForText('Hello World', 1);
```

你可以使用 `waitUntilMissingText` 方法等待显示的文本从页面上移除：

```php
// 最多等待 5 秒，直到文本被移除...
$browser->waitUntilMissingText('Hello World');

// 最多等待 1 秒，直到文本被移除...
$browser->waitUntilMissingText('Hello World', 1);
```

#### 等待链接

`waitForLink` 方法可用于等待指定链接文本显示在页面上：

```php
// 最多等待 5 秒，直到链接出现...
$browser->waitForLink('Create');

// 最多等待 1 秒，直到链接出现...
$browser->waitForLink('Create', 1);
```

#### 等待输入框

`waitForInput` 方法可用于等待指定输入字段显示在页面上：

```php
// 最多等待 5 秒，直到输入字段出现...
$browser->waitForInput($field);

// 最多等待 1 秒，直到输入字段出现...
$browser->waitForInput($field, 1);
```

#### 等待页面地址

在做出路径断言时，例如 `$browser->assertPathIs('/home')`，如果 `window.location.pathname` 是异步更新的，断言可能会失败。你可以使用 `waitForLocation` 方法等待位置变为给定值：

```php
$browser->waitForLocation('/secret');
```

`waitForLocation` 方法也可以用于等待当前窗口地址变为完全限定的 URL：

```php
$browser->waitForLocation('https://example.com/path');
```

你也可以等待 [命名路由](https://learnku.com/docs/laravel/11.x/routing#named-routes) 的地址：

```php
$browser->waitForRoute($routeName, $parameters);
```

#### 等待页面重新加载

如果你需要在执行操作后等待页面重新加载，请使用 `waitForReload` 方法：

```php
use Laravel\Dusk\Browser;

$browser->waitForReload(function (Browser $browser) {
    $browser->press('Submit');
})
->assertSee('Success!');
```

由于需要等待页面重新加载通常发生在点击按钮之后，你可以使用 `clickAndWaitForReload` 方法以方便地实现这一点：

```php
$browser->clickAndWaitForReload('.selector')
        ->assertSee('something');
```

#### 等待 JavaScript 表达式

有时你可能想要暂停测试的执行，直到给定的 JavaScript 表达式评估为 `true`。你可以使用 `waitUntil` 方法轻松实现这一点。当传递一个表达式给此方法时，你不需要包含 `return` 关键字或结尾的分号：

```php
// 最多等待 5 秒，直到表达式为真...
$browser->waitUntil('App.data.servers.length > 0');

// 最多等待 1 秒，直到表达式为真...
$browser->waitUntil('App.data.servers.length > 0', 1);
```

#### 等待 Vue 表达式

`waitUntilVue` 和 `waitUntilVueIsNot` 方法可用于等待 [Vue 组件](https://vuejs.org/) 属性具有指定值：

```php
// 等待直到组件属性包含给定值...
$browser->waitUntilVue('user.name', 'Taylor', '@user');

// 等待直到组件属性不包含给定值...
$browser->waitUntilVueIsNot('user.name', null, '@user');
```

#### 等待 JavaScript 事件

`waitForEvent` 方法可用于暂停测试的执行，直到发生 JavaScript 事件：

```php
$browser->waitForEvent('load');
```

事件监听器附加到当前范围，默认情况下是 `body` 元素。当使用范围选择器时，事件监听器将附加到匹配的元素上：

```php
$browser->with('iframe', function (Browser $iframe) {
    // 等待 iframe 的加载事件...
    $iframe->waitForEvent('load');
});
```

你可以将选择器作为 `waitForEvent` 方法的第二个参数提供，以将事件监听器附加到特定元素上：

```php
$browser->waitForEvent('load', '.selector');
```

你也可以等待 `document` 和 `window` 对象上的事件：

```php
// 等待直到文档被滚动...
$browser->waitForEvent('scroll', 'document');

// 最多等待 5 秒，直到窗口大小被调整...
$browser->waitForEvent('resize', 'window', 5);
```

#### 使用回调等待

Dusk 中的许多「等待」方法都依赖于底层的 `waitUsing` 方法。你可以直接使用此方法来等待给定的闭包返回 `true`。`waitUsing` 方法接受等待的最大秒数、闭包应被评估的间隔、闭包以及一个可选的失败消息：

```php
$browser->waitUsing(10, 1, function () use ($something) {
    return $something->isReady();
}, "Something wasn't ready in time.");
```

### 滚动元素到视图中

有时你可能无法点击一个元素，因为它在浏览器的可视区域之外。`scrollIntoView` 方法将滚动浏览器窗口，直到指定选择器处的元素位于视图中：

```php
$browser->scrollIntoView('.selector')
        ->click('.selector');
```

## 可以的断言

Dusk 提供了各种针对应用程序的断言。所有可用的断言都记录在下面的列表中：

#### assertTitle

断言页面标题与给定文本相同：

```php
$browser->assertTitle($title);
```

#### assertTitleContains

断言页面标题包含给定文本：

```php
$browser->assertTitleContains($title);
```

#### assertUrlIs

断言当前 URL（不包括查询字符串）与给定字符串相同：

```php
$browser->assertUrlIs($url);
```

#### assertSchemeIs

断言当前 URL scheme 与给定 scheme 相同：

```php
$browser->assertSchemeIs($scheme);
```

#### assertSchemeIsNot

断言当前 URL scheme 与给定 scheme 不同：

```php
$browser->assertSchemeIsNot($scheme);
```

#### assertHostIs

断言当前 URL 主机与给定主机相同：

```php
$browser->assertHostIs($host);
```

#### assertHostIsNot

断言当前 URL host 与给定 host 不同：

```php
$browser->assertHostIsNot($host);
```

#### assertPortIs

断言当前 URL 端口与给定端口相同：

```php
$browser->assertPortIs($port);
```

#### assertPortIsNot

断言当前 URL 端口与给定端口不同：

```php
$browser->assertPortIsNot($port);
```

#### assertPathBeginsWith

断言当前 URL 路径以给定路径开头：

```php
$browser->assertPathBeginsWith('/home');
```

#### assertPathEndsWith

断言当前 URL 路径以给定路径结尾：

```php
$browser->assertPathEndsWith('/home');
```

#### assertPathContains

断言当前 URL 路径包含给定路径：

```php
$browser->assertPathContains('/home');
```

#### assertPathIs

断言当前路径与给定路径相同：

```php
$browser->assertPathIs('/home');
```

#### assertPathIsNot

断言当前路径与给定路径不同：

```php
$browser->assertPathIsNot('/home');
```

#### assertRouteIs

断言当前 URL 与给定 [命名路由](https://learnku.com/docs/laravel/11.x/routing#named-routes) 的 URL 相同：

```php
$browser->assertRouteIs($name, $parameters);
```

#### assertQueryStringHas

断言给定的查询字符串参数存在：

```php
$browser->assertQueryStringHas($name);
```

断言给定的查询字符串参数存在并具有给定值：

```php
$browser->assertQueryStringHas($name, $value);
```

#### assertQueryStringMissing

断言给定的查询字符串参数缺失：

```php
$browser->assertQueryStringMissing($name);
```

#### assertFragmentIs

断言 URL 的当前哈希片段与给定片段相同：

```php
$browser->assertFragmentIs('anchor');
```

#### assertFragmentBeginsWith

断言 URL 的当前哈希片段以给定片段开头：

```php
$browser->assertFragmentBeginsWith('anchor');
```

#### assertFragmentIsNot

断言 URL 的当前哈希片段与给定片段不同：

```php
$browser->assertFragmentIsNot('anchor');
```

#### assertHasCookie

断言给定的加密 cookie 存在：

```php
$browser->assertHasCookie($name);
```

#### assertHasPlainCookie

断言给定的未加密 cookie 存在：

```php
$browser->assertHasPlainCookie($name);
```

#### assertCookieMissing

断言给定的加密 cookie 不存在：

```php
$browser->assertCookieMissing($name);
```

#### assertPlainCookieMissing

断言给定的未加密 cookie 不存在：

```php
$browser->assertPlainCookieMissing($name);
```

#### assertCookieValue

断言加密 cookie 具有给定值：

```php
$browser->assertCookieValue($name, $value);
```

#### assertPlainCookieValue

断言未加密 cookie 具有给定值：

```php
$browser->assertPlainCookieValue($name, $value);
```

#### assertSee

断言给定文本存在于页面上：

```php
$browser->assertSee($text);
```

#### assertDontSee

断言给定的文本不存在于页面上：

```php
$browser->assertDontSee($text);
```

#### assertSeeIn

断言给定的文本存在于选择器中:

```php
$browser->assertSeeIn($selector, $text);
```

#### assertDontSeeIn

断言给定的文本不存在于选择器中:

```php
$browser->assertDontSeeIn($selector, $text);
```

#### assertSeeAnythingIn

断言选择器中存在任何文本:

```php
$browser->assertSeeAnythingIn($selector);
```

#### assertSeeNothingIn

断言选择器中没有文本:

```php
$browser->assertSeeNothingIn($selector);
```

#### assertScript

断言给定 JavaScript 表达式的计算结果为给定值:

```php
$browser->assertScript('window.isLoaded')
        ->assertScript('document.readyState', 'complete');
```

#### assertSourceHas

断言给定的源代码存在于页面上:

```php
$browser->assertSourceHas($code);
```

#### assertSourceMissing

断言给定的源代码不存在于页面上:

```php
$browser->assertSourceMissing($code);
```

#### assertSeeLink

断言给定的链接存在于页面上:

```php
$browser->assertSeeLink($linkText);
```

#### assertDontSeeLink

断言给定的链接不存在于页面上:

```php
$browser->assertDontSeeLink($linkText);
```

#### assertInputValue

断言给定的输入字段具有给定的值:

```php
$browser->assertInputValue($field, $value);
```

#### assertInputValueIsNot

断言给定的输入字段没有给定的值:

```php
$browser->assertInputValueIsNot($field, $value);
```

#### assertChecked

断言给定的复选框被选中:

```php
$browser->assertChecked($field);
```

#### assertNotChecked

断言给定的复选框未被选中:

```php
$browser->assertNotChecked($field);
```

#### assertIndeterminate

断言给定的复选框处于不确定状态:

```php
$browser->assertIndeterminate($field);
```

#### assertRadioSelected

断言给定的单选字段已被选中:

```php
$browser->assertRadioSelected($field, $value);
```

#### assertRadioNotSelected

断言给定的单选字段未被选中：

```php
$browser->assertRadioNotSelected($field, $value);
```

#### assertSelected

断言给定的下拉框已经选中了给定的值:

```php
$browser->assertSelected($field, $value);
```

#### assertNotSelected

断言给定的下拉列表没有选中给定的值:

```php
$browser->assertNotSelected($field, $value);
```

#### assertSelectHasOptions

断言给定的值数组可供选择：

```php
$browser->assertSelectHasOptions($field, $values);
```

#### assertSelectMissingOptions

断言给定的值数组不可选:

```php
$browser->assertSelectMissingOptions($field, $values);
```

#### assertSelectHasOption

断言给定的值可以在给定的字段上选择:

```php
$browser->assertSelectHasOption($field, $value);
```

#### assertSelectMissingOption

断言给定的值不可选:

```php
$browser->assertSelectMissingOption($field, $value);
```

#### assertValue

断言与给定选择器匹配的元素具有给定的值:

```php
$browser->assertValue($selector, $value);
```

#### assertValueIsNot

断言与给定选择器匹配的元素不具有给定的值:

```php
$browser->assertValueIsNot($selector, $value);
```

#### assertAttribute

断言与给定选择器匹配的元素在提供的属性中具有指定的值:

```php
$browser->assertAttribute($selector, $attribute, $value);
```

#### assertAttributeContains

断言与给定选择器匹配的元素在提供的属性中包含指定的值：

```php
$browser->assertAttributeContains($selector, $attribute, $value);
```

#### assertAttributeDoesntContain

断言与给定选择器匹配的元素在提供的属性中不包含指定的值:

```php
$browser->assertAttributeDoesntContain($selector, $attribute, $value);
```

#### assertAriaAttribute

断言与给定选择器匹配的元素在提供的 aria 属性中具有指定的值:

```php
$browser->assertAriaAttribute($selector, $attribute, $value);
```

例如，给定标记 `<button aria-label="Add"></button>`，你可以这样对 `aria-label` 属性进行断言：

```php
$browser->assertAriaAttribute('button', 'label', 'Add')
```

#### assertDataAttribute

断言与给定选择器匹配的元素在提供的 data 属性中具有给定值：

```php
$browser->assertDataAttribute($selector, $attribute, $value);
```

例如，给定标记 `<tr id="row-1" data-content="attendees"></tr>`，你可以这样对 `data-label` 属性进行断言：

```php
$browser->assertDataAttribute('#row-1', 'content', 'attendees')
```

#### assertVisible

断言与给定选择器匹配的元素是可见的:

```php
$browser->assertVisible($selector);
```

#### assertPresent

断言与给定选择器匹配的元素在源代码中存在:

```php
$browser->assertPresent($selector);
```

#### assertNotPresent

断言与给定选择器匹配的元素在源代码中不存在:

```php
$browser->assertNotPresent($selector);
```

#### assertMissing

断言与给定选择器匹配的元素不可见:

```php
$browser->assertMissing($selector);
```

#### assertInputPresent

断言具有给定名称的输入字段存在：

```php
$browser->assertInputPresent($name);
```

#### assertInputMissing

断言具有给定名称的输入字段不存在于源代码中:

```php
$browser->assertInputMissing($name);
```

#### assertDialogOpened

断言一个带有给定消息的 JavaScript 对话框已经打开:

```php
$browser->assertDialogOpened($message);
```

#### assertEnabled

断言给定字段已启用:

```php
$browser->assertEnabled($field);
```

#### assertDisabled

断言给定的字段是禁用的:

```php
$browser->assertDisabled($field);
```

#### assertButtonEnabled

断言给定的按钮是启用的:

```php
$browser->assertButtonEnabled($button);
```

#### assertButtonDisabled

断言给定的按钮被禁用:

```php
$browser->assertButtonDisabled($button);
```

#### assertFocused

断言给定字段被聚焦：

```php
$browser->assertFocused($field);
```

#### assertNotFocused

断言给定的字段没有聚焦:

```php
$browser->assertNotFocused($field);
```

#### assertAuthenticated

断言用户已身份认证:

```php
$browser->assertAuthenticated();
```

#### assertGuest

断言用户没有身份认证:

#### assertAuthenticatedAs

断言用户以给定用户身份进行的认证：

```php
$browser->assertAuthenticatedAs($user);
```

#### assertVue

Dusk 甚至允许你对 [Vue 组件](https://vuejs.org/) 的状态进行断言。例如，假设你的应用程序包含以下 Vue 组件：

```php
// HTML...

<profile dusk="profile-component"></profile>

// 组件定义...

Vue.component('profile', {
    template: '<div>{{ user.name }}</div>',

    data: function () {
        return {
            user: {
                name: 'Taylor'
            }
        };
    }
});
```

你可以对 Vue 组件的状态进行这样的断言：

```php
// Pest
test('vue', function () {
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
                ->assertVue('user.name', 'Taylor', '@profile-component');
    });
});
```

```php
// PHPUnit
/**
 * 一个基本的 Vue 测试示例。
 */
public function test_vue(): void
{
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
                ->assertVue('user.name', 'Taylor', '@profile-component');
    });
}
```

#### assertVueIsNot

断言给定的 Vue 组件数据属性与给定值不同：

```php
$browser->assertVueIsNot($property, $value, $componentSelector = null);
```

#### assertVueContains

断言给定的 Vue 组件数据属性是一个数组并包含给定值：

```php
$browser->assertVueContains($property, $value, $componentSelector = null);
```

#### assertVueDoesntContain

断言给定的 Vue 组件数据属性是一个数组并且不包含给定值：

```php
$browser->assertVueDoesntContain($property, $value, $componentSelector = null);
```

## 页面

有时，测试需要按顺序执行多个复杂的操作。这会使你的测试更难阅读和理解。Dusk 页面允许你定义可以在给定页面上通过单个方法执行的表达性操作。页面还允许你为应用程序或单个页面定义常用选择器的快捷方式。

### 生成页面

要生成一个页面对象，请执行 `dusk:page` Artisan 命令。所有页面对象都将放置在你的应用程序的 `tests/Browser/Pages` 目录中：

```php
php artisan dusk:page Login
```

### 配置页面

默认情况下，页面有三个方法：`url`、`assert` 和 `elements`。我们现在将讨论 `url` 和 `assert` 方法。`elements` 方法将在 [下面更详细地讨论](#shorthand-selectors)。

#### `url` 方法

`url` 方法应返回表示页面的 URL 的路径。Dusk 将在浏览器中导航到该页面时使用此 URL：

```php
/**
 * 获取页面的 URL。
 */
public function url(): string
{
    return '/login';
}
```

#### `assert` 方法

`assert` 方法可以进行任何必要的断言，以验证浏览器实际上位于给定页面上。实际上不需要在此方法中放置任何内容；但是，如果你愿意，可以自由进行这些断言。导航到页面时，这些断言将自动运行：

```php
/**
 * 断言浏览器位于页面上。
 */
public function assert(Browser $browser): void
{
    $browser->assertPathIs($this->url());
}
```

### 导航到页面

页面一旦确定，你就可以使用 `visit` 方法导航到该页面：

```php
use Tests\Browser\Pages\Login;

$browser->visit(new Login);
```

有时你可能已经位于给定页面上，并且需要「加载」页面的选择器和方法到当前测试上下文中。这在按下按钮并被重定向到给定页面而没有显式导航到该页面时很常见。在这种情况下，你可以使用 `on` 方法来加载页面：

```php
use Tests\Browser\Pages\CreatePlaylist;

$browser->visit('/dashboard')
        ->clickLink('Create Playlist')
        ->on(new CreatePlaylist)
        ->assertSee('@create');
```

### 快捷选择器

页面类中的 `elements` 方法允许你为页面上的任何 CSS 选择器定义快速、易于记忆的快捷方式。例如，让我们为应用程序登录页面的「email」输入字段定义一个快捷方式：

```php
/**
 * 获取页面的元素快捷方式。
 *
 * @return array<string, string>
 */
public function elements(): array
{
    return [
        '@email' => 'input[name=email]',
    ];
}
```

一旦定义好快捷方式，你就可以在任何通常使用完整 CSS 选择器的地方使用简写选择器：

```php
$browser->type('@email', 'taylor@laravel.com');
```

#### 全局快捷选择器

安装 Dusk 后，一个基础 `Page` 类将放置在你的 `tests/Browser/Pages` 目录中。这个类包含一个 `siteElements` 方法，可用于定义应在应用程序的每个页面上可用的全局简写选择器：

```php
/**
 * 获取站点的全局元素快捷方式。
 *
 * @return array<string, string>
 */
public static function siteElements(): array
{
    return [
        '@element' => '#selector',
    ];
}
```

### 页面方法

除了页面上定义的默认方法外，你还可以定义其他方法，这些方法可以在你的测试中使用。例如，假设我们正在构建一个音乐管理应用程序。应用程序的一个页面的常见操作可能是创建播放列表。你可以在页面类上定义一个 `createPlaylist` 方法，而不是在每个测试中重写创建播放列表的逻辑：

```php
<?php

namespace Tests\Browser\Pages;

use Laravel\Dusk\Browser;
use Laravel\Dusk\Page;

class Dashboard extends Page
{
    // 其他页面方法...

    /**
     * 创建一个新的播放列表。
     */
    public function createPlaylist(Browser $browser, string $name): void
    {
        $browser->type('name', $name)
                ->check('share')
                ->press('Create Playlist');
    }
}
```

一旦定义好方法，你就可以在任何使用该页面的测试中使用它。浏览器实例将自动作为第一个参数传递给自定义页面方法：

```php
use Tests\Browser\Pages\Dashboard;

$browser->visit(new Dashboard)
        ->createPlaylist('My Playlist')
        ->assertSee('My Playlist');
```

## 组件

组件类似于 Dusk 的「页面对象」，但旨在用于在整个应用程序中重用的 UI 和功能部分，例如导航栏或通知窗口。因此，组件不绑定到特定 URL。

### 生成组件

要生成一个组件，请执行 `dusk:component` Artisan 命令。新组件放置在 `tests/Browser/Components` 目录中：

```php
php artisan dusk:component DatePicker
```

如上所示，「日期选择器」是一个可能在应用程序的多种页面上存在的组件示例。在测试套件中的数十个测试中手动编写浏览器自动化逻辑来选择日期可能会变得繁琐。相反，我们可以定义一个 Dusk 组件来表示日期选择器，从而将该逻辑封装在组件中：

```php
<?php

namespace Tests\Browser\Components;

use Laravel\Dusk\Browser;
use Laravel\Dusk\Component as BaseComponent;

class DatePicker extends BaseComponent
{
    /**
     * 获取组件的根选择器。
     */
    public function selector(): string
    {
        return '.date-picker';
    }

    /**
     * 断言浏览器页面包含该组件。
     */
    public function assert(Browser $browser): void
    {
        $browser->assertVisible($this->selector());
    }

    /**
     * 获取组件的元素快捷方式。
     *
     * @return array<string, string>
     */
    public function elements(): array
    {
        return [
            '@date-field' => 'input.datepicker-input',
            '@year-list' => 'div > div.datepicker-years',
            '@month-list' => 'div > div.datepicker-months',
            '@day-list' => 'div > div.datepicker-days',
        ];
    }

    /**
     * 选择给定的日期。
     */
    public function selectDate(Browser $browser, int $year, int $month, int $day): void
    {
        $browser->click('@date-field')
                ->within('@year-list', function (Browser $browser) use ($year) {
                    $browser->click($year);
                })
                ->within('@month-list', function (Browser $browser) use ($month) {
                    $browser->click($month);
                })
                ->within('@day-list', function (Browser $browser) use ($day) {
                    $browser->click($day);
                });
    }
}
```

### 使用组件

组件一旦定义好后，我们就可以轻松地从任何测试中在日期选择器中选择日期。而且，如果选择日期所需的逻辑发生变化，我们只需要更新组件：

```php
// Pest
<?php

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\Browser\Components\DatePicker;

uses(DatabaseMigrations::class);

test('basic example', function () {
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
                ->within(new DatePicker, function (Browser $browser) {
                    $browser->selectDate(2019, 1, 30);
                })
                ->assertSee('January');
    });
});
```

```php
// PHPUnit
<?php

namespace Tests\Browser;

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\Browser\Components\DatePicker;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    /**
     * 一个基本的组件测试示例。
     */
    public function test_basic_example(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->visit('/')
                    ->within(new DatePicker, function (Browser $browser) {
                        $browser->selectDate(2019, 1, 30);
                    })
                    ->assertSee('January');
        });
    }
}
```

## Continuous Integration

> \[! 警告\]  
> 大多数 Dusk 持续集成配置期望你的 Laravel 应用程序使用内置的 PHP 开发服务器在端口 8000 上提供服务。因此，在继续之前，你应该确保你的持续集成环境具有 `APP_URL` 环境变量值为 `http://127.0.0.1:8000`。

### Heroku CI

要在 [Heroku CI](https://www.heroku.com/continuous-integration) 上运行 Dusk 测试，请将以下 Google Chrome 构建包和脚本添加到你的 Heroku `app.json` 文件中：

```php
{
  "environments": {
    "test": {
      "buildpacks": [
        { "url": "heroku/php" },
        { "url": "https://github.com/heroku/heroku-buildpack-google-chrome" }
      ],
      "scripts": {
        "test-setup": "cp .env.testing .env",
        "test": "nohup bash -c './vendor/laravel/dusk/bin/chromedriver-linux > /dev/null 2>&1 &' && nohup bash -c 'php artisan serve --no-reload > /dev/null 2>&1 &' && php artisan dusk"
      }
    }
  }
}
```

### Travis CI

要在 [Travis CI](https://travis-ci.org/) 上运行你的 Dusk 测试，请使用以下 `.travis.yml` 配置。由于 Travis CI 不是一个图形环境，我们需要采取一些额外的步骤来启动 Chrome 浏览器。此外，我们将使用 `php artisan serve` 来启动 PHP 的内置 Web 服务器：

```ini
language: php

php:
  - 7.3

addons:
  chrome: stable

install:
  - cp .env.testing .env
  - travis_retry composer install --no-interaction --prefer-dist
  - php artisan key:generate
  - php artisan dusk:chrome-driver

before_script:
  - google-chrome-stable --headless --disable-gpu --remote-debugging-port=9222 http://localhost &
  - php artisan serve --no-reload &

script:
  - php artisan dusk
```

### GitHub Actions

如果你使用 [GitHub Actions](https://github.com/features/actions) 来运行你的 Dusk 测试，你可以使用以下配置文件作为起点。与 TravisCI 一样，我们将使用 `php artisan serve` 命令来启动 PHP 的内置 Web 服务器：

```ini
name: CI
on: [push]
jobs:

  dusk-php:
    runs-on: ubuntu-latest
    env:
      APP_URL: "http://127.0.0.1:8000"
      DB_USERNAME: root
      DB_PASSWORD: root
      MAIL_MAILER: log
    steps:
      - uses: actions/checkout@v4
      - name: Prepare The Environment
        run: cp .env.example .env
      - name: Create Database
        run: |
          sudo systemctl start mysql
          mysql --user="root" --password="root" -e "CREATE DATABASE \`my-database\` character set UTF8mb4 collate utf8mb4_bin;"
      - name: Install Composer Dependencies
        run: composer install --no-progress --prefer-dist --optimize-autoloader
      - name: Generate Application Key
        run: php artisan key:generate
      - name: Upgrade Chrome Driver
        run: php artisan dusk:chrome-driver --detect
      - name: Start Chrome Driver
        run: ./vendor/laravel/dusk/bin/chromedriver-linux &
      - name: Run Laravel Server
        run: php artisan serve --no-reload &
      - name: Run Dusk Tests
        run: php artisan dusk
      - name: Upload Screenshots
        if: failure()
        uses: actions/upload-artifact@v2
        with:
          name: screenshots
          path: tests/Browser/screenshots
      - name: Upload Console Logs
        if: failure()
        uses: actions/upload-artifact@v2
        with:
          name: console
          path: tests/Browser/console
```

### Chipper CI

如果你使用 [Chipper CI](https://chipperci.com/) 来运行你的 Dusk 测试，你可以使用以下配置文件作为起点。我们将使用 PHP 的内置服务器来运行 Laravel，以便我们可以监听请求：

```ini
# .chipperci.yml 文件
version: 1

environment:
  php: 8.2
  node: 16

# 在构建环境中包含 Chrome
services:
  - dusk

# 构建所有提交
on:
   push:
      branches: .*

pipeline:
  - name: Setup
    cmd: |
      cp -v .env.example .env
      composer install --no-interaction --prefer-dist --optimize-autoloader
      php artisan key:generate

      # 创建一个 dusk 环境文件，确保 APP_URL 使用 BUILD_HOST：
      cp -v .env .env.dusk.ci
      sed -i "s@APP_URL=.*@APP_URL=http://$BUILD_HOST:8000@g" .env.dusk.ci

  - name: Compile Assets
    cmd: |
      npm ci --no-audit
      npm run build

  - name: Browser Tests
    cmd: |
      php -S [::0]:8000 -t public 2>server.log &
      sleep 2
      php artisan dusk:chrome-driver $CHROME_DRIVER
      php artisan dusk --env=ci
```

要了解更多关于在 Chipper CI 上运行 Dusk 测试的信息，包括如何使用数据库，请查阅 [官方 Chipper CI 文档](https://chipperci.com/docs/testing/laravel-dusk-new/)。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/du...](https://learnku.com/docs/laravel/11.x/duskmd/16712)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/du...](https://learnku.com/docs/laravel/11.x/duskmd/16712)