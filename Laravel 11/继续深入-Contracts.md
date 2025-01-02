本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## 契约（Contract）

+   [简介](#introduction)
    +   [Contract 对比 Facade](#contracts-vs-facades)
+   [何时使用 Contract](#when-to-use-contracts)
+   [如何使用 Contract](#how-to-use-contracts)
+   [Contract 参考](#contract-reference)

## 简介

Laravel 的「契约（Contract）」是一组接口，它们定义由框架提供的核心服务。例如，`illuste\Contracts\Queue\Queue` Contract 定义了队列所需的方法，而 `illuste\Contracts\Mail\Mailer` Contract 定义了发送邮件所需的方法。

每个契约都有由框架提供的相应实现。例如，Laravel 提供了一个支持各种驱动的队列实现，还有一个由 [Symfony Mailer](https://symfony.com/doc/7.0/mailer.html) 提供支持的邮件程序实现等等。

所有的 Laravel Contract 都存在于它们各自的 [GitHub 仓库](https://github.com/illuminate/contracts)。这为所有可用的契约提供了一个快速的参考点，同时也可做为一个解耦的独立组件用于开发需要与 Laravel 交互的第三方包。

### Contract 对比 Facade

Laravel 的 [Facades](https://learnku.com/docs/laravel/11.x/facades) 和辅助函数提供了一种利用 Laravel 服务的简单方法，无需类型提示并可以从服务容器中解析 Contract。在大多数情况下，每个 Facade 都有一个等效的 Contract。

和 Facade（不需要在构造函数中引入）不同，Contract 允许你为类定义显式依赖关系。一些开发者更喜欢以这种方式显式定义其依赖项，所以更喜欢使用 Contract，而其他开发者则享受 Facade 带来的便利。**通常，大多数应用都可以在开发过程中使用 Facade。**

## 何时使用 Contract

使用 Contract 或 Facades 取决于个人喜好和开发团队的喜好。Contract 和 Facade 均可用于创建功能强大且经过良好测试的 Laravel 应用。Contract 和 Facade 并不是一道单选题，你可以在同一个应用内同时使用 Contract 和 Facade。只要你在类中贯彻「单一职责」原则，你会发现 Contract 和 Facade 的实际差异其实很小。

通常情况下，大部分使用 Facade 的应用都不会在开发中遇到问题。但如果你在建立一个可以由多个 PHP 框架使用的扩展包，你可能会希望使用 `illuminate/contracts` 扩展包来定义该包和 Laravel 集成，而不需要引入完整的 Laravel 实现（不需要在 `composer.json` 中具体显式引入 Laravel 框架来实现）。

## 如何使用 Contract

那么，如何实现契约呢？它其实很简单。

Laravel 中的许多类都是通过 [服务容器](https://learnku.com/docs/laravel/11.x/container) 解析的，包括控制器、事件侦听器、中间件、队列任务，甚至路由闭包。因此，要实现契约，你只需要在被解析的类的构造函数中写好「类型提示」接口。

例如，看看下面的这个事件监听器：

```php
<?php

namespace App\Listeners;

use App\Events\OrderWasPlaced;
use App\Models\User;
use Illuminate\Contracts\Redis\Factory;

class CacheOrderInformation
{
    /**
     * 创建一个新的事件监听器实例
     */
    public function __construct(
        protected Factory $redis,
    ) {}

    /**
     * 处理该事件。
     */
    public function handle(OrderWasPlaced $event): void
    {
        // ...
    }
}
```

当解析事件监听器时，服务容器将读取构造函数上的类型提示，并注入适当的值。 要了解更多有关在服务容器中注册内容的信息，请查看 [其文档](https://learnku.com/docs/laravel/11.x/container).

## Contract 参考

下表提供了所有 Laravel Contract 及对应的 Facade 的便捷参考：

| Contract | 对应的 Facade |
| --- | --- |
| [Illuminate\\Contracts\\Auth\\Access\\Authorizable](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/Access/Authorizable.php) |    |
| [Illuminate\\Contracts\\Auth\\Access\\Gate](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/Access/Gate.php) | `Gate` |
| [Illuminate\\Contracts\\Auth\\Authenticatable](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/Authenticatable.php) |    |
| [Illuminate\\Contracts\\Auth\\CanResetPassword](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/CanResetPassword.php) |   |
| [Illuminate\\Contracts\\Auth\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/Factory.php) | `Auth` |
| [Illuminate\\Contracts\\Auth\\Guard](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/Guard.php) | `Auth::guard()` |
| [Illuminate\\Contracts\\Auth\\PasswordBroker](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/PasswordBroker.php) | `Password::broker()` |
| [Illuminate\\Contracts\\Auth\\PasswordBrokerFactory](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/PasswordBrokerFactory.php) | `Password` |
| [Illuminate\\Contracts\\Auth\\StatefulGuard](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/StatefulGuard.php) |   |
| [Illuminate\\Contracts\\Auth\\SupportsBasicAuth](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/SupportsBasicAuth.php) |   |
| [Illuminate\\Contracts\\Auth\\UserProvider](https://github.com/illuminate/contracts/blob/laravel/11.x/Auth/UserProvider.php) |   |
| [Illuminate\\Contracts\\Bus\\Dispatcher](https://github.com/illuminate/contracts/blob/laravel/11.x/Bus/Dispatcher.php) | `Bus` |
| [Illuminate\\Contracts\\Bus\\QueueingDispatcher](https://github.com/illuminate/contracts/blob/laravel/11.x/Bus/QueueingDispatcher.php) | `Bus::dispatchToQueue()` |
| [Illuminate\\Contracts\\Broadcasting\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/Broadcasting/Factory.php) | `Broadcast` |
| [Illuminate\\Contracts\\Broadcasting\\Broadcaster](https://github.com/illuminate/contracts/blob/laravel/11.x/Broadcasting/Broadcaster.php) | `Broadcast::connection()` |
| [Illuminate\\Contracts\\Broadcasting\\ShouldBroadcast](https://github.com/illuminate/contracts/blob/laravel/11.x/Broadcasting/ShouldBroadcast.php) |   |
| [Illuminate\\Contracts\\Broadcasting\\ShouldBroadcastNow](https://github.com/illuminate/contracts/blob/laravel/11.x/Broadcasting/ShouldBroadcastNow.php) |   |
| [Illuminate\\Contracts\\Cache\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/Cache/Factory.php) | `Cache` |
| [Illuminate\\Contracts\\Cache\\Lock](https://github.com/illuminate/contracts/blob/laravel/11.x/Cache/Lock.php) |   |
| [Illuminate\\Contracts\\Cache\\LockProvider](https://github.com/illuminate/contracts/blob/laravel/11.x/Cache/LockProvider.php) |   |
| [Illuminate\\Contracts\\Cache\\Repository](https://github.com/illuminate/contracts/blob/laravel/11.x/Cache/Repository.php) | `Cache::driver()` |
| [Illuminate\\Contracts\\Cache\\Store](https://github.com/illuminate/contracts/blob/laravel/11.x/Cache/Store.php) |   |
| [Illuminate\\Contracts\\Config\\Repository](https://github.com/illuminate/contracts/blob/laravel/11.x/Config/Repository.php) | `Config` |
| [Illuminate\\Contracts\\Console\\Application](https://github.com/illuminate/contracts/blob/laravel/11.x/Console/Application.php) |   |
| [Illuminate\\Contracts\\Console\\Kernel](https://github.com/illuminate/contracts/blob/laravel/11.x/Console/Kernel.php) | `Artisan` |
| [Illuminate\\Contracts\\Container\\Container](https://github.com/illuminate/contracts/blob/laravel/11.x/Container/Container.php) | `App` |
| [Illuminate\\Contracts\\Cookie\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/Cookie/Factory.php) | `Cookie` |
| [Illuminate\\Contracts\\Cookie\\QueueingFactory](https://github.com/illuminate/contracts/blob/laravel/11.x/Cookie/QueueingFactory.php) | `Cookie::queue()` |
| [Illuminate\\Contracts\\Database\\ModelIdentifier](https://github.com/illuminate/contracts/blob/laravel/11.x/Database/ModelIdentifier.php) |   |
| [Illuminate\\Contracts\\Debug\\ExceptionHandler](https://github.com/illuminate/contracts/blob/laravel/11.x/Debug/ExceptionHandler.php) |   |
| [Illuminate\\Contracts\\Encryption\\Encrypter](https://github.com/illuminate/contracts/blob/laravel/11.x/Encryption/Encrypter.php) | `Crypt` |
| [Illuminate\\Contracts\\Events\\Dispatcher](https://github.com/illuminate/contracts/blob/laravel/11.x/Events/Dispatcher.php) | `Event` |
| [Illuminate\\Contracts\\Filesystem\\Cloud](https://github.com/illuminate/contracts/blob/laravel/11.x/Filesystem/Cloud.php) | `Storage::cloud()` |
| [Illuminate\\Contracts\\Filesystem\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/Filesystem/Factory.php) | `Storage` |
| [Illuminate\\Contracts\\Filesystem\\Filesystem](https://github.com/illuminate/contracts/blob/laravel/11.x/Filesystem/Filesystem.php) | `Storage::disk()` |
| [Illuminate\\Contracts\\Foundation\\Application](https://github.com/illuminate/contracts/blob/laravel/11.x/Foundation/Application.php) | `App` |
| [Illuminate\\Contracts\\Hashing\\Hasher](https://github.com/illuminate/contracts/blob/laravel/11.x/Hashing/Hasher.php) | `Hash` |
| [Illuminate\\Contracts\\Http\\Kernel](https://github.com/illuminate/contracts/blob/laravel/11.x/Http/Kernel.php) |   |
| [Illuminate\\Contracts\\Mail\\MailQueue](https://github.com/illuminate/contracts/blob/laravel/11.x/Mail/MailQueue.php) | `Mail::queue()` |
| [Illuminate\\Contracts\\Mail\\Mailable](https://github.com/illuminate/contracts/blob/laravel/11.x/Mail/Mailable.php) |   |
| [Illuminate\\Contracts\\Mail\\Mailer](https://github.com/illuminate/contracts/blob/laravel/11.x/Mail/Mailer.php) | `Mail` |
| [Illuminate\\Contracts\\Notifications\\Dispatcher](https://github.com/illuminate/contracts/blob/laravel/11.x/Notifications/Dispatcher.php) | `Notification` |
| [Illuminate\\Contracts\\Notifications\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/Notifications/Factory.php) | `Notification` |
| [Illuminate\\Contracts\\Pagination\\LengthAwarePaginator](https://github.com/illuminate/contracts/blob/laravel/11.x/Pagination/LengthAwarePaginator.php) |   |
| [Illuminate\\Contracts\\Pagination\\Paginator](https://github.com/illuminate/contracts/blob/laravel/11.x/Pagination/Paginator.php) |   |
| [Illuminate\\Contracts\\Pipeline\\Hub](https://github.com/illuminate/contracts/blob/laravel/11.x/Pipeline/Hub.php) |   |
| [Illuminate\\Contracts\\Pipeline\\Pipeline](https://github.com/illuminate/contracts/blob/laravel/11.x/Pipeline/Pipeline.php) | `Pipeline`; |
| [Illuminate\\Contracts\\Queue\\EntityResolver](https://github.com/illuminate/contracts/blob/laravel/11.x/Queue/EntityResolver.php) |   |
| [Illuminate\\Contracts\\Queue\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/Queue/Factory.php) | `Queue` |
| [Illuminate\\Contracts\\Queue\\Job](https://github.com/illuminate/contracts/blob/laravel/11.x/Queue/Job.php) |   |
| [Illuminate\\Contracts\\Queue\\Monitor](https://github.com/illuminate/contracts/blob/laravel/11.x/Queue/Monitor.php) | `Queue` |
| [Illuminate\\Contracts\\Queue\\Queue](https://github.com/illuminate/contracts/blob/laravel/11.x/Queue/Queue.php) | `Queue::connection()` |
| [Illuminate\\Contracts\\Queue\\QueueableCollection](https://github.com/illuminate/contracts/blob/laravel/11.x/Queue/QueueableCollection.php) |   |
| [Illuminate\\Contracts\\Queue\\QueueableEntity](https://github.com/illuminate/contracts/blob/laravel/11.x/Queue/QueueableEntity.php) |   |
| [Illuminate\\Contracts\\Queue\\ShouldQueue](https://github.com/illuminate/contracts/blob/laravel/11.x/Queue/ShouldQueue.php) |   |
| [Illuminate\\Contracts\\Redis\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/Redis/Factory.php) | `Redis` |
| [Illuminate\\Contracts\\Routing\\BindingRegistrar](https://github.com/illuminate/contracts/blob/laravel/11.x/Routing/BindingRegistrar.php) | `Route` |
| [Illuminate\\Contracts\\Routing\\Registrar](https://github.com/illuminate/contracts/blob/laravel/11.x/Routing/Registrar.php) | `Route` |
| [Illuminate\\Contracts\\Routing\\ResponseFactory](https://github.com/illuminate/contracts/blob/laravel/11.x/Routing/ResponseFactory.php) | `Response` |
| [Illuminate\\Contracts\\Routing\\UrlGenerator](https://github.com/illuminate/contracts/blob/laravel/11.x/Routing/UrlGenerator.php) | `URL` |
| [Illuminate\\Contracts\\Routing\\UrlRoutable](https://github.com/illuminate/contracts/blob/laravel/11.x/Routing/UrlRoutable.php) |   |
| [Illuminate\\Contracts\\Session\\Session](https://github.com/illuminate/contracts/blob/laravel/11.x/Session/Session.php) | `Session::driver()` |
| [Illuminate\\Contracts\\Support\\Arrayable](https://github.com/illuminate/contracts/blob/laravel/11.x/Support/Arrayable.php) |   |
| [Illuminate\\Contracts\\Support\\Htmlable](https://github.com/illuminate/contracts/blob/laravel/11.x/Support/Htmlable.php) |   |
| [Illuminate\\Contracts\\Support\\Jsonable](https://github.com/illuminate/contracts/blob/laravel/11.x/Support/Jsonable.php) |   |
| [Illuminate\\Contracts\\Support\\MessageBag](https://github.com/illuminate/contracts/blob/laravel/11.x/Support/MessageBag.php) |   |
| [Illuminate\\Contracts\\Support\\MessageProvider](https://github.com/illuminate/contracts/blob/laravel/11.x/Support/MessageProvider.php) |   |
| [Illuminate\\Contracts\\Support\\Renderable](https://github.com/illuminate/contracts/blob/laravel/11.x/Support/Renderable.php) |   |
| [Illuminate\\Contracts\\Support\\Responsable](https://github.com/illuminate/contracts/blob/laravel/11.x/Support/Responsable.php) |   |
| [Illuminate\\Contracts\\Translation\\Loader](https://github.com/illuminate/contracts/blob/laravel/11.x/Translation/Loader.php) |   |
| [Illuminate\\Contracts\\Translation\\Translator](https://github.com/illuminate/contracts/blob/laravel/11.x/Translation/Translator.php) | `Lang` |
| [Illuminate\\Contracts\\Validation\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/Validation/Factory.php) | `Validator` |
| [Illuminate\\Contracts\\Validation\\ImplicitRule](https://github.com/illuminate/contracts/blob/laravel/11.x/Validation/ImplicitRule.php) |   |
| [Illuminate\\Contracts\\Validation\\Rule](https://github.com/illuminate/contracts/blob/laravel/11.x/Validation/Rule.php) |   |
| [Illuminate\\Contracts\\Validation\\ValidatesWhenResolved](https://github.com/illuminate/contracts/blob/laravel/11.x/Validation/ValidatesWhenResolved.php) |   |
| [Illuminate\\Contracts\\Validation\\Validator](https://github.com/illuminate/contracts/blob/laravel/11.x/Validation/Validator.php) | `Validator::make()` |
| [Illuminate\\Contracts\\View\\Engine](https://github.com/illuminate/contracts/blob/laravel/11.x/View/Engine.php) |   |
| [Illuminate\\Contracts\\View\\Factory](https://github.com/illuminate/contracts/blob/laravel/11.x/View/Factory.php) | `View` |
| [Illuminate\\Contracts\\View\\View](https://github.com/illuminate/contracts/blob/laravel/11.x/View/View.php) | `View::make()` |

本文章首发在 [LearnKu.com](https://learnku.com/) 网站上。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/co...](https://learnku.com/docs/laravel/11.x/contractsmd/16676)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/co...](https://learnku.com/docs/laravel/11.x/contractsmd/16676)