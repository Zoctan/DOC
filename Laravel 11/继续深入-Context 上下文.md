本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Context 上下文

+   [介绍](#introduction)
    +   [工作原理](#how-it-works)
+   [捕获 Context 上下文](#capturing-context)
    +   [堆栈](#stacks)
+   [检索 Context 上下文](#retrieving-context)
    +   [确定项目是否存在](#determining-item-existence)
+   [移除 Context 上下文](#removing-context)
+   [隐藏 Context 上下文](#hidden-context)
+   [事件](#events)
    +   [脱水](#dehydrating)
    +   [注水](#hydrated)

## 介绍

Laravel 的「Context 上下文」功能使你能够在应用程序中执行的请求、任务和命令中捕获、检索和共享信息。这些捕获的信息也包含在应用程序写入的日志中，使你深入了解在写入日志条目之前发生的代码执行历史，并允许你跟踪整个分布式系统的执行流程。

### 工作原理

理解 Laravel Context 上下文功能的最佳方式是使用内置的日志功能来观察它的实际情况。首先，你可以使用 `Context` Facade [向 Context 上下文添加信息](#capturing-context)。在此示例中，我们将使用 [中间件](https://learnku.com/docs/laravel/11.x/middleware) 将请求 URL 和唯一的跟踪 ID 添加到每个传入请求的 Context 上下文中：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;
use Symfony\Component\HttpFoundation\Response;

class AddContext
{
    /**
     * 处理传入的请求
     */
    public function handle(Request $request, Closure $next): Response
    {
        Context::add('url', $request->url());
        Context::add('trace_id', Str::uuid()->toString());

        return $next($request);
    }
}
```

添加到 Context 上下文中的信息会自动作为元数据附加到在请求期间写入的任何 [日志条目](https://learnku.com/docs/laravel/11.x/logging) 中。将 Context 上下文作为元数据附加到日志条目中，可以将传递到单个日志条目的信息与通过 `Context` 共享的信息区分开来。例如，假设我们写入以下日志条目：

```php
Log::info('User authenticated.', ['auth_id' => Auth::id()]);
```

写入的日志将包含传递给日志条目的 `auth_id`，但也将包含上下文的 `url` 和 `trace_id` 作为元数据：

```php
User authenticated. {"auth_id":27} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

添加到上下文的信息也可用于分发到队列的任务。比如，假设我们在将一些信息添加到上下文后，将 `ProcessPodcast` 任务分发给队列：

```php
// 在中间件中...
Context::add('url', $request->url());
Context::add('trace_id', Str::uuid()->toString());

// 在控制器中...
ProcessPodcast::dispatch($podcast);
```

任务分发时，当前保存在上下文中的所有信息，都可以在任务中捕获和分享。然后，在执行任务时，这些捕获的信息填充回到当前上下文中。因此，如果我们的任务的 handle 方法是写入日志：

```php
class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    // ...

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        Log::info('Processing podcast.', [
            'podcast_id' => $this->podcast->id,
        ]);

        // ...
    }
}
```

生成的日志条目将包含在最初调度该任务的请求期间添加到上下文中的信息：

```php
Processing podcast. {"podcast_id":95} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

尽管我们关注的是 Laravel 上下文的内置日志相关功能，但后续文档将说明上下文如何允许跨 HTTP 请求 / 队列任务边界共享信息，甚至如何添加未使用日志条目编写的[隐藏上下文数据](#hidden-context)。

## 捕获上下文

使用 `Context` 门面的 `add` 方法，你可以在当前上下文中存储信息：

```php
use Illuminate\Support\Facades\Context;

Context::add('key', 'value');
```

要一次添加多个项目，你可以将关联数组传递给 `add` 方法：

```php
Context::add([
    'first_key' => 'value',
    'second_key' => 'value',
]);
```

该 `add` 方法将覆盖共享同一个键名的现有值。如果你只想将还不存在键名的信息添加到上下文中，你可以使用 `addIf` 方法：

```php
Context::add('key', 'first');

Context::get('key');
// "first"

Context::addIf('key', 'second');

Context::get('key');
// "first"
```

#### 条件上下文

`when` 方法可以用于基于给定条件将数据添加到上下文中。如果给定的条件等于 `true`，`when` 方法中的第一个闭包将被调用；而如果给定的条件等于 `false`，则会调用第二个闭包：

```php
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Context;

Context::when(
    Auth::user()->isAdmin(),
    fn ($context) => $context->add('permissions', Auth::user()->permissions),
    fn ($context) => $context->add('permissions', []),
);
```

### 堆栈

上下文提供了创建「堆栈」的能力，堆栈是按添加顺序存储的数据列表。你可以通过调用 `push` 方法将信息添加到堆栈中：

```php
use Illuminate\Support\Facades\Context;

Context::push('breadcrumbs', 'first_value');

Context::push('breadcrumbs', 'second_value', 'third_value');

Context::get('breadcrumbs');
// [
//     'first_value',
//     'second_value',
//     'third_value',
// ]
```

堆栈可用于捕获有关请求的历史信息，例如整个应用中发生的事件。比如，你可以创建一个事件监听器，以便在每次执行查询时推送到堆栈，将查询 SQL 和持续时间捕获为元组：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\DB;

DB::listen(function ($event) {
    Context::push('queries', [$event->time, $event->sql]);
});
```

## 检索上下文

你可以使用 `Context` 门面的 `get` 方法，从上下文中检索信息：

```php
use Illuminate\Support\Facades\Context;

$value = Context::get('key');
```

`only` 方法可用于在上下文中检索信息子集。

```php
$data = Context::only(['first_key', 'second_key']);
```

`pull` 方法可用于从上下文中检索信息并立即将其从上下文中删除：

```php
$value = Context::pull('key');
```

如果你想检上下文中存储的所有信息，你可以调用 `all` 方法：

### 检查项目是否存在

你可以使用 `has` 方法检测上下文是否为给定的键存有任何值：

```php
use Illuminate\Support\Facades\Context;

if (Context::has('key')) {
    // ...
}
```

不管存储的值是什么，`has` 方法都将返回 `true`。因此，比如，具有 `null` 值的键将被视为存在：

```php
Context::add('key', null);

Context::has('key');
// true
```

## 删除上下文

`forget` 方法可用于在当前上下文中删除键及它的值：

```php
use Illuminate\Support\Facades\Context;

Context::add(['first_key' => 1, 'second_key' => 2]);

Context::forget('first_key');

Context::all();

// ['second_key' => 2]
```

通过给 `forget` 方法提供一个数组，你可以一次性删除多个键：

```php
Context::forget(['first_key', 'second_key']);
```

## 隐藏上下文

上下文提供存储隐藏数据的能力。这些隐藏信息不会追加到日志中，也无法通过上述数据检索方法进行访问。上下文提供了一组不同的方法来与隐藏的上下文信息交互：

```php
use Illuminate\Support\Facades\Context;

Context::addHidden('key', 'value');

Context::getHidden('key');
// 'value'

Context::get('key');
// null
```

这些「隐藏」方法对上述非隐藏方法的功能进行了镜像：

```php
Context::addHidden(/* ... */);
Context::addHiddenIf(/* ... */);
Context::pushHidden(/* ... */);
Context::getHidden(/* ... */);
Context::pullHidden(/* ... */);
Context::onlyHidden(/* ... */);
Context::allHidden(/* ... */);
Context::hasHidden(/* ... */);
Context::forgetHidden(/* ... */);
```

## 事件

上下文分发了两个事件，允许你插入钩子到上下文的注水和脱水处理过程。

为了说明如何使用这些事件，假设在应用的中间件中，你根据传入的 HTTP 请求的 `Accept Language` 标头设置 `app.locate` 配置值。上下文的事件允许你在请求期间捕获此值并将其还原到队列中，从而确保在队列上发送的通知具有正确的 `app.locale` 值。我们可以使用上下文的事件和[隐藏的](#hidden-context) 数据来实现这一点，下面的文档将对此进行说明。

### 脱水

每当任务被调度到队列时，上下文中的数据都会被「脱水」，并与任务业的有效负载一起捕获。`Context::dehydrating` 方法允许你对将在脱水过程中调用的闭包进行注册。在此闭包中，你可以对将与队列任务共享的数据进行更改。

通常，你应该在应用的 `AppServiceProvider` 类的 `boot` 方法中注册 `dehydrating` 回调：

```php
use Illuminate\Log\Context\Repository;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Context;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Context::dehydrating(function (Repository $context) {
        $context->addHidden('locale', Config::get('app.locale'));
    });
}
```

> 注意  
> 你不应在 `dehydrating` 回调中使用 `Context` 门面，因为这会将修改当前进程的上下文。请确保仅对传递给回调的存储库进行更改。

### 注水

每当队列中的任务开始在队列上执行时，与该任务共享的任何上下文都将注水回当前上下文。`Context::hydrated` 方法允许你注册将在注水过程中调用的闭包。

通常，你应该在应用的 `AppServiceProvider` 类中的 `boot` 方法里注册 `hydrated` 回调：

```php
use Illuminate\Log\Context\Repository;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Context;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Context::hydrated(function (Repository $context) {
        if ($context->hasHidden('locale')) {
            Config::set('app.locale', $context->getHidden('locale'));
        }
    });
}
```

> 注  
> 你不应在 `hydrated` 回调中使用 `Context` 门面，并请确保仅对传递给回调的存储库进行更改。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/co...](https://learnku.com/docs/laravel/11.x/contextmd/16675)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/co...](https://learnku.com/docs/laravel/11.x/contextmd/16675)