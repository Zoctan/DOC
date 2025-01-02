本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Artisan Console

+   [介绍](#introduction)
    +   [Tinker 命令 （REPL）](#tinker)
+   [编写命令](#writing-commands)
    +   [生成命令](#generating-commands)
    +   [命令结构](#command-structure)
    +   [闭包命令](#closure-commands)
    +   [可隔离命令](#isolatable-commands)
+   [定义输入期望值](#defining-input-expectations)
    +   [参数](#arguments)
    +   [选项](#options)
    +   [输入数组](#input-arrays)
    +   [输入说明](#input-descriptions)
    +   [提示缺少的输入](#prompting-for-missing-input)
+   [I/O 命令](#command-io)
    +   [检索输入](#retrieving-input)
    +   [输入提示](#prompting-for-input)
    +   [编写输出](#writing-output)
+   [注册命令](#registering-commands)
+   [编程式执行命令](#programmatically-executing-commands)
    +   [从其他命令调用命令](#calling-commands-from-other-commands)
+   [信号处理](#signal-handling)
+   [Stub 自定义](#stub-customization)
+   [事件](#events)

## 介绍

Artisan 是 Laravel 自带的命令行接口。Artisan 以 `artisan` 脚本的方式存在于应用的根目录中，并提供了许多有用的命令，来帮你构建应用程序。你可以使用 `list` 命令查看所有可用的 Artisan 命令：

每个命令都有一个「help」帮助界面，它展示并描述了该命令的可用参数和选项。要查看帮助界面，请在命令前加上 `help` 即可：

#### Laravel Sail

如果你使用 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sailmd) 作为本地开发环境，记得使用 `sail` 命令行来调用 Artisan 命令。Sail 会在应用的 Docker 容器中执行 Artisan 命令：

```shell
./vendor/bin/sail artisan list
```

### Tinker （REPL）

Laravel Tinker 是为 Laravel 提供的一个强大的 REPL（交互式解释器），由 [PsySH](https://github.com/bobthecow/psysh) 驱动支持。

#### 介绍

所有的 Laravel 应用默认都自带 Tinker。 不过，如果你此前删除了它，你可以使用 Composer 安装：

```shell
composer require laravel/tinker
```

> **注意**  
> 在寻找与 Laravel 应用程序交互时能热加载、多行代码编辑和自动补全的功能？试试 [Tinkerwell](https://tinkerwell.app/)！

#### 使用

Tinker 允许你在命令行中和整个 Laravel 应用交互，包括 Eloquent 模型、队列、事件等等。要进入 Tinker 环境，只需运行 `tinker` Artisan 命令：

你可以使用 `vendor:publish` 命令发布 Tinker 的配置文件：

```shell
php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"
```

> **警告**  
> `dispatch` 辅助函数及 `Dispatchable` 类中 `dispatch` 方法依赖于垃圾回收机制将任务放置到队列中。因此，使用 tinker 时，请使用 `Bus::dispath` 或 `Queue::push` 来分发任务。

#### 命令白名单

Tinker 使用一个 “白名单” 来确定哪些 Artisan 命令可以在其 shell 中运行。默认情况下，你可以允许  
`clear-compiled`，`down`， `env`，`inspire`，`migrate`，`optimize`， 和 `up` 命令。如果你想允许更多命令，你可以将它们添加到 `tinker.php` 配置文件的 `commands` 数组中：

```php
'commands' => [
    // App\Console\Commands\ExampleCommand::class,
],
```

#### 别名黑名单

一般来说， Tinker 会在你引入类时自动为其添加别名。不过，你可能希望永远不为某些类设置别名。你可以在 `thinker.php` 配置文件的 `dont_alias` 数组中列举这些类来完成此操作：

```php
'dont_alias' => [
    App\Models\User::class,
],
```

## 编写命令

除了 Artisan 提供的命令之外，你还可以创建自定义命令。一般来说，命令会保存在 `app/Console/Commands` 目录中；不过，你可以自由选择命令的存储位置，只要它能够被 Composer 加载即可。

### 生成命令

要创建新命令，你可以使用 `make:command` Artisan 命令。该命令会在 `app/Console/Commands` 目录下创建一个新的命令类。不用担心你的应用程序中不存在这个目录 - 它会在第一次运行 `make:command` Artisan 命令的时自动创建：

```shell
php artisan make:command SendEmails
```

### 命令结构

生成命令后，你应该为类的 `signature` 和 `description` 属性定义适当的值。这些属性会在 `list` 屏幕展示你的命令时被用到。`signature` 属性还允许你定义[命令输入预期值](#defining-input-expectations)。 `handle` 方法会在命令执行时被调用。你可以在该方法中编写命令逻辑。

让我们看一个示例命令。请注意，我们能够通过命令的 handle 方法请求我们需要的任何依赖项。Laravel [服务容器](https://learnku.com/docs/laravel/11.x/containermd) 将自动注入此方法签名中带有类型提示的所有依赖项：

```php
<?php

namespace App\Console\Commands;

use App\Models\User;
use App\Support\DripEmailer;
use Illuminate\Console\Command;

class SendEmails extends Command
{
    /**
     * 控制台命令的名称和签名
     *
     * @var string
     */
    protected $signature = 'mail:send {user}';

    /**
     * 命令描述
     *
     * @var string
     */
    protected $description = 'Send a marketing email to a user';

    /**
     * 执行命令
     */
    public function handle(DripEmailer $drip): void
    {
        $drip->send(User::find($this->argument('user')));
    }
}
```

> **注意**  
> 为了更好地复用代码，让你的控制台命令保持轻量级并委托应用服务来完成它们的任务。在上面的例子中，请注意我们注入了一个服务类来进行发送电子邮件的「繁重工作」。

#### 退出代码

如果 `handle` 方法没有返回任何内容并且命令执行成功，则命令将以退出代码 `0` 来退出，表示成功。当然，`handle` 方法也可以选择返回一个整数来手动指定命令的退出代码：

```php
$this->error('Something went wrong.');

return 1;
```

如果你想在命令里的任意方法里让命令以「失败」结束，你可以使用 `fail` 方法。`fail` 方法将立即终止命令的执行并返回 `1` 的退出代码：

```php
$this->fail('Something went wrong.');
```

### 闭包命令

闭包命令提供了定义控制台命令为类的另一种选择。就像路由闭包是对控制器的一种替代，可以认为命令闭包是对命令类的一种替代。

尽管 `routes/console.php` 文件不定义 HTTP 路由，它定义了进入你的应用程序的基于控制台的入口点（路由）。在这个文件中，你可以使用 `Artisan::command` 方法定义所有的基于闭包的控制台命令。`command` 方法接受两个参数：[命令的签名](#defining-input-expectations) 和一个闭包，闭包接收命令的参数和选项：

```php
Artisan::command('mail:send {user}', function (string $user) {
    $this->info("Sending email to: {$user}!");
});
```

该闭包绑定到底层的命令实例，因而你可以完全访问所有你通常能够在完整命令类上访问的辅助方法

#### 类型约束依赖

除了接受命令参数及选项外，命令闭包也可以使用类型约束从 [服务容器](https://learnku.com/docs/laravel/11.x/containermd) 中解析其他的依赖关系：

```php
use App\Models\User;
use App\Support\DripEmailer;

Artisan::command('mail:send {user}', function (DripEmailer $drip, string $user) {
    $drip->send(User::find($user));
});
```

#### 闭包命令的描述

定义基于闭包的命令时，可以使用 `purpose` 方法为命令添加描述。这个描述将在运行 `php artisan list` 或 `php artisan help` 命令时显示：

```php
Artisan::command('mail:send {user}', function (string $user) {
    // ...
})->purpose('向一个用户发送营销邮件');
```

### 可隔离的命令

> \[警告\]  
> 要使用此功能，您的应用必须将 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 缓存驱动设置为默认缓存驱动。此外，所有服务器都必须与同一个中央缓存服务器进行通信。

有时候，您可能希望确保一次只能运行一个命令实例。为此，您可以在命令类上实现 `Illuminate\Contracts\Console\Isolatable` 接口：

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Contracts\Console\Isolatable;

class SendEmails extends Command implements Isolatable
{
    // ...
}
```

当一个命令被标记为 `Isolatable` 时，Laravel 会自动给该命令添加一个 `--isolated` 选项。使用该选项调用命令时，Laravel 会确保没有其他实例的该命令正在运行。Laravel 通过尝试使用应用的默认缓存驱动获取一个原子锁来实现这一点。如果有其他实例的命令正在运行，那么该命令将不会执行；不过，命令仍然会以一个成功的退出状态码结束：

```shell
php artisan mail:send 1 --isolated
```

如果您希望指定命令在无法执行时应返回的退出状态码，可以通过 `isolated` 选项提供所需的状态码：

```shell
php artisan mail:send 1 --isolated=12
```

#### 锁 ID

默认情况下，Laravel 会使用命令的名称生成字符串的键，用于获取应用程序缓存中的原子锁。当然，你可以在 Artisan 命令类中定义一个 `isolatableId` 方法来自定义这个键，从而允许你将命令的参数或选项集成到键中：

```php
/**
 * 获取命令的可隔离ID
 */
public function isolatableId(): string
{
    return $this->argument('user');
}
```

#### 锁过期时间

默认情况下，隔离锁会在命令完成后过期。或者，如果命令被中断而无法完成，锁将在一小时后过期。但是，您可以通过在命令上定义一个 `isolationLockExpiresAt` 方法来调整锁的过期时间：

```php
use DateTimeInterface;
use DateInterval;

/**
 * 指定命令的隔离锁何时过期
 */
public function isolationLockExpiresAt(): DateTimeInterface|DateInterval
{
    return now()->addMinutes(5);
}
```

## 定义输入期望值

编写控制台命令时，通常是通过参数和选项来收集用户输入的。 Laravel 让你可以非常方便地在 `signature` 属性中定义你期望用户输入的内容。`signature` 属性允许使用单一且可读性高，类似路由的语法来定义命令的名称、参数和选项。

### 参数

用户提供的所有参数和选项都用花括号括起来。在下面的示例中，该命令定义了一个必需的参数 `user`:

```php
/**
 * 命令的名称和标识。
 *
 * @var string
 */
protected $signature = 'mail:send {user}';
```

你也可以使参数可选或为参数定义默认值：

```php
// 可选参数...
'mail:send {user?}'

// 带有默认值的可选参数...
'mail:send {user=foo}'
```

### 选项

选项，如同参数，是另一种形式的用户输入。选项在通过命令行提供时，前缀为两个连字符（`--`）。选项有两种类型：接收值的选项和不接收值的选项。不接收值的选项作为一个布尔「开关」。让我们来看一个这种类型选项的例子：

```php
/**
 * 控制台命令的名称和签名。
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue}';
```

在这个例子中，调用 Artisan 命令时可以指定 `--queue` 开关。如果传递了 `--queue` 开关，选项的值为 `true`。否则，值为 `false`：

```shell
php artisan mail:send 1 --queue
```

#### 带值的选项

接下来，让我们看一个期望值的选项。如果用户必须为选项指定一个值，你应该在选项名称后加上一个 `=` 符号：

```php
/**
 * 控制台命令的名称和签名。
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue=}';
```

在这个例子中，用户可以这样为选项传递一个值。如果在调用命令时没有指定选项，其值将为 `null`：

```shell
php artisan mail:send 1 --queue=default
```

你可以通过在选项名称后指定默认值来为选项分配默认值。如果用户没有传递选项值，将使用默认值：

```php
'mail:send {user} {--queue=default}'
```

#### 选项快捷方式

在定义选项时，你可以在选项名称前指定一个快捷方式，并使用 `|` 字符作为分隔符来分隔快捷方式和完整的选项名称：

```php
'mail:send {user} {--Q|queue}'
```

在终端上调用命令时，选项快捷方式应以单个连字符为前缀，并且在为选项指定值时不应包含 `=` 字符：

```shell
php artisan mail:send 1 -Qdefault
```

### 输入数组

如果你想定义期望接收多个输入值的参数或选项，你可以使用 `*` 字符。首先，让我们看一个指定这种参数的例子：

调用这个方法时，`user` 参数可以按顺序传递给命令行。例如，以下命令将设置 `user` 的值为一个包含 `1` 和 `2` 的数组：

```shell
php artisan mail:send 1 2
```

这个 `*` 字符可以与可选参数定义结合使用，以允许零个或多个参数实例：

#### 选项数组

在定义期望接收多个输入值的选项时，传递给命令的每个选项值都应以选项名称为前缀：

通过传递多个 `--id` 参数可以调用这样的命令：

```shell
php artisan mail:send --id=1 --id=2
```

### 输入描述

你可以通过使用冒号分隔参数名称和描述来为输入参数和选项分配描述。如果你需要更多空间来定义你的命令，可以随意将定义分布在多行上：

```php
/**
 * 控制台命令的名称和签名。
 *
 * @var string
 */
protected $signature = 'mail:send
                        {user : 用户的ID}
                        {--queue : 作业是否应该被加入队列}';
```

### 提示参数缺失

如果你的命令包含必需的参数，当这些参数未提供时用户会收到一个错误消息。作为替代，你可以配置你的命令在必需的参数缺失时自动提示用户，通过实现 `PromptsForMissingInput` 接口来实现：

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Contracts\Console\PromptsForMissingInput;

class SendEmails extends Command implements PromptsForMissingInput
{
    /**
     * console命令的名称和签名。
     *
     * @var string
     */
    protected $signature = 'mail:send {user}';

    // ...
}
```

如果 Laravel 需要从用户那里收集一个必需的参数，它将自动询问用户该参数，通过智能地使用参数名称或描述来提问。如果你希望自定义用于收集必需参数的问题，你可以实现 `promptForMissingArgumentsUsing` 方法，返回一个以参数名称为键的问题数组：

```php
/**
 * 使用返回的问题来提示缺失的输入参数
 *
 * @return array<string, string>
 */
protected function promptForMissingArgumentsUsing(): array
{
    return [
        'user' => '应该向哪个用户发送邮件？',
    ];
}
```

你也可以通过使用包含问题和占位符的元组来提供占位文本：

```php
return [
    'user' => ['Which user ID should receive the mail?', 'E.g. 123'],
];
```

如果你想要完全控制提示，你可以提供一个闭包，该闭包应该提示用户并返回他们的答案：

```php
use App\Models\User;
use function Laravel\Prompts\search;

// ...

return [
    'user' => fn () => search(
        label: 'Search for a user:',
        placeholder: 'E.g. Taylor Otwell',
        options: fn ($value) => strlen($value) > 0
            ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
            : []
    ),
];
```

> \[!NOTE\]  
> 关于可用提示及其使用的更多信息，请参阅全面的 [Laravel Prompts](https://learnku.com/docs/laravel/11.x/prompts) 文档。

如果您希望提示用户选择或输入[选项](#options)，您可以在命令的 `handle` 方法中包含提示。但是，如果您只希望在用户因缺失参数而自动被提示时再向他们提示，那么您可以实现 `afterPromptingForMissingArguments` 方法：

```php
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use function Laravel\Prompts\confirm;

// ...

/**
 * 用户被提示缺失参数后执行的操作。
 */
protected function afterPromptingForMissingArguments(InputInterface $input, OutputInterface $output): void
{
    $input->setOption('queue', confirm(
        label: 'Would you like to queue the mail?',
        default: $this->option('queue')
    ));
}
```

## 命令输入 / 输出

### 获取输入

在命令执行过程中，您可能需要访问命令接受的参数和选项的值。要做到这一点，您可以使用 `argument` 和 `option` 方法。如果参数或选项不存在，将返回 `null`：

```php
/**
 * 执行控制台命令。
 */
public function handle(): void
{
    $userId = $this->argument('user');
}
```

如果您需要以数组形式检索所有参数，调用 `arguments` 方法：

```php
$arguments = $this->arguments();
```

选项可以像参数一样轻松地使用 `option` 方法检索。要以数组形式检索所有选项，调用 `options` 方法：

```php
// 检索一个特定的选项...
$queueName = $this->option('queue');

// 以数组形式检索所有选项...
$options = $this->options();
```

### 提示输入

> \[!NOTE\]  
> [Laravel Prompts](https://learnku.com/docs/laravel/11.x/prompts) 是一个 PHP 包，为您的命令行应用程序添加了美观且用户友好的表单，具备浏览器类特性，包括占位符文本和验证功能。

除了显示输出以外，你还可以要求用户在执行命令期间提供输入。`ask` 方法将询问用户指定的问题来接收用户输入，然后用户输入将会传到你的命令中：

```php
/**
 * 执行命令
 */
public function handle(): void
{
    $name = $this->ask('What is your name?');

    // ...
}
```

`ask` 方法还接受一个可选的第二个参数，该参数指定了如果没有用户提供输入时的默认值：

```php
$name = $this->ask('What is your name?', 'Taylor');
```

`secret` 方法与 `ask` 相似，区别在于用户的输入将不可见。这个方法在需要输入一些诸如密码之类的敏感信息时是非常有用的：

```php
$password = $this->secret('What is the password?');
```

#### 请求确认

如果你需要请求用户进行一个简单的确认，可以使用 `confirm` 方法来实现。默认情况下，这个方法会返回 `false`。当然，如果用户输入 `y` 或 `yes`，这个方法将会返回 `true`。

```php
if ($this->confirm('Do you wish to continue?')) {
    // ...
}
```

必要的话，你可以将 `true` 作为第二个参数传递给 `confirm` 方法，这样就可以在默认情况下返回 `true`：

```php
if ($this->confirm('Do you wish to continue?', true)) {
    // ...
}
```

#### 自动补全

`anticipate` 方法可用于为可能的选项提供自动补全功能。用户依然可以忽略自动补全的提示，进行任意回答：

```php
$name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);
```

另外，您可以把一个闭包作为第二个参数传递给 `anticipate` 方法。每当用户输入一个字符时，该闭包就会被调用。此闭包应接收一个包含用户截至目前输入内容的字符串形式的参数，并返回一个自动补全选项的数组：

```php
$name = $this->anticipate('What is your address?', function (string $input) {
    // Return auto-completion options...
});
```

#### 多选问题

如果你需要在询问问题时给用户提供一组预先设定好的选项，你可以使用 `choice` 方法。你可以给方法的第三个参数传入数组的索引，来设置没有选项被选择时的默认值：

```php
$name = $this->choice(
    'What is your name?',
    ['Taylor', 'Dayle'],
    $defaultIndex
);
```

此外， `choice` 方法接受第四和第五个可选参数 ，用于确定选择有效响应的最大尝试次数以及是否允许多次选择：

```php
$name = $this->choice(
    'What is your name?',
    ['Taylor', 'Dayle'],
    $defaultIndex,
    $maxAttempts = null,
    $allowMultipleSelections = false
);
```

### 编写输出

你可以使用 `line`，`info`，`comment`，`question` ，`warn` 和 `error` 方法，发送输出到控制台。 这些方法中的每一个都会使用合适的 ANSI 颜色以展示不同的用途。例如，我们要为用户展示一些常规信息。通常，`info` 将会以绿色文本在控制台展示。

```php
/**
 * 执行命令
 */
public function handle(): void
{
    // ...

    $this->info('The command was successful!');
}
```

展示错误信息，使用 `error` 方法。错误信息通常使用红色字体显示：

```php
$this->error('Something went wrong!');
```

你可以使用 `line` 方法输出普通的无色文本：

```php
$this->line('Display this on the screen');
```

你可以使用 `newLine` 方法输出空白行：

```php
// 输出单行空白...
$this->newLine();

// 输出三行空白...
$this->newLine(3);
```

#### 表格

`table` 方法可以轻松正确地格式化多行 / 多列数据。你需要做的就是提供表的列名和数据，Laravel 会自动为你计算合适的表格宽度和高度：

```php
use App\Models\User;

$this->table(
    ['Name', 'Email'],
    User::all(['name', 'email'])->toArray()
);
```

#### 进度条

对于长时间运行的任务，显示一个进度条来通知用户任务的完成程度会很有帮助。使用 `withProgressBar` 方法，Laravel 将显示一个进度条，并在给定的可迭代值上推进每次迭代的进度：

```php
use App\Models\User;

$users = $this->withProgressBar(User::all(), function (User $user) {
    $this->performTask($user);
});
```

有时，你可能需要更多手动控制进度条的前进方式。首先，定义流程将迭代的步骤总数。然后，在处理完每个项目后推进进度条：

```php
$users = App\Models\User::all();

$bar = $this->output->createProgressBar(count($users));

$bar->start();

foreach ($users as $user) {
    $this->performTask($user);

    $bar->advance();
}

$bar->finish();
```

> **注意**  
> 有关更多高级选项，请查看 [Symfony Progress Bar component documentation](https://symfony.com/doc/7.0/components/console/helpers/progressbar.html).

## 注册命令

默认情况下，Laravel 会自动注册 `app/Console/Commands` 目录中的所有命令。但是，你可以让 Laravel 使用应用程序的 `bootstrap/app.php` 文件中的 `withCommands` 方法扫描其他目录以查找 Artisan 命令：

```php
->withCommands([
    __DIR__.'/../app/Domain/Orders/Commands',
])
```

必要的话，您还可以通过将命令的类名提供给 `withCommands` 方法来手动注册命令：

```php
use App\Domain\Orders\Commands\SendEmails;

->withCommands([
    SendEmails::class,
])
```

当 Artisan 启动时，应用程序中的所有命令都将被[服务容器](https://learnku.com/docs/laravel/11.x/container)并注册到 Artisan。

## 编程式执行命令

有时你可能希望在 CLI 之外执行 Artisan 命令。例如，你可能希望从路由或控制器执行 Artisan 命令。你可以使用 `Artisan` 门面的 `call` 方法来完成此操作。 `call` 方法接受命令的签名名称或类名作为其第一个参数，并接受一个命令参数数组作为第二个参数。退出代码会最后被返回：

```php
use Illuminate\Support\Facades\Artisan;

Route::post('/user/{user}/mail', function (string $user) {
    $exitCode = Artisan::call('mail:send', [
        'user' => $user, '--queue' => 'default'
    ]);

    // ...
});
```

或者，你可以将整个 Artisan 命令作为字符串传递给 `call` 方法：

```php
Artisan::call('mail:send 1 --queue=default');
```

#### 传递数组值

如果你的命令定义了一个接受数组的选项，你可以将一个数组值传递给该选项：

```php
use Illuminate\Support\Facades\Artisan;

Route::post('/mail', function () {
    $exitCode = Artisan::call('mail:send', [
        '--id' => [5, 13]
    ]);
});
```

#### 传递布尔值

如果你需要指定不接受字符串值的选项的值，例如 `migrate:refresh` 命令上的 `--force` 标志，则应传递 `true` 或 `false` 作为 选项：

```php
$exitCode = Artisan::call('migrate:refresh', [
    '--force' => true,
]);
```

#### 队列 Artisan 命令

使用 `Artisan` 门面的 `queue` 方法，你甚至可以对 Artisan 命令进行排队，以便你的 [队列工作者](https://learnku.com/docs/laravel/10.x/queues) 在后台处理它们。在使用此方法之前，请确保你已配置队列并正在运行队列侦听器：

```php
use Illuminate\Support\Facades\Artisan;

Route::post('/user/{user}/mail', function (string $user) {
    Artisan::queue('mail:send', [
        'user' => $user, '--queue' => 'default'
    ]);

    // ...
});
```

你可以使用 `onConnection` 和 `onQueue` 方法来指定 Artisan 命令应分发到的连接或队列：

```php
Artisan::queue('mail:send', [
    'user' => 1, '--queue' => 'default'
])->onConnection('redis')->onQueue('commands');
```

### 从其他命令调用命令

有时你可能希望从现有的 Artisan 命令调用其他命令。你可以使用 `call` 方法来执行此操作。`call` 方法接受命令名称和命令参数 / 选项的数组：

```php
/**
 * Execute the console command.
 */
public function handle(): void
{
    $this->call('mail:send', [
        'user' => 1, '--queue' => 'default'
    ]);

    // ...
}
```

如果你想调用另一个控制台命令并禁用它的输出，你可以使用 `callSilently` 方法。 `callSilently` 方法与 `call` 方法具有相同的签名：

```php
$this->callSilently('mail:send', [
    'user' => 1, '--queue' => 'default'
]);
```

## 信号处理

如你所知，操作系统可以向运行中的进程发送信号。例如，「SIGTERM」信号是操作系统要求程序终止的方式。如果你想在 Artisan 控制台命令中监听信号，并在信号出现时时执行代码，你可以使用 `trap` 方法。

```php
/**
 * 执行命令
 */
public function handle(): void
{
    $this->trap(SIGTERM, fn () => $this->shouldKeepRunning = false);

    while ($this->shouldKeepRunning) {
        // ...
    }
}
```

要一次侦听多个信号，你可以向 `trap` 方法提供一个信号数组。

```php
$this->trap([SIGTERM, SIGQUIT], function (int $signal) {
    $this->shouldKeepRunning = false;

    dump($signal); // SIGTERM / SIGQUIT
});
```

## Stub 定制

Artisan 控制台的 `make` 命令用于创建各种类，例如控制器、队列、迁移和测试。这些类是使用「stub」文件生成的，这些文件中会根据你的输入填充值。但是，你可能需要对 Artisan 生成的文件进行少量更改。为此，你可以使用 `stub:publish` 命令将最常见的 stub 命令发布到你的应用程序中，以便可以自定义它们：

已发布的 stub 将存放于你的应用根目录下的 `stubs` 目录中。对这些 stub 进行任何改动都将在你使用 Artisan 的 `make` 命令生成相应的类的时候反映出来。

## 事件

Artisan 在运行命令时会调度三个事件： `Illuminate\Console\Events\ArtisanStarting`， `Illuminate\Console\Events\CommandStarting`，和 `Illuminate\Console\Events\CommandFinished`。当 Artisan 开始运行时，会立即调度 `ArtisanStarting` 事件。接下来，在命令运行之前立即调度 `CommandStarting` 事件。最后，一旦命令执行完毕，就会调度 `CommandFinished` 事件。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/ar...](https://learnku.com/docs/laravel/11.x/artisanmd/16671)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/ar...](https://learnku.com/docs/laravel/11.x/artisanmd/16671)