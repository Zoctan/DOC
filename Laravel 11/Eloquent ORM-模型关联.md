本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Eloquent: Relationships

+   [简介](#introduction)
+   [定义关联](#defining-relationships)
    +   [一对一](#one-to-one)
    +   [一对多](#one-to-many)
    +   [一对多 (反向) / 属于](#one-to-many-inverse)
    +   [一对多检索](#has-one-of-many)
    +   [远程一对一](#has-one-through)
    +   [远程一对多](#has-many-through)
+   [多对多关联](#many-to-many)
    +   [获取中间表字段](#retrieving-intermediate-table-columns)
    +   [通过中间表字段过滤查询](#filtering-queries-via-intermediate-table-columns)
    +   [通过中间表字段排序查询](#ordering-queries-via-intermediate-table-columns)
    +   [自定义中间表模型](#defining-custom-intermediate-table-models)
+   [多态关联](#polymorphic-relationships)
    +   [一对一](#one-to-one-polymorphic-relations)
    +   [一对多](#one-to-many-polymorphic-relations)
    +   [一对多检索](#one-of-many-polymorphic-relations)
    +   [多对多](#many-to-many-polymorphic-relations)
    +   [自定义多态模型](#custom-polymorphic-types)
+   [动态关联](#dynamic-relationships)
+   [查询关联](#querying-relations)
    +   [关联方法与动态属性](#relationship-methods-vs-dynamic-properties)
    +   [基于存在的关联查询](#querying-relationship-existence)
    +   [基于不存在的关联查询](#querying-relationship-absence)
    +   [基于多态的关联查询](#querying-morph-to-relationships)
+   [统计关联模型](#aggregating-related-models)
    +   [关联模型计数](#counting-related-models)
    +   [其他统计函数](#other-aggregate-functions)
    +   [多态关联数据计数](#counting-related-models-on-morph-to-relationships)
+   [预加载](#eager-loading)
    +   [约束预加载](#constraining-eager-loads)
    +   [延迟预加载](#lazy-eager-loading)
    +   [阻止懒加载](#preventing-lazy-loading)
+   [插入及更新关联模型](#inserting-and-updating-related-models)
    +   [save 方法](#the-save-method)
    +   [create 方法](#the-create-method)
    +   [属于关联](#updating-belongs-to-relationships)
    +   [多对多关联](#updating-many-to-many-relationships)
+   [更新父级时间戳](#touching-parent-timestamps)

## 简介

数据库表通常相互关联。例如，一篇博客文章可能有许多评论，或者一个订单对应一个下单用户。`Eloquent` 让这些关联的管理和使用变得简单，并支持多种常用的关联类型：

+   [一对一](#one-to-one)
+   [一对多](#one-to-many)
+   [多对多](#many-to-many)
+   [远程一对一](#has-one-through)
+   [远程一对多](#has-many-through)
+   [多态一对一](#one-to-one-polymorphic-relations)
+   [多态一对多](#one-to-many-polymorphic-relations)
+   [多态多对多](#many-to-many-polymorphic-relations)

## 定义关联

Eloquent 关联是在 Eloquent 模型类中作为方法定义的。关联同时也是强大的 [查询构造器](https://learnku.com/docs/laravel/11.x/queriesmd)，定义关联提供了强大的链式调用和查询功能。例如，可以在 `post` 关联的链式调用中附加一个约束条件：

$user->posts()->where(‘active’, 1)->get();

在深入使用关联之前，让我们先学习如何定义 Eloquent 支持的每种关联类型

### 一对一

一对一是最基本的数据库关系。例如，一个 `User` 模型一个 `Phone` 模型关联。定义这个关联，要在 `User` 模型写一个 `Phone` 方法。在这个 `Phone` 方法中调用 `hasOne` 方法并返回其结果。`hasOne` 方法被定义在 `Illuminate\Database\Eloquent\Model` 这个模型基类中：

```php
  <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Relations\HasOne;

    class User extends Model
    {
        /**
         * 获取与用户关联的电话。
  */
        public function phone(): HasOne
        {
            return $this->hasOne(Phone::class);
        }
    }
```

`hasOne` 方法的第一个参数是关联模型的类名。关联一旦定义，就可以使用 Eloquent 的动态属性获得相关记录。动态属性允许访问该关联方法，就像访问模型中定一个属性一样：

```php
  $phone = User::find(1)->phone;
```

Eloquent 基于父模型的名称来确定关联模型的外键名称。在本例中，`Phone` 模型会被自动假定有个 `user_id` 的外键。如果想要覆盖这个约定，可以传递第二个参数给 `hasOne` 方法：

```php
return $this->hasOne(Phone::class, 'foreign_key');
```

另外，Eloquent 假设外键的值是与父模型的主键（Primary Key）相同的。换句话说，Eloquent 将会通过 `Phone` 记录的 `user_id` 列中查找与用户表的 `id` 列相匹配的值。如果你希望使用自定义的主键值，而不是使用 `id` 或者模型中的 `$primaryKey` 属性，你可以给 `hasOne` 方法传递第三个参数：

```php
  return $this->hasOne(Phone::class, 'foreign_key', 'local_key');
```

#### 定义反向关联

现在已经能从 `User` 模型访问到 `Phone` 模型了。接下来，让我们再在 `Phone` 模型上定义一个关联，它能让我们访问到拥有该电话的用户。我们可以使用 `belongsTo` 方法来定义反向关联， `belongsTo` 方法与 `hasOne` 方法相对应：

```php
 <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Relations\BelongsTo;

    class Phone extends Model
    {
        /**
         * 获取拥有该手机的用户。
  */
        public function user(): BelongsTo
        {
            return $this->belongsTo(User::class);
        }
    }
```

在调用 `user` 方法时，Eloquent 会尝试查找一个 `User` 模型，该 `User` 模型上的 `id` 字段会与 `Phone` 模型上的 `user_id` 字段相匹配。

Eloquent 通过关联方法（`user`）的名称并使用 `_id` 作为后缀名来确定外键名称。因此，在本例中，Eloquent 会假设 `Phone` 模型有一个 `user_id` 字段。但是，如果 `Phone` 模型的外键不是 `user_id`，这时你可以给 `belongsTo` 方法的第二个参数传递一个自定义键名：

```php
  /**
    * 获取拥有该手机的用户。
    */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'foreign_key');
    }
```

如果父模型的主键未使用 `id` 作为字段名，或者你想要使用其他的字段来匹配相关联的模型，那么你可以向 `belongsTo` 方法传递第三个参数，这个参数是在父模型中自己定义的字段名称：

```php
  /**
    * 获取拥有此电话的用户
    */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'foreign_key', 'owner_key');
    }
```

### 一对多

当要定义一个模型是其他 （一个或者多个）模型的父模型这种关系时，可以使用一对多关联。例如，一篇博客可以有很多条评论。和其他模型关联一样，一对多关联也是在 Eloquent 模型文件中用一个方法来定义的：

```php
  <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Relations\HasMany;

    class Post extends Model
    {
        /**
        * 获取这篇博客的所有评论
        */
        public function comments(): HasMany
        {
            return $this->hasMany(Comment::class);
        }
    }
```

注意，Eloquent 将会自动为 `Comment` 模型选择一个合适的外键。通常，这个外键是通过使用父模型的「蛇形命名」方式，然后再加上 `_id`. 的方式来命名的。因此，在上面这个例子中，Eloquent 将会默认 `Comment` 模型的外键是 `post_id` 字段。

如果关联方法被定义，那么我们就可以通过 `comments` 属性来访问相关的评论 [集合](https://learnku.com/docs/laravel/11.x/eloquent-collections)。注意，由于 Eloquent 提供了「动态属性」，所以我们就可以像访问模型属性一样来访问关联方法：

```php
    use App\Models\Post;

    $comments = Post::find(1)->comments;

    foreach ($comments as $comment) {
        // ...
    }
```

由于所有的关系都可以看成是查询构造器，所以你也可以通过链式调用的方式，在 `comments` 方法中继续添加条件约束：

```php
 $comment = Post::find(1)
    ->comments()
    ->where('title', 'foo')
    ->first();
```

像 `hasOne` 方法一样，你也可以通过将附加参数传递给 `hasMany` 方法来覆盖外键和本地键：

```php
  // 覆盖外键
  return $this->hasMany(Comment::class, 'foreign_key');

  // 覆盖外键和本地键
  return $this->hasMany(Comment::class, 'foreign_key', 'local_key');
```

### 一对多 (反向) / 属于

现在我们可以访问一篇文章的所有评论，下面我们可以定义一个关联关系，从而让我们可以通过一条评论来获取到它所属的文章。这个关联关系是 `hasMany` 的反向，可以在子模型中通过 `belongsTo` 方法来定义这种关联关系：

```php
  <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Relations\BelongsTo;

    class Comment extends Model
    {
        /**
        * 获取这条评论所属的文章。
        */
        public function post(): BelongsTo
        {
            return $this->belongsTo(Post::class);
        }
    }
```

如果定义了这种关联关系，那么我们就可以通过 `Comment` 模型中的 `post` 「动态属性」来获取到这条评论所属的文章：

```php
    use App\Models\Comment;

    $comment = Comment::find(1);
    return $comment->post->title;
```

在上面这个例子中，Eloquent 将会尝试寻找 `Post` 模型中的 `id` 字段与 `Comment` 模型中的 `post_id` 字段相匹配。

Eloquent 通过检查关联方法的名称，从而在关联方法名称后面加上 `_` ，然后再加上父模型 （Post）的主键名称，以此来作为默认的外键名。因此，在上面这个例子中，Eloquent 将会默认 `Post` 模型在 `comments` 表中的外键是 `post_id`。

但是，如果你的外键不遵循这种约定的话，那么你可以传递一个自定义的外键名来作为 `belongsTo` 方法的第二个参数：

```php
  /**
    * 获取这条评论所属的文章。
    */
    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class, 'foreign_key');
    }
```

如果你的父表不使用 `id` 作为主键，或者你希望使用不同的列来关联模型，你可以将第三个参数传递给 `belongsTo` 方法，指定父表的自定义键：

```php
   /**
    * 获取这条评论所属的文章。
   */
    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class, 'foreign_key', 'owner_key');
    }
```

#### 默认模型

当 `belongsTo`，`hasOne`，`hasOneThrough` 和 `morphOne` 这些关联方法返回 `null` 的时候，你可以定义一个默认的模型返回。该模式通常被称为 [空对象模式](https://en.wikipedia.org/wiki/Null_Object_pattern)，它可以帮你省略代码中的一些条件判断。在下面这个例子中，如果 `Post` 模型中没有用户，那么 `user` 关联关系将会返回一个空的 `App\Models\User` 模型：

```php
   /**
    * 获取文章的作者。
    */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class)->withDefault();
    }
```

可以向 `withDefault` 方法传递数组或者闭包来填充默认模型的属性。

```php
   /**
    * 获取文章的作者。
    */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class)->withDefault([
            'name' => 'Guest Author',
        ]);
    }

   /**
    * 获取文章的作者。
    */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class)->withDefault(function (User $user, Post $post) {
            $user->name = 'Guest Author';
        });
    }
```

#### 查询所属关系

在查询「所属」的子模型时，可以构建 `where` 语句来检索相应的 Eloquent 模型：

```php
    use App\Models\Post;
    $posts = Post::where('user_id', $user->id)->get();
```

但是，你会发现使用 `whereBelongsTo` 方法更方便，它会自动确定给定模型的正确关系和外键：

```php
    $posts = Post::whereBelongsTo($user)->get();
```

你还可以向 `whereBelongsTo` 方法提供一个 [集合](https://learnku.com/docs/laravel/11.x/eloquent-collections) 实例。 这样 Laravel 将检索属于集合中任何父模型的子模型：

```php
    $users = User::where('vip', true)->get();

    $posts = Post::whereBelongsTo($users)->get();
```

默认情况下，Laravel 将根据模型的类名来确定给定模型的关联关系； 你也可以通过将关系名称作为 `whereBelongsTo` 方法的第二个参数来手动指定关系名称：

```php
    $posts = Post::whereBelongsTo($user, 'author')->get();
```

### 一对多检索

有时一个模型可能有许多相关模型，如果你想很轻松的检索「最新」或「最旧」的相关模型。例如，一个 `User` 模型可能与许多 `Order` 模型相关，但你想定义一种方便的方式来与用户最近下的订单进行交互。 可以使用 `hasOne` 关系类型结合 `ofMany` 方法来完成此操作：

```php
   /**
    * 获取用户最新的订单。
    */
    public function latestOrder(): HasOne
    {
        return $this->hasOne(Order::class)->latestOfMany();
    }
```

同样，你可以定义一个方法来检索 「oldest」或第一个相关模型：

```php
   /**
    * 获取用户最早的订单。
    */
    public function oldestOrder(): HasOne
    {
        return $this->hasOne(Order::class)->oldestOfMany();
    }
```

默认情况下，`latestOfMany` 和 `oldestOfMany` 方法将根据模型的主键检索最新或最旧的相关模型，该主键必须是可排序的。 但是，有时你可能希望使用不同的排序条件从更大的关系中检索单个模型。

例如，使用 `ofMany` 方法，可以检索用户最昂贵的订单。 `ofMany` 方法接受可排序列作为其第一个参数，以及在查询相关模型时应用哪个聚合函数（`min` 或 `max`）：

```php
   /**
    * 获取用户最昂贵的订单。
    */
    public function largestOrder(): HasOne
    {
        return $this->hasOne(Order::class)->ofMany('price', 'max');
    }
```

> \[!WARNING\]  
> 因为 PostgreSQL 不支持对 UUID 列执行 `MAX` 函数，所以目前无法将一对多关系与 PostgreSQL UUID 列结合使用。

#### 转换 一对多关联 为 一对一关联

通常，当使用 `latestOfMany`， `oldestOfMany`， 或者 `ofMany` 方法检索单个模型时，该模型已经有一个 「 has many 」 关联。为了方便，Laravel 允许调用已有「 has many 」关联的 `one` 方法 ， 轻松转换此关系为 「 has one 」关联。

```php
   /**
    * 获取用户的订单。
    */
    public function orders(): HasMany
    {
        return $this->hasMany(Order::class);
    }

    /**
     * 获取用户最昂贵的订单。
     */
    public function largestOrder(): HasOne
    {
        return $this->orders()->one()->ofMany('price', 'max');
    }
```

#### 进阶一对多关联

可以构建更高级的「一对多」关联。例如，一个 `Product` 模型可能有许多关联的 `Price` 模型，即使在新定价发布后，这些模型也会保留在系统中。此外，产品的新定价数据能够通过 `published_at` 列提前发布，以便在未来某日生效。  
因此，我们需要检索最新的发布定价。 此外，如果两个价格的发布日期相同，我们优先选择 ID 更大的价格。 为此，我们必须将一个数组传递给 `ofMany` 方法，其中包含确定最新价格的可排序列。此外，将提供一个闭包作为 `ofMany` 方法的第二个参数。此闭包将负责向关系查询添加额外的发布日期约束：

```php
   /**
    * 获取产品的当前定价。
    */
    public function currentPricing(): HasOne
    {
        return $this->hasOne(Price::class)->ofMany([
            'published_at' => 'max',
            'id' => 'max',
        ], function (Builder $query) {
            $query->where('published_at', '<', now());
        });
    }
```

### 远程一对一

「远程一对一」关联定义了与另一个模型的一对一的关联。然而，这种关联是声明的模型通过第三个模型来与另一个模型的一个实例相匹配。  
例如，在一个汽车维修的应用程序中，每一个 `Mechanic` 模型都与一个 `Car` 模型相关联，同时每一个 `Car` 模型也和一个 `Owner` 模型相关联。虽然维修师（mechanic）和车主（owner）在数据库中并没有直接的关联，但是维修师可以通过 `Car` 模型来找到车主。让我们来看看定义这种关联所需要的数据表：

```yaml

  mechanics
        id - integer
        name - string

    cars
        id - integer
        model - string
        mechanic_id - integer

    owners
        id - integer
        name - string
        car_id - integer
```

既然我们已经了解了远程一对一的表结构，那么我们就可以在 `Mechanic` 模型中定义这种关联：

```php
<?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Relations\HasOneThrough;

    class Mechanic extends Model
    {
       /**
        * 获取汽车的主人。
        */
        public function carOwner(): HasOneThrough
        {
            return $this->hasOneThrough(Owner::class, Car::class);
        }
    }
```

传递给 `hasOneThrough` 方法的第一个参数是我们希望访问的最终模型的名称，而第二个参数是中间模型的名称。

或者，如果相关的关联已经在关联中涉及的所有模型上被定义，你可以通过调用 `through` 方法和提供这些关联的名称来流式定义一个「远程一对一」关联。例如，`Mechanic` 模型有一个 `cars` 关联，`Car` 模型有一个 `owner` 关联，你可以这样定义一个连接维修师和车主的「远程一对一」关联：

```php
// 基于字符串的语法...
return $this->through('cars')->has('owner');

// 动态语法...
return $this->throughCars()->hasOwner();
```

#### 键名约定

当使用远程一对一进行关联查询时，Eloquent 将会使用约定的外键名。如果你想要自定义相关联的键名的话，可以传递两个参数来作为 `hasOneThrough` 方法的第三个和第四个参数。第三个参数是中间表的外键名。第四个参数是最终想要访问的模型的外键名。第五个参数是当前模型的本地键名，第六个参数是中间模型的本地键名：

```php
  class Mechanic extends Model
    {
       /**
        * 获取汽车的主人
        */
        public function carOwner(): HasOneThrough
        {
            return $this->hasOneThrough(
                Owner::class,
                Car::class,
                'mechanic_id', // 汽车表的外键mechanic_id...
                'car_id', // 车主表的外键car_id...
                'id', // 机械师表的本地键...
                'id' // 汽车表的本地键...
            );
        }
    }
```

如果所涉及的模型已经定义了相关关系，可以调用 `through` 方法并提供关系名来定义「远程一对一」关联。该方法的优点是重复使用已有关系上定义的主键约定：

```php
// 基本语法...
return $this->through('cars')->has('owner');

// 动态语法...
return $this->throughCars()->hasOwner();
```

### 远程一对多

「远程一对多」关联是可以通过中间关系来实现远程一对多的。例如，我们正在构建一个像 [Laravel Vapor](https://vapor.laravel.com/) 这样的部署平台。一个 `Project` 模型可以通过一个中间的 `Environment` 模型来访问许多个 `Deployment` 模型。就像上面的这个例子，可以在给定的 environment 中很方便的获取所有的 deployments。下面是定义这种关联关系所需要的数据表：

```yaml
  projects
        id - integer
        name - string

    environments
        id - integer
        project_id - integer
        name - string

    deployments
        id - integer
        environment_id - integer
        commit_hash - string
```

既然我们已经检查了关系的表结构，现在让我们在 `Project` 模型上定义该关系：

```php
  <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Relations\HasManyThrough;

    class Project extends Model
    {
        /**
         * Get all of the deployments for the project.
         */
        public function deployments(): HasManyThrough
        {
            return $this->hasManyThrough(Deployment::class, Environment::class);
        }
    }
```

`hasManyThrough` 方法中传递的第一个参数是我们希望访问的最终模型名称，而第二个参数是中间模型的名称。

或者，所有模型上都定义好了关系，你可以通过调用 `through` 方法并提供这些关系的名称来定义「has-many-through」关系。例如，如果 `Project` 模型具有 `environments` 关系，而 `Environment` 模型具有 `deployments` 关系，则可以定义连接 project 和 deployments 的「has-many-through」关系，如下所示：

```php
    // 基于字符串的语法...
    return $this->through('environments')->has('deployments');

    // 动态语法...
    return $this->throughEnvironments()->hasDeployments();
```

虽然 `Deployment` 模型的表格不包含 `project_id` 列，但 `hasManyThrough` 关系通过 `$project->deployments` 提供了访问项目的部署方式。为了检索这些模型，Eloquent 在中间的 `Environment` 模型表中检查 `project_id` 列。在找到相关的 environment ID 后，它们被用来查询 `Deployment` 模型。

#### 键名约定

在执行关系查询时，通常会使用典型的 Eloquent 外键约定。如果你想要自定义关系键名，可以将它们作为 `hasManyThrough` 方法的第三个和第四个参数传递。第三个参数是中间模型上的外键名称。第四个参数是最终模型上的外键名称。第五个参数是本地键，而第六个参数是中间模型的本地键：

```php
 class Project extends Model
 {
    public function deployments(): HasManyThrough
    {
        return $this->hasManyThrough(
            Deployment::class,
            Environment::class,
            'project_id', // 在 environments 表上的外键...
            'environment_id', // 在 deployments 表上的外键...
            'id', // 在 projects 表上的本地键...
            'id' // 在 environments 表上的本地键...
        );
    }
}
```

或者，所有模型上都定义好了关系，你可以通过调用 `through` 方法并提供这些关系的名称来流畅的定义「has-many-through」关系。这种方法的优点是可以复用现有关系中已定义的键的约束：

```php
    // 基于字符串的语法...
    return $this->through('environments')->has('deployments');

    // 动态语法...
    return $this->throughEnvironments()->hasDeployments();
```

## 多对多关联

多对多关联比 `hasOne` 和 `hasMany` 关联略微复杂。一个多对多关系的例子，在应用中一个用户可以拥有多个角色，同时这些角色也可以分配给其他用户。例如，一个用户可是「作者」和「编辑」；但是，这些角色也可以分配给其他用户。所以，一个用户可以拥有多个角色，一个角色可以分配给多个用户。

### 表结构

要定义这种关联，需要三个数据库表: `users`, `roles` 和 `role_user`。  
`role_user` 表的命名是由关联的两个模型按照字母顺序来的，并且包含了 `user_id` 和 `role_id` 字段。该表用作链接 `users` 和 `roles` 的中间表。

特别提醒，由于角色可以属于多个用户，因此我们不能简单地在 `roles` 表上放置 `user_id` 列。如果这样，这意味着角色只能属于一个用户。为了支持将角色分配给多个用户，需要使用 `role_user` 表。我们可以这样总结关系的表结构：

```yaml
    users
        id - integer
        name - string

    roles
        id - integer
        name - string

    role_user
        user_id - integer
        role_id - integer
```

### 模型结构

多对多关联是通过调用 `belongsToMany` 方法结果的方法来定义的。`belongsToMany` 方法由 `Illuminate\Database\Eloquent\Model` 基类提供，所有应用程序的 Eloquent 模型都使用该基类。例如，让我们在 `User` 模型上定义一个 `roles` 方法。传递给此方法的第一个参数是相关模型类的名称：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class User extends Model
{
   /**
    * 属于用户的角色。
    */
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }
}
```

定义关系后，可以使用 `roles` 动态关系属性访问用户的角色：

```php
    use App\Models\User;

    $user = User::find(1);

    foreach ($user->roles as $role) {
        // ...
    }
```

由于所有的关系也可以作为查询构建器，你可以通过调用 `roles()` 方法链式的在查询上添加关系条件：

```php
    $roles = User::find(1)->roles()->orderBy('name')->get();
```

为了确定关系的中间表的表名，Eloquent 会按字母顺序连接两个相关的模型名。你也可以随意覆盖此约定。通过将第二个参数传递给 `belongsToMany` 方法来做到这一点：

```php
    return $this->belongsToMany(Role::class, 'role_user');
```

除了自定义连接表的表名，你还可以通过传递额外的参数到 `belongsToMany` 方法来定义该表中字段的键名。第三个参数是定义此关联的模型在连接表里的外键名，第四个参数是另一个模型在连接表里的外键名：

```php
    return $this->belongsToMany(Role::class, 'role_user', 'user_id', 'role_id');
```

#### 定义反向关系

要定义多对多的「反向」关联，只需要在关联模型中定义一个方法并返回调用 `belongsToMany` 方法的结果。为了完成我们的用户 / 角色例子，让我们在 `Role` 模型中定义 `users` 方法：

```php
    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Relations\BelongsToMany;

    class Role extends Model
    {
       /**
        * 属于角色的用户。
        */
        public function users(): BelongsToMany
        {
            return $this->belongsToMany(User::class);
        }
    }
```

如你所见，除了引用 `App\Models\User` 模型之外，该关系的定义与其对应的 `User` 模型完全相同。 由于我们复用了 `belongsToMany` 方法，所以在定义多对多关系的「反向」关系时，所有常用的表和键自定义选项都可用。

### 获取中间表字段

如上所述，处理多对多关系需要一个中间表。 Eloquent 提供了一些非常有用的方式来与这张表进行交互。 假设我们的 `User` 模型关联了多个 `Role` 模型。在获得这些关联对象后，我们可以使用模型的 `pivot` 属性访问中间表的属性：

```php
    use App\Models\User;

    $user = User::find(1);

    foreach ($user->roles as $role) {
        echo $role->pivot->created_at;
    }
```

需要注意的是，我们获取的每个 `Role` 模型对象，都会被自动赋予 `pivot` 属性。这些属性包含一个代表中间表的模型。

默认情况下，`pivot` 对象只包含两个关联模型的主键，如果你的中间表里还有其他额外字段，你必须在定义关联时明确指出：

```php
    return $this->belongsToMany(Role::class)->withPivot('active', 'created_by');
```

如果你想让中间表自动维护 `created_at` 和 `updated_at` 时间戳，那么在定义关联时附加上 `withTimestamps` 方法即可：

```php
    return $this->belongsToMany(Role::class)->withTimestamps();
```

> **注意**  
> 使用 Eloquent 自动维护时间戳的中间表需要同时具有 `created_at` 和 `updated_at` 时间戳字段。

#### 自定义 pivot 属性名称

如前所述，可以通过 `pivot` 属性在模型上访问中间表中的属性。 但是，你可以随意自定义此属性的名称，以更好地反映其在应用程序中的用途。

例如，如果你的应用程序包含可能订阅播客的用户，则用户和播客之间可能存在多对多关系。 如果是这种情况，你可能希望将中间表属性重命名为 `subscription` 而不是 `pivot`。 这可以在定义关系时使用 `as` 方法来完成：

```php
    return $this->belongsToMany(Podcast::class)
                ->as('subscription')
                ->withTimestamps();
```

一旦定义中间表属性被指定，你可以使用自定义名称访问中间表数据：

```php
    $users = User::with('podcasts')->get();

    foreach ($users->flatMap->podcasts as $podcast) {
        echo $podcast->subscription->created_at;
    }
```

### 通过中间表过滤查询

你还可以在定义关系时使用 `wherePivot`、`wherePivotIn`、`wherePivotNotIn`、`wherePivotBetween`、`wherePivotNotBetween`、`wherePivotNull` 和 `wherePivotNotNull` 方法过滤 `belongsToMany` 关系查询返回的结果：

```php
    return $this->belongsToMany(Role::class)
            ->wherePivot('approved', 1);

    return $this->belongsToMany(Role::class)
            ->wherePivotIn('priority', [1, 2]);

    return $this->belongsToMany(Role::class)
            ->wherePivotNotIn('priority', [1, 2]);

    return $this->belongsToMany(Podcast::class)
            ->as('subscriptions')
            ->wherePivotBetween('created_at', ['2020-01-01 00:00:00', '2020-12-31 00:00:00']);

    return $this->belongsToMany(Podcast::class)
            ->as('subscriptions')
            ->wherePivotNotBetween('created_at', ['2020-01-01 00:00:00', '2020-12-31 00:00:00']);

    return $this->belongsToMany(Podcast::class)
            ->as('subscriptions')
            ->wherePivotNull('expired_at');

    return $this->belongsToMany(Podcast::class)
            ->as('subscriptions')
            ->wherePivotNotNull('expired_at');
```

### 按中间表的列字段对查询结果进行排序

你可以使用 `orderByPivot` 方法对 `belongsToMany` 关系查询返回的结果进行排序。在以下示例中，我们将检索用户的最新徽章：

```php
return $this->belongsToMany(Badge::class)
        ->where('rank', 'gold')
        ->orderByPivot('created_at', 'desc');
```

### 自定义中间表模型

如果你希望为多对多关系的中间表定义一个自定义模型，可以在定义关系时使用 `using` 方法。这样可以在模型中定义额外的功能，比如自定义方法和类型转换。

自定义多对多中间表模型应继承 `Illuminate\Database\Eloquent\Relations\Pivot` 类，而自定义多对多（多态）中间表模型应继承 `Illuminate\Database\Eloquent\Relations\MorphPivot` 类。例如，我们在定义 `Role` 模型的关联模型时，使用自定义中间表模型 `RoleUser`：

```php
  <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Relations\BelongsToMany;

    class Role extends Model
    {
       /**
        * 属于该角色的用户。
        */
        public function users(): BelongsToMany
        {
            return $this->belongsToMany(User::class)->using(RoleUser::class);
        }
    }
```

定义 `RoleUser` 模型时，应该继承 `Illuminate\Database\Eloquent\Relations\Pivot` 类：

```php
  <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Relations\Pivot;

    class RoleUser extends Pivot
    {
        // ...
    }
```

> **注意**  
> Pivot 模型不能使用 `SoftDeletes` trait。如果你需要中间表模型进行软删除，请考虑将你的中间表模型转换为一个实际的 Eloquent 模型。

#### 自定义中间模型和自增 ID

如果你用一个自定义的中间模型定义了多对多的关系，而且这个中间模型拥有一个自增的主键，你应当确保这个自定义中间模型类中定义了一个 `incrementing` 属性且其值为 `true`.

```php

    /**
     * 标识 ID 是否自增
     *
     * @var bool
     */
    public $incrementing = true;
```

## 多态关系

多态关联允许子模型使用单个关联属于多种类型的模型。例如，假设你正在构建一个应用程序，允许用户共享博客文章和视频。在这样的应用程序中， `Comment` 模型可能同时属于 `Post` 和 `Video` 模型。

### 一对一 (多态)

#### 表结构

一对一多态关联类似于典型的一对一关系，但是子模型可以使用单个关联属于多个类型的模型。例如，一个博客 `Post` 和一个 `User` 可以共享到一个 `Image` 模型的多态关联。使用一对一多态关联允许你拥有一个唯一图像的单个表，这些图像可以与帖子和用户关联。首先，让我们查看表结构:

```yaml
 posts
        id - integer
        name - string

    users
        id - integer
        name - string

    images
        id - integer
        url - string
        imageable_id - integer
        imageable_type - string
```

请注意 `images` 表上的 `imageable_id` 和 `imageable_type` 两列。 `imageable_id` 列将包含帖子或用户的 ID 值， 而 `imageable_type` 列将包含父模型的类名。`imageable_type` 列用于 Eloquent 在访问 `imageable` 关联时确定要返回哪种类型的父模型。在本例中，该列将包含 `App\Models\Post` 或 `App\Models\User`。

#### 模型结构

接下来，让我们看一下构建这种关系所需的模型定义：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class Image extends Model
{
    /**
     * 获取父级可变模型（用户或帖子）。
  */
    public function imageable(): MorphTo
    {
        return $this->morphTo();
    }
}

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphOne;

class Post extends Model
{
    /**
     * 获取帖子的图片。
  */
    public function image(): MorphOne
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphOne;

class User extends Model
{
    /**
     * 获取用户的图片。
  */
    public function image(): MorphOne
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}
```

#### 检索关系

一旦你定义了数据库表和模型，你可以通过你的模型访问这些关系。例如，要检索帖子的图片，我们可以访问 `image` 动态关系属性：

```php
use App\Models\Post;

$post = Post::find(1);

$image = $post->image;
```

你可以通过访问执行对 `morphTo` 的调用的方法的名称来检索多态模型的父级。在这种情况下，这是 `Image` 模型上的 `imageable` 方法。因此，我们将访问该方法作为动态关系属性：

```php
use App\Models\Image;

$image = Image::find(1);

$imageable = $image->imageable;
```

`Image` 模型上的 `imageable` 关系将返回一个 `Post` 或 `User` 实例，取决于哪种类型的模型拥有该图片。

#### 键约定

如果需要，你可以指定多态子模型使用的「id」和「type」列的名称。如果这样做，请确保始终将关系的名称作为 `morphTo` 方法的第一个参数传递。通常，这个值应该与方法名称匹配，因此你可以使用 PHP 的 `__FUNCTION__` 常量：

```php
/**
 * 获取图片所属的模型。
  */
public function imageable(): MorphTo
{
    return $this->morphTo(__FUNCTION__, 'imageable_type', 'imageable_id');
}
```

### 一对多（多态）

#### 表结构

一对多多态关系类似于典型的一对多关系；然而，子模型可以使用单个关联属于多种类型的模型。例如，假设你的应用程序的用户可以在帖子和视频上「评论」。使用多态关系，你可以使用单个 `comments` 表来包含帖子和视频的评论。首先，让我们看一下构建这种关系所需的表结构：

```yaml
posts
    id - integer
    title - string
    body - text

videos
    id - integer
    title - string
    url - string

comments
    id - integer
    body - text
    commentable_id - integer
    commentable_type - string
```

#### 模型结构

接下来，让我们看一下构建这种关系所需的模型定义：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class Comment extends Model
{
    /**
     * 获取父级可变评论模型（帖子或视频）。
  */
    public function commentable(): MorphTo
    {
        return $this->morphTo();
    }
}

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphMany;

class Post extends Model
{
    /**
     * 获取所有帖子的评论。
  */
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphMany;

class Video extends Model
{
    /**
     * 获取所有视频的评论。
  */
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

#### 检索关系

一旦你定义了数据库表和模型，你可以通过模型的动态关系属性访问这些关系。例如，要访问帖子的所有评论，我们可以使用 `comments` 动态属性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->comments as $comment) {
    // ...
}
```

你也可以通过访问执行对 `morphTo` 的调用的方法的名称来检索多态子模型的父模型。在这种情况下，这是 `Comment` 模型上的 `commentable` 方法。因此，我们将访问该方法作为动态关系属性，以便访问评论的父模型：

```php
use App\Models\Comment;

$comment = Comment::find(1);

$commentable = $comment->commentable;
```

`Comment` 模型上的 `commentable` 关系将返回一个 `Post` 或 `Video` 实例，取决于评论的父模型是哪种类型。

### 多个中的一个（多态）

有时，一个模型可能有许多相关模型，但你想要轻松地检索关系中的「最新」或「最旧」相关模型。例如，一个 `User` 模型可能与许多 `Image` 模型相关联，但你想要定义一种方便的方式与用户最近上传的图片进行交互。你可以使用 `morphOne` 关系类型结合 `ofMany` 方法来实现这一点：

```php
/**
 * 获取用户最新的图片。
  */
public function latestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->latestOfMany();
}
```

同样，你可以定义一个方法来检索关系中的「最旧」或第一个相关模型：

```php
/**
 * 获取用户最早的图片。
  */
public function oldestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->oldestOfMany();
}
```

默认情况下，`latestOfMany` 和 `oldestOfMany` 方法将基于模型的主键检索最新或最旧的相关模型，这些主键必须是可排序的。然而，有时你可能希望使用不同的排序标准从更大的关系中检索单个模型。

例如，使用 `ofMany` 方法，你可以检索用户最「喜欢」的图片。`ofMany` 方法接受可排序列作为第一个参数，并在查询相关模型时应用哪种聚合函数（`min` 或 `max`）：

```php
/**
 * 获取用户最受欢迎的图片。
  */
public function bestImage(): MorphOne
{
    return $this->morphOne(Image::class, 'imageable')->ofMany('likes', 'max');
}
```

> **注意**  
> 可以构建更复杂的「多个中的一个」关系。有关更多信息，请参阅[一个中的多个文档](#advanced-has-one-of-many-relationships)。

### 多对多（多态）

#### 表结构

多对多多态关系比「morph one」和「morph many」关系稍微复杂一些。例如，`Post` 模型和 `Video` 模型可以共享与 `Tag` 模型的多态关系。在这种情况下使用多对多多态关系，你的应用程序可以拥有一个包含可与帖子或视频关联的唯一标签的单个表。首先，让我们看一下构建这种关系所需的表结构：

```yaml
posts
    id - integer
    name - string

videos
    id - integer
    name - string

tags
    id - integer
    name - string

taggables
    tag_id - integer
    taggable_id - integer
    taggable_type - string
```

> **注意**  
> 在深入研究多对多多态关系之前，你可能会从阅读关于典型[多对多关系](#many-to-many)的文档中受益。

#### 模型结构

接下来，我们准备在模型上定义关系。`Post` 和 `Video` 模型都将包含一个 `tags` 方法，该方法调用基础 Eloquent 模型类提供的 `morphToMany` 方法。

`morphToMany` 方法接受相关模型的名称以及「关系名称」。根据我们分配给中间表名称及其包含的键的名称，我们将将该关系称为「taggable」：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphToMany;

class Post extends Model
{
    /**
     * 获取该帖子的所有标签。
  */
    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}
```

#### 定义关系的反向关系

接下来，在 `Tag` 模型上，你应该为每个可能的父模型定义一个方法。因此，在这个例子中，我们将定义一个 `posts` 方法和一个 `videos` 方法。这两个方法都应该返回 `morphedByMany` 方法的结果。

`morphedByMany` 方法接受相关模型的名称以及「关系名称」。根据我们分配给中间表名称及其包含的键的名称，我们将将该关系称为「taggable」：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphToMany;

class Tag extends Model
{
    /**
     * 获取分配了该标签的所有帖子。
  */
    public function posts(): MorphToMany
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }

    /**
     * 获取分配了该标签的所有视频。
  */
    public function videos(): MorphToMany
    {
        return $this->morphedByMany(Video::class, 'taggable');
    }
}
```

#### 检索关系

一旦定义了数据库表和模型，你可以通过你的模型访问关系。例如，要访问帖子的所有标签，你可以使用 `tags` 动态关系属性：

```php
use App\Models\Post;

$post = Post::find(1);

foreach ($post->tags as $tag) {
    // ...
}
```

你可以通过访问执行对 `morphedByMany` 调用的方法的名称，从多态子模型中检索多态关系的父模型。在这种情况下，就是 `Tag` 模型上的 `posts` 或 `videos` 方法：

```php
use App\Models\Tag;

$tag = Tag::find(1);

foreach ($tag->posts as $post) {
    // ...
}

foreach ($tag->videos as $video) {
    // ...
}
```

### 自定义多态类型

默认情况下，Laravel 将使用完全限定的类名来存储相关模型的「类型」。例如，考虑上面的一对多关系示例，其中 `Comment` 模型可以属于 `Post` 或 `Video` 模型，默认的 `commentable_type` 分别为 `App\Models\Post` 或 `App\Models\Video`。然而，你可能希望将这些值与应用程序的内部结构解耦。

例如，我们可以使用简单的字符串（如 `post` 和 `video`）而不是使用模型名称作为「类型」。通过这样做，即使模型被重命名，我们数据库中的多态「类型」列值也将保持有效：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);
```

你可以在 `App\Providers\AppServiceProvider` 类的 `boot` 方法中调用 `enforceMorphMap` 方法，或者如果你愿意，也可以创建一个单独的服务提供程序。

你可以使用模型的 `getMorphClass` 方法在运行时确定给定模型的多态别名。反之，你可以使用 `Relation::getMorphedModel` 方法确定与多态别名关联的完全限定类名：

```php
use Illuminate\Database\Eloquent\Relations\Relation;

$alias = $post->getMorphClass();

$class = Relation::getMorphedModel($alias);
```

> **警告**  
> 在向现有应用程序添加「morph map」时，数据库中仍包含完全限定类的每个可多态化的 `*_type` 列值都需要转换为其「映射」名称。

### 动态关系

你可以使用 `resolveRelationUsing` 方法在运行时定义 Eloquent 模型之间的关系。虽然在正常应用程序开发中通常不建议使用，但在开发 Laravel 包时偶尔可能会很有用。

`resolveRelationUsing` 方法接受所需的关系名称作为其第一个参数。传递给该方法的第二个参数应该是一个接受模型实例并返回有效的 Eloquent 关系定义的闭包。通常，你应该在[服务提供程序](https://learnku.com/docs/laravel/11.x/providers)的 `boot` 方法中配置动态关系：

```php
use App\Models\Order;
use App\Models\Customer;

Order::resolveRelationUsing('customer', function (Order $orderModel) {
    return $orderModel->belongsTo(Customer::class, 'customer_id');
});
```

> **警告**  
> 在定义动态关系时，始终为 Eloquent 关系方法提供明确的键名参数。

## 查询关系

由于所有 Eloquent 关系都是通过方法定义的，你可以调用这些方法以获取关系的实例，而无需实际执行查询来加载相关模型。此外，所有类型的 Eloquent 关系也充当[查询构建器](https://learnku.com/docs/laravel/11.x/queries)，允许你在最终执行 SQL 查询之前继续在关系查询上链约束。

例如，想象一个博客应用程序，其中 `User` 模型有许多关联的 `Post` 模型：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class User extends Model
{
    /**
     * 获取用户的所有帖子。
  */
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }
}
```

你可以查询 `posts` 关系并向关系添加额外约束，如下所示：

```php
use App\Models\User;

$user = User::find(1);

$user->posts()->where('active', 1)->get();
```

你可以在关系上使用 Laravel [查询构建器的](https://learnku.com/docs/laravel/11.x/queries)任何方法，因此请务必查阅查询构建器文档，了解所有可用的方法。

### 在关系之后链式使用 `orWhere` 子句

如上面的示例所示，你可以在查询关系时添加额外约束。但是，在将 `orWhere` 子句链接到关系时要小心，因为 `orWhere` 子句将在与关系约束相同级别逻辑分组：

```php
$user->posts()
        ->where('active', 1)
        ->orWhere('votes', '>=', 100)
        ->get();
```

上面的示例将生成以下 SQL。如你所见，`or` 子句指示查询返回具有超过 100 票的 *任何* 帖子。查询不再受限于特定用户：

```sql
select *
from posts
where user_id = ? and active = 1 or votes >= 100
```

在大多数情况下，你应该使用[逻辑分组](https://learnku.com/docs/laravel/11.x/queries#logical-grouping)将条件检查分组在括号之间：

```php
use Illuminate\Database\Eloquent\Builder;

$user->posts()
        ->where(function (Builder $query) {
            return $query->where('active', 1)
                    ->orWhere('votes', '>=', 100);
        })->get();
```

上面的示例将产生以下 SQL。请注意，逻辑分组正确地将约束分组，查询仍然受限于特定用户：

```sql
select *
from posts
where user_id = ? and (active = 1 or votes >= 100)
```

### 关系方法 vs. 动态属性

如果你不需要向 Eloquent 关系查询添加额外约束，可以像访问属性一样访问关系。例如，继续使用我们的 `User` 和 `Post` 示例模型，你可以这样访问用户的所有帖子：

```php
use App\Models\User;

$user = User::find(1);

foreach ($user->posts as $post) {
    // ...
}
```

动态关系属性执行「懒加载」，意味着只有在你实际访问它们时才会加载它们的关系数据。因此，开发人员通常使用[预加载](#eager-loading)来预加载他们知道在加载模型后将被访问的关系。预加载大大减少了必须执行的 SQL 查询，以加载模型的关系。

### 查询关系存在性

在检索模型记录时，你可能希望根据关系的存在性限制结果。例如，想象一下，你想检索所有至少有一条评论的博客文章。为此，你可以将关系的名称传递给 `has` 和 `orHas` 方法：

```php
use App\Models\Post;

// 检索至少有一条评论的所有帖子...
$posts = Post::has('comments')->get();
```

你还可以指定运算符和计数值以进一步自定义查询：

```php
// 检索至少有三条或更多评论的所有帖子...
$posts = Post::has('comments', '>=', 3)->get();
```

可以使用「点」符号构建嵌套的 `has` 语句。例如，你可以检索至少有一条具有至少一张图片的评论的所有帖子：

```php
// 检索至少有一条带有图片的评论的帖子...
$posts = Post::has('comments.images')->get();
```

如果你需要更多功能，可以使用 `whereHas` 和 `orWhereHas` 方法在 `has` 查询上定义额外的查询约束，例如检查评论的内容：

```php
use Illuminate\Database\Eloquent\Builder;

// 检索至少有一条包含类似 code% 的单词的评论的帖子...
$posts = Post::whereHas('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();

// 检索至少有十条包含类似 code% 的单词的评论的帖子...
$posts = Post::whereHas('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
}, '>=', 10)->get();
```

> **警告**  
> 目前 Eloquent 不支持跨数据库查询关系存在性。这些关系必须存在于同一个数据库中。

#### 内联关系存在性查询

如果你想要通过一个简单的 where 条件查询关系的存在性，你可能会发现使用 `whereRelation`、`orWhereRelation`、`whereMorphRelation` 和 `orWhereMorphRelation` 方法更方便。例如，我们可以查询所有具有未批准评论的帖子：

```php
use App\Models\Post;

$posts = Post::whereRelation('comments', 'is_approved', false)->get();
```

当然，类似于查询构建器的 `where` 方法调用，你也可以指定一个操作符：

```php
$posts = Post::whereRelation(
    'comments', 'created_at', '>=', now()->subHour()
)->get();
```

### 查询关系缺失

在检索模型记录时，你可能希望根据关系的缺失限制结果。例如，假设你想要检索所有**没有**任何评论的博客帖子。为此，你可以将关系的名称传递给 `doesntHave` 和 `orDoesntHave` 方法：

```php
use App\Models\Post;

$posts = Post::doesntHave('comments')->get();
```

如果你需要更多的功能，你可以使用 `whereDoesntHave` 和 `orWhereDoesntHave` 方法为你的 `doesntHave` 查询添加额外的查询约束，比如检查评论的内容：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments', function (Builder $query) {
    $query->where('content', 'like', 'code%');
})->get();
```

你可以使用「点」符号对嵌套关系执行查询。例如，以下查询将检索所有没有评论的帖子；然而，带有来自未被禁止的作者的评论的帖子将包含在结果中：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::whereDoesntHave('comments.author', function (Builder $query) {
    $query->where('banned', 0);
})->get();
```

### 查询 Morph To 关系

要查询「morph to」关系的存在性，你可以使用 `whereHasMorph` 和 `whereDoesntHaveMorph` 方法。这些方法将关系的名称作为第一个参数。接下来，方法接受你希望在查询中包含的相关模型的名称。最后，你可以提供一个自定义关系查询的闭包：

```php
use App\Models\Comment;
use App\Models\Post;
use App\Models\Video;
use Illuminate\Database\Eloquent\Builder;

// 检索与标题类似于 code% 的帖子或视频关联的评论...
$comments = Comment::whereHasMorph(
    'commentable',
    [Post::class, Video::class],
    function (Builder $query) {
        $query->where('title', 'like', 'code%');
    }
)->get();

// 检索与标题不类似于 code% 的帖子关联的评论...
$comments = Comment::whereDoesntHaveMorph(
    'commentable',
    Post::class,
    function (Builder $query) {
        $query->where('title', 'like', 'code%');
    }
)->get();
```

偶尔你可能需要根据多态关联模型的「类型」添加查询约束。传递给 `whereHasMorph` 方法的闭包可以接收 `$type` 值作为第二个参数。这个参数允许你检查正在构建的查询的「类型」：

```php
use Illuminate\Database\Eloquent\Builder;

$comments = Comment::whereHasMorph(
    'commentable',
    [Post::class, Video::class],
    function (Builder $query, string $type) {
        $column = $type === Post::class ? 'content' : 'title';

        $query->where($column, 'like', 'code%');
    }
)->get();
```

#### 查询所有相关的 Morph To 模型

你可以将 `*` 作为通配符值，而不是提供可能的多态模型数组。这将指示 Laravel 从数据库中检索所有可能的多态类型。为了执行这个操作，Laravel 将执行额外的查询：

```php
use Illuminate\Database\Eloquent\Builder;

$comments = Comment::whereHasMorph('commentable', '*', function (Builder $query) {
    $query->where('title', 'like', 'foo%');
})->get();
```

## 聚合相关模型

### 计算相关模型数量

有时你可能想要计算给定关系的相关模型数量，而不实际加载这些模型。为了实现这一点，你可以使用 `withCount` 方法。`withCount` 方法将在结果模型上放置一个 `{relation}_count` 属性：

```php
use App\Models\Post;

$posts = Post::withCount('comments')->get();

foreach ($posts as $post) {
    echo $post->comments_count;
}
```

通过向 `withCount` 方法传递一个数组，你可以为多个关系添加「计数」，并为查询添加额外约束：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::withCount(['votes', 'comments' => function (Builder $query) {
    $query->where('content', 'like', 'code%');
}])->get();

echo $posts[0]->votes_count;
echo $posts[0]->comments_count;
```

你还可以为关系计数结果设置别名，允许在同一关系上进行多次计数：

```php
use Illuminate\Database\Eloquent\Builder;

$posts = Post::withCount([
    'comments',
    'comments as pending_comments_count' => function (Builder $query) {
        $query->where('approved', false);
    },
])->get();

echo $posts[0]->comments_count;
echo $posts[0]->pending_comments_count;
```

#### 延迟计数加载

使用 `loadCount` 方法，你可以在已经检索到父模型之后加载关系计数：

```php
$book = Book::first();

$book->loadCount('genres');
```

如果你需要在计数查询上设置额外的查询约束，你可以传递一个以你希望计数的关系为键的数组。数组值应该是接收查询构建器实例的闭包：

```php
$book->loadCount(['reviews' => function (Builder $query) {
    $query->where('rating', 5);
}])
```

#### 关联计数和自定义选择语句

如果你将 `withCount` 与 `select` 语句结合使用，请确保在 `select` 方法之后调用 `withCount`：

```php
$posts = Post::select(['title', 'body'])
                ->withCount('comments')
                ->get();
```

### 其他聚合函数

除了 `withCount` 方法外，Eloquent 还提供了 `withMin`、`withMax`、`withAvg`、`withSum` 和 `withExists` 方法。这些方法将在你的结果模型上放置一个 `{relation}_{function}_{column}` 属性：

```php
use App\Models\Post;

$posts = Post::withSum('comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->comments_sum_votes;
}
```

如果你希望使用另一个名称访问聚合函数的结果，你可以指定自己的别名：

```php
$posts = Post::withSum('comments as total_comments', 'votes')->get();

foreach ($posts as $post) {
    echo $post->total_comments;
}
```

与 `loadCount` 方法类似，这些方法的延迟版本也是可用的。这些额外的聚合操作可以在已经检索到的 Eloquent 模型上执行：

```php
$post = Post::first();

$post->loadSum('comments', 'votes');
```

如果你将这些聚合方法与 `select` 语句结合使用，请确保在 `select` 方法之后调用聚合方法：

```php
$posts = Post::select(['title', 'body'])
                ->withExists('comments')
                ->get();
```

### 在 Morph To 关系上计数相关模型

如果你想要预加载「morph to」关系，并且为该关系可能返回的各种实体的相关模型计数，你可以在 `with` 方法中结合 `morphTo` 关系的 `morphWithCount` 方法。

在这个例子中，假设 `Photo` 和 `Post` 模型可以创建 `ActivityFeed` 模型。我们假设 `ActivityFeed` 模型定义了一个名为 `parentable` 的「morph to」关系，允许我们为给定的 `ActivityFeed` 实例检索父 `Photo` 或 `Post` 模型。此外，让我们假设 `Photo` 模型「拥有多个」 `Tag` 模型，而 `Post` 模型「拥有多个」 `Comment` 模型。

现在，让我们想象我们想要检索 `ActivityFeed` 实例，并预加载每个 `ActivityFeed` 实例的 `parentable` 父模型。此外，我们希望检索与每个父照片关联的标签数量，以及与每个父帖子关联的评论数量：

```php
use Illuminate\Database\Eloquent\Relations\MorphTo;

$activities = ActivityFeed::with([
    'parentable' => function (MorphTo $morphTo) {
        $morphTo->morphWithCount([
            Photo::class => ['tags'],
            Post::class => ['comments'],
        ]);
    }])->get();
```

#### 延迟计数加载

假设我们已经检索了一组 `ActivityFeed` 模型，现在我们想要加载与活动源关联的各种 `parentable` 模型的嵌套关系计数。你可以使用 `loadMorphCount` 方法来实现这一点：

```php
    $activities = ActivityFeed::with('parentable')->get();

    $activities->loadMorphCount('parentable', [
        Photo::class => ['tags'],
        Post::class => ['comments'],
    ]);
```

## 预加载

当将 Eloquent 关系视为属性访问时，相关模型是「懒加载」的。这意味着直到你首次访问属性时，关系数据才会被实际加载。然而，Eloquent 可以在查询父模型时「预加载」关系。预加载可以减轻「N + 1」查询问题。为了说明 N + 1 查询问题，考虑一个 `Book` 模型，它「belongs to」 一个 `Author` 模型：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Book extends Model
{
    /**
     * 获取写作这本书的作者。
  */
    public function author(): BelongsTo
    {
        return $this->belongsTo(Author::class);
    }
}
```

现在，让我们检索所有书籍及其作者：

```php
use App\Models\Book;

$books = Book::all();

foreach ($books as $book) {
    echo $book->author->name;
}
```

这个循环将执行一次查询以检索数据库表中的所有书籍，然后为每本书执行另一个查询以检索书籍的作者。因此，如果我们有 25 本书，上面的代码将运行 26 次查询：一次是为原始书籍，另外 25 次是为每本书的作者。

幸运的是，我们可以使用预加载来将此操作减少到只需两次查询。在构建查询时，你可以使用 `with` 方法指定应该预加载哪些关系：

```php
$books = Book::with('author')->get();

foreach ($books as $book) {
    echo $book->author->name;
}
```

对于这个操作，只会执行两次查询 - 一次查询以检索所有书籍，另一次查询以检索所有书籍的作者：

```sql
select * from books

select * from authors where id in (1, 2, 3, 4, 5, ...)
```

#### 预加载多个关联

有时你可能需要预加载几个不同的关联。要做到这一点，只需将关系数组传递给 `with` 方法：

```php
$books = Book::with(['author', 'publisher'])->get();
```

#### 嵌套预加载

要预加载关联的关联，你可以使用「点」语法。例如，让我们一次性加载所有书籍的作者和作者的个人联系方式：

```php
$books = Book::with('author.contacts')->get();
```

或者，你可以通过向 `with` 方法提供嵌套数组来指定嵌套预加载的关系，当预加载多个嵌套关系时，这种方法可能更方便：

```php
$books = Book::with([
    'author' => [
        'contacts',
        'publisher',
    ],
])->get();
```

#### 嵌套预加载 `morphTo` 关联

如果你想要预加载一个 `morphTo` 关联，以及可能由该关联返回的各个实体上的嵌套关联，你可以在 `with` 方法中结合使用 `morphTo` 关联的 `morphWith` 方法。为了帮助说明这种方法，让我们考虑以下模型：

```php
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class ActivityFeed extends Model
{
    /**
     * 获取活动源记录的父级。
  */
    public function parentable(): MorphTo
    {
        return $this->morphTo();
    }
}
```

在这个例子中，假设 `Event`、`Photo` 和 `Post` 模型可以创建 `ActivityFeed` 模型。此外，假设 `Event` 模型属于 `Calendar` 模型，`Photo` 模型与 `Tag` 模型相关联，`Post` 模型属于 `Author` 模型。

使用这些模型定义和关系，我们可以检索 `ActivityFeed` 模型实例，并预加载所有 `parentable` 模型及其各自的嵌套关联：

```php
use Illuminate\Database\Eloquent\Relations\MorphTo;

$activities = ActivityFeed::query()
    ->with(['parentable' => function (MorphTo $morphTo) {
        $morphTo->morphWith([
            Event::class => ['calendar'],
            Photo::class => ['tags'],
            Post::class => ['author'],
        ]);
    }])->get();
```

#### 预加载特定列

你并非总是需要检索你所获取的关联的每一列。因此，Eloquent 允许你指定你想要检索的关联的哪些列：

```php
$books = Book::with('author:id,name,book_id')->get();
```

> **警告**  
> 当使用此功能时，你应该始终在希望检索的列列表中包括 `id` 列和任何相关的外键列。

#### 默认预加载

有时候在检索模型时，你可能希望总是加载一些关联数据。为了实现这一点，你可以在模型上定义一个 `$with` 属性：

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Book extends Model
{
    /**
     * 应该始终加载的关联。
  *
     * @var array
     */
    protected $with = ['author'];

    /**
     * 获取撰写本书的作者。
  */
    public function author(): BelongsTo
    {
        return $this->belongsTo(Author::class);
    }

    /**
     * 获取本书的流派。
  */
    public function genre(): BelongsTo
    {
        return $this->belongsTo(Genre::class);
    }
}
```

如果你想要在单个查询中从 `$with` 属性中移除一个项目，你可以使用 `without` 方法：

```php
$books = Book::without('author')->get();
```

如果你想要覆盖 `$with` 属性中的所有项目以进行单个查询，你可以使用 `withOnly` 方法：

```php
$books = Book::withOnly('genre')->get();
```

### 约束预加载

有时候你可能希望预加载一个关联数据，但同时为预加载查询指定额外的查询条件。你可以通过将一个关联数据数组传递给 `with` 方法来实现这一点，其中数组键是关联名称，数组值是一个闭包，该闭包添加额外的约束条件到预加载查询中：

```php
use App\Models\User;
use Illuminate\Contracts\Database\Eloquent\Builder;

$users = User::with(['posts' => function (Builder $query) {
    $query->where('title', 'like', '%code%');
}])->get();
```

在这个例子中，Eloquent 只会预加载那些标题列包含单词 `code` 的帖子。你可以调用其他 [查询构建器](https://learnku.com/docs/laravel/11.x/queries) 方法来进一步定制预加载操作：

```php
$users = User::with(['posts' => function (Builder $query) {
    $query->orderBy('created_at', 'desc');
}])->get();
```

#### 对 `morphTo` 关联的预加载进行约束

如果你正在预加载一个 `morphTo` 关联，Eloquent 将运行多个查询以获取每种相关模型。你可以使用 `MorphTo` 关联的 `constrain` 方法为每个查询添加额外约束：

```php
    use Illuminate\Database\Eloquent\Relations\MorphTo;

    $comments = Comment::with(['commentable' => function (MorphTo $morphTo) {
        $morphTo->constrain([
            Post::class => function ($query) {
                $query->whereNull('hidden_at');
            },
            Video::class => function ($query) {
                $query->where('type', 'educational');
            },
        ]);
    }])->get();
```

在这个例子中，Eloquent 只会预加载未隐藏的帖子和具有「educational」类型值的视频。

#### 根据关联存在性约束预加载

有时你可能需要同时检查关联的存在性并根据相同条件加载关联。例如，你可能希望仅检索符合给定查询条件的子 `Post` 模型的 `User` 模型，同时预加载匹配的帖子。你可以使用 `withWhereHas` 方法来实现：

```php
    use App\Models\User;

    $users = User::withWhereHas('posts', function ($query) {
        $query->where('featured', true);
    })->get();
```

### 惰性预加载

有时你可能需要在已检索父模型之后预加载关联。例如，如果你需要动态决定是否加载相关模型，则这可能会很有用：

```php
    use App\Models\Book;

    $books = Book::all();

    if ($someCondition) {
        $books->load('author', 'publisher');
    }
```

如果你需要在预加载查询上设置额外的查询约束，你可以传递一个由你希望加载的关联作为键的数组。数组值应该是接收查询实例的闭包实例：

```php
    $author->load(['books' => function (Builder $query) {
        $query->orderBy('published_date', 'asc');
    }]);
```

要仅在尚未加载时加载关联，请使用 `loadMissing` 方法：

```php
$book->loadMissing('author');
```

#### 嵌套预加载和 `morphTo`

如果你想要预加载一个 `morphTo` 关联，以及可能由该关联返回的各种实体上的嵌套关联，你可以使用 `loadMorph` 方法。

该方法接受 `morphTo` 关联的名称作为第一个参数，以及一个模型 / 关联对的数组作为第二个参数。为了帮助说明这个方法，让我们考虑以下模型：

```php
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class ActivityFeed extends Model
{
    /**
     * 获取活动源记录的父级。
  */
    public function parentable(): MorphTo
    {
        return $this->morphTo();
    }
}
```

在这个例子中，假设 `Event`、`Photo` 和 `Post` 模型可以创建 `ActivityFeed` 模型。此外，假设 `Event` 模型属于 `Calendar` 模型，`Photo` 模型与 `Tag` 模型相关联，而 `Post` 模型属于 `Author` 模型。

使用这些模型定义和关联，我们可以检索 `ActivityFeed` 模型实例，并预加载所有 `parentable` 模型及其各自的嵌套关联：

```php
$activities = ActivityFeed::with('parentable')
    ->get()
    ->loadMorph('parentable', [
        Event::class => ['calendar'],
        Photo::class => ['tags'],
        Post::class => ['author'],
    ]);
```

### 防止懒加载

如前所述，预加载关联通常可以为你的应用程序提供显著的性能优势。因此，如果你希望的话，你可以指示 Laravel 始终阻止关联的懒加载。为了实现这一点，你可以调用基础 Eloquent 模型类提供的 `preventLazyLoading` 方法。通常情况下，你应该在应用程序的 `AppServiceProvider` 类的 `boot` 方法中调用这个方法。

`preventLazyLoading` 方法接受一个可选的布尔参数，指示是否应禁止懒加载。例如，你可能希望仅在非生产环境中禁用懒加载，这样即使在生产代码中意外存在懒加载关系，生产环境仍将正常运行：

```php
use Illuminate\Database\Eloquent\Model;

/**
 * 初始化任何应用程序服务。
  */
public function boot(): void
{
    Model::preventLazyLoading(! $this->app->isProduction());
}
```

在禁止懒加载后，当你的应用程序尝试懒加载任何 Eloquent 关系时，Eloquent 将抛出一个 `Illuminate\Database\LazyLoadingViolationException` 异常。

你可以使用 `handleLazyLoadingViolationUsing` 方法自定义懒加载违规行为。例如，使用这个方法，你可以指示懒加载违规行为仅被记录，而不是用异常中断应用程序的执行：

```php
Model::handleLazyLoadingViolationUsing(function (Model $model, string $relation) {
    $class = $model::class;

    info("Attempted to lazy load [{$relation}] on model [{$class}].");
});
```

## 插入和更新相关模型

### `save` 方法

Eloquent 提供了方便的方法来将新模型添加到关系中。例如，也许你需要向帖子添加一条新评论。你可以使用关系的 `save` 方法来插入评论，而不是手动设置 `Comment` 模型上的 `post_id` 属性：

```php
use App\Models\Comment;
use App\Models\Post;

$comment = new Comment(['message' => 'A new comment.']);

$post = Post::find(1);

$post->comments()->save($comment);
```

请注意，我们没有将 `comments` 关系作为动态属性访问。相反，我们调用了 `comments` 方法来获取关系的实例。`save` 方法将自动将适当的 `post_id` 值添加到新的 `Comment` 模型中。

如果需要保存多个相关模型，可以使用 `saveMany` 方法：

```php
$post = Post::find(1);

$post->comments()->saveMany([
    new Comment(['message' => 'A new comment.']),
    new Comment(['message' => 'Another new comment.']),
]);
```

`save` 和 `saveMany` 方法会持久化给定的模型实例，但不会将新持久化的模型添加到已加载到父模型的任何内存关系中。如果计划在使用 `save` 或 `saveMany` 方法后访问关系，可以使用 `refresh` 方法重新加载模型及其关系：

```php
$post->comments()->save($comment);

$post->refresh();

// 所有评论，包括新保存的评论...
$post->comments;
```

#### 递归保存模型和关系

如果想要 `save` 你的模型及其所有关联关系，可以使用 `push` 方法。在这个例子中，`Post` 模型将被保存，以及它的评论和评论的作者：

```php
$post = Post::find(1);

$post->comments[0]->message = 'Message';
$post->comments[0]->author->name = 'Author Name';

$post->push();
```

`pushQuietly` 方法可用于保存模型及其关联关系而不触发任何事件：

### `create` 方法

除了 `save` 和 `saveMany` 方法之外，还可以使用 `create` 方法，它接受一个属性数组，创建一个模型，并将其插入数据库。`save` 和 `create` 之间的区别在于 `save` 接受一个完整的 Eloquent 模型实例，而 `create` 接受一个普通的 PHP `array`。`create` 方法将返回新创建的模型：

```php
use App\Models\Post;

$post = Post::find(1);

$comment = $post->comments()->create([
    'message' => 'A new comment.',
]);
```

你可以使用 `createMany` 方法来创建多个相关模型：

```php
$post = Post::find(1);

$post->comments()->createMany([
    ['message' => 'A new comment.'],
    ['message' => 'Another new comment.'],
]);
```

`createQuietly` 和 `createManyQuietly` 方法可用于创建模型而不触发任何事件：

```php
$user = User::find(1);

$user->posts()->createQuietly([
    'title' => 'Post title.',
]);

$user->posts()->createManyQuietly([
    ['title' => 'First post.'],
    ['title' => 'Second post.'],
]);
```

你还可以使用 `findOrNew`、`firstOrNew`、`firstOrCreate` 和 `updateOrCreate` 方法来[在关系上创建和更新模型](https://learnku.com/docs/laravel/11.x/eloquent#upserts)。

> **注意**  
> 在使用 `create` 方法之前，请确保查看[批量赋值](https://learnku.com/docs/laravel/11.x/eloquent#mass-assignment)文档。

### 属于关系

如果想将子模型分配给新的父模型，可以使用 `associate` 方法。在这个例子中，`User` 模型定义了一个到 `Account` 模型的 `belongsTo` 关系。`associate` 方法将在子模型上设置外键：

```php
use App\Models\Account;

$account = Account::find(10);

$user->account()->associate($account);

$user->save();
```

要从子模型中移除父模型，可以使用 `dissociate` 方法。该方法将关系的外键设置为 `null`：

```php
$user->account()->dissociate();

$user->save();
```

### 多对多关系

#### 附加 / 分离

Eloquent 还提供了一些方法，使得处理多对多关系更加方便。例如，假设一个用户可以拥有多个角色，而一个角色也可以拥有多个用户。你可以使用 `attach` 方法将一个角色附加到一个用户，通过在关系的中间表中插入一条记录：

```php
use App\Models\User;

$user = User::find(1);

$user->roles()->attach($roleId);
```

当向模型附加关系时，你也可以传递一个包含要插入到中间表的附加数据的数组：

```php
$user->roles()->attach($roleId, ['expires' => $expires]);
```

有时需要从用户中移除一个角色。要移除多对多关系记录，可以使用 `detach` 方法。`detach` 方法将从中间表中删除适当的记录；但是，两个模型将仍然保留在数据库中：

```php
// 从用户中分离单个角色...
$user->roles()->detach($roleId);

// 从用户中分离所有角色...
$user->roles()->detach();
```

为了方便起见，`attach` 和 `detach` 也接受 ID 数组作为输入：

```php
$user = User::find(1);

$user->roles()->detach([1, 2, 3]);

$user->roles()->attach([
    1 => ['expires' => $expires],
    2 => ['expires' => $expires],
]);
```

#### 同步关联

你还可以使用 `sync` 方法构建多对多关联。`sync` 方法接受一个 ID 数组，放置在中间表上。不在给定数组中的任何 ID 将从中间表中删除。因此，在此操作完成后，只有给定数组中的 ID 将存在于中间表中：

```php
$user->roles()->sync([1, 2, 3]);
```

你也可以传递附加的中间表值与 ID：

```php
$user->roles()->sync([1 => ['expires' => true], 2, 3]);
```

如果希望将相同的中间表值与每个同步模型 ID 一起插入，可以使用 `syncWithPivotValues` 方法：

```php
$user->roles()->syncWithPivotValues([1, 2, 3], ['active' => true]);
```

如果不想分离缺少在给定数组中的现有 ID，可以使用 `syncWithoutDetaching` 方法：

```php
$user->roles()->syncWithoutDetaching([1, 2, 3]);
```

#### 切换关联

多对多关系还提供了一个 `toggle` 方法，用于「切换」给定相关模型的附加状态。如果给定的 ID 当前已附加，它将被分离。同样，如果它当前处于分离状态，它将被附加：

```php
$user->roles()->toggle([1, 2, 3]);
```

你也可以传递附加表的额外中间值与 ID：

```php
$user->roles()->toggle([
    1 => ['expires' => true],
    2 => ['expires' => true],
]);
```

#### 更新中间表上的记录

如果你需要更新关系的中间表中的现有行，你可以使用 `updateExistingPivot` 方法。该方法接受中间记录外键和要更新的属性数组：

```php
$user = User::find(1);

$user->roles()->updateExistingPivot($roleId, [
    'active' => false,
]);
```

## 更新父级时间戳

当一个模型定义了一个 `belongsTo` 或 `belongsToMany` 关系到另一个模型，比如一个 `Comment` 属于一个 `Post`，有时在更新子模型时更新父模型的时间戳是有帮助的。

例如，当一个 `Comment` 模型被更新时，你可能希望自动「触摸」拥有的 `Post` 的 `updated_at` 时间戳，使其设置为当前日期和时间。为了实现这一点，你可以在子模型中添加一个 `touches` 属性，其中包含应在更新子模型时更新其 `updated_at` 时间戳的关系名称：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Comment extends Model
{
    /**
     * 所有要被触摸的关系。
  *
     * @var array
     */
    protected $touches = ['post'];

    /**
     * 获取评论所属的帖子。
  */
    public function post(): BelongsTo
    {
        return $this->belongsTo(Post::class);
    }
}
```

> **警告**  
> 只有当子模型使用 Eloquent 的 `save` 方法进行更新时，父模型的时间戳才会被更新。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-relationshipsmd/16703)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-relationshipsmd/16703)