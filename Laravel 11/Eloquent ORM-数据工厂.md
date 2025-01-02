本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Eloquent: 数据工厂

+   [介绍](#introduction)
+   [定义模型工厂](#defining-model-factories)
    +   [生成工厂](#generating-factories)
    +   [工厂状态](#factory-states)
    +   [工厂回调](#factory-callbacks)
+   [使用工厂创建模型](#creating-models-using-factories)
    +   [实例化模型](#instantiating-models)
    +   [持久化模型](#persisting-models)
    +   [顺序](#sequences)
+   [工厂关联](#factory-relationships)
    +   [一对多关系](#has-many-relationships)
    +   [属于关系](#belongs-to-relationships)
    +   [多对多关系](#many-to-many-relationships)
    +   [多态关系](#polymorphic-relationships)
    +   [在工厂中定义关系](#defining-relationships-within-factories)
    +   [重复利用现有的模型建立关系](#recycling-an-existing-model-for-relationships)

## 介绍

当测试你的应用程序或向数据库填充数据时，你可能需要插入一些记录到数据库中。Laravel 允许你使用模型工厂为每个 [Eloquent 模型](https://learnku.com/docs/laravel/11.x/eloquent) 定义一组默认属性，而不是手动指定每个列的值。

要查看如何编写工厂的示例，请查看你的应用程序中的 `database/factories/UserFactory.php` 文件。这个工厂已经包含在所有新的 Laravel 应用程序中，并包含以下工厂定义：

```php
    namespace Database\Factories;

    use Illuminate\Database\Eloquent\Factories\Factory;
    use Illuminate\Support\Facades\Hash;
    use Illuminate\Support\Str;

    /**
     * @extends \Illuminate\Database\Eloquent\Factories\Factory<\App\Models\User>
     */
    class UserFactory extends Factory
    {
        /**
         * 被用在数据工厂的当前密码。
  */
        protected static ?string $password;

        /**
         * 定义模型的默认状态。
  *
         * @return array<string, mixed>
         */
        public function definition(): array
        {
            return [
                'name' => fake()->name(),
                'email' => fake()->unique()->safeEmail(),
                'email_verified_at' => now(),
                'password' => static::$password ??= Hash::make('password'),
                'remember_token' => Str::random(10),
            ];
        }

        /**
         * 标识模型的电子邮件地址未经验证。
  */
        public function unverified(): static
        {
            return $this->state(fn (array $attributes) => [
                'email_verified_at' => null,
            ]);
        }
    }
```

正如你所见，在最基本的形式中，数据工厂是继承 Laravel 基础工厂类并定义 `definition` 方法的类。`definition` 方法返回应在使用工厂创建模型时应用的默认属性值集合。  
通过 `fake` 辅助器，工厂可以访问 [Faker](https://github.com/FakerPHP/Faker) PHP 库， 它允许你方便地生成各种用于测试和填充的随机数据。

> \[! 注意\]  
> 你可以通过更新 `config/app.php` 配置文件中的 `faker_locale` 选项来更改应用程序的 Faker 区域设置。

## 定义模型工厂

### 生成工厂

为了创建工厂，请执行 `make:factory` [Artisan 命令](https://learnku.com/docs/laravel/11.x/artisan):

```shell
php artisan make:factory PostFactory
```

新的工厂类将被放置在你的 `database/factories` 目录中。

#### 模型和工厂的发现规范

一旦你定义了工厂，你可以使用 `Illuminate\Database\Eloquent\Factories\HasFactory` 特性提供的静态 `factory` 方法来为该模型实例化一个工厂实例。  
`HasFactory` 特性的 `factory` 方法将使用约定来确定分配给该特性的模型的适当工厂。具体而言，该方法将在 `Database\Factories` 命名空间中查找一个类名与模型名称匹配且以 `Factory` 为后缀的工厂。如果这些约定不适用于你的特定应用程序或工厂，则可以重写模型上的 `newFactory` 方法以直接返回相应工厂的实例：

use Illuminate\\Database\\Eloquent\\Factories\\Factory;  
use Database\\Factories\\Administration\\FlightFactory;

```php
/**
 * 为该模型创建一个新的工厂实例.
 */
protected static function newFactory(): Factory
{
    return FlightFactory::new();
}
```

接下来，为相应工厂定义一个 `model` 属性：

```php
    use App\Administration\Flight;
    use Illuminate\Database\Eloquent\Factories\Factory;

    class FlightFactory extends Factory
    {
        /**
         * 工厂对应的模型类名。
  *
         * @var class-string<\Illuminate\Database\Eloquent\Model>
         */
        protected $model = Flight::class;
    }
```

### 工厂状态

状态操作方法可以让你定义离散的修改，这些修改可以在任意组合中应用于你的模型工厂。例如，你的 `Database\Factories\UserFactory` 工厂可能包含一个 `suspended` 状态方法，该方法可以修改其默认属性值之一。

状态转换方法通常会调用 Laravel 基础工厂类提供的 `state` 方法。`state` 方法接受一个闭包函数，该函数将接收为工厂定义的原始属性数组，并应返回一个要修改的属性数组：

```php
    use Illuminate\Database\Eloquent\Factories\Factory;

    /**
     * 表示用户已被暂停。
  */
    public function suspended(): Factory
    {
        return $this->state(function (array $attributes) {
            return [
                'account_status' => 'suspended',
            ];
        });
    }
```

#### 「Trashed」状态

如果你的 Eloquent 模型可以进行[软删除](https://learnku.com/docs/laravel/11.x/eloquentmd#soft-deleting)，你可以调用内置的 `trashed` 状态方法来表示创建的模型应该已经被「软删除」。你不需要手动定义 `trashed` 状态，因为它对所有工厂都是自动可用的：

```php
    use App\Models\User;

    $user = User::factory()->trashed()->create();
```

### 工厂回调

工厂回调是使用 `afterMaking` 和 `afterCreating` 方法注册的，它们允许你在生成或新创建型后执行额外任务。你应该通过在工厂类中定义一个 `configure` 方法来注册这些回调。当工厂被实例化时，Laravel 将自动调用这个方法：

```php
    namespace Database\Factories;

    use App\Models\User;
    use Illuminate\Database\Eloquent\Factories\Factory;

    class UserFactory extends Factory
    {
        /**
         * 配置模型工厂。
  */
        public function configure(): static
        {
            return $this->afterMaking(function (User $user) {
                // ...
            })->afterCreating(function (User $user) {
                // ...
            });
        }

        // ...
    }
```

你也可以结合 `state` 方法注册工厂回调来实现指定状态下的额外任务。

```php
    use App\Models\User;
    use Illuminate\Database\Eloquent\Factories\Factory;

    /**
     * 表示用户被停用。
  */
    public function suspended(): Factory
    {
        return $this->state(function (array $attributes) {
            return [
                'account_status' => 'suspended',
            ];
        })->afterMaking(function (User $user) {
            // ...
        })->afterCreating(function (User $user) {
            // ...
        });
    }
```

## 使用工厂创建模型

### 实例化模型

一旦你定义了工厂，你可以使用由 `Illuminate\Database\Eloquent\Factories\HasFactory` trait 为你的模型提供的 `factory` 静态方法来实例化该模型的工厂对象。让我们看一些创建模型的示例。首先，我们将使用 `make` 方法创建模型，而不将其持久化到数据库中：

```php
    use App\Models\User;

    $user = User::factory()->make();
```

你可以使用 count 方法创建多个模型的集合：

```php
    $users = User::factory()->count(3)->make();
```

#### 应用状态

你也可以将任何[状态](#factory-states)应用于这些模型。如果你想要对这些模型应用多个状态转换，只需直接调用状态转换方法即可：

```php
    $users = User::factory()->count(5)->suspended()->make();
```

#### 覆盖属性

如果你想要覆盖模型的一些默认值，可以将一个值数组传递给 `make` 方法。只有指定的属性将被替换，而其余的属性将保持设置为工厂指定的默认值：

```php
    $user = User::factory()->make([
        'name' => 'Abigail Otwell',
    ]);
```

或者，可以直接在工厂实例上调用 `state` 方法进行内联状态转换：

```php
    $user = User::factory()->state([
        'name' => 'Abigail Otwell',
    ])->make();
```

> **技巧**  
> 使用工厂创建模型时，[批量赋值保护](https://learnku.com/docs/laravel/11.x/eloquentmd#mass-assignment)会自动被禁用。

### 持久化模型

`create` 方法会实例化模型并使用 Eloquent 的 `save` 方法将它们持久化到数据库中：

```php
    use App\Models\User;

    // 创建一个单独的 App\Models\User 实例...
    $user = User::factory()->create();

    // 创建三个 App\Models\User 实例...
    $users = User::factory()->count(3)->create();
```

你可以通过将属性数组传递给 `create` 方法来覆盖工厂的默认模型属性：

```php
    $user = User::factory()->create([
        'name' => 'Abigail',
    ]);
```

### 顺序

有时，你可能希望为每个创建的模型交替更改给定模型属性的值。你可以通过将状态转换定义为顺序来实现此目的。例如，你可能希望为每个创建的用户在 `admin` 列中在 `Y` 和 `N` 之间交替更改：

```php
    use App\Models\User;
    use Illuminate\Database\Eloquent\Factories\Sequence;

    $users = User::factory()
                    ->count(10)
                    ->state(new Sequence(
                        ['admin' => 'Y'],
                        ['admin' => 'N'],
                    ))
                    ->create();
```

在这个例子中，将创建五个 `admin` 值为 `Y` 的用户和五个 `admin` 值为 `N` 的用户。

如果需要，你可以将闭包作为顺序值包含在内。每次顺序需要一个新值时，都会调用闭包：

```php
    use Illuminate\Database\Eloquent\Factories\Sequence;

    $users = User::factory()
                    ->count(10)
                    ->state(new Sequence(
                        fn (Sequence $sequence) => ['role' => UserRoles::all()->random()],
                    ))
                    ->create();
```

在顺序闭包内，你可以访问注入到闭包中的顺序实例上的 `$index` 或 `$count` 属性。 `$index` 属性包含到目前为止已经进行的顺序迭代次数，而 `$count` 属性包含顺序将被调用的总次数：

```php
    $users = User::factory()
                    ->count(10)
                    ->sequence(fn (Sequence $sequence) => ['name' => 'Name '.$sequence->index])
                    ->create();
```

为了方便，顺序的应用也可以使用 `sequence` 方法，该方法简单的在内部调用了 `state` 方法。 `sequence` 方法接受一个闭包或顺序的属性的数组：

```php
    $users = User::factory()
                    ->count(2)
                    ->sequence(
                        ['name' => 'First User'],
                        ['name' => 'Second User'],
                    )
                    ->create();
```

## 工厂关联

### 一对多关系

接下来，让我们探讨如何使用 Laravel 流畅的工厂方法构建 Eloquent 模型关系。首先，假设我们的应用程序有一个 `App\Models\User` 模型和一个 `App\Models\Post` 模型。同时，假设 `User` 模型定义了与 `Post` 的一对多关系。我们可以使用 Laravel 工厂提供的 `has` 方法创建一个有三篇文章的用户。`has` 方法接受一个工厂实例：

```php
    use App\Models\Post;
    use App\Models\User;

    $user = User::factory()
                ->has(Post::factory()->count(3))
                ->create();
```

按照约定，当将 `Post` 模型传递给 `has` 方法时，Laravel 将假定 `User` 模型必须有一个 `posts` 方法来定义关系。如果需要，你可以显式指定你想要操作的关系名称：

```php
    $user = User::factory()
                ->has(Post::factory()->count(3), 'posts')
                ->create();
```

当然，你可以对相关模型执行状态操作。此外，如果你的状态更改需要访问父模型，你可以传递基于闭包的状态转换：

```php
$user = User::factory()
            ->has(
                Post::factory()
                        ->count(3)
                        ->state(function (array $attributes, User $user) {
                            return ['user_type' => $user->type];
                        })
            )
            ->create();
```

#### 使用魔术方法

为了方便起见，你可以使用 Laravel 的魔术工厂关系方法来构建关系。例如，以下示例将使用约定确定相关模型应通过 `posts` 关系方法在 `User` 模型上创建：

```php
$user = User::factory()
            ->hasPosts(3)
            ->create();
```

在使用魔术方法创建工厂关系时，你可以传递一个属性数组，用于在相关模型上进行覆盖：

```php
$user = User::factory()
            ->hasPosts(3, [
                'published' => false,
            ])
            ->create();
```

如果你的状态更改需要访问父模型，你可以提供基于闭包的状态转换：

```php
$user = User::factory()
            ->hasPosts(3, function (array $attributes, User $user) {
                return ['user_type' => $user->type];
            })
            ->create();
```

### 属于关系

现在我们已经探讨了如何使用工厂构建「has many」关系，让我们来探讨关系的反向。`for` 方法可用于定义工厂创建的模型所属的父模型。例如，我们可以创建属于单个用户的三个 `App\Models\Post` 模型实例：

```php
use App\Models\Post;
use App\Models\User;

$posts = Post::factory()
            ->count(3)
            ->for(User::factory()->state([
                'name' => 'Jessica Archer',
            ]))
            ->create();
```

如果你已经有一个应与你正在创建的模型关联的父模型实例，你可以将该模型实例传递给 `for` 方法：

```php
$user = User::factory()->create();

$posts = Post::factory()
            ->count(3)
            ->for($user)
            ->create();
```

#### 使用魔术方法

为了方便起见，你可以使用 Laravel 的魔术工厂关系方法来定义「属于」关系。例如，以下示例将使用约定确定这三篇文章应该属于 `Post` 模型上的 `user` 关系：

```php
$posts = Post::factory()
            ->count(3)
            ->forUser([
                'name' => 'Jessica Archer',
            ])
            ->create();
```

### 多对多关系

类似于[一对多关系](#has-many-relationships)，可以使用 `has` 方法创建「多对多」关系：

```php
use App\Models\Role;
use App\Models\User;

$user = User::factory()
            ->has(Role::factory()->count(3))
            ->create();
```

#### 中间表属性

如果你需要定义应设置在连接模型之间的中间表 / 中间表上的属性，你可以使用 `hasAttached` 方法。该方法接受一个包含中间表属性名称和值的数组作为其第二个参数：

```php
use App\Models\Role;
use App\Models\User;

$user = User::factory()
            ->hasAttached(
                Role::factory()->count(3),
                ['active' => true]
            )
            ->create();
```

如果你的状态更改需要访问相关模型，你可以提供基于闭包的状态转换：

```php
$user = User::factory()
            ->hasAttached(
                Role::factory()
                    ->count(3)
                    ->state(function (array $attributes, User $user) {
                        return ['name' => $user->name.' Role'];
                    }),
                ['active' => true]
            )
            ->create();
```

如果你已经有模型实例希望附加到你正在创建的模型上，你可以将模型实例传递给 `hasAttached` 方法。在这个示例中，相同的三个角色将附加给所有三个用户：

```php
$roles = Role::factory()->count(3)->create();

$user = User::factory()
            ->count(3)
            ->hasAttached($roles, ['active' => true])
            ->create();
```

#### 使用魔术方法

为了方便起见，你可以使用 Laravel 的魔术工厂关系方法来定义多对多关系。例如，以下示例将使用约定确定相关模型应通过 `roles` 关系方法在 `User` 模型上创建：

```php
$user = User::factory()
            ->hasRoles(1, [
                'name' => 'Editor'
            ])
            ->create();
```

### 多态关系

[多态关系](https://learnku.com/docs/laravel/11.x/eloquent-relationships#polymorphic-relationships) 也可以使用工厂创建。多态「morph many」关系的创建方式与典型的「has many」关系相同。例如，如果 `App\Models\Post` 模型与 `App\Models\Comment` 模型具有 `morphMany` 关系：

```php
use App\Models\Post;

$post = Post::factory()->hasComments(3)->create();
```

#### Morph To 关系

不能使用魔术方法创建 `morphTo` 关系。相反，必须直接使用 `for` 方法，并明确提供关系的名称。例如，假设 `Comment` 模型具有一个 `commentable` 方法定义了一个 `morphTo` 关系。在这种情况下，我们可以通过直接使用 `for` 方法创建属于单个帖子的三条评论：

```php
$comments = Comment::factory()->count(3)->for(
    Post::factory(), 'commentable'
)->create();
```

#### 多态多对多关系

多态「多对多」（`morphToMany` / `morphedByMany`）关系的创建方式与非多态「多对多」关系相同：

```php
use App\Models\Tag;
use App\Models\Video;

$videos = Video::factory()
            ->hasAttached(
                Tag::factory()->count(3),
                ['public' => true]
            )
            ->create();
```

当然，魔术 `has` 方法也可以用于创建多态「多对多」关系：

```php
$videos = Video::factory()
            ->hasTags(3, ['public' => true])
            ->create();
```

### 在工厂中定义关系

要在你的模型工厂中定义关系，通常会将一个新的工厂实例分配给关系的外键。这通常用于「反向」关系，如 `belongsTo` 和 `morphTo` 关系。例如，如果你想在创建帖子时创建一个新用户，你可以这样做：

```php
use App\Models\User;

/**
 * 定义模型的默认状态。
  *
 * @return array<string, mixed>
 */
public function definition(): array
{
    return [
        'user_id' => User::factory(),
        'title' => fake()->title(),
        'content' => fake()->paragraph(),
    ];
}
```

如果关系的列取决于定义它的工厂，你可以将闭包分配给一个属性。闭包将接收工厂评估的属性数组：

```php
/**
 * 定义模型的默认状态。
  *
 * @return array<string, mixed>
 */
public function definition(): array
{
    return [
        'user_id' => User::factory(),
        'user_type' => function (array $attributes) {
            return User::find($attributes['user_id'])->type;
        },
        'title' => fake()->title(),
        'content' => fake()->paragraph(),
    ];
}
```

### 重复使用现有模型来建立关系

如果你有多个模型与另一个模型共享一个常见关系，你可以使用 `recycle` 方法确保相关模型的单个实例被重复使用，用于工厂创建的所有关系。

举例来说，假设你有 `Airline` 、 `Flight` 和 `Ticket` 模型，其中机票属于一家航空公司和一个航班，而航班也隶属于一家航空公司。在创建机票时，你可能希望为机票和航班选择相同的航空公司，因此可以将一家航空公司实例传递给 `recycle` 方法：

Ticket::factory()  
\->recycle(Airline::factory()->create())  
\->create();

如果你的模型属于公共用户或团队，你可能会发现 `recycle` 方法特别有用。

`recycle` 方法还接受一组现有模型。当将一组模型提供给 `recycle` 方法时，工厂需要该类型的模型时会从集合中随机选择一个模型：

Ticket::factory()  
\->recycle($airlines)  
\->create();

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-factoriesmd/16708)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/el...](https://learnku.com/docs/laravel/11.x/eloquent-factoriesmd/16708)