本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Eloquent: 修改器与类型转换

+   [简介](#introduction)
+   [访问器与修改器](#accessors-and-mutators)
    +   [定义访问器](#defining-an-accessor)
    +   [定义修改器](#defining-a-mutator)
+   [属性类型转换](#attribute-casting)
    +   [数组与 JSON 类型转换](#array-and-json-casting)
    +   [日期类型转换](#date-casting)
    +   [枚举类型转换](#enum-casting)
    +   [加密类型转换](#encrypted-casting)
    +   [查询时间类型转换](#query-time-casting)
+   [自定义类型转换](#custom-casts)
    +   [值对象类型转换](#value-object-casting)
    +   [数组 / JSON 序列化](#array-json-serialization)
    +   [入站类型转换](#inbound-casting)
    +   [类型转换参数](#cast-parameters)
    +   [可转换类](#castables)

## 简介

访问器、修改器和属性类型转换允许你在获取或设置 Eloquent 模型实例的属性值时对其进行转换。例如，你可能想将值存储到数据库时使用 [Laravel 加密器](https://learnku.com/docs/laravel/11.x/encryption) 对其进行加密，之后在访问 Eloquent 模型上的属性时自动解密该值。或者，你可能想通过 Eloquent 模型访问时，将存储在数据库中的 JSON 字符串转换为数组。

## 访问器与修改器

### 定义访问器

访问器在访问 Eloquent 属性值时对其进行转换。要定义一个访问器，请在模型上创建一个受保护的方法来代表可访问的属性。该方法名称应与实际基础模型属性 / 数据库列的「驼峰式」表示法相对应（如果适用）。

在这个例子中，我们将为 `first_name` 属性定义一个访问器。当尝试获取 `first_name` 属性的值时，访问器将自动被 Eloquent 调用。所有属性访问器 / 修改器方法必须声明为 `Illuminate\Database\Eloquent\Casts\Attribute` 的返回类型提示 ：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 获取用户的名字。
     */
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
        );
    }
}
```

所有访问器方法返回一个 `Attribute` 实例，该实例定义了属性如何被访问，以及可选地如何被修改。在这个例子中，我们只定义了属性如何被访问。为此，我们向 `Attribute` 类构造函数提供了 `get` 参数。

如你所见，原始列的值被传递给访问器，允许你对其进行篡改并返该值。要访问访问器的值，你可以只访问模型实例的 `first_name` 属性：

```php
use App\Models\User;

$user = User::find(1);

$firstName = $user->first_name;
```

> \[! 注意\]  
> 如果你想让这些计算值被添加到模型的数组 / JSON 表示中，[你需要将它们附加到其中](https://learnku.com/docs/laravel/11.x/eloquent-serialization#appending-values-to-json)。

#### 从多个属性构建值对象

有时你的访问器可能需要将多个模型属性转换为一个「值对象」。为此，你的 `get` 闭包可以接受第二个参数 `$attributes`，它将包含模型当前所有属性的数组自动提供给闭包：

```php
use App\Support\Address;
use Illuminate\Database\Eloquent\Casts\Attribute;

/**
 * 与用户的地址交互。
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn (mixed $value, array $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
    );
}
```

#### 访问器缓存

当从访问器返回值对象时，对值对象所做的任何更改都会在模型保存之前自动同步回模型。这是可能的，因为 Eloquent 保留了访问器返回的实例，因此每次调用访问器时都可以返回相同的实例：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Line 1 Value';
$user->address->lineTwo = 'Updated Address Line 2 Value';

$user->save();
```

然而，有时你可能希望为字符串和布尔等原始值启用缓存，特别是如果它们是计算密集型的。为实现这个，你可以在定义访问器时调用 `shouldCache` 方法：

```php
protected function hash(): Attribute
{
    return Attribute::make(
        get: fn (string $value) => bcrypt(gzuncompress($value)),
    )->shouldCache();
}
```

如果你想禁用属性的对象缓存行为，你可以在定义属性时调用 `withoutObjectCaching` 方法：

```php
/**
 * 与用户的地址交互。
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn (mixed $value, array $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
    )->withoutObjectCaching();
}
```

### 定义修改器

修改器在设置时转换 Eloquent 属性值。要定义一个修改器，你可以在定义属性时提供 `set` 参数。让我们为 `first_name` 属性定义一个修改器。当我们尝试在模型上设置 `first_name` 属性的值时，这个修改器将自动被调用：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 与用户的名字交互。
     */
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => strtolower($value),
        );
    }
}
```

修改器闭包将接收正在设置到属性上的值，允许你对其进行篡改并返回处理后的值。要使用我们的修改器，我们只需要在 Eloquent 模型上设置 `first_name` 属性：

```php
use App\Models\User;

$user = User::find(1);

$user->first_name = 'Sally';
```

在这个例子中，`set` 回调将被调用并传入值 `Sally`。修改器随后会对该名称应用 `strtolower` 函数，并将其结果值设置到模型的内部 `$attributes` 数组中。

#### 修改多个属性

有时你的修改器可能需要设置底层模型上的多个属性。为此，你可以从 `set` 闭包返回一个数组。数组中的每个键应与模型关联的基础属性 / 数据库列相对应：

```php
use App\Support\Address;
use Illuminate\Database\Eloquent\Casts\Attribute;

/**
 * 与用户的地址交互。
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn (mixed $value, array $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
        set: fn (Address $value) => [
            'address_line_one' => $value->lineOne,
            'address_line_two' => $value->lineTwo,
        ],
    );
}
```

## 属性类型转换

属性类型转换提供了类似于访问器和修改器的功能，但不需要在模型上定义任何额外的方法。相反，模型的 `casts` 方法提供了一种便利的方式来将属性转换为常见数据类型。

`casts` 方法应返回一个数组，其中键是要转换的属性的名称，值是你希望将列转换为的类型。支持的转换类型有：

+   `array`
+   `AsStringable::class`
+   `boolean`
+   `collection`
+   `date`
+   `datetime`
+   `immutable_date`
+   `immutable_datetime`
+   `decimal:<precision>`
+   `double`
+   `encrypted`
+   `encrypted:array`
+   `encrypted:collection`
+   `encrypted:object`
+   `float`
+   `hashed`
+   `integer`
+   `object`
+   `real`
+   `string`
+   `timestamp`

为了演示属性类型转换，我们将在数据库中存储为整数 （`0` 或 `1`） 的 `is_admin` 属性转换为布尔值：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 获取应该强制转换的属性。
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'is_admin' => 'boolean',
        ];
    }
}
```

定义了类型转换后，即使底层值在数据库中存储的是整数，当你访问 `is_admin` 属性时，它总是会被转换为布尔值：

```php
$user = App\Models\User::find(1);

if ($user->is_admin) {
    // ...
}
```

如果你需要在运行时添加一个新的临时类型转换，你可以使用 `mergeCasts` 方法。这些类型转换定义将添加到模型上已定义的类型转换中：

```php
$user->mergeCasts([
    'is_admin' => 'integer',
    'options' => 'object',
]);
```

> \[! 警告\]  
> 值为 `null` 的属性不会被转换。此外，你不应该定义一个与关系同名的类型转换（或属性），也不应该给模型的主键配置类型转换。

#### 可字符串化的类型转换

你可以使用 `Illuminate\Database\Eloquent\Casts\AsStringable` 转换类将模型属性转换为[流畅的 `Illuminate\Support\Stringable` 对象](https://learnku.com/docs/laravel/11.x/strings#fluent-strings-method-list)：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\AsStringable;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 获取应该类型转换的属性。
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'directory' => AsStringable::class,
        ];
    }
}
```

### 数组和 JSON 类型转换

`array` 类型转换在处理存储为序列化 JSON 的列时特别有用。例如，如果你的数据库有一个包含序列化 JSON 的 `JSON` 或 `TEXT` 字段类型，将 `array` 类型转换添加到该属性，将在你访问 Eloquent 模型上的该属性时自动将其反序列化为 PHP 数组：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 获取应类型转换的属性。
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'options' => 'array',
        ];
    }
}
```

类型转换一旦定义，你就可以访问 `options` 属性，它将自动从 JSON 反序列化为 PHP 数组。当你设置 `options` 属性的值时，给定的数组将自动序列化回 JSON 以进行存储：

```php
use App\Models\User;

$user = User::find(1);

$options = $user->options;

$options['key'] = 'value';

$user->options = $options;

$user->save();
```

要使用更精简的语法更新 JSON 属性的单个字段，你可以[使属性可批量赋值](https://learnku.com/docs/laravel/11.x/eloquent#mass-assignment-json-columns)，并在调用 `update` 方法时使用 `->` 操作符：

```php
$user = User::find(1);

$user->update(['options->key' => 'value']);
```

#### 数组对象和集合类型转换

虽然标准的 `array` 类型转换对于许多应用来说已经足够，但它也有一些缺点。由于 `array` 转换返回一个原始类型，因此无法直接修改数组的偏移量。例如，以下代码将触发 PHP 错误：

```php
$user = User::find(1);

$user->options['key'] = $value;
```

为解决这个问题，Laravel 提供了一个 `AsArrayObject` 类型转换，它将你的 JSON 属性转换为一个 [ArrayObject](https://www.php.net/manual/en/class.arrayobject.php) 类。这一功能是通过 Laravel 的[自定义转换](#custom-casts)实现的，它允许 Laravel 智能地缓存和转换被修改的对象，从而可以修改单个偏移量而不会触发 PHP 错误。要使用 `AsArrayObject` 转换，只需将其分配给一个属性：

```php
use Illuminate\Database\Eloquent\Casts\AsArrayObject;

/**
 * 获取应类型转换的属性。
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsArrayObject::class,
    ];
}
```

同样，Laravel 提供了一个 `AsCollection` 类型转换，它将你的 JSON 属性转换为一个 Laravel [集合](https://learnku.com/docs/laravel/11.x/collections)实例：

```php
use Illuminate\Database\Eloquent\Casts\AsCollection;

/**
 * 获取应类型转换的属性。
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsCollection::class,
    ];
}
```

如果你想 `AsCollection` 类型转换实例化一个自定义集合类而不是 Laravel 的基础集合类，你可以将集合类名称作为转换参数提供：

```php
use App\Collections\OptionCollection;
use Illuminate\Database\Eloquent\Casts\AsCollection;

/**
 * 获取应类型转换的属性。
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'options' => AsCollection::using(OptionCollection::class),
    ];
}
```

### 日期类型转换

默认情况下，Eloquent 会将 `created_at` 和 `updated_at` 列转换为 [Carbon](https://github.com/briannesbitt/Carbon) 实例，该实例扩展了 PHP 的 `DateTime` 类并提供了许多有用的方法。你可以通过在模型的 `casts` 方法中定义额外的日期转换来转换额外的日期属性。通常，日期应该使用 `datetime` 或 `immutable_datetime` 转换类型进行转换。

在定义 `date` 或 `datetime` 类型转换时，你还可以指定日期的格式。该格式将在[模型序列化为数组或 JSON](https://learnku.com/docs/laravel/11.x/eloquent-serialization) 时使用：

```php
/**
 * 获取应类型转换的属性。
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'created_at' => 'datetime:Y-m-d',
    ];
}
```

当一个列被转换为日期时，你可以将相应的模型属性值设置为 UNIX 时间戳、日期字符串（`Y-m-d`）、日期时间字符串、或 `DateTime` / `Carbon` 实例。日期的值将被正确转换并存储在你的数据库中。

你可以通过在模型上定义一个 `serializeDate` 方法来自定义所有模型日期的默认序列化格式。此方法不会影响日期在数据库中的存储格式：

```php
/**
 * 为数组/ JSON 序列化准备一个日期。
 */
protected function serializeDate(DateTimeInterface $date): string
{
    return $date->format('Y-m-d');
}
```

要指定在实际存储模型日期时应使用的格式，你应在模型上定义一个 `$dateFormat` 属性：

```php
/**
 * 模型日期列的存储格式。
 *
 * @var string
 */
protected $dateFormat = 'U';
```

#### 日期转换、序列化和时区

默认情况下，不管你的应用程序的 `timezone` 配置选项中指定在哪个时区，`date` 和 `datetime` 转换会将日期序列化为 UTC ISO-8601 日期字符串（`YYYY-MM-DDTHH:MM:SS.uuuuuuZ`）。强烈建议你始终使用这种序列化格式，并通过不变更应用程序的 `timezone` 配置选项的默认的 `UTC` 值，来将应用程序的日期存储在 UTC 时区。在整个应用程序中一致地使用 UTC 时区，将提供与其他用 PHP 和 JavaScript 编写的日期操作库最大的互操作性。

如果将自定义格式应用于 `date` 或 `datetime` 转换，例如 `datetime:Y-m-d H:i:s`，在日期序列化期间将使用 Carbon 实例的内部时区。通常，这将是应用程序的 `timezone` 配置选项中指定的时区。

### 枚举类型转换

Eloquent 还允许你将属性值转换为 PHP [枚举](https://www.php.net/manual/en/language.enumerations.backed.php)。为实现这点，你可以在模型的 `casts` 方法中指定要转换的属性和枚举：

```php
use App\Enums\ServerStatus;

/**
 * 获取应类型转换的属性。
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'status' => ServerStatus::class,
    ];
}
```

一旦你在模型上定义了转换，当你与该属性交互时，指定的属性将自动转换为枚举或从枚举转换回来：

```php
if ($server->status == ServerStatus::Provisioned) {
    $server->status = ServerStatus::Ready;

    $server->save();
}
```

#### 枚举数组类型转换

有时你可能需要在单个列中存储一组枚举值。为此，你可以利用 Laravel 提供的 `AsEnumArrayObject` 或 `AsEnumCollection` 转换：

```php
use App\Enums\ServerStatus;
use Illuminate\Database\Eloquent\Casts\AsEnumCollection;

/**
 * 获取应类型转换的属性。
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'statuses' => AsEnumCollection::of(ServerStatus::class),
    ];
}
```

### 加密转换

`encrypted` 转换将使用 Laravel 内置的[加密](https://learnku.com/docs/laravel/11.x/encryption)功能对模型的属性值进行加密。此外，`encrypted:array`、`encrypted:collection`、`encrypted:object`、`AsEncryptedArrayObject` 和 `AsEncryptedCollection` 转换的工作方式与其未加密的对应项类似；不过，正如你可能预期的那样，底层值在存储到数据库时会被加密。

由于加密文本的最终长度是不可预测的，并且比其对应明文长，因此请确保关联的数据库列是 `TEXT` 或更大的类型。此外，由于值在数据库中被加密，你将无法查询或搜索加密的属性值。

#### 密钥轮换

如你所知，Laravel 使用应用程序的 `app` 配置文件中指定的 `key` 配置值来加密字符串。通常，该值对应于 `APP_KEY` 环境变量的值。如果你需要轮换应用程序的加密密钥，你将需要使用新密钥手动重新加密你的加密属性。

### 查询时间类型转换

有时你可能需要在执行查询时应用类型转换，比如在从一个表中选择原始值时。例如，考虑以下查询：

```php
use App\Models\Post;
use App\Models\User;

$users = User::select([
    'users.*',
    'last_posted_at' => Post::selectRaw('MAX(created_at)')
            ->whereColumn('user_id', 'users.id')
])->get();
```

此查询结果中的 `last_posted_at` 属性将是一个简单的字符串。如果我们能在执行查询时对此属性应用 `datetime` 类型转换，那就太好了。谢天谢地，我们可以使用 `withCasts` 方法来实现这一点：

```php
$users = User::select([
    'users.*',
    'last_posted_at' => Post::selectRaw('MAX(created_at)')
            ->whereColumn('user_id', 'users.id')
])->withCasts([
    'last_posted_at' => 'datetime'
])->get();
```

## 自定义类型转换

Laravel 有许多内置的很有用的转换类型；然而，你有时可能需要定义自己的转换类型。要创建一个类型转换，请执行 `make:cast` Artisan 命令。新的转换类将放置在你的 `app/Casts` 目录中：

```shell
php artisan make:cast Json
```

所有自定义转换类都实现了 `CastsAttributes` 接口。实现此接口的类必须定义一个 `get` 和 `set` 方法。`get` 方法负责将数据库中的原始值变换为转换值，而 `set` 方法应该将转换值变换为可以存储在数据库中的原始值。作为一个例子，我们将内置的 `json` 转换类型重新实现为一个自定义转换类型：

```php
<?php

namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

class Json implements CastsAttributes
{
    /**
     * 转换给定的值。
     *
     * @param  array<string, mixed>  $attributes
     * @return array<string, mixed>
     */
    public function get(Model $model, string $key, mixed $value, array $attributes): array
    {
        return json_decode($value, true);
    }

    /**
     * 为存储准备给定的值。
     *
     * @param  array<string, mixed>  $attributes
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): string
    {
        return json_encode($value);
    }
}
```

一旦明确好一个自定义转换类型，你就可以使用其类名将其附加到模型属性上：

```php
<?php

namespace App\Models;

use App\Casts\Json;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 获取应类型转换的属性。
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'options' => Json::class,
        ];
    }
}
```

### 值对象类型转换

不限于将值转换为原始类型，你还可以将值转换为对象。定义将值转换为对象的自定义转换与转换为原始类型非常相似；但是，`set` 方法应返回一个键 / 值对数组，这些键 / 值对将用于在模型上设置原始的可存储值。

作为一个例子，我们将定义一个自定义转换类，该类将多个模型值转换为一个 `Address` 值对象。我们假设 `Address` 值对象有两个公共属性：`lineOne` 和 `lineTwo` ：

```php
<?php

namespace App\Casts;

use App\ValueObjects\Address as AddressValueObject;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;
use InvalidArgumentException;

class Address implements CastsAttributes
{
    /**
     * 转换给定的值。
     *
     * @param  array<string, mixed>  $attributes
     */
    public function get(Model $model, string $key, mixed $value, array $attributes): AddressValueObject
    {
        return new AddressValueObject(
            $attributes['address_line_one'],
            $attributes['address_line_two']
        );
    }

    /**
     * 为存储准备给定的值。
     *
     * @param  array<string, mixed>  $attributes
     * @return array<string, string>
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): array
    {
        if (! $value instanceof AddressValueObject) {
            throw new InvalidArgumentException('The given value is not an Address instance.');
        }

        return [
            'address_line_one' => $value->lineOne,
            'address_line_two' => $value->lineTwo,
        ];
    }
}
```

在类型转换为值对象时，对值对象所做的任何更改都会在模型保存之前自动同步回模型：

```php
use App\Models\User;

$user = User::find(1);

$user->address->lineOne = 'Updated Address Value';

$user->save();
```

> \[! 注意\]  
> 如果你打算将包含值对象的 Eloquent 模型序列化为 JSON 或数组，你应该在值对象上实现 `Illuminate\Contracts\Support\Arrayable` 和 `JsonSerializable` 接口。

#### 值对象缓存

解析转换为值对象的属性时，它们会被 Eloquent 缓存。因此，如果再次访问该属性，将返回相同的对象实例。

如果你想禁用自定义类型转换类的对象缓存行为，可以在自定义转换类上声明一个公共的 `withoutObjectCaching` 属性：

```php
class Address implements CastsAttributes
{
    public bool $withoutObjectCaching = true;

    // ...
}
```

### 数组 / JSON 序列化

当使用 `toArray` 和 `toJson` 方法将 Eloquent 模型转换为数组或 JSON 时，只要你的自定义转换值对象有实现 `Illuminate\Contracts\Support\Arrayable` 和 `JsonSerializable` 接口，它们通常也会被序列化。可是，当使用第三方库提供的值对象时，你可能没有能力向对象添加这些接口。

因此，你可以指定你的自定义转换类用于负责序列化值对象。为此，你的自定义转换类应该实现 `Illuminate\Contracts\Database\Eloquent\SerializesCastableAttributes` 接口。该接口表明你的类应该包含一个 `serialize` 方法，该方法应返回值对象的序列化形式：

```php
/**
 * 获取值的序列化表示形式。
 *
 * @param  array<string, mixed>  $attributes
 */
public function serialize(Model $model, string $key, mixed $value, array $attributes): string
{
    return (string) $value;
}
```

### 入站类型转换

有时，你可能需要编写一个自定义转换类，该类仅转换正在设置到模型上的值，并且在从模型检索属性时不执行任何操作。

仅入站的自定义转换应实现 `CastsInboundAttributes` 接口，该接口仅需要定义一个 `set` 方法。调用带 `--inbound` 选项的 `make:cast` Artisan 命令来生成仅入站的转换类：

```shell
php artisan make:cast Hash --inbound
```

仅入站转换的一个经典例子是「哈希」转换。例如，我们可以定义一个通过给定算法对入站值进行哈希处理的转换：

```php
<?php

namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsInboundAttributes;
use Illuminate\Database\Eloquent\Model;

class Hash implements CastsInboundAttributes
{
    /**
     * 创建一个新的类型转换类实例。
     */
    public function __construct(
        protected string|null $algorithm = null,
    ) {}

    /**
     * 为存储准备给定的值。
     *
     * @param  array<string, mixed>  $attributes
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): string
    {
        return is_null($this->algorithm)
                    ? bcrypt($value)
                    : hash($this->algorithm, $value);
    }
}
```

### 类型转换参数

当将自定义转换附加到模型时，可以通过使用 `:` 字符分隔类名称并将多个参数用逗号分隔来指定转换参数。参数将被传递给转换类的构造函数：

```php
/**
 * 获取应类型转换的属性。
 *
 * @return array<string, string>
 */
protected function casts(): array
{
    return [
        'secret' => Hash::class.':sha256',
    ];
}
```

### 可转换类

你可能想要允许应用程序的值对象定义它们自己的自定义转换类。你不需要将自定义转换类附加到模型上，而是可以附加一个实现了 `Illuminate\Contracts\Database\Eloquent\Castable` 接口的值对象类：

```php
use App\ValueObjects\Address;

protected function casts(): array
{
    return [
        'address' => Address::class,
    ];
}
```

实现 `Castable` 接口的对象必须定义一个 `castUsing` 方法，该方法返回负责从 `Castable` 类转换的自定义转换类类名：

```php
<?php

namespace App\ValueObjects;

use Illuminate\Contracts\Database\Eloquent\Castable;
use App\Casts\Address as AddressCast;

class Address implements Castable
{
    /**
     * 获取在从/向此转换目标转换时要使用的转换类名称。
     *
     * @param  array<string, mixed>  $arguments
     */
    public static function castUsing(array $arguments): string
    {
        return AddressCast::class;
    }
}
```

当使用 `Castable` 类时，你仍然可以在 `casts` 方法定义中提供参数。参数将被传递给 `castUsing` 方法：

```php
use App\ValueObjects\Address;

protected function casts(): array
{
    return [
        'address' => Address::class.':argument',
    ];
}
```

#### 可转换类和匿名转换类

通过将「可转换类」与 PHP 的[匿名类](https://www.php.net/manual/en/language.oop5.anonymous.php)结合，你可以将值对象及其转换逻辑定义为一个单一的可转换对象。为此，从值对象的 `castUsing` 方法返回一个匿名类。匿名类应实现 `CastsAttributes` 接口：

```php
<?php

namespace App\ValueObjects;

use Illuminate\Contracts\Database\Eloquent\Castable;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;

class Address implements Castable
{
    // ...

    /**
     * 获取在从/向此转换目标转换时要使用的转换类。
     *
     * @param  array<string, mixed>  $arguments
     */
    public static function castUsing(array $arguments): CastsAttributes
    {
        return new class implements CastsAttributes
        {
            public function get(Model $model, string $key, mixed $value, array $attributes): Address
            {
                return new Address(
                    $attributes['address_line_one'],
                    $attributes['address_line_two']
                );
            }

            public function set(Model $model, string $key, mixed $value, array $attributes): array
            {
                return [
                    'address_line_one' => $value->lineOne,
                    'address_line_two' => $value->lineTwo,
                ];
            }
        };
    }
}
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-mutatorsmd/16705)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-mutatorsmd/16705)