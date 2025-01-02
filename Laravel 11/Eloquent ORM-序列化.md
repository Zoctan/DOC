本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Eloquent: 序列化

+   [简介](#introduction)
+   [序列化模型 & 集合](#serializing-models-and-collections)
    +   [序列化为数组](#serializing-to-arrays)
    +   [序列化为 JSON](#serializing-to-json)
+   [隐藏 JSON 属性](#hiding-attributes-from-json)
+   [追加 JSON 值](#appending-values-to-json)
+   [序列化日期](#date-serialization)

## 简介

使用 Laravel 构建 API 时，您经常需要将模型和关系转换为数组或 JSON。Eloquent 包含进行这些转换的便捷方法，以及控制哪些属性包含在模型的序列化表示中。

> 有关处理 Eloquent 模型和集合 JSON 序列化的更强大方法，请查看 [Eloquent API 资源](https://learnku.com/docs/laravel/11.x/eloquent-resources).

## 序列化模型 & 集合

### 序列化为数组

要将模型加载的关系转换为数组，应使用 `toArray` 方法。此方法是递归的，因此所有属性和所有关系（包括关系的关系）都将转换为数组：

```php
use App\Models\User;

$user = User::with('roles')->first();

return $user->toArray();
```

`attributesToArray` 方法可用于将模型的属性转换为数组，但不能将其关系转换为数组：

```php
$user = User::first();

return $user->attributesToArray();
```

还可以通过调用集合实例上的 `toArray` 方法将模型的整个 [集合](https://learnku.com/docs/laravel/11.x/eloquent-collections) 转换为数组：

```php
$users = User::all();

return $users->toArray();
```

### 序列化为 JSON

要将模型转换为 JSON，您应该使用 `toJson` 方法。与 `toArray` 一样，`toJson` 方法是递归的，因此所有属性和关系都将转换为 JSON。您还可以指定 [PHP 支持的](https://secure.php.net/manual/en/function.json-encode.php) 的任何 JSON 编码选项：

```php
use App\Models\User;

$user = User::find(1);

return $user->toJson();

return $user->toJson(JSON_PRETTY_PRINT);
```

或者，您可以将模型或集合转换为字符串，这将自动调用模型或集合上的 `toJson` 方法：

```php
return (string) User::find(1);
```

由于模型和集合在转换为字符串时会转换为 JSON，因此您可以直接从应用程序的路由或控制器返回 Eloquent 对象。当从路由或控制器返回 Eloquent 模型和集合时，Laravel 会自动将它们序列化为 JSON：

```php
Route::get('users', function () {
    return User::all();
});
```

#### 关联关系

当 Eloquent 模型转换为 JSON 时，其加载的关系将自动作为属性包含在 JSON 对象中。此外，尽管 Eloquent 关系方法是使用`驼峰式`方法名称定义的，但关系的 JSON 属性将采用`蛇形式`。

## 隐藏 JSON 属性

有时您可能希望限制模型数组或 JSON 表示中包含的属性（例如密码）。为此，请向您的模型添加 `$hidden` 属性。`$hidden` 属性数组中列出的属性将不会包含在模型的序列化表示中：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The attributes that should be hidden for arrays.
     *
     * @var array
     */
    protected $hidden = ['password'];
}
```

> 要隐藏关系，请将关系的方法名称添加到 Eloquent 模型的 `$hidden` 属性中。

此外，也可以使用属性 `visible` 定义一个模型数组和 JSON 可见的「白名单」。转化后的数组或 JSON 不会出现其他的属性：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The attributes that should be visible in arrays.
     *
     * @var array
     */
    protected $visible = ['first_name', 'last_name'];
}
```

#### 临时修改可见属性

如果你想要在一个模型实例中显示隐藏的属性，可以使用 `makeVisible` 方法。`makeVisible` 方法返回模型实例：

```php
return $user->makeVisible('attribute')->toArray();
```

相应地，如果你想要在一个模型实例中隐藏可见的属性，可以使用 `makeHidden` 方法。

```php
return $user->makeHidden('attribute')->toArray();
```

如果你希望暂时覆盖所有可见或隐藏属性，你可以分别使用 `setVisible` 和 `setHidden` 方法：

```php
return $user->setVisible(['id', 'name'])->toArray();

return $user->setHidden(['email', 'password', 'remember_token'])->toArray();
```

## 追加 JSON 值

有时，在将模型转换为数组或 JSON 时，你可能希望添加数据库中没有对应列的属性。为此，首先为该值定义一个 [访问器](https://learnku.com/docs/laravel/11.x/eloquent-mutatorsmd)：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 确定用户是否为管理员
     */
    protected function isAdmin(): Attribute
    {
        return new Attribute(
            get: fn () => 'yes',
        );
    }
}
```

如果你希望访问器始终附加到模型的数组和 JSON 表示，则可以将属性名称添加到模型的「appends」属性中。请注意，属性名称通常使用其「蛇形命名法」序列化表示来引用，即使访问器的 PHP 方法是使用「驼峰命名法」定义的：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 要附加到模型的数组形式的访问器。
     *
     * @var array
     */
    protected $appends = ['is_admin'];
}
```

使用 `appends` 方法追加属性后，它将包含在模型的数组和 JSON 中。`appends` 数组中的属性也将遵循模型上配置的 `visible` 和 `hidden` 设置

#### 运行时追加

在运行时，你可以指示模型实例使用 `append` 方法附加其他属性。或者，您可以使用 `setAppends` 方法来覆盖给定模型实例的整个附加属性数组：

```php
return $user->append('is_admin')->toArray();

return $user->setAppends(['is_admin'])->toArray();
```

## 日期序列化

#### 自定义默认日期格式

可以通过重写 `serializeDate` 方法来自定义默认序列化格式。此方法不会影响日期在数据库中的存储格式：

```php
/**
 * 准备一个用于数组 / JSON 序列化
 */
protected function serializeDate(DateTimeInterface $date): string
{
    return $date->format('Y-m-d');
}
```

#### 自定义默认日期格式

可以通过在模型的 [转换声明](https://laravel.com/docs/11.x/eloquent-mutators#attribute-casting) 中指定日期格式来自定义单个 Eloquent 日期属性的序列化格式：

```php
protected function casts(): array
{
    return [
        'birthday' => 'date:Y-m-d',
        'joined_at' => 'datetime:Y-m-d H:00',
    ];
}
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-serializationmd/16707)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-serializationmd/16707)