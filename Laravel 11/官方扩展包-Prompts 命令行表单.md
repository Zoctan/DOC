本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Prompts

+   [介绍](#introduction)
+   [安装](#installation)
+   [可用 Prompts](#available-prompts)
    +   [文本](#text)
    +   [文本框](#textarea)
    +   [密码](#password)
    +   [确认](#confirm)
    +   [单选](#select)
    +   [多选](#multiselect)
    +   [建议](#suggest)
    +   [搜索](#search)
    +   [多选搜索](#multisearch)
    +   [暂停](#pause)
+   [表单](#forms)
+   [提示信息](#informational-messages)
+   [表格](#tables)
+   [旋转](#spin)
+   [进度条](#progress)
+   [终端考虑因素](#terminal-considerations)
+   [不支持的环境与回退](#fallbacks)

## 介绍

[Laravel Prompts](https://github.com/laravel/prompts) 是一个 PHP 包，用于向命令行应用程序添加漂亮且用户友好的表单，并具有类似浏览器的特性，包括占位符文本和验证。

![prompts-example.png](https://laravel.com/img/docs/prompts-example.png)

Laravel Prompts 非常适合在你的 [Artisan 控制台命令](https://learnku.com/docs/laravel/11.x/artisan#writing-commands) 中接收用户输入，但是它也可以用在任何命令行 PHP 项目中。

> **注意**  
> Laravel Prompts 通过 WSL 支持 macOS、 Linux 和 Windows。有关更多信息，请参阅我们的文档 [不支持的环境 & 回退](#fallbacks)。

## 安装

Laravel Prompts 已经包含在最新版本的 Laravel 中。

你也可以通过使用 Composer 包管理器，将 Laravel Prompts 安装在你其他 PHP 项目中:

```shell
composer require laravel/prompts
```

## 可用提示

### Text

`text` 函数将提示用户指定的问题，接收他们的输入，然后返回:

```php
use function Laravel\Prompts\text;

$name = text('What is your name?');
```

你还可以引入占位文本、默认值和信息提示:

```php
$name = text(
    label: 'What is your name?',
    placeholder: 'E.g. Taylor Otwell',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```

#### 必传的值

如果你要求一个值必传，可以传递 `required` 参数：

```php
$name = text(
    label: 'What is your name?',
    required: true
);
```

如果你想自定义验证信息，你也可以传递一个字符串：

```php
$name = text(
    label: 'What is your name?',
    required: 'Your name is required.'
);
```

#### 额外验证

最后，如果你想执行额外的验证逻辑，可以给 `validate` 参数传递一个闭包：

```php
$name = text(
    label: 'What is your name?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

闭包接收输入的值，可能会返回一个错误信息，如果验证通过会返回 `null`。

或者，你也可以使用 Laravel [验证器](https://learnku.com/docs/laravel/11.x/validation) 这个强大功能。要想使用它，你可以给 `validate` 传递一个数组，数组包含属性名称和所需的验证规则：

```php
$name = text(
    label: 'What is your name?',
    validate: ['name' => 'required|max:255|unique:users,name']
);
```

### 文本框

`textarea` 函数给用户提示指定的问题，通过多行文本框接收输入，然后返回：

```php
use function Laravel\Prompts\textarea;

$story = textarea('Tell me a story.');
```

你也可以引入占位文本、默认值和信息提示：

```php
$story = textarea(
    label: 'Tell me a story.',
    placeholder: 'This is a story about...',
    hint: 'This will be displayed on your profile.'
);
```

#### 必传的值

如果你要求一个值必传，可以传递 `required` 参数：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: true
);
```

如果你想自定义验证信息，你也可以传递一个字符串：

```php
$story = textarea(
    label: 'Tell me a story.',
    required: 'A story is required.'
);
```

#### 额外验证

最后，如果你想执行额外的验证逻辑，可以给 `validate` 参数传递一个闭包：

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: fn (string $value) => match (true) {
        strlen($value) < 250 => 'The story must be at least 250 characters.',
        strlen($value) > 10000 => 'The story must not exceed 10,000 characters.',
        default => null
    }
);
```

闭包接收输入的值，可能会返回一个错误信息，如果验证通过会返回 `null`。

或者，你也可以使用 Laravel [验证器](https://learnku.com/docs/laravel/11.x/validation) 这个强大功能。要想使用它，你可以给 `validate` 传递一个数组，数组包含属性名称和所需的验证规则：

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: ['story' => 'required|max:10000']
);
```

### 密码

`password` 函数和 `text` 函数类似，但是当用户在控制台输入时，内容将被遮蔽。这在询问诸如密码这种敏感信息时十分有用：

```php
use function Laravel\Prompts\password;

$password = password('What is your password?');
```

你也可以引入占位文本、默认值和信息提示：

```php
$password = password(
    label: 'What is your password?',
    placeholder: 'password',
    hint: 'Minimum 8 characters.'
);
```

#### 必传的值

如果你要求一个值必传，可以传递 `required` 参数：

```php
$password = password(
    label: 'What is your password?',
    required: true
);
```

如果你想自定义验证信息，你也可以传递一个字符串：

```php
$password = password(
    label: 'What is your password?',
    required: 'The password is required.'
);
```

#### 额外验证

最后，如果你想执行额外的验证逻辑，可以给 `validate` 参数传递一个闭包：

```php
$password = password(
    label: 'What is your password?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 8 => 'The password must be at least 8 characters.',
        default => null
    }
);
```

闭包接收输入的值，可能会返回一个错误信息，如果验证通过会返回 `null`。

或者，你也可以使用 Laravel [验证器](https://learnku.com/docs/laravel/11.x/validation) 这个强大功能。要想使用它，你可以给 `validate` 传递一个数组，数组包含属性名称和所需的验证规则：

```php
$password = password(
    label: 'What is your password?',
    validate: ['password' => 'min:8']
);
```

### 确认

如果你需要询问用户一个「是」或「否」的确认，可以使用 `confirm` 函数。用户可以使用箭头键或者字母 `y` 或者 `n` 键，来选择他们的响应。该函数会返回 `true` 或者 `false`。

```php
use function Laravel\Prompts\confirm;

$confirmed = confirm('Do you accept the terms?');
```

你也可以引入默认值，「是」和「否」标签的自定义措辞，以及信息提示：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    default: false,
    yes: 'I accept',
    no: 'I decline',
    hint: 'The terms must be accepted to continue.'
);
```

#### 强制选择「是」

如有必要，你可以通过传递 `required` 参数强制用户选择「是」：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: true
);
```

如果你想自定义验证信息，你也可以传递一个字符串：

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: 'You must accept the terms to continue.'
);
```

### 单选

如果你需要用户从一组预定义选项中进行选择，可以使用 `select` 函数：

```php
use function Laravel\Prompts\select;

$role = select(
    'What role should the user have?',
    ['Member', 'Contributor', 'Owner'],
);
```

你也可以指定默认选项和信息提示：

```php
$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner'],
    default: 'Owner',
    hint: 'The role may be changed at any time.'
);
```

你也可以给 `options` 传递一个关联数组，让选中的键返回，而不是返回它的值:

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner'
    ],
    default: 'owner'
);
```

当列表开始滚动前，最多展示五个选项。可以通过传递 `scroll` 参数自定义这个值：

```php
$role = select(
    label: 'Which category would you like to assign?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```

#### 验证

不同于其他提示函数，`select` 函数不接受 `required` 参数，因为什么都不选是不可能的。然而，如果你需要展示一个选项但又阻止其被选中，可以给 `validate` 参数传递一个闭包：

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner'
    ],
    validate: fn (string $value) =>
        $value === 'owner' && User::where('role', 'owner')->exists()
            ? 'An owner already exists.'
            : null
);
```

如果 `options` 参数是一个关联数组，闭包将接收选中的键，否则将接收选中的值。闭包可能会返回一个错误信息，如果验证通过则返回 `null`。

### 多选

如果你需要用户能够选择多个选项，可以使用 `multiselect` 函数：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    'What permissions should be assigned?',
    ['Read', 'Create', 'Update', 'Delete']
);
```

你也可以指定默认选项和信息提示：

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete'],
    default: ['Read', 'Create'],
    hint: 'Permissions may be updated at any time.'
);
```

你也可以给 `options` 传递一个关联数组，返回选中选项的键，而不是它们的值:

```php
$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete'
    ],
    default: ['read', 'create']
);
```

当列表开始滚动前，最多展示五个选项。可以通过传递 `scroll` 参数自定义这个值：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```

你也可用通过 `canSelectAll` 参数，让用户轻松选择所有选项：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    canSelectAll: true
);
```

#### 强制选择至少一个值

默认情况下，用户可以选择零个或多个选项。你可以通过传递 `required` 参数，强制用户选择一个或多个选项：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: true,
);
```

如果你想自定义验证信息，可以给 `required` 参数传递一个字符串：

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: 'You must select at least one category',
);
```

#### 验证

如果你想展示一个选项又想阻止其被选中的话，可以给 `validate` 传递一个闭包：

```php
$permissions = multiselect(
    label: 'What permissions should the user have?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete'
    ],
    validate: fn (array $values) => ! in_array('read', $values)
        ? 'All users require the read permission.'
        : null
);
```

如果 `options` 参数是一个关联数组，闭包将接收选中的键，否则将接收选中的值。闭包可能会返回一个错误信息，如果验证通过则返回 `null`。

### 建议

`suggest` 函数可以用于为可能的选项提供自动补全功能。不过用户仍然可以忽视自动补全的提示，提供任意一个答案：

```php
use function Laravel\Prompts\suggest;

$name = suggest('What is your name?', ['Taylor', 'Dayle']);
```

另外，你也可以给 `suggest` 函数传递一个闭包作为第二个参数。当用户每次输入一个字符时，闭包将会被调用。闭包应该接收一个字符串参数，其中包含用户到目前为止的输入，并返回一个自动完成选项的数组：

```php
$name = suggest(
    'What is your name?',
    fn ($value) => collect(['Taylor', 'Dayle'])
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
)
```

你也可以引入占位文本、默认值和信息提示：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    placeholder: 'E.g. Taylor',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```

#### 必传的值

如果你要求一个值必传，可以传递 `required` 参数：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: true
);
```

如果你想自定义验证信息，你也可以传递一个字符串：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: 'Your name is required.'
);
```

#### 额外验证

最后，如果你想执行额外的验证逻辑，可以给 `validate` 参数传递一个闭包：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

闭包接收输入的值，可能会返回一个错误信息，如果验证通过会返回 `null`。

或者，你也可以使用 Laravel [验证器](https://learnku.com/docs/laravel/11.x/validation) 这个强大功能。要想使用它，你可以给 `validate` 传递一个数组，数组包含属性名称和所需的验证规则：

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: ['name' => 'required|min:3|max:255']
);
```

### 搜索

如果你有大量选项供用户选择，在用户使用箭头键选中一个选项之前，可以使用 `search` 函数让用户输入一个搜索查询条件来过滤结果：

```php
use function Laravel\Prompts\search;

$id = search(
    'Search for the user that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

闭包接收用户到目前为止输入的文本，而且必须返回一个选项数组。如果返回一个关联数组，将返回选中选项的键，否则将返回它们的值：

你也可以引入占位文本和信息提示：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

当列表开始滚动前，最多展示五个选项。可以通过传递 `scroll` 参数自定义这个值：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```

#### 验证

如果你想执行额外验证逻辑，可以给 `validate` 参数传递一个闭包：

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (int|string $value) {
        $user = User::findOrFail($value);

        if ($user->opted_out) {
            return 'This user has opted-out of receiving mail.';
        }
    }
);
```

如果 `options` 参数是一个关联数组，闭包将接收选中的键，否则将接收选中的值。闭包可能会返回一个错误信息，如果验证通过则返回 `null`。

### 多选搜索

如果你有大量可查找的选项，并且需要用户能够选择多个选项，在用户使用箭头键和空格键选中选项之前，可以使用 `multisearch` 函数让用户输入一个搜索查询条件来过滤结果：

```php
use function Laravel\Prompts\multisearch;

$ids = multisearch(
    'Search for the users that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

闭包接收用户到目前为止输入的文本，而且必须返回一个选项数组。如果返回一个关联数组，将返回选中选项的键，否则将返回它们的值：

你也可以引入占位文本和信息提示：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

当列表开始滚动前，最多展示五个选项。可以通过传递 `scroll` 参数自定义这个值：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```

#### 强制选择至少一个值

默认情况下，用户可以选择零个或多个选项。你可以通过传递 `required` 参数，强制用户选择一个或多个选项：

```php
$ids = multisearch(
    'Search for the users that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: true,
);
```

如果你想自定义验证信息，可以给 `required` 参数传递一个字符串：

```php
$ids = multisearch(
    'Search for the users that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: 'You must select at least one user.'
);
```

#### 验证

如果你想执行额外的验证逻辑，可以给 `validate` 参数传递一个闭包：

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (array $values) {
        $optedOut = User::where('name', 'like', '%a%')->findMany($values);

        if ($optedOut->isNotEmpty()) {
            return $optedOut->pluck('name')->join(', ', ', and ').' have opted out.';
        }
    }
);
```

如果 `options` 闭包返回一个关联数组，闭包将接收选中的键，否则将接收选中的值。闭包可能会返回一个错误信息，如果验证通过则返回 `null`。

### 暂停

`pause` 函数可用户向用户展示信息文本，通过按下 回车 / 返回 键，等待用户确认其继续操作的意愿：

```php
use function Laravel\Prompts\pause;

pause('Press ENTER to continue.');
```

## 表单

通常，在执行其他操作之前，将按顺序显示多个提示，以收集信息。你可以使用 `form` 函数创建一组提示，供用户完成：

```php
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true)
    ->password('What is your password?', validate: ['password' => 'min:8'])
    ->confirm('Do you accept the terms?')
    ->submit();
```

`submit` 方法将返回一个数字索引数组，其中包含来自表单提示的所有响应。不过，你可以通过 `name` 参数为每个提示提供一个名称。当提供了一个名称时，可以通过该名称访问指定提示的响应:

```php
use App\Models\User;
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->password(
        'What is your password?',
        validate: ['password' => 'min:8'],
        name: 'password',
    )
    ->confirm('Do you accept the terms?')
    ->submit();

User::create([
    'name' => $responses['name'],
    'password' => $responses['password']
]);
```

使用 `form` 函数最大的好处在于，它能够让用户在表单中通过使用 `CTRL + U` 键，返回到之前的提示。这让用户可以调整错误，或者修改选项，而不用取消或者重新启动整个表单。

如果需要对表单中的提示进行更细粒度的控制，可以调用 `add` 方法，而不是直接调用某个提示函数。用户以前提供的所有响应都会传递给 `add` 方法：

```php
use function Laravel\Prompts\form;
use function Laravel\Prompts\outro;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->add(function ($responses) {
        return text("How old are you, {$responses['name']}?");
    }, name: 'age')
    ->submit();

outro("Your name is {$responses['name']} and you are {$responses['age']} years old.");
```

## 提示信息

`note`、`info`、`warning`、`error` 和 `alert` 函数可以用来展示提示信息：

```php
use function Laravel\Prompts\info;

info('Package installed successfully.');
```

## 表格

`table` 函数让展示多行多列的数据变得十分简单。你唯一要做的就是为表格提供列名称和数据：

```php
use function Laravel\Prompts\table;

table(
    ['Name', 'Email'],
    User::all(['name', 'email'])->toArray()
);
```

## 旋转

在执行指定的回调时，`spin` 函数会显示一个带有可选消息的旋转器。它用于指示正在运行的进程，并在完成时返回回调执行的结果:

```php
use function Laravel\Prompts\spin;

$response = spin(
    fn () => Http::get('http://example.com'),
    'Fetching response...'
);
```

> **警告**  
> `spin` 函数要求开启 `pcntl` PHP 扩展，才能让旋转器动起来。当扩展不可用时，将会用一个静态版本的旋转器代替。

## 进度条

对于长时间运行的任务，给用户展示一个进度条是很有帮助的，这可以告知用户任务完成的情况。通过使用 `progress` 函数，Laravel 会展示一个进度条，并在给定的迭代值上推进每次迭代的进度：

```php
use function Laravel\Prompts\progress;

$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: fn ($user) => $this->performTask($user),
);
```

`progress` 函数的作用类似于一个映射函数，它会返回一个数组，包含你的回调每次迭代的返回值：

回调也可以接受 `\Laravel\Prompts\Progress` 的实例，这允许你修改标签并在每次迭代中给出提示：

```php
$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: function ($user, $progress) {
        $progress
            ->label("Updating {$user->name}")
            ->hint("Created on {$user->created_at}");

        return $this->performTask($user);
    },
    hint: 'This may take some time.',
);
```

有时，针对进度条如何推进，你可能需要更多的手动控制。为此，首先，定义流程需要迭代的步骤总数。然后，在处理完每个任务后，通过 `advance` 函数推进进度条：

```php
$progress = progress(label: 'Updating users', steps: 10);

$users = User::all();

$progress->start();

foreach ($users as $user) {
    $this->performTask($user);

    $progress->advance();
}

$progress->finish();
```

## 终端考虑因素

#### 终端宽度

如果任何标签、选项或验证消息的长度超过用户终端中「列」的数量，为了适应都将被自动截断。如果你的用户可能使用较窄的终端，请考虑最小化这些字符串的长度。对于支持 80 个字符长度的终端来说，通常安全的最大长度是 74 个字符。

#### 终端高度

为了适应用户终端的高度，任何接受 `scroll` 参数的提示，配置的值都会被自动减小，包括验证信息的空格。

## 不支持的环境与回退

Laravel Prompts 通过 WSL 支持 macOS、 Linux 和 Windows。由于 PHP Windows 版本的限制，目前不可能在 WSL 之外的 Windows 上使用 Laravel Prompts 。

出于这个原因，Laravel Prompts 支持回退到另一种实现，例如 [Symfony Console Question Helper](https://symfony.com/doc/7.0/components/console/helpers/questionhelper.html) 。

> **注意**  
> 当在 Laravel 框架中使用 Laravel Prompts 时，每个提示的回退都已经为你配置好，当环境不支持时会自动启用。

#### 回退条件

如果你没有使用 Laravel 或者需要自定义何时使用回退行为，可以给 `Prompt` 类的 `fallbackWhen` 静态方法传递一个布尔值：

```php
use Laravel\Prompts\Prompt;

Prompt::fallbackWhen(
    ! $input->isInteractive() || windows_os() || app()->runningUnitTests()
);
```

#### 回退行为

如果你没有使用 Laravel 或者需要自定义回退行为，可以给每个提示类的 `fallbackUsing` 静态方法传递一个闭包：

```php
use Laravel\Prompts\TextPrompt;
use Symfony\Component\Console\Question\Question;
use Symfony\Component\Console\Style\SymfonyStyle;

TextPrompt::fallbackUsing(function (TextPrompt $prompt) use ($input, $output) {
    $question = (new Question($prompt->label, $prompt->default ?: null))
        ->setValidator(function ($answer) use ($prompt) {
            if ($prompt->required && $answer === null) {
                throw new \RuntimeException(is_string($prompt->required) ? $prompt->required : 'Required.');
            }

            if ($prompt->validate) {
                $error = ($prompt->validate)($answer ?? '');

                if ($error) {
                    throw new \RuntimeException($error);
                }
            }

            return $answer;
        });

    return (new SymfonyStyle($input, $output))
        ->askQuestion($question);
});
```

回退闭包必须针对每个提示类单独配置。闭包将接收提示类的实例，并且必须为提示返回适当的类型。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/pr...](https://learnku.com/docs/laravel/11.x/promptsmd/16728)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/pr...](https://learnku.com/docs/laravel/11.x/promptsmd/16728)