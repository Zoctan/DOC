本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Eloquent: 集合

+   [介绍](#introduction)
+   [可用的方法](#available-methods)
+   [自定义集合](#custom-collections)

## 介绍

所有返回一个以上模型结果的 Eloquent 方法都会返回 `Illuminate\Database\Eloquent\Collection` 类的实例，包括通过 `get` 方法获取或者通过关联访问的结果。 Eloquent 集合对象扩展了 Laravel 的 [基本集合](https://learnku.com/docs/laravel/11.x/collections) , 所以它很自然地继承了数十种 Eloquent 模型底层方法。可以查看 Laravel 集合文档，了解这些有用的方法！

所有集合还可以作为迭代器使用，可以像使用 PHP 数组一样对它们进行循环：

```php
use App\Models\User;

$users = User::where('active', 1)->get();

foreach ($users as $user) {
    echo $user->name;
}
```

不过，如前所述，集合比数组强大得多，它提供了各种映射 / 还原操作，可以使用直观的界面进行连锁操作。例如，我们可以删除所有不活动的模型，然后收集每个剩余用户的名字：

```php
$names = User::all()->reject(function (User $user) {
    return $user->active === false;
})->map(function (User $user) {
    return $user->name;
});
```

#### Eloquent 集合转换

大多数 Eloquent 集合方法都会返回一个 Eloquent 集合的新实例，而 `collapse`, `flatten`, `flip`, `keys`, `pluck`, 和 `zip` 方法则会返回一个[基础集合](https://learnku.com/docs/laravel/11.x/collections) 实例。 同样，如果 `map` 操作返回的集合不包含任何 Eloquent 模型，它将被转换为基础集合实例。

## 可用的方法

所有 Eloquent 的集合都继承了 [Laravel collection](https://learnku.com/docs/laravel/11.x/collectionsmd#available-methods) 对象；因此， 他们也继承了所有集合基类提供的强大的方法。

另外， `Illuminate\Database\Eloquent\Collection` 类提供了一套上层的方法来帮你管理你的模型集合。大多数方法返回 `Illuminate\Database\Eloquent\Collection` 实例；然而，也会有一些方法， 例如 `modelKeys`， 它们会返回 `Illuminate\Support\Collection` 类的实例。

#### `append($attributes)`

可以使用 `append` 方法来为集合中的模型[追加属性](https://learnku.com/docs/laravel/11.x/eloquent-serializationmd#appending-values-to-json)。 该方法接受一个数组追加属性或追加单个属性：

```php
    $users->append('team');

    $users->append(['team', 'is_admin']);
```

#### `contains($key, $operator = null, $value = null)`

`contains` 方法可用于判断集合中是否包含指定的模型实例。这个方法接收一个主键或者模型实例：

```php
    $users->contains(1);

    $users->contains(User::find(1));
```

#### `diff($items)`

`diff` 方法返回不在给定集合中的所有模型：

```php
    use App\Models\User;

    $users = $users->diff(User::whereIn('id', [1, 2, 3])->get());
```

#### `except($keys)`

`except` 方法返回给定主键外的所有模型：

```php
    $users = $users->except([1, 2, 3]);
```

#### `find($key)`

`find` 方法查找给定主键的模型。如果 `$key` 是一个模型实例， `find` 将会尝试返回与主键匹配的模型。 如果 `$key` 是一个关联数组， `find` 将返回所有数组主键匹配的模型：

```php
    $users = User::all();

    $user = $users->find(1);
```

#### `fresh($with = [])`

`fresh` 方法用于从数据库中检索集合中每个模型的新实例。此外，还将预加载任何指定的关联关系：

```php
    $users = $users->fresh();

    $users = $users->fresh('comments');
```

#### `intersect($items)`

`intersect` 方法返回给定集合与当前模型的交集：

```php
    use App\Models\User;

    $users = $users->intersect(User::whereIn('id', [1, 2, 3])->get());
```

#### `load($relations)`

`load` 方法为集合中的所有模型加载给定关联关系：

```php
    $users->load(['comments', 'posts']);

    $users->load('comments.author');

    $users->load(['comments', 'posts' => fn ($query) => $query->where('active', 1)]);
```

#### `loadMissing($relations)`

如果尚未加载关联关系，则 `loadMissing` 方法将加载集合中所有模型的给定关联关系：

```php
    $users->loadMissing(['comments', 'posts']);

    $users->loadMissing('comments.author');

    $users->loadMissing(['comments', 'posts' => fn ($query) => $query->where('active', 1)]);
```

#### `modelKeys()`

`modelKeys` 方法返回集合中所有模型的主键：

```php
    $users->modelKeys();

    // [1, 2, 3, 4, 5]
```

#### `makeVisible($attributes)`

`makeVisible` 方法针对集合中平常「隐藏」的属性进行[模型隐藏属性可见](https://learnku.com/docs/laravel/11.x/eloquent-serializationmd#hiding-attributes-from-json) ：

```php
    $users = $users->makeVisible(['address', 'phone_number']);
```

#### `makeHidden($attributes)`

`makeHidden` 方法针对集合中平常「可见」的属性进行[模型属性隐藏](https://learnku.com/docs/laravel/11.x/eloquent-serializationmd#hiding-attributes-from-json) ：

```php
    $users = $users->makeHidden(['address', 'phone_number']);
```

#### `only($keys)`

`only` 方法返回具有给定主键的所有模型：

```php
    $users = $users->only([1, 2, 3]);
```

#### `setVisible($attributes)`

`setVisible` 方法[临时覆盖](https://learnku.com/docs/laravel/11.x/eloquent-serializationmd#temporarily-modifying-attribute-visibility)集合中每个模型的所有可见属性：

```php
    $users = $users->setVisible(['id', 'name']);
```

#### `setHidden($attributes)`

`setHidden` 方法[临时覆盖](https://learnku.com/docs/laravel/11.x/eloquent-serializationmd#temporarily-modifying-attribute-visibility)集合中每个模型的所有隐藏属性:

```php
    $users = $users->setHidden(['email', 'password', 'remember_token']);
```

#### `toQuery()`

`toQuery` 方法返回一个 Eloquent 查询生成器实例，该实例包含集合模型主键上的 `whereIn` 约束：

```php
    use App\Models\User;

    $users = User::where('status', 'VIP')->get();

    $users->toQuery()->update([
        'status' => 'Administrator',
    ]);
```

#### `unique($key = null, $strict = false)`

`unique` 方法返回集合中所有不重复的模型，若模型在集合中存在相同类型且相同主键的另一模型，该模型将被从集合中删除：

```php
    $users = $users->unique();
```

## 自定义集合

如果你想在操作某个模型时使用自定义的 `Collection` 对象，你可以在模型上定义一个 `newCollection` 方法：

```php
<?php

namespace App\Models;

use App\Support\UserCollection;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 创建一个新的合集实例。
     *
     * @param  array<int, \Illuminate\Database\Eloquent\Model>  $models
     * @return \Illuminate\Database\Eloquent\Collection<int, \Illuminate\Database\Eloquent\Model>
     */
    public function newCollection(array $models = []): Collection
    {
        return new UserCollection($models);
    }
}
```

一旦定义了 `newCollection` 方法，每当 Eloquent 返回一个 `Illuminate\Database\Eloquent\Collection` 实例时，你将得到一个你的自定义集合实例。如果你想为应用程序中的所有模型使用自定义集合，你应该在所有应用程序模型都继承的基础模型类上定义 `newCollection` 方法。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-collectionsmd/16704)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-collectionsmd/16704)