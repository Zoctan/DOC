本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## 交易工具包 (Stripe)

+   [引言](#introduction)
+   [交易工具包升级](#upgrading-cashier)
+   [安装](#installation)
+   [配置](#configuration)
    +   [可计费模型](#billable-model)
    +   [API 密钥](#api-keys)
    +   [货币设置](#currency-configuration)
    +   [税务设置](#tax-configuration)
    +   [日志记录](#logging)
    +   [使用自定义模型](#using-custom-models)
+   [快速上手](#quickstart)
    +   [销售产品](#quickstart-selling-products)
    +   [销售订阅服务](#quickstart-selling-subscriptions)
+   [客户管理](#customers)
    +   [获取客户信息](#retrieving-customers)
    +   [创建客户](#creating-customers)
    +   [更新客户信息](#updating-customers)
    +   [余额管理](#balances)
    +   [税号管理](#tax-ids)
    +   [与 Stripe 同步客户数据](#syncing-customer-data-with-stripe)
    +   [账单门户](#billing-portal)
+   [支付方式](#payment-methods)
    +   [保存支付方式](#storing-payment-methods)
    +   [获取支付方式](#retrieving-payment-methods)
    +   [检查支付方式是否存在](#payment-method-presence)
    +   [更新默认支付方式](#updating-the-default-payment-method)
    +   [添加支付方式](#adding-payment-methods)
    +   [删除支付方式](#deleting-payment-methods)
+   [订阅管理](#subscriptions)
    +   [创建订阅](#creating-subscriptions)
    +   [检查订阅状态](#checking-subscription-status)
    +   [更改价格](#changing-prices)
    +   [订阅数量](#subscription-quantity)
    +   [多产品订阅](#subscriptions-with-multiple-products)
    +   [多重订阅](#multiple-subscriptions)
    +   [按量计费](#metered-billing)
    +   [订阅税费](#subscription-taxes)
    +   [订阅起始日期](#subscription-anchor-date)
    +   [取消订阅](#cancelling-subscriptions)
    +   [恢复订阅](#resuming-subscriptions)
+   [订阅试用期](#subscription-trials)
    +   [预先提供支付方式](#with-payment-method-up-front)
    +   [无需预先提供支付方式](#without-payment-method-up-front)
    +   [延长试用期](#extending-trials)
+   [处理 Stripe Webhooks](#handling-stripe-webhooks)
    +   [定义 Webhook 事件处理器](#defining-webhook-event-handlers)
    +   [验证 Webhook 签名](#verifying-webhook-signatures)
+   [一次性收费](#single-charges)
    +   [简单收费](#simple-charge)
    +   [带发票的收费](#charge-with-invoice)
    +   [创建支付意向](#creating-payment-intents)
    +   [退款处理](#refunding-charges)
+   [结账流程](#checkout)
    +   [产品结账](#product-checkouts)
    +   [一次性收费结账](#single-charge-checkouts)
    +   [订阅结账](#subscription-checkouts)
    +   [收集税号](#collecting-tax-ids)
    +   [访客结账](#guest-checkouts)
+   [发票管理](#invoices)
    +   [获取发票](#retrieving-invoices)
    +   [即将到来的发票](#upcoming-invoices)
    +   [预览订阅发票](#previewing-subscription-invoices)
    +   [生成发票 PDF](#generating-invoice-pdfs)
+   [处理支付失败](#handling-failed-payments)
    +   [确认支付](#confirming-payments)
+   [强客户认证 (SCA)](#strong-customer-authentication)
    +   [需要额外确认的支付](#payments-requiring-additional-confirmation)
    +   [离线支付通知](#off-session-payment-notifications)
+   [Stripe SDK](#stripe-sdk)
+   [测试](#testing)

## 引言

[Laravel 交易工具包](https://github.com/laravel/cashier-stripe) 为 [Stripe](https://stripe.com/) 的订阅计费服务提供了一个富有表现力且流畅的接口。它几乎处理了所有你不愿编写的订阅计费样板代码。除了基本的订阅管理外，交易工具包还可以处理优惠券、切换订阅、订阅「数量」、取消宽限期，甚至生成发票 PDF。

## 升级交易工具包

在升级到新版本的工具包时，务必仔细阅读 [升级指南](https://github.com/laravel/cashier-stripe/blob/master/UPGRADE.md)。

> **警告**  
> 为了防止破坏性变更，交易工具包使用固定的 Stripe API 版本。交易工具包 15 使用 Stripe API 版本 `2023-10-16` 。Stripe API 版本将在次要版本中更新，以便利用 Stripe 的新功能和改进。

## 安装

首先，使用 Composer 包管理器安装 Stripe 的交易工具包：

```shell
composer require laravel/cashier
```

安装包后，使用 `vendor:publish` Artisan 命令发布 Cashier 的迁移文件：

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

然后，迁移你的数据库：

交易工具包的迁移将在你的 `users` 表中添加几个列。它们还将创建一个新的 `subscriptions` 表来保存所有客户的订阅，以及一个 `subscription_items` 表用于多价格订阅。

如果你愿意，也可以使用 `vendor:publish` Artisan 命令发布 Cashier 的配置文件：

```shell
php artisan vendor:publish --tag="cashier-config"
```

最后，为确保交易工具包正确处理所有 Stripe 事件，请记得 [配置交易工具包的 webhook 处理](#handling-stripe-webhooks)。

> **警告**  
> Stripe 建议用于存储 Stripe 标识符的任何列都应区分大小写。因此，在使用 MySQL 时，你应确保 stripe\_id 列的排序规则设置为 utf8\_bin。有关此内容的更多信息可以在 [Stripe 文档](https://stripe.com/docs/upgrades#what-changes-does-stripe-consider-to-be-backwards-compatible) 中找到。

## 配置

### 可计费模型

在使用 Cashier 之前，请将 `Billable` 特性添加到你的可计费模型定义中。通常，这将是 `App\Models\User` 模型。该特性提供了各种方法，允许你执行常见的计费任务，如创建订阅、应用优惠券和更新付款方式信息：

```php
use Laravel\Cashier\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

Cashier 假定你的可计费模型将是 Laravel 提供的 `App\Models\User` 类。如果你希望更改这一点，可以通过 `useCustomerModel` 方法指定不同的模型。这个方法通常应该在你的 `AppServiceProvider` 类的 `boot` 方法中调用：

```php
use App\Models\Cashier\User;
use Laravel\Cashier\Cashier;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Cashier::useCustomerModel(User::class);
}
```

> **警告**  
> 如果你使用的模型不是 Laravel 提供的 `App\Models\User` 模型，你需要发布并修改提供的 [Cashier 迁移](#installation)，以匹配你的替代模型的表名。

### API 密钥

接下来，你应在应用程序的 `.env` 文件中配置你的 Stripe API 密钥。你可以从 Stripe 控制面板中检索你的 Stripe API 密钥：

```ini
STRIPE_KEY=your-stripe-key
STRIPE_SECRET=your-stripe-secret
STRIPE_WEBHOOK_SECRET=your-stripe-webhook-secret
```

> **警告**  
> 请确保在应用程序的 `.env` 文件中定义了 `STRIPE_WEBHOOK_SECRET` 环境变量，因为该变量用于确保传入的 Webhooks 实际来自 Stripe。

### 货币配置

Cashier 的默认货币是美元（USD）。你可以通过在应用程序的 `.env` 文件中设置 `CASHIER_CURRENCY` 环境变量来更改默认货币：

除了配置 Cashier 的货币外，你还可以指定一个区域设置，用于在发票上显示货币值时格式化。在内部，Cashier 使用 [PHP 的 `NumberFormatter` 类](https://www.php.net/manual/en/class.numberformatter.php) 来设置货币区域设置：

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> **警告**  
> 若要使用除 `en` 外的区域设置，请确保在服务器上安装并配置了 `ext-intl` PHP 扩展。

### 税务配置

由于 [Stripe Tax](https://stripe.com/tax) 的支持，可以自动为 Stripe 生成的所有发票计算税费。你可以通过在应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中调用 `calculateTaxes` 方法来启用自动税费计算：

```php
use Laravel\Cashier\Cashier;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Cashier::calculateTaxes();
}
```

一旦启用了税费计算，任何生成的新订阅和任何一次性发票都将接收到自动税费计算。

为使此功能正常工作，客户的账单详细信息，如客户姓名、地址和税号，需要与 Stripe 同步。你可以使用 Cashier 提供的 [客户数据同步](#syncing-customer-data-with-stripe) 和 [税号](#tax-ids) 方法来实现这一点。

> **警告**  
> 不会为 [单次收费](#single-charges) 或 [单次收费结账](#single-charge-checkouts) 计算税费。

### 日志记录

Cashier 允许你指定在记录致命的 Stripe 错误时要使用的日志通道。你可以通过在应用程序的 `.env` 文件中定义 `CASHIER_LOGGER` 环境变量来指定日志通道：

通过对 Stripe 的 API 调用生成的异常将通过你的应用程序的默认日志通道进行记录。

### 使用自定义模型

你可以通过定义自己的模型并扩展相应的 Cashier 模型来自由扩展 Cashier 内部使用的模型：

```php
use Laravel\Cashier\Subscription as CashierSubscription;

class Subscription extends CashierSubscription
{
    // ...
}
```

定义完你的模型后，你可以通过 `Laravel\Cashier\Cashier` 类指示 Cashier 使用你的自定义模型。通常，你应该在应用程序的 `App\Providers\AppServiceProvider` 类的 `boot` 方法中告知 Cashier 关于你的自定义模型：

```php
use App\Models\Cashier\Subscription;
use App\Models\Cashier\SubscriptionItem;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Cashier::useSubscriptionModel(Subscription::class);
    Cashier::useSubscriptionItemModel(SubscriptionItem::class);
}
```

## 快速入门

### 销售产品

> **注意**  
> 在使用 Stripe Checkout 之前，你应该在 Stripe 仪表板中定义具有固定价格的产品。此外，你应该[配置 Cashier 的 Webhook 处理](#handling-stripe-webhooks)。

通过你的应用程序提供产品和订阅计费可能会让人望而生畏。然而，由于 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout) 的支持，你可以轻松构建现代、强大的支付集成。

要为非重复的单次收费产品向客户收费，我们将利用 Cashier 将客户引导到 Stripe Checkout，在那里他们将提供他们的付款信息并确认购买。一旦通过 Checkout 进行付款，客户将被重定向到你在应用程序中选择的成功 URL：

```php
use Illuminate\Http\Request;

Route::get('/checkout', function (Request $request) {
    $stripePriceId = 'price_deluxe_album';

    $quantity = 1;

    return $request->user()->checkout([$stripePriceId => $quantity], [
        'success_url' => route('checkout-success'),
        'cancel_url' => route('checkout-cancel'),
    ]);
})->name('checkout');

Route::view('/checkout/success', 'checkout.success')->name('checkout-success');
Route::view('/checkout/cancel', 'checkout.cancel')->name('checkout-cancel');
```

正如上面的示例所示，我们将利用 Cashier 提供的 `checkout` 方法将客户重定向到 Stripe Checkout，针对给定的「价格标识符」。在使用 Stripe 时，「价格」指的是[针对特定产品定义的价格](https://stripe.com/docs/products-prices/how-products-and-prices-work)。

如果需要，`checkout` 方法将自动在 Stripe 中创建一个客户，并将该 Stripe 客户记录连接到你应用程序数据库中相应的用户。完成结账会话后，客户将被重定向到专用的成功或取消页面，你可以在那里向客户显示信息性消息。

#### 向 Stripe Checkout 提供元数据

在销售产品时，通过你自己的应用程序定义的 `Cart` 和 `Order` 模型通常用于跟踪已完成的订单和已购买的产品。当将客户重定向到 Stripe Checkout 完成购买时，你可能需要提供一个现有的订单标识符，以便在客户被重定向回你的应用程序时将已完成的购买与相应的订单关联起来。

为实现这一点，你可以向 `checkout` 方法提供一个 `metadata` 数组。假设在我们的应用程序中，当用户开始结账流程时，会创建一个待处理的 `Order`。请记住，本示例中的 `Cart` 和 `Order` 模型仅用于说明，并非由 Cashier 提供。你可以根据你自己应用程序的需求自由实现这些概念：

```php
use App\Models\Cart;
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/cart/{cart}/checkout', function (Request $request, Cart $cart) {
    $order = Order::create([
        'cart_id' => $cart->id,
        'price_ids' => $cart->price_ids,
        'status' => 'incomplete',
    ]);

    return $request->user()->checkout($order->price_ids, [
        'success_url' => route('checkout-success').'?session_id={CHECKOUT_SESSION_ID}',
        'cancel_url' => route('checkout-cancel'),
        'metadata' => ['order_id' => $order->id],
    ]);
})->name('checkout');
```

正如上面的示例所示，当用户开始结账流程时，我们将为 `checkout` 方法提供所有购物车 / 订单关联的 Stripe 价格标识符。当客户将它们添加时，你的应用程序负责将这些项目与「购物车」或订单关联起来。我们还通过 `metadata` 数组向 Stripe Checkout 会话提供订单的 ID。最后，我们在结账成功路由中添加了 `CHECKOUT_SESSION_ID` 模板变量。当 Stripe 将客户重定向回你的应用程序时，此模板变量将自动填充为 Checkout 会话 ID。

接下来，让我们构建结账成功路由。这是用户在通过 Stripe Checkout 完成购买后将被重定向到的路由。在这个路由中，我们可以检索 Stripe Checkout 会话 ID 和关联的 Stripe Checkout 实例，以便访问我们提供的元数据并相应地更新客户的订单：

```php
use App\Models\Order;
use Illuminate\Http\Request;
use Laravel\Cashier\Cashier;

Route::get('/checkout/success', function (Request $request) {
    $sessionId = $request->get('session_id');

    if ($sessionId === null) {
        return;
    }

    $session = Cashier::stripe()->checkout->sessions->retrieve($sessionId);

    if ($session->payment_status !== 'paid') {
        return;
    }

    $orderId = $session['metadata']['order_id'] ?? null;

    $order = Order::findOrFail($orderId);

    $order->update(['status' => 'completed']);

    return view('checkout-success', ['order' => $order]);
})->name('checkout-success');
```

请参考 Stripe 的文档以获取有关 [Checkout 会话对象中包含的数据](https://stripe.com/docs/api/checkout/sessions/object)的更多信息。

### 销售订阅

> **注意**  
> 在使用 Stripe Checkout 之前，你应该在 Stripe 仪表板中定义具有固定价格的产品。此外，你应该[配置 Cashier 的 Webhook 处理](#handling-stripe-webhooks)。

通过你的应用程序提供产品和订阅计费可能会让人望而生畏。然而，借助 Cashier 和 [Stripe Checkout](https://stripe.com/payments/checkout)，你可以轻松构建现代、强大的支付集成。

要学习如何使用 Cashier 和 Stripe Checkout 出售订阅，让我们考虑一个简单的情景：一个具有基本月度（`price_basic_monthly`）和年度（`price_basic_yearly`）计划的订阅服务。这两个价格可以在我们的 Stripe 仪表板中作为「Basic」产品（`pro_basic`）进行分组。此外，我们的订阅服务可能还提供 `pro_expert` 作为专家计划。

首先，让我们了解客户如何订阅我们的服务。当然，你可以想象客户可能会在我们应用程序的定价页面上为基本计划点击「订阅」按钮。这个按钮或链接应该将用户重定向到一个 Laravel 路由，为他们选择的计划创建 Stripe Checkout 会话：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_basic_monthly')
        ->trialDays(5)
        ->allowPromotionCodes()
        ->checkout([
            'success_url' => route('your-success-route'),
            'cancel_url' => route('your-cancel-route'),
        ]);
});
```

正如上面的示例所示，我们将客户重定向到一个 Stripe Checkout 会话，让他们订阅我们的基本计划。在成功结账或取消后，客户将被重定向回我们在 `checkout` 方法中提供的 URL。为了知道他们的订阅实际开始的时间（因为一些支付方式需要几秒钟来处理），我们还需要[配置 Cashier 的 Webhook 处理](#handling-stripe-webhooks)。

现在客户可以开始订阅了，我们需要限制应用程序的某些部分，以便只有订阅用户才能访问它们。当然，我们可以始终通过 Cashier 的 `Billable` trait 提供的 `subscribed` 方法来确定用户当前的订阅状态：

```blade
@if ($user->subscribed())
    <p>你已订阅。</p>
@endif
```

我们甚至可以轻松确定用户是否订阅了特定产品或价格：

```blade
@if ($user->subscribedToProduct('pro_basic'))
    <p>你已订阅我们的基础产品。</p>
@endif

@if ($user->subscribedToPrice('price_basic_monthly'))
    <p>你已订阅我们的月度基础计划。</p>
@endif
```

#### 创建订阅中间件

为了方便起见，你可能希望创建一个[中间件](https://learnku.com/docs/laravel/11.x/middleware)，用于确定传入请求是否来自已订阅用户。一旦定义了这个中间件，你可以轻松地将其分配给一个路由，以防止未订阅的用户访问该路由：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class Subscribed
{
    /**
     * 处理传入请求。
  */
    public function handle(Request $request, Closure $next): Response
    {
        if (! $request->user()?->subscribed()) {
            // 将用户重定向到计费页面并要求他们订阅...
            return redirect('/billing');
        }

        return $next($request);
    }
}
```

一旦定义了中间件，你可以将其分配给一个路由：

```php
use App\Http\Middleware\Subscribed;

Route::get('/dashboard', function () {
    // ...
})->middleware([Subscribed::class]);
```

#### 允许客户管理他们的计费计划

当然，客户可能希望将他们的订阅计划更改为另一个产品或「层级」。允许这样做的最简单方法是将客户引导到 Stripe 的[客户计费门户](https://stripe.com/docs/no-code/customer-portal)，该门户提供了一个托管用户界面，允许客户下载发票、更新他们的付款方式以及更改订阅计划。

首先，在你的应用程序中定义一个链接或按钮，将用户重定向到一个 Laravel 路由，我们将利用该路由来启动一个计费门户会话：

```blade
<a href="{{ route('billing') }}">
    计费
</a>
```

接下来，让我们定义一个路由，启动一个 Stripe 客户计费门户会话，并将用户重定向到门户。`redirectToBillingPortal` 方法接受用户在退出门户时应返回的 URL：

```php
use Illuminate\Http\Request;

Route::get('/billing', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('dashboard'));
})->middleware(['auth'])->name('billing');
```

> **注意**  
> 只要你已经配置了 Cashier 的 Webhook 处理，Cashier 将通过检查来自 Stripe 的传入 Webhook 自动保持你的应用程序的 Cashier 相关数据库表同步。因此，例如，当用户通过 Stripe 的客户计费门户取消订阅时，Cashier 将接收相应的 Webhook 并在你的应用程序数据库中将订阅标记为「已取消」。

## 客户

### 检索客户

你可以使用 `Cashier::findBillable` 方法通过他们的 Stripe ID 检索客户。该方法将返回一个可计费模型的实例：

```php
use Laravel\Cashier\Cashier;

$user = Cashier::findBillable($stripeId);
```

### 创建客户

有时，你可能希望创建一个 Stripe 客户而不开始订阅。你可以使用 `createAsStripeCustomer` 方法来实现这一点：

```php
$stripeCustomer = $user->createAsStripeCustomer();
```

一旦客户在 Stripe 中创建，你可以在以后的某个日期开始订阅。你可以提供一个可选的 `$options` 数组，以传递任何额外的[由 Stripe API 支持的客户创建参数](https://stripe.com/docs/api/customers/create)：

```php
$stripeCustomer = $user->createAsStripeCustomer($options);
```

如果你想为可计费模型返回 Stripe 客户对象，可以使用 `asStripeCustomer` 方法：

```php
$stripeCustomer = $user->asStripeCustomer();
```

如果你想要检索给定可计费模型的 Stripe 客户对象，但不确定该可计费模型是否已经是 Stripe 中的客户，可以使用 `createOrGetStripeCustomer` 方法。如果该客户在 Stripe 中不存在，该方法将创建一个新的客户：

```php
$stripeCustomer = $user->createOrGetStripeCustomer();
```

### 更新客户

有时，你可能希望直接使用额外信息更新 Stripe 客户。你可以使用 `updateStripeCustomer` 方法来实现这一点。该方法接受一个 [Stripe API 支持的客户更新选项数组](https://stripe.com/docs/api/customers/update)：

```php
$stripeCustomer = $user->updateStripeCustomer($options);
```

### 余额

Stripe 允许你为客户的「余额」增加或减少金额。之后，这个余额将在新的发票上得到增加或减少。你可以使用可计费模型上提供的 `balance` 方法来检查客户的总余额。`balance` 方法将以客户的货币形式返回一个格式化的余额字符串：

```php
$balance = $user->balance();
```

要为客户的余额增加金额，你可以向 `creditBalance` 方法提供一个值。如果需要，你也可以提供一个描述：

```php
$user->creditBalance(500, '高级客户充值。');
```

向 `debitBalance` 方法提供一个值将减少客户的余额：

```php
$user->debitBalance(300, '不良使用惩罚。');
```

`applyBalance` 方法将为客户创建新的余额交易记录。你可以使用 `balanceTransactions` 方法检索这些交易记录，这可能对提供客户审查的积分和借记日志很有用：

```php
// 检索所有交易记录...
$transactions = $user->balanceTransactions();

foreach ($transactions as $transaction) {
    // 交易金额...
    $amount = $transaction->amount(); // $2.31

    // 在可能的情况下检索相关发票...
    $invoice = $transaction->invoice();
}
```

### 税务识别号

Cashier 提供了一个简单的方法来管理客户的税务识别号。例如，可以使用 `taxIds` 方法来检索分配给客户的所有[税务识别号](https://stripe.com/docs/api/customer_tax_ids/object)作为一个集合：

```php
$taxIds = $user->taxIds();
```

你也可以通过其标识符为客户检索特定的税务识别号：

```php
$taxId = $user->findTaxId('txi_belgium');
```

通过向 `createTaxId` 方法提供有效的 [type](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-type) 和值，你可以创建一个新的税务识别号：

```php
$taxId = $user->createTaxId('eu_vat', 'BE0123456789');
```

`createTaxId` 方法将立即将 VAT ID 添加到客户的账户中。[Stripe 还会对 VAT ID 进行验证](https://stripe.com/docs/invoicing/customer/tax-ids#validation)；但是，这是一个异步过程。你可以通过订阅 `customer.tax_id.updated` Webhook 事件并检查 [VAT ID 的 `verification` 参数](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-verification)来获得验证更新的通知。有关处理 Webhooks 的更多信息，请参阅[定义 Webhook 处理程序的文档](#handling-stripe-webhooks)。

你可以使用 `deleteTaxId` 方法删除税务识别号：

```php
$user->deleteTaxId('txi_belgium');
```

### 将客户数据与 Stripe 同步

通常，当你的应用程序用户更新他们的姓名、电子邮件地址或其他也存储在 Stripe 中的信息时，你应该通知 Stripe 进行更新。通过这样做，Stripe 中的信息副本将与你的应用程序保持同步。

要自动化这一过程，你可以在可计费模型上定义一个事件监听器，以响应模型的 `updated` 事件。然后，在你的事件监听器中，你可以在模型上调用 `syncStripeCustomerDetails` 方法：

```php
use App\Models\User;
use function Illuminate\Events\queueable;

/**
 * 模型的 "booted" 方法。
  */
protected static function booted(): void
{
    static::updated(queueable(function (User $customer) {
        if ($customer->hasStripeId()) {
            $customer->syncStripeCustomerDetails();
        }
    }));
}
```

现在，每当你的客户模型被更新时，其信息将与 Stripe 同步。为了方便起见，在创建客户时，Cashier 将自动将客户的信息与 Stripe 同步。

你可以通过覆盖 Cashier 提供的各种方法来自定义用于将客户信息与 Stripe 同步的列。例如，你可以覆盖 `stripeName` 方法，以自定义应被视为客户「姓名」的属性，当 Cashier 将客户信息同步到 Stripe 时：

```php
/**
 * 获取应同步到 Stripe 的客户名称。
  */
public function stripeName(): string|null
{
    return $this->company_name;
}
```

类似地，你可以覆盖 `stripeEmail`、`stripePhone`、`stripeAddress` 和 `stripePreferredLocales` 方法。这些方法将在[更新 Stripe 客户对象](https://stripe.com/docs/api/customers/update)时将信息同步到相应的客户参数。如果你希望完全控制客户信息同步过程，你可以覆盖 `syncStripeCustomerDetails` 方法。

### 计费门户

Stripe 提供了[一个简单的设置计费门户的方法](https://stripe.com/docs/billing/subscriptions/customer-portal)，让你的客户可以管理他们的订阅、支付方式，并查看他们的账单历史。你可以通过在控制器或路由中在可计费模型上调用 `redirectToBillingPortal` 方法来将用户重定向到计费门户：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal();
});
```

默认情况下，当用户完成管理他们的订阅后，他们可以通过 Stripe 计费门户内的链接返回到你应用程序的 `home` 路由。你可以通过将自定义 URL 作为参数传递给 `redirectToBillingPortal` 方法来提供用户应返回的自定义 URL：

```php
use Illuminate\Http\Request;

Route::get('/billing-portal', function (Request $request) {
    return $request->user()->redirectToBillingPortal(route('billing'));
});
```

如果你想要生成到计费门户的 URL 而不生成 HTTP 重定向响应，你可以调用 `billingPortalUrl` 方法：

```php
$url = $request->user()->billingPortalUrl(route('billing'));
```

## 付款方式

### 存储付款方式

为了使用 Stripe 创建订阅或执行「一次性」收费，你需要存储一个付款方式并从 Stripe 检索其标识符。为了实现这一目的，根据你计划将付款方式用于订阅还是单次收费，采取的方法有所不同，因此我们将在下面分别讨论这两种情况。

#### 订阅的付款方式

当为将来使用订阅的客户存储信用卡信息时，必须使用 Stripe 的「设置意向」API 来安全地收集客户的付款方式详细信息。「设置意向」向 Stripe 指示要收取客户的付款方式。Cashier 的 `Billable` 特性包括 `createSetupIntent` 方法，用于轻松创建新的设置意向。你应该从将呈现收集客户付款方式详细信息的表单的路由或控制器中调用此方法：

```php
return view('update-payment-method', [
    'intent' => $user->createSetupIntent()
]);
```

在创建设置意向并将其传递给视图后，你应该将其密钥附加到将收集付款方式的元素上。例如，考虑以下「更新付款方式」表单：

```html
<input id="card-holder-name" type="text">

<!-- Stripe 元素占位符 -->
<div id="card-element"></div>

<button id="card-button" data-secret="{{ $intent->client_secret }}">
    更新付款方式
</button>
```

接下来，可以使用 Stripe.js 库将 [Stripe 元素](https://stripe.com/docs/stripe-js)附加到表单中，并安全地收集客户的付款详细信息：

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

接下来，可以验证卡片并使用 [Stripe 的 `confirmCardSetup` 方法](https://stripe.com/docs/js/setup_intents/confirm_card_setup)从 Stripe 获取安全的「付款方式标识符」：

```js
const cardHolderName = document.getElementById('card-holder-name');
const cardButton = document.getElementById('card-button');
const clientSecret = cardButton.dataset.secret;

cardButton.addEventListener('click', async (e) => {
    const { setupIntent, error } = await stripe.confirmCardSetup(
        clientSecret, {
            payment_method: {
                card: cardElement,
                billing_details: { name: cardHolderName.value }
            }
        }
    );

    if (error) {
        // 向用户显示「error.message」...
    } else {
        // 卡片已成功验证...
    }
});
```

在 Stripe 验证卡片后，你可以将生成的 `setupIntent.payment_method` 标识符传递给你的 Laravel 应用程序，在那里它可以附加到客户。付款方式可以作为[新的付款方式添加](#adding-payment-methods)或[用于更新默认付款方式](#updating-the-default-payment-method)。你还可以立即使用付款方式标识符来[创建新的订阅](#creating-subscriptions)。

> **注意**  
> 如果你想要了解有关设置意向和收集客户付款详细信息的更多信息，请[查看 Stripe 提供的概述](https://stripe.com/docs/payments/save-and-reuse#php)。

#### 单次收费的付款方式

当针对客户的付款方式进行单次收费时，我们只需要一次使用付款方式标识符。由于 Stripe 的限制，你可能无法在单次收费中使用客户的存储默认付款方式。你必须允许客户使用 Stripe.js 库输入他们的付款方式详细信息。例如，考虑以下表单：

```html
<input id="card-holder-name" type="text">

<!-- Stripe 元素占位符 -->
<div id="card-element"></div>

<button id="card-button">
    处理付款
</button>
```

在定义这样的表单后，可以使用 Stripe.js 库将 [Stripe 元素](https://stripe.com/docs/stripe-js)附加到表单中，并安全地收集客户的付款详细信息：

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

接下来，可以验证卡片并使用 [Stripe 的 `createPaymentMethod` 方法](https://stripe.com/docs/stripe-js/reference#stripe-create-payment-method)从 Stripe 获取安全的「付款方式标识符」：

```js
const cardHolderName = document.getElementById('card-holder-name');
const cardButton = document.getElementById('card-button');

cardButton.addEventListener('click', async (e) => {
    const { paymentMethod, error } = await stripe.createPaymentMethod(
        'card', cardElement, {
            billing_details: { name: cardHolderName.value }
        }
    );

    if (error) {
        // 向用户显示「error.message」...
    } else {
        // 卡片已成功验证...
    }
});
```

如果卡片验证成功，你可以将 `paymentMethod.id` 传递给你的 Laravel 应用程序并处理[单次收费](#simple-charge)。

### 获取付款方式

在可账单化模型实例上的 `paymentMethods` 方法返回一个 `Laravel\Cashier\PaymentMethod` 实例的集合：

```php
$paymentMethods = $user->paymentMethods();
```

默认情况下，此方法将返回每种类型的付款方式。要检索特定类型的付款方式，可以将 `type` 作为参数传递给方法：

```php
$paymentMethods = $user->paymentMethods('sepa_debit');
```

要检索客户的默认付款方式，可以使用 `defaultPaymentMethod` 方法：

```php
$paymentMethod = $user->defaultPaymentMethod();
```

使用 `findPaymentMethod` 方法可以检索附加到可账单化模型的特定付款方式：

```php
$paymentMethod = $user->findPaymentMethod($paymentMethodId);
```

### 付款方式存在性

要确定可账单化模型是否附加了默认付款方式到他们的帐户，请调用 `hasDefaultPaymentMethod` 方法：

```php
if ($user->hasDefaultPaymentMethod()) {
    // ...
}
```

你可以使用 `hasPaymentMethod` 方法来确定可账单化模型是否至少附加了一个付款方式到他们的帐户：

```php
if ($user->hasPaymentMethod()) {
    // ...
}
```

此方法将确定可账单化模型是否有任何付款方式。要确定特定类型的付款方式是否存在于模型中，可以将 `type` 作为参数传递给方法：

```php
if ($user->hasPaymentMethod('sepa_debit')) {
    // ...
}
```

### 更新默认付款方式

`updateDefaultPaymentMethod` 方法可用于更新客户的默认付款方式信息。此方法接受一个 Stripe 付款方式标识符，并将新付款方式分配为默认的账单付款方式：

```php
$user->updateDefaultPaymentMethod($paymentMethod);
```

要将默认付款方式信息与 Stripe 中客户的默认付款方式信息同步，可以使用 `updateDefaultPaymentMethodFromStripe` 方法：

```php
$user->updateDefaultPaymentMethodFromStripe();
```

> **警告**  
> 客户的默认付款方式只能用于开具发票和创建新订阅。由于 Stripe 的限制，它可能无法用于单次收费。

### 添加付款方式

要添加新的付款方式，你可以在可账单化模型上调用 `addPaymentMethod` 方法，传递付款方式标识符：

```php
$user->addPaymentMethod($paymentMethod);
```

> **注意**  
> 要了解如何检索付款方式标识符，请查看[付款方式存储文档](#storing-payment-methods)。

### 删除付款方式

要删除付款方式，你可以在要删除的 `Laravel\Cashier\PaymentMethod` 实例上调用 `delete` 方法：

```php
$paymentMethod->delete();
```

`deletePaymentMethod` 方法将从可账单化模型中删除特定的付款方式：

```php
$user->deletePaymentMethod('pm_visa');
```

`deletePaymentMethods` 方法将删除可账单化模型的所有付款方式信息：

```php
$user->deletePaymentMethods();
```

默认情况下，此方法将删除每种类型的付款方式。要删除特定类型的付款方式，可以将 `type` 作为参数传递给方法：

```php
$user->deletePaymentMethods('sepa_debit');
```

> **警告**  
> 如果用户有活跃订阅，你的应用程序不应允许他们删除他们的默认付款方式。

## 订阅

订阅提供了一种为客户设置循环付款的方式。由 Cashier 管理的 Stripe 订阅支持多个订阅价格、订阅数量、试用期等功能。

### 创建订阅

要创建订阅，首先检索可账单化模型的实例，通常将是 `App\Models\User` 的实例。一旦检索到模型实例，可以使用 `newSubscription` 方法创建模型的订阅：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription(
        'default', 'price_monthly'
    )->create($request->paymentMethodId);

    // ...
});
```

传递给 `newSubscription` 方法的第一个参数应该是订阅的内部类型。如果你的应用程序只提供单个订阅，你可以称之为 `default` 或 `primary`。这种订阅类型仅供内部应用程序使用，不应向用户显示。此外，它不应包含空格，并且在创建订阅后不应更改。第二个参数是用户要订阅的具体价格。这个值应该对应于 Stripe 中价格的标识符。

`create` 方法接受 [Stripe 付款方式标识符](#storing-payment-methods)或 Stripe `PaymentMethod` 对象，将开始订阅，并更新你的数据库，其中包含可账单化模型的 Stripe 客户 ID 和其他相关的计费信息。

> **警告**  
> 直接将付款方式标识符传递给 `create` 订阅方法还将自动将其添加到用户存储的付款方式中。

### 通过发票电子邮件收取循环付款

与自动收取客户的循环付款不同，你可以要求 Stripe 在每次客户的循环付款到期时向客户发送一封发票电子邮件。然后，客户可以在收到发票后手动支付。在通过发票收取循环付款时，客户无需提前提供付款方式：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice();
```

客户在取消订阅之前必须支付发票的时间取决于 `days_until_due` 选项。默认情况下，这是 30 天；但是，如果需要，你可以为此选项提供一个特定值：

```php
$user->newSubscription('default', 'price_monthly')->createAndSendInvoice([], [
    'days_until_due' => 30
]);
```

### 订阅数量

如果你想在创建订阅时为价格设置特定的[数量](https://stripe.com/docs/billing/subscriptions/quantities)，你应该在创建订阅之前在订阅构建器上调用 `quantity` 方法：

```php
$user->newSubscription('default', 'price_monthly')
     ->quantity(5)
     ->create($paymentMethod);
```

#### 附加详情

如果你想指定额外的[客户](https://stripe.com/docs/api/customers/create)或[订阅](https://stripe.com/docs/api/subscriptions/create)选项（由 Stripe 支持），你可以将它们作为第二个和第三个参数传递给 `create` 方法：

```php
$user->newSubscription('default', 'price_monthly')->create($paymentMethod, [
    'email' => $email,
], [
    'metadata' => ['note' => '一些额外信息。'],
]);
```

#### 优惠券

如果你想在创建订阅时应用优惠券，你可以使用 `withCoupon` 方法：

```php
$user->newSubscription('default', 'price_monthly')
     ->withCoupon('code')
     ->create($paymentMethod);
```

或者，如果你想应用 [Stripe 促销代码](https://stripe.com/docs/billing/subscriptions/discounts/codes)，你可以使用 `withPromotionCode` 方法：

```php
$user->newSubscription('default', 'price_monthly')
     ->withPromotionCode('promo_code_id')
     ->create($paymentMethod);
```

给定的促销代码 ID 应该是分配给促销代码的 Stripe API ID，而不是客户可见的促销代码。如果你需要根据给定的客户可见促销代码查找促销代码 ID，你可以使用 `findPromotionCode` 方法：

```php
// 根据客户可见代码查找促销代码ID...
$promotionCode = $user->findPromotionCode('SUMMERSALE');

// 根据客户可见代码查找活动促销代码ID...
$promotionCode = $user->findActivePromotionCode('SUMMERSALE');
```

在上面的示例中，返回的 `$promotionCode` 对象是 `Laravel\Cashier\PromotionCode` 的实例。这个类装饰了一个底层的 `Stripe\PromotionCode` 对象。你可以通过调用 `coupon` 方法来获取与促销代码相关联的优惠券：

```php
$coupon = $user->findPromotionCode('SUMMERSALE')->coupon();
```

优惠券实例允许你确定折扣金额以及优惠券是固定折扣还是基于百分比的折扣：

```php
if ($coupon->isPercentage()) {
    return $coupon->percentOff().'%'; // 21.5%
} else {
    return $coupon->amountOff(); // $5.99
}
```

你还可以检索当前应用于客户或订阅的折扣：

```php
$discount = $billable->discount();

$discount = $subscription->discount();
```

返回的 `Laravel\Cashier\Discount` 实例装饰了一个底层的 `Stripe\Discount` 对象实例。你可以通过调用 `coupon` 方法来获取与该折扣相关联的优惠券：

```php
$coupon = $subscription->discount()->coupon();
```

如果你想将新的优惠券或促销代码应用于客户或订阅，你可以通过 `applyCoupon` 或 `applyPromotionCode` 方法来实现：

```php
$billable->applyCoupon('coupon_id');
$billable->applyPromotionCode('promotion_code_id');

$subscription->applyCoupon('coupon_id');
$subscription->applyPromotionCode('promotion_code_id');
```

请记住，你应该使用分配给促销代码的 Stripe API ID，而不是客户可见的促销代码。在特定时间内，只能向客户或订阅应用一个优惠券或促销代码。

有关更多信息，请参考 Stripe 关于[优惠券](https://stripe.com/docs/billing/subscriptions/coupons)和[促销代码](https://stripe.com/docs/billing/subscriptions/coupons/codes)的文档。

### 添加订阅

如果你想给已经有默认支付方式的客户添加订阅，你可以在订阅构建器上调用 `add` 方法：

```php
use App\Models\User;

$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->add();
```

#### 从 Stripe 仪表板创建订阅

你也可以直接从 Stripe 仪表板创建订阅。这样做时，Cashier 将同步新添加的订阅并将它们分配一个类型为 `default`。如果要自定义分配给仪表板创建的订阅的订阅类型，请[定义 Webhook 事件处理程序](#defining-webhook-event-handlers)。

此外，你只能通过 Stripe 仪表板创建一种类型的订阅。如果你的应用程序提供使用不同类型的多个订阅，只能通过 Stripe 仪表板添加一种类型的订阅。

最后，你应该始终确保每种应用程序提供的订阅类型只添加一个活动订阅。如果客户有两个 `default` 订阅，Cashier 只会使用最近添加的订阅，尽管两者都会与你的应用程序数据库同步。

### 检查订阅状态

一旦客户订阅了你的应用程序，你可以使用各种便利方法轻松检查他们的订阅状态。首先，`subscribed` 方法在客户有活动订阅时返回 `true`，即使订阅目前处于试用期内。`subscribed` 方法接受订阅类型作为其第一个参数：

```php
if ($user->subscribed('default')) {
    // ...
}
```

`subscribed` 方法还可以作为[路由中间件](https://learnku.com/docs/laravel/11.x/middleware)的一个很好的选择，让你可以根据用户的订阅状态过滤对路由和控制器的访问：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserIsSubscribed
{
    /**
     * 处理传入请求。
  *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->user() && ! $request->user()->subscribed('default')) {
            // 这个用户不是付费客户...
            return redirect('billing');
        }

        return $next($request);
    }
}
```

如果你想确定用户是否仍处于试用期内，你可以使用 `onTrial` 方法。这个方法可以用于确定是否应向用户显示警告，告诉他们他们仍处于试用期内：

```php
if ($user->subscription('default')->onTrial()) {
    // ...
}
```

`subscribedToProduct` 方法可用于确定用户是否订阅了给定产品，基于给定的 Stripe 产品标识符。在 Stripe 中，产品是价格的集合。在这个例子中，我们将确定用户的 `default` 订阅是否积极订阅了应用程序的「premium」产品。给定的 Stripe 产品标识符应该对应于 Stripe 仪表板中一个产品的标识符：

```php
if ($user->subscribedToProduct('prod_premium', 'default')) {
    // ...
}
```

通过向 `subscribedToProduct` 方法传递一个数组，你可以确定用户的 `default` 订阅是否积极订阅了应用程序的「basic」或「premium」产品：

```php
if ($user->subscribedToProduct(['prod_basic', 'prod_premium'], 'default')) {
    // ...
}
```

`subscribedToPrice` 方法可用于确定客户的订阅是否对应于给定的价格 ID：

```php
if ($user->subscribedToPrice('price_basic_monthly', 'default')) {
    // ...
}
```

`recurring` 方法可用于确定用户当前是否订阅且不再处于试用期内：

```php
if ($user->subscription('default')->recurring()) {
    // ...
}
```

> **警告**  
> 如果用户有两个相同类型的订阅，`subscription` 方法始终会返回最近的订阅。例如，用户可能有两个类型为 `default` 的订阅记录；然而，其中一个订阅可能是旧的、已过期的订阅，而另一个是当前的、活动的订阅。最近的订阅将始终返回，而旧的订阅会保留在数据库中供历史审查。

#### 取消订阅状态

要确定用户曾经是活跃订阅者但已取消他们的订阅，你可以使用 `canceled` 方法：

```php
if ($user->subscription('default')->canceled()) {
    // ...
}
```

你还可以确定用户是否已取消订阅，但仍处于「宽限期」直到订阅完全到期。例如，如果用户在 3 月 5 日取消了原定于 3 月 10 日到期的订阅，则用户在 3 月 10 日之前处于「宽限期」。请注意，在此期间 `subscribed` 方法仍然返回 `true`：

```php
if ($user->subscription('default')->onGracePeriod()) {
    // ...
}
```

要确定用户是否已取消订阅且不再处于「宽限期」，你可以使用 `ended` 方法：

```php
if ($user->subscription('default')->ended()) {
    // ...
}
```

#### 不完整和过期状态

如果订阅创建后需要进行第二次付款操作，则订阅将被标记为 `incomplete`。订阅状态存储在 Cashier 的 `subscriptions` 数据库表的 `stripe_status` 列中。

类似地，如果在更改价格时需要进行第二次付款操作，则订阅将被标记为 `past_due`。当订阅处于这两种状态之一时，直到客户确认付款为止，订阅将不处于活动状态。可以使用 billable 模型或订阅实例上的 `hasIncompletePayment` 方法来确定订阅是否存在不完整的付款：

```php
if ($user->hasIncompletePayment('default')) {
    // ...
}

if ($user->subscription('default')->hasIncompletePayment()) {
    // ...
}
```

当订阅存在不完整的付款时，你应该将用户引导至 Cashier 的付款确认页面，传递 `latestPayment` 标识符。你可以使用订阅实例上提供的 `latestPayment` 方法来检索这个标识符：

```html
<a href="{{ route('cashier.payment', $subscription->latestPayment()->id) }}">
    请确认你的付款。
</a>
```

如果希望在订阅处于 `past_due` 或 `incomplete` 状态时仍然将其视为活跃状态，可以使用 Cashier 提供的 `keepPastDueSubscriptionsActive` 和 `keepIncompleteSubscriptionsActive` 方法。通常，这些方法应该在 `App\Providers\AppServiceProvider` 的 `register` 方法中调用：

```php
use Laravel\Cashier\Cashier;

/**
 * Register any application services.
 */
public function register(): void
{
    Cashier::keepPastDueSubscriptionsActive();
    Cashier::keepIncompleteSubscriptionsActive();
}
```

> **警告**  
> 当订阅处于 `incomplete` 状态时，直到付款确认之前无法更改订阅。因此，当订阅处于 `incomplete` 状态时，`swap` 和 `updateQuantity` 方法将抛出异常。

#### 订阅范围

大多数订阅状态也作为查询范围可用，这样你可以轻松地查询处于特定状态的订阅：

```php
// 获取所有活跃订阅...
$subscriptions = Subscription::query()->active()->get();

// 获取用户的所有已取消订阅...
$subscriptions = $user->subscriptions()->canceled()->get();
```

下面是可用的所有范围的完整列表：

```php
Subscription::query()->active();
Subscription::query()->canceled();
Subscription::query()->ended();
Subscription::query()->incomplete();
Subscription::query()->notCanceled();
Subscription::query()->notOnGracePeriod();
Subscription::query()->notOnTrial();
Subscription::query()->onGracePeriod();
Subscription::query()->onTrial();
Subscription::query()->pastDue();
Subscription::query()->recurring();
```

### 更改价格

当客户订阅你的应用程序后，他们可能偶尔想要更改到新的订阅价格。要将客户切换到新的价格，请将 Stripe 价格标识符传递给 `swap` 方法。在更改价格时，假设用户希望重新激活他们之前取消的订阅。给定的价格标识符应该对应于 Stripe 仪表板中可用的 Stripe 价格标识符：

```php
use App\Models\User;

$user = App\Models\User::find(1);

$user->subscription('default')->swap('price_yearly');
```

如果客户正在试用期内，试用期将被保留。此外，如果订阅存在「数量」，该数量也将被保留。

如果你想要切换价格并取消客户当前正在进行的试用期，你可以调用 `skipTrial` 方法：

```php
$user->subscription('default')
    ->skipTrial()
    ->swap('price_yearly');
```

如果你想要切换价格并立即向客户开具发票，而不是等待他们的下一个计费周期，你可以使用 `swapAndInvoice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->swapAndInvoice('price_yearly');
```

#### 比例调整

默认情况下，在不同价格之间进行切换时，Stripe 会按比例调整费用。可以使用 `noProrate` 方法来更新订阅的价格，而不会按比例调整费用：

```php
$user->subscription('default')->noProrate()->swap('price_yearly');
```

有关订阅比例调整的更多信息，请参考 [Stripe 文档](https://stripe.com/docs/billing/subscriptions/prorations)。

> **警告**  
> 在执行 `swapAndInvoice` 方法之前执行 `noProrate` 方法将不会影响比例调整。发票将始终被开具。

### 订阅数量

有时订阅会受到「数量」的影响。例如，一个项目管理应用程序可能会按每个项目每月 $10 收费。你可以使用 `incrementQuantity` 和 `decrementQuantity` 方法轻松增加或减少订阅数量：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->incrementQuantity();

// 将当前数量增加五个...
$user->subscription('default')->incrementQuantity(5);

$user->subscription('default')->decrementQuantity();

// 将当前数量减少五个...
$user->subscription('default')->decrementQuantity(5);
```

另外，你可以使用 `updateQuantity` 方法设置特定的数量：

```php
$user->subscription('default')->updateQuantity(10);
```

可以使用 `noProrate` 方法来更新订阅的数量，而不会按比例调整费用：

```php
$user->subscription('default')->noProrate()->updateQuantity(10);
```

有关订阅数量的更多信息，请参考 [Stripe 文档](https://stripe.com/docs/subscriptions/quantities)。

#### 多产品订阅的数量

如果你的订阅是一个[具有多个产品的订阅](#subscriptions-with-multiple-products)，你应该将你希望增加或减少数量的价格的 ID 作为第二个参数传递给增加 / 减少方法：

```php
$user->subscription('default')->incrementQuantity(1, 'price_chat');
```

### 具有多个产品的订阅

[具有多个产品的订阅](https://stripe.com/docs/billing/subscriptions/multiple-products)允许你将多个计费产品分配给单个订阅。例如，假设你正在构建一个客户服务「帮助台」应用程序，基本订阅价格为每月 $10，但提供额外的每月 $15 的实时聊天附加产品。具有多个产品的订阅的信息存储在 Cashier 的 `subscription_items` 数据库表中。

你可以通过将价格数组作为 `newSubscription` 方法的第二个参数传递来为给定订阅指定多个产品：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default', [
        'price_monthly',
        'price_chat',
    ])->create($request->paymentMethodId);

    // ...
});
```

在上面的示例中，客户将有两个价格附加到他们的 `default` 订阅上。两个价格将在各自的计费间隔上收费。如果需要，你可以使用 `quantity` 方法为每个价格指定特定数量：

```php
$user = User::find(1);

$user->newSubscription('default', ['price_monthly', 'price_chat'])
    ->quantity(5, 'price_chat')
    ->create($paymentMethod);
```

如果你想要向现有订阅添加另一个价格，你可以调用订阅的 `addPrice` 方法：

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat');
```

上面的示例将添加新价格，客户将在下一个计费周期收取费用。如果你想要立即向客户收费，你可以使用 `addPriceAndInvoice` 方法：

```php
$user->subscription('default')->addPriceAndInvoice('price_chat');
```

如果你想要添加一个具有特定数量的价格，你可以将数量作为 `addPrice` 或 `addPriceAndInvoice` 方法的第二个参数传递：

```php
$user = User::find(1);

$user->subscription('default')->addPrice('price_chat', 5);
```

你可以使用 `removePrice` 方法从订阅中移除价格：

```php
$user->subscription('default')->removePrice('price_chat');
```

> **警告**  
> 你不能移除订阅中的最后一个价格。相反，你应该简单地取消订阅。

#### 替换价格

你也可以更改附加到具有多个产品的订阅的价格。例如，假设客户有一个 `price_basic` 订阅和一个 `price_chat` 附加产品，你想要将客户从 `price_basic` 升级到 `price_pro` 价格：

```php
use App\Models\User;

$user = User::find(1);

$user->subscription('default')->swap(['price_pro', 'price_chat']);
```

在执行上面的示例时，`price_basic` 的基础订阅项目将被删除，`price_chat` 的项目将被保留。此外，将为 `price_pro` 创建一个新的订阅项目。

你还可以通过向 `swap` 方法传递键 / 值对数组来指定订阅项目选项。例如，你可能需要指定订阅价格的数量：

```php
$user = User::find(1);

$user->subscription('default')->swap([
    'price_pro' => ['quantity' => 5],
    'price_chat'
]);
```

如果你想在订阅中替换单个价格，可以使用订阅项本身的 `swap` 方法。如果你想保留订阅的其他价格上的所有现有元数据，这种方法特别有用：

```php
$user = User::find(1);

$user->subscription('default')
        ->findItemOrFail('price_basic')
        ->swap('price_pro');
```

#### 按比例计价

默认情况下，当在具有多个产品的订阅中添加或删除价格时，Stripe 会按比例计费。如果你想进行价格调整而不按比例计费，应在价格操作中链接 `noProrate` 方法：

```php
$user->subscription('default')->noProrate()->removePrice('price_chat');
```

#### 数量

如果你想在单个订阅价格上更新数量，可以使用[现有的数量方法](#subscription-quantity)，通过将价格的 ID 作为额外参数传递给方法：

```php
$user = User::find(1);

$user->subscription('default')->incrementQuantity(5, 'price_chat');

$user->subscription('default')->decrementQuantity(3, 'price_chat');

$user->subscription('default')->updateQuantity(10, 'price_chat');
```

> **警告**  
> 当订阅有多个价格时，`Subscription` 模型上的 `stripe_price` 和 `quantity` 属性将为 `null`。要访问各个价格属性，应使用 `Subscription` 模型上可用的 `items` 关系。

#### 订阅项

当订阅有多个价格时，它将在你的数据库的 `subscription_items` 表中存储多个订阅「项」。你可以通过订阅的 `items` 关系访问这些项：

```php
use App\Models\User;

$user = User::find(1);

$subscriptionItem = $user->subscription('default')->items->first();

// 检索特定项的 Stripe 价格和数量...
$stripePrice = $subscriptionItem->stripe_price;
$quantity = $subscriptionItem->quantity;
```

你也可以使用 `findItemOrFail` 方法检索特定的价格：

```php
$user = User::find(1);

$subscriptionItem = $user->subscription('default')->findItemOrFail('price_chat');
```

#### 多个订阅

Stripe 允许你的客户同时拥有多个订阅。例如，你可能经营一家健身房，提供游泳订阅和举重订阅，每个订阅可能有不同的定价。当然，客户应该能够订阅其中一个或两个计划。

当你的应用程序创建订阅时，可以向 `newSubscription` 方法提供订阅的类型。类型可以是表示用户正在启动的订阅类型的任何字符串：

```php
use Illuminate\Http\Request;

Route::post('/swimming/subscribe', function (Request $request) {
    $request->user()->newSubscription('swimming')
        ->price('price_swimming_monthly')
        ->create($request->paymentMethodId);

    // ...
});
```

在这个例子中，我们为客户启动了一个月度游泳订阅。然而，他们可能希望在以后切换到年度订阅。在调整客户的订阅时，我们可以简单地在 `swimming` 订阅上替换价格：

```php
$user->subscription('swimming')->swap('price_swimming_yearly');
```

当然，你也可以完全取消订阅：

```php
$user->subscription('swimming')->cancel();
```

#### 计量计费

[计量计费](https://stripe.com/docs/billing/subscriptions/metered-billing)允许你根据客户在计费周期内的产品使用量向他们收费。例如，你可以根据客户每月发送的短信或电子邮件数量来收费。

要开始使用计量计费，首先需要在 Stripe 仪表板中创建一个具有计量价格的新产品。然后，使用 `meteredPrice` 将计量价格 ID 添加到客户订阅中：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default')
        ->meteredPrice('price_metered')
        ->create($request->paymentMethodId);

    // ...
});
```

你也可以通过 [Stripe Checkout](#checkout) 来启动计量订阅：

```php
$checkout = Auth::user()
        ->newSubscription('default', [])
        ->meteredPrice('price_metered')
        ->checkout();

return view('your-checkout-view', [
    'checkout' => $checkout,
]);
```

#### 报告使用情况

当客户使用你的应用程序时，你需要向 Stripe 报告他们的使用情况，以便能够准确计费。要增加计量订阅的使用量，可以使用 `reportUsage` 方法：

```php
$user = User::find(1);

$user->subscription('default')->reportUsage();
```

默认情况下，会在计费周期中添加一个「使用数量」为 1。或者，你可以传递一个特定的「使用量」以添加到客户在计费周期内的使用量：

```php
$user = User::find(1);

$user->subscription('default')->reportUsage(15);
```

如果你的应用程序在单个订阅上提供多个价格，你需要使用 `reportUsageFor` 方法来指定要报告使用情况的计量价格：

```php
$user = User::find(1);

$user->subscription('default')->reportUsageFor('price_metered', 15);
```

有时，你可能需要更新先前报告的使用情况。为此，可以将时间戳或 `DateTimeInterface` 实例作为 `reportUsage` 的第二个参数传递。这样做时，Stripe 将更新在给定时间报告的使用情况。只要给定的日期和时间仍在当前计费周期内，就可以继续更新先前的使用记录：

```php
$user = User::find(1);

$user->subscription('default')->reportUsage(5, $timestamp);
```

#### 检索使用记录

要检索客户的过去使用情况，可以使用订阅实例的 `usageRecords` 方法：

```php
$user = User::find(1);

$usageRecords = $user->subscription('default')->usageRecords();
```

如果你的应用程序在单个订阅上提供多个价格，你可以使用 `usageRecordsFor` 方法来指定要检索使用记录的计量价格：

```php
$user = User::find(1);

$usageRecords = $user->subscription('default')->usageRecordsFor('price_metered');
```

`usageRecords` 和 `usageRecordsFor` 方法返回一个包含使用记录的关联数组的 Collection 实例。你可以遍历这个数组以显示客户的总使用量：

```php
@foreach ($usageRecords as $usageRecord)
    - Period Starting: {{ $usageRecord['period']['start'] }}
    - Period Ending: {{ $usageRecord['period']['end'] }}
    - Total Usage: {{ $usageRecord['total_usage'] }}
@endforeach
```

要查看返回的所有使用数据以及如何使用 Stripe 的基于游标的分页，请参考[官方 Stripe API 文档](https://stripe.com/docs/api/usage_records/subscription_item_summary_list)。

#### 订阅税费

> **警告**  
> 代替手动计算税率，你可以使用 [Stripe Tax 自动计算税费](#tax-configuration)

为了指定用户在订阅上支付的税率，你应该在可账单模型上实现 `taxRates` 方法，并返回一个包含 Stripe 税率 ID 的数组。你可以在 [Stripe 仪表板](https://dashboard.stripe.com/test/tax-rates) 中定义这些税率：

```php
/**
 * 用户订阅应用的税率。
  *
 * @return array<int, string>
 */
public function taxRates(): array
{
    return ['txr_id'];
}
```

`taxRates` 方法使你能够根据每个客户应用税率，这对于跨多个国家和税率的用户群可能很有帮助。

如果你提供具有多个产品的订阅，你可以通过在可账单模型上实现 `priceTaxRates` 方法来为每个价格定义不同的税率：

```php
/**
 * 用户订阅应用的税率。
  *
 * @return array<string, array<int, string>>
 */
public function priceTaxRates(): array
{
    return [
        'price_monthly' => ['txr_id'],
    ];
}
```

> **警告**  
> `taxRates` 方法仅适用于订阅费用。如果你使用 Cashier 进行「一次性」收费，你需要在那时手动指定税率。

#### 同步税率

当更改 `taxRates` 方法返回的硬编码税率 ID 时，用户现有订阅的税费设置将保持不变。如果你希望使用新的 `taxRates` 值更新现有订阅的税费值，你应该在用户的订阅实例上调用 `syncTaxRates` 方法：

```php
$user->subscription('default')->syncTaxRates();
```

这也将同步具有多个产品的订阅的任何项目税率。如果你的应用程序提供具有多个产品的订阅，你应该确保你的可账单模型实现了上文中讨论的 `priceTaxRates` 方法。

#### 免税

Cashier 还提供了 `isNotTaxExempt`、`isTaxExempt` 和 `reverseChargeApplies` 方法，用于确定客户是否免税。这些方法将调用 Stripe API 来确定客户的税收豁免状态：

```php
use App\Models\User;

$user = User::find(1);

$user->isTaxExempt();
$user->isNotTaxExempt();
$user->reverseChargeApplies();
```

> **警告**  
> 这些方法也适用于任何 `Laravel\Cashier\Invoice` 对象。但是，当在 `Invoice` 对象上调用时，这些方法将确定发票创建时的豁免状态。

#### 订阅锚定日期

默认情况下，计费周期锚点是订阅创建的日期，或者如果使用试用期，则是试用期结束的日期。如果你想修改计费锚定日期，可以使用 `anchorBillingCycleOn` 方法：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $anchor = Carbon::parse('first day of next month');

    $request->user()->newSubscription('default', 'price_monthly')
                ->anchorBillingCycleOn($anchor->startOfDay())
                ->create($request->paymentMethodId);

    // ...
});
```

要了解更多关于管理订阅计费周期的信息，请查阅 [Stripe 计费周期文档](https://stripe.com/docs/billing/subscriptions/billing-cycle)。

#### 取消订阅

要取消订阅，请在用户的订阅上调用 `cancel` 方法：

```php
$user->subscription('default')->cancel();
```

当取消订阅时，Cashier 会自动设置你的 `subscriptions` 数据库表中的 `ends_at` 列。这一列用于确定 `subscribed` 方法何时应开始返回 `false`。

例如，如果客户在 3 月 1 日取消订阅，但订阅直到 3 月 5 日才结束，`subscribed` 方法将继续返回 `true`，直到 3 月 5 日。这是因为通常允许用户在其计费周期结束前继续使用应用程序。

你可以使用 `onGracePeriod` 方法确定用户是否已取消订阅但仍处于「宽限期」：

```php
if ($user->subscription('default')->onGracePeriod()) {
    // ...
}
```

如果你希望立即取消订阅，请在用户的订阅上调用 `cancelNow` 方法：

```php
$user->subscription('default')->cancelNow();
```

如果你希望立即取消订阅并对任何剩余未计费的计量使用量或新的 / 待处理的调整发票项目进行结算，请在用户的订阅上调用 `cancelNowAndInvoice` 方法：

```php
$user->subscription('default')->cancelNowAndInvoice();
```

你也可以选择在特定时间取消订阅：

```php
$user->subscription('default')->cancelAt(
    now()->addDays(10)
);
```

最后，在删除相关的用户模型之前，你应该始终取消用户订阅：

```php
$user->subscription('default')->cancelNow();

$user->delete();
```

#### 恢复订阅

如果客户取消了他们的订阅，而你希望恢复它，你可以在订阅上调用 `resume` 方法。客户必须仍处于他们的「宽限期」内才能恢复订阅：

```php
$user->subscription('default')->resume();
```

如果客户取消了订阅，然后在订阅完全到期之前恢复了订阅，客户不会立即被收费。相反，他们的订阅将重新激活，并且他们将按照原始的计费周期进行收费。

## 订阅试用期

### 预先收集付款方式

如果你想向客户提供试用期，同时仍然在前期收集付款方式信息，你应该在创建订阅时使用 `trialDays` 方法：

```php
use Illuminate\Http\Request;

Route::post('/user/subscribe', function (Request $request) {
    $request->user()->newSubscription('default', 'price_monthly')
                ->trialDays(10)
                ->create($request->paymentMethodId);

    // ...
});
```

这个方法会在数据库中的订阅记录上设置试用期结束日期，并指示 Stripe 在此日期之后开始向客户收费。使用 `trialDays` 方法时，Cashier 会覆盖 Stripe 中为价格配置的任何默认试用期。

> **警告**  
> 如果客户的订阅在试用期结束日期之前未取消，他们将在试用期到期后立即收费，因此你应该确保通知用户他们的试用期结束日期。

`trialUntil` 方法允许你提供一个指定试用期应该结束的 `DateTime` 实例：

```php
use Carbon\Carbon;

$user->newSubscription('default', 'price_monthly')
            ->trialUntil(Carbon::now()->addDays(10))
            ->create($paymentMethod);
```

你可以使用用户实例的 `onTrial` 方法或订阅实例的 `onTrial` 方法来确定用户是否在他们的试用期内。下面的两个示例是等效的：

```php
if ($user->onTrial('default')) {
    // ...
}

if ($user->subscription('default')->onTrial()) {
    // ...
}
```

你可以使用 `endTrial` 方法立即结束订阅试用期：

```php
$user->subscription('default')->endTrial();
```

要确定现有试用期是否已过期，你可以使用 `hasExpiredTrial` 方法：

```php
if ($user->hasExpiredTrial('default')) {
    // ...
}

if ($user->subscription('default')->hasExpiredTrial()) {
    // ...
}
```

#### 在 Stripe / Cashier 中定义试用天数

你可以选择在 Stripe 仪表板中定义价格的试用天数，或者始终通过 Cashier 显式传递它们。如果你选择在 Stripe 中定义价格的试用天数，你应该知道新订阅，包括过去曾有订阅的客户的新订阅，将始终获得试用期，除非你显式调用 `skipTrial()` 方法。

### 无需预先收集付款方式

如果你想在不预先收集用户付款方式信息的情况下提供试用期，你可以将用户记录中的 `trial_ends_at` 列设置为你期望的试用结束日期。这通常在用户注册过程中完成：

```php
use App\Models\User;

$user = User::create([
    // ...
    'trial_ends_at' => now()->addDays(10),
]);
```

> **警告**  
> 请确保在你的可计费模型类定义中为 `trial_ends_at` 属性添加一个[日期转换](https://learnku.com/docs/laravel/11.x/eloquent-mutators#date-casting)。

Cashier 将这种类型的试用期称为「通用试用期」，因为它不附加到任何现有订阅。如果当前日期未超过 `trial_ends_at` 的值，可计费模型实例上的 `onTrial` 方法将返回 `true`：

```php
if ($user->onTrial()) {
    // 用户在试用期内...
}
```

当你准备为用户创建实际订阅时，你可以像往常一样使用 `newSubscription` 方法：

```php
$user = User::find(1);

$user->newSubscription('default', 'price_monthly')->create($paymentMethod);
```

要检索用户的试用结束日期，你可以使用 `trialEndsAt` 方法。如果用户正在试用中，该方法将返回一个 Carbon 日期实例，如果他们不在试用中，则返回 `null`。如果你想获取除默认订阅以外其他特定订阅的试用结束日期，你还可以传递一个可选的订阅类型参数：

```php
if ($user->onTrial()) {
    $trialEndsAt = $user->trialEndsAt('main');
}
```

如果你希望明确知道用户是否在他们的「通用」试用期内并且尚未创建实际订阅，你可以使用 `onGenericTrial` 方法：

```php
if ($user->onGenericTrial()) {
    // 用户在他们的「通用」试用期内...
}
```

### 延长试用期

`extendTrial` 方法允许你在创建订阅后延长订阅的试用期。如果试用期已经过期并且客户已经为订阅付费，你仍然可以为他们提供延长的试用期。试用期内的时间将从客户的下一个账单中扣除：

```php
use App\Models\User;

$subscription = User::find(1)->subscription('default');

// 从现在起延长试用期 7 天...
$subscription->extendTrial(
    now()->addDays(7)
);

// 在试用期上再增加 5 天...
$subscription->extendTrial(
    $subscription->trial_ends_at->addDays(5)
);
```

## 处理 Stripe Webhooks

> **注意**  
> 你可以使用 [Stripe CLI](https://stripe.com/docs/stripe-cli) 在本地开发过程中帮助测试 Webhooks。

Stripe 可以通过 Webhooks 通知你的应用程序各种事件。默认情况下，Cashier 服务提供程序会自动注册一个指向 Cashier 的 Webhook 控制器的路由。这个控制器将处理所有传入的 Webhook 请求。

默认情况下，Cashier 的 Webhook 控制器将自动处理取消订阅（由你的 Stripe 设置定义）次数过多的失败扣款、客户更新、客户删除、订阅更新和支付方式更改；然而，正如我们将很快发现的那样，你可以扩展这个控制器来处理任何你喜欢的 Stripe Webhook 事件。

为确保你的应用程序能够处理 Stripe Webhooks，请确保在 Stripe 控制面板中配置 Webhook URL。默认情况下，Cashier 的 Webhook 控制器响应于 `/stripe/webhook` URL 路径。在 Stripe 控制面板中应启用的所有 Webhooks 的完整列表包括：

+   `customer.subscription.created`
+   `customer.subscription.updated`
+   `customer.subscription.deleted`
+   `customer.updated`
+   `customer.deleted`
+   `payment_method.automatically_updated`
+   `invoice.payment_action_required`
+   `invoice.payment_succeeded`

为方便起见，Cashier 包含一个 `cashier:webhook` Artisan 命令。该命令将在 Stripe 中创建一个 Webhook，监听 Cashier 所需的所有事件：

```shell
php artisan cashier:webhook
```

默认情况下，创建的 Webhook 将指向由 `APP_URL` 环境变量和 Cashier 附带的 `cashier.webhook` 路由定义的 URL。如果你想使用不同的 URL，可以在调用命令时提供 `--url` 选项：

```shell
php artisan cashier:webhook --url "https://example.com/stripe/webhook"
```

创建的 Webhook 将使用与你的 Cashier 版本兼容的 Stripe API 版本。如果你想使用不同的 Stripe 版本，你可以提供 `--api-version` 选项：

```shell
php artisan cashier:webhook --api-version="2019-12-03"
```

创建后，Webhook 将立即生效。如果你希望创建 Webhook，但在准备好之前将其禁用，你可以在调用命令时提供 `--disabled` 选项：

```shell
php artisan cashier:webhook --disabled
```

> **警告**  
> 确保使用 Cashier 包含的 [Webhook 签名验证](#verifying-webhook-signatures) 中间件保护传入的 Stripe Webhook 请求。

#### Webhooks 和 CSRF 保护

由于 Stripe Webhooks 需要绕过 Laravel 的 [CSRF 保护](https://learnku.com/docs/laravel/11.x/csrf)，你应确保 Laravel 不会尝试验证传入的 Stripe Webhooks 的 CSRF 令牌。为此，你应在应用程序的 `bootstrap/app.php` 文件中排除 `stripe/*` 从 CSRF 保护中：

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(except: [
        'stripe/*',
    ]);
})
```

### 定义 Webhook 事件处理程序

Cashier 自动处理订阅取消、失败扣款和其他常见的 Stripe Webhook 事件。但是，如果你有其他 Webhook 事件需要处理，你可以通过监听 Cashier 分发的以下事件来处理：

+   `Laravel\Cashier\Events\WebhookReceived`
+   `Laravel\Cashier\Events\WebhookHandled`

这两个事件包含了完整的 Stripe Webhook 负载。例如，如果你希望处理 `invoice.payment_succeeded` Webhook，你可以注册一个 [监听器](https://learnku.com/docs/laravel/11.x/events#defining-listeners) 来处理该事件：

```php
<?php

namespace App\Listeners;

use Laravel\Cashier\Events\WebhookReceived;

class StripeEventListener
{
    /**
     * 处理接收到的 Stripe Webhooks。
  */
    public function handle(WebhookReceived $event): void
    {
        if ($event->payload['type'] === 'invoice.payment_succeeded') {
            // 处理传入的事件...
        }
    }
}
```

### 验证 Webhook 签名

为了保护你的 Webhooks，你可以使用 [Stripe 的 Webhook 签名](https://stripe.com/docs/webhooks/signatures)。为了方便起见，Cashier 自动包含了一个中间件，用于验证传入的 Stripe Webhook 请求是否有效。

要启用 Webhook 验证，请确保在你的应用程序的 `.env` 文件中设置了 `STRIPE_WEBHOOK_SECRET` 环境变量。Webhook 的 `secret` 可以从你的 Stripe 账户仪表板中获取。

## 单笔收费

### 简单收费

如果你想对客户进行一次性收费，你可以在可账单模型实例上使用 `charge` 方法。你需要将第二个参数作为 `charge` 方法的支付方法标识符：

```php
use Illuminate\Http\Request;

Route::post('/purchase', function (Request $request) {
    $stripeCharge = $request->user()->charge(
        100, $request->paymentMethodId
    );

    // ...
});
```

`charge` 方法接受一个数组作为其第三个参数，允许你传递任何你希望传递给底层 Stripe 收费创建的选项。有关在创建收费时可用的选项的更多信息，请参阅 [Stripe 文档](https://stripe.com/docs/api/charges/create)：

```php
$user->charge(100, $paymentMethod, [
    'custom_option' => $value,
]);
```

你也可以在没有底层客户或用户的情况下使用 `charge` 方法。为此，在你的应用程序的可账单模型的新实例上调用 `charge` 方法：

```php
use App\Models\User;

$stripeCharge = (new User)->charge(100, $paymentMethod);
```

如果收费失败，`charge` 方法将抛出异常。如果收费成功，方法将返回一个 `Laravel\Cashier\Payment` 的实例：

```php
try {
    $payment = $user->charge(100, $paymentMethod);
} catch (Exception $e) {
    // ...
}
```

> **警告**  
> `charge` 方法接受以你的应用程序使用的货币的最低单位为单位的支付金额。例如，如果客户以美元支付，金额应该以美分为单位指定。

### 带发票的收费

有时候你可能需要进行一次性收费并为客户提供 PDF 发票。`invoicePrice` 方法可以帮助你实现这一目的。例如，让我们为客户开具五件新衬衫的发票：

```php
$user->invoicePrice('price_tshirt', 5);
```

发票将立即从用户的默认支付方式中扣款。`invoicePrice` 方法还接受一个数组作为其第三个参数。这个数组包含发票项目的计费选项。方法接受的第四个参数也是一个数组，应该包含发票本身的计费选项：

```php
$user->invoicePrice('price_tshirt', 5, [
    'discounts' => [
        ['coupon' => 'SUMMER21SALE']
    ],
], [
    'default_tax_rates' => ['txr_id'],
]);
```

类似于 `invoicePrice`，你可以使用 `tabPrice` 方法通过将项目添加到客户的「标签」来为多个项目（每张发票最多 250 个项目）创建一次性收费，然后向客户开具发票。例如，我们可以为客户开具五件衬衫和两只杯子的发票：

```php
$user->tabPrice('price_tshirt', 5);
$user->tabPrice('price_mug', 2);
$user->invoice();
```

或者，你可以使用 `invoiceFor` 方法对客户的默认支付方式进行「一次性」收费：

```php
$user->invoiceFor('一次性费用', 500);
```

虽然 `invoiceFor` 方法可供使用，但建议你使用预定义价格的 `invoicePrice` 和 `tabPrice` 方法。这样做可以让你在 Stripe 仪表板中更好地了解关于每种产品销售的分析和数据。

> **警告**  
> `invoice`、`invoicePrice` 和 `invoiceFor` 方法将创建一个 Stripe 发票，该发票将重试失败的扣款尝试。如果你不希望发票重试失败的扣款，请在首次扣款失败后使用 Stripe API 关闭它们。

### 创建支付意图

你可以通过在可账单模型实例上调用 `pay` 方法来创建一个新的 Stripe 支付意图。调用此方法将创建一个包含在 `Laravel\Cashier\Payment` 实例中的支付意图：

```php
use Illuminate\Http\Request;

Route::post('/pay', function (Request $request) {
    $payment = $request->user()->pay(
        $request->get('amount')
    );

    return $payment->client_secret;
});
```

创建支付意图后，你可以将客户端密钥返回给你的应用前端，以便用户可以在他们的浏览器中完成支付。要了解有关使用 Stripe 支付意图构建整个支付流程的更多信息，请查阅 [Stripe 文档](https://stripe.com/docs/payments/accept-a-payment?platform=web)。

在使用 `pay` 方法时，默认启用的支付方式将在你的 Stripe 仪表板中对客户可用。或者，如果你只想允许使用一些特定的支付方式，你可以使用 `payWith` 方法：

```php
use Illuminate\Http\Request;

Route::post('/pay', function (Request $request) {
    $payment = $request->user()->payWith(
        $request->get('amount'), ['card', 'bancontact']
    );

    return $payment->client_secret;
});
```

> \[!WARNING\]  
> `pay` 和 `payWith` 方法接受以你的应用程序使用的货币的最低单位为单位的支付金额。例如，如果客户以美元支付，金额应该以美分为单位指定。

### 退款

如果你需要退款 Stripe 的一笔交易，你可以使用 `refund` 方法。该方法将接受 Stripe 的[支付意图 ID](#payment-methods-for-single-charges) 作为其第一个参数：

```php
$payment = $user->charge(100, $paymentMethodId);

$user->refund($payment->id);
```

## 发票

### 获取发票

你可以使用 `invoices` 方法轻松地获取可账单模型的发票数组。`invoices` 方法返回一个 `Laravel\Cashier\Invoice` 实例的集合：

```php
$invoices = $user->invoices();
```

如果你想在结果中包括待处理的发票，你可以使用 `invoicesIncludingPending` 方法：

```php
$invoices = $user->invoicesIncludingPending();
```

你可以使用 `findInvoice` 方法通过发票 ID 检索特定发票：

```php
$invoice = $user->findInvoice($invoiceId);
```

#### 显示发票信息

在为客户列出发票时，你可以使用发票的方法来显示相关的发票信息。例如，你可以希望在表格中列出每张发票，让用户可以轻松下载其中任何一张：

```php
<table>
    @foreach ($invoices as $invoice)
        <tr>
            <td>{{ $invoice->date()->toFormattedDateString() }}</td>
            <td>{{ $invoice->total() }}</td>
            <td><a href="/user/invoice/{{ $invoice->id }}">下载</a></td>
        </tr>
    @endforeach
</table>
```

### 即将到期的发票

要获取客户的即将到期的发票，你可以使用 `upcomingInvoice` 方法：

```php
$invoice = $user->upcomingInvoice();
```

类似地，如果客户有多个订阅，你也可以为特定订阅获取即将到期的发票：

```php
$invoice = $user->subscription('default')->upcomingInvoice();
```

### 预览订阅发票

使用 `previewInvoice` 方法，你可以在进行价格更改之前预览发票。这将允许你确定当进行特定价格更改时，客户的发票会是什么样子：

```php
$invoice = $user->subscription('default')->previewInvoice('price_yearly');
```

你可以向 `previewInvoice` 方法传递价格数组，以便预览具有多个新价格的发票：

```php
$invoice = $user->subscription('default')->previewInvoice(['price_yearly', 'price_metered']);
```

### 生成发票 PDF

在生成发票 PDF 之前，你应该使用 Composer 安装 Dompdf 库，这是 Cashier 的默认发票渲染器：

```php
composer require dompdf/dompdf
```

在路由或控制器中，你可以使用 `downloadInvoice` 方法来生成给定发票的 PDF 下载。这个方法将自动生成下载发票所需的正确 HTTP 响应：

```php
use Illuminate\Http\Request;

Route::get('/user/invoice/{invoice}', function (Request $request, string $invoiceId) {
    return $request->user()->downloadInvoice($invoiceId);
});
```

默认情况下，发票上的所有数据都来自于 Stripe 中存储的客户和发票数据。文件名基于你的 `app.name` 配置值。但是，你可以通过将数组作为 `downloadInvoice` 方法的第二个参数来自定义部分数据。这个数组允许你自定义诸如公司和产品详细信息之类的信息：

```php
return $request->user()->downloadInvoice($invoiceId, [
    'vendor' => 'Your Company',
    'product' => 'Your Product',
    'street' => 'Main Str. 1',
    'location' => '2000 Antwerp, Belgium',
    'phone' => '+32 499 00 00 00',
    'email' => 'info@example.com',
    'url' => 'https://example.com',
    'vendorVat' => 'BE123456789',
]);
```

`downloadInvoice` 方法还允许通过第三个参数设置自定义文件名。这个文件名将自动添加 `.pdf` 后缀：

```php
return $request->user()->downloadInvoice($invoiceId, [], 'my-invoice');
```

#### 自定义发票渲染器

Cashier 还可以使用自定义发票渲染器。默认情况下，Cashier 使用 `DompdfInvoiceRenderer` 实现，它利用 [dompdf](https://github.com/dompdf/dompdf) PHP 库来生成 Cashier 的发票。但是，你可以通过实现 `Laravel\Cashier\Contracts\InvoiceRenderer` 接口来使用任何你希望的渲染器。例如，你可以希望使用 API 调用到第三方 PDF 渲染服务来渲染发票 PDF：

```php
use Illuminate\Support\Facades\Http;
use Laravel\Cashier\Contracts\InvoiceRenderer;
use Laravel\Cashier\Invoice;

class ApiInvoiceRenderer implements InvoiceRenderer
{
    /**
     * 渲染给定的发票并返回原始的PDF字节。
  */
    public function render(Invoice $invoice, array $data = [], array $options = []): string
    {
        $html = $invoice->view($data)->render();

        return Http::get('https://example.com/html-to-pdf', ['html' => $html])->body();
    }
}
```

一旦你实现了发票渲染器合同，你应该在你的应用程序的 `config/cashier.php` 配置文件中更新 `cashier.invoices.renderer` 配置值。这个配置值应该设置为你自定义渲染器实现的类名。

## 结账

Cashier Stripe 还支持 [Stripe Checkout](https://stripe.com/payments/checkout)。Stripe Checkout 通过提供一个预构建的托管支付页面，简化了实现自定义接受支付页面的痛苦。

以下文档包含了如何开始在 Cashier 中使用 Stripe Checkout 的信息。要了解更多关于 Stripe Checkout 的信息，你还应考虑查看 [Stripe 自己的 Checkout 文档](https://stripe.com/docs/payments/checkout)。

### 产品结账

你可以在可账单模型上使用 `checkout` 方法为已在你的 Stripe 仪表板中创建的现有产品执行结账。`checkout` 方法将启动一个新的 Stripe Checkout 会话。默认情况下，你需要传递一个 Stripe 价格 ID：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout('price_tshirt');
});
```

如果需要，你也可以指定产品数量：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 15]);
});
```

当客户访问此路由时，他们将被重定向到 Stripe 的结账页面。默认情况下，当用户成功完成或取消购买时，他们将被重定向到你的 `home` 路由位置，但你可以使用 `success_url` 和 `cancel_url` 选项指定自定义回调 URL：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 1], [
        'success_url' => route('your-success-route'),
        'cancel_url' => route('your-cancel-route'),
    ]);
});
```

在定义你的 `success_url` 结账选项时，你可以指示 Stripe 在调用 URL 时将结账会话 ID 作为查询字符串参数添加。为此，将字面字符串 `{CHECKOUT_SESSION_ID}` 添加到你的 `success_url` 查询字符串中。Stripe 将用实际的结账会话 ID 替换这个占位符：

```php
use Illuminate\Http\Request;
use Stripe\Checkout\Session;
use Stripe\Customer;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()->checkout(['price_tshirt' => 1], [
        'success_url' => route('checkout-success').'?session_id={CHECKOUT_SESSION_ID}',
        'cancel_url' => route('checkout-cancel'),
    ]);
});

Route::get('/checkout-success', function (Request $request) {
    $checkoutSession = $request->user()->stripe()->checkout->sessions->retrieve($request->get('session_id'));

    return view('checkout.success', ['checkoutSession' => $checkoutSession]);
})->name('checkout-success');
```

#### 促销代码

默认情况下，Stripe Checkout 不允许[用户可兑换的促销代码](https://stripe.com/docs/billing/subscriptions/discounts/codes)。幸运的是，有一种简单的方法可以为你的结账页面启用这些促销代码。为此，你可以调用 `allowPromotionCodes` 方法：

```php
use Illuminate\Http\Request;

Route::get('/product-checkout', function (Request $request) {
    return $request->user()
        ->allowPromotionCodes()
        ->checkout('price_tshirt');
});
```

### 单次收费结账

你还可以为未在 Stripe 仪表板中创建的临时产品执行简单的收费。为此，你可以在可账单模型上使用 `checkoutCharge` 方法，并传递一个可收费金额、产品名称和可选数量。当客户访问此路由时，他们将被重定向到 Stripe 的结账页面：

```php
use Illuminate\Http\Request;

Route::get('/charge-checkout', function (Request $request) {
    return $request->user()->checkoutCharge(1200, 'T-Shirt', 5);
});
```

> **警告**  
> 使用 `checkoutCharge` 方法时，Stripe 将始终在你的 Stripe 仪表板中创建一个新的产品和价格。因此，我们建议你提前在 Stripe 仪表板中创建产品，并改用 `checkout` 方法。

### 订阅结账

> **警告**  
> 使用 Stripe Checkout 进行订阅需要在 Stripe 仪表板中启用 `customer.subscription.created` Webhook。该 Webhook 将在你的数据库中创建订阅记录并存储所有相关的订阅项目。

你也可以使用 Stripe Checkout 来启动订阅。在使用 Cashier 的订阅构建器方法定义订阅后，你可以调用 `checkout` 方法。当客户访问此路由时，他们将被重定向到 Stripe 的结账页面：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_monthly')
        ->checkout();
});
```

与产品结账一样，你可以自定义成功和取消的 URL：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_monthly')
        ->checkout([
            'success_url' => route('your-success-route'),
            'cancel_url' => route('your-cancel-route'),
        ]);
});
```

当然，你也可以为订阅结账启用促销代码：

```php
use Illuminate\Http\Request;

Route::get('/subscription-checkout', function (Request $request) {
    return $request->user()
        ->newSubscription('default', 'price_monthly')
        ->allowPromotionCodes()
        ->checkout();
});
```

> **警告**  
> 不幸的是，当开始订阅时，Stripe Checkout 不支持所有订阅计费选项。在 Stripe Checkout 会话期间，使用订阅构建器上的 `anchorBillingCycleOn` 方法、设置按比例计价行为或设置付款行为都不会产生任何效果。请参阅 [Stripe Checkout 会话 API 文档](https://stripe.com/docs/api/checkout/sessions/create)以查看可用的参数。

#### Stripe Checkout 和试用期

当构建使用 Stripe Checkout 完成的订阅时，你当然可以定义一个试用期：

```php
$checkout = Auth::user()->newSubscription('default', 'price_monthly')
    ->trialDays(3)
    ->checkout();
```

然而，试用期必须至少为 48 小时，这是 Stripe Checkout 支持的最短试用时间。

#### 订阅和 Webhooks

请记住，Stripe 和 Cashier 通过 Webhooks 更新订阅状态，因此当客户在输入付款信息后返回应用程序时，可能会出现订阅尚未激活的情况。为了处理这种情况，你可能希望显示一条消息，告知用户他们的付款或订阅正在等待处理。

### 收集税号

Checkout 还支持收集客户的税号。要在结账会话上启用此功能，请在创建会话时调用 `collectTaxIds` 方法：

```php
$checkout = $user->collectTaxIds()->checkout('price_tshirt');
```

当调用此方法时，客户将可以看到一个新复选框，允许他们指示是否以公司名义购买。如果是，他们将有机会提供他们的税号。

> **警告**  
> 如果你已经在应用程序的服务提供者中配置了[自动税收](#tax-configuration)，那么此功能将自动启用，无需调用 `collectTaxIds` 方法。

### 访客结账

使用 `Checkout::guest` 方法，你可以为你的应用程序中没有「账户」的访客启动结账会话：

```php
use Illuminate\Http\Request;
use Laravel\Cashier\Checkout;

Route::get('/product-checkout', function (Request $request) {
    return Checkout::guest()->create('price_tshirt', [
        'success_url' => route('your-success-route'),
        'cancel_url' => route('your-cancel-route'),
    ]);
});
```

类似于为现有用户创建结账会话时，你可以利用 `Laravel\Cashier\CheckoutBuilder` 实例上提供的其他方法来自定义访客结账会话：

```php
use Illuminate\Http\Request;
use Laravel\Cashier\Checkout;

Route::get('/product-checkout', function (Request $request) {
    return Checkout::guest()
        ->withPromotionCode('promo-code')
        ->create('price_tshirt', [
            'success_url' => route('your-success-route'),
            'cancel_url' => route('your-cancel-route'),
        ]);
});
```

完成访客结账后，Stripe 可以发送一个 `checkout.session.completed` 的 Webhook 事件，所以确保[配置你的 Stripe Webhook](https://dashboard.stripe.com/webhooks) 以确实将此事件发送到你的应用程序。一旦在 Stripe 仪表板中启用了 Webhook，你可以[使用 Cashier 处理 Webhook](#handling-stripe-webhooks)。Webhook 负载中包含的对象将是一个 [`checkout` 对象](https://stripe.com/docs/api/checkout/sessions/object)，你可以检查该对象以完成客户的订单。

## 处理失败支付

有时，订阅或单笔付款可能会失败。当发生这种情况时，Cashier 将抛出一个 `Laravel\Cashier\Exceptions\IncompletePayment` 异常，通知你发生了这种情况。在捕获此异常后，你有两种选择如何继续。

首先，你可以将客户重定向到 Cashier 中包含的专用支付确认页面。该页面已经有一个通过 Cashier 的服务提供者注册的关联命名路由。因此，你可以捕获 `IncompletePayment` 异常并将用户重定向到支付确认页面：

```php
use Laravel\Cashier\Exceptions\IncompletePayment;

try {
    $subscription = $user->newSubscription('default', 'price_monthly')
                            ->create($paymentMethod);
} catch (IncompletePayment $exception) {
    return redirect()->route(
        'cashier.payment',
        [$exception->payment->id, 'redirect' => route('home')]
    );
}
```

在支付确认页面上，客户将被提示重新输入他们的信用卡信息，并执行 Stripe 要求的任何其他操作，比如「3D Secure」确认。确认支付后，用户将被重定向到上面指定的 `redirect` 参数提供的 URL。重定向时，将向 URL 添加 `message`（字符串）和 `success`（整数）查询字符串变量。当前支付页面支持以下支付方式类型：

+   信用卡
+   支付宝
+   Bancontact
+   BECS 直接借记
+   EPS
+   Giropay
+   iDEAL
+   SEPA 直接借记

或者，你可以允许 Stripe 为你处理支付确认。在这种情况下，你可以在 Stripe 仪表板中[设置 Stripe 的自动账单邮件](https://dashboard.stripe.com/account/billing/automatic)，而不是重定向到支付确认页面。然而，如果捕获到 `IncompletePayment` 异常，你仍应通知用户他们将收到一封包含进一步支付确认说明的电子邮件。

以下方法可能会抛出支付异常：在使用 `Billable` 特性的模型上的 `charge`、`invoiceFor` 和 `invoice` 方法。在与订阅交互时，`SubscriptionBuilder` 上的 `create` 方法，以及 `Subscription` 和 `SubscriptionItem` 模型上的 `incrementAndInvoice` 和 `swapAndInvoice` 方法可能会抛出不完整支付异常。

可以使用可计费模型或订阅实例上的 `hasIncompletePayment` 方法来确定现有订阅是否存在不完整支付：

```php
if ($user->hasIncompletePayment('default')) {
    // ...
}

if ($user->subscription('default')->hasIncompletePayment()) {
    // ...
}
```

你可以通过检查异常实例上的 `payment` 属性来推导不完整支付的具体状态：

```php
use Laravel\Cashier\Exceptions\IncompletePayment;

try {
    $user->charge(1000, 'pm_card_threeDSecure2Required');
} catch (IncompletePayment $exception) {
    // 获取支付意图状态...
    $exception->payment->status;

    // 检查特定条件...
    if ($exception->payment->requiresPaymentMethod()) {
        // ...
    } elseif ($exception->payment->requiresConfirmation()) {
        // ...
    }
}
```

### 确认支付

某些支付方法需要额外数据才能确认支付。例如，SEPA 支付方法在支付过程中需要额外的「授权」数据。你可以使用 `withPaymentConfirmationOptions` 方法将这些数据提供给 Cashier：

```php
$subscription->withPaymentConfirmationOptions([
    'mandate_data' => '...',
])->swap('price_xxx');
```

你可以查阅 [Stripe API 文档](https://stripe.com/docs/api/payment_intents/confirm)以查看确认支付时接受的所有选项。

## 强制客户认证

如果你的业务或你的某位客户位于欧洲，你将需要遵守欧盟的强制客户认证（SCA）规定。这些规定是欧盟于 2019 年 9 月实施的，旨在防止支付欺诈。幸运的是，Stripe 和 Cashier 已经为构建符合 SCA 标准的应用程序做好了准备。

> **警告**  
> 在开始之前，请查阅 [Stripe 关于 PSD2 和 SCA 的指南](https://stripe.com/guides/strong-customer-authentication)以及他们关于新 SCA API 的[文档](https://stripe.com/docs/strong-customer-authentication)。

### 需要额外确认的支付

SCA 规定通常需要额外的验证来确认和处理支付。当发生这种情况时，Cashier 将抛出一个 `Laravel\Cashier\Exceptions\IncompletePayment` 异常，通知你需要额外验证。如何处理这些异常的更多信息可以在[处理失败支付](#handling-failed-payments)的文档中找到。

由 Stripe 或 Cashier 呈现的支付确认屏幕可能会针对特定银行或卡发行商的支付流程进行定制，并可能包括额外的卡确认、临时小额扣款、单独的设备验证或其他形式的验证。

#### 不完整和过期状态

当支付需要额外确认时，订阅将保持在 `不完整（incomplete）` 或 `过期（past_due）` 状态，如其 `stripe_status` 数据库列所示。一旦支付确认完成，并且你的应用程序通过 Stripe 的 Webhook 收到通知，Cashier 将自动激活客户的订阅。

有关 `不完整（incomplete）` 和 `过期（past_due）` 状态的更多信息，请参考我们关于这些状态的[额外文档](#incomplete-and-past-due-status)。

### 离线支付通知

由于 SCA 规定要求客户偶尔验证他们的付款细节，即使他们的订阅仍然有效，Cashier 可以在需要离线支付确认时向客户发送通知。例如，当订阅续订时可能会发生这种情况。通过将 `CASHIER_PAYMENT_NOTIFICATION` 环境变量设置为通知类，可以启用 Cashier 的付款通知。默认情况下，此通知被禁用。当然，Cashier 包含一个你可以用于此目的的通知类，但如果需要，你可以自由提供自己的通知类：

```ini
CASHIER_PAYMENT_NOTIFICATION=Laravel\Cashier\Notifications\ConfirmPayment
```

为确保离线支付确认通知被传递，请验证为你的应用程序配置了 [Stripe Webhooks](#handling-stripe-webhooks)，并且在你的 Stripe 仪表板中启用了 `invoice.payment_action_required` Webhook。此外，你的 `Billable` 模型还应该使用 Laravel 的 `Illuminate\Notifications\Notifiable` 特性。

> **警告**  
> 即使客户手动进行需要额外确认的支付，通知也会被发送。不幸的是，Stripe 无法知道支付是手动完成还是「离线」完成。但是，如果客户在确认支付后访问付款页面，他们将简单地看到「支付成功」消息。客户不会被允许意外确认相同的付款两次并产生意外的第二笔费用。

## Stripe SDK

Cashier 的许多对象都是 Stripe SDK 对象的封装。如果你想直接与 Stripe 对象交互，你可以使用 `asStripe` 方法方便地检索它们：

```php
$stripeSubscription = $subscription->asStripeSubscription();

$stripeSubscription->application_fee_percent = 5;

$stripeSubscription->save();
```

你还可以使用 `updateStripeSubscription` 方法直接更新 Stripe 订阅：

```php
$subscription->updateStripeSubscription(['application_fee_percent' => 5]);
```

如果你想直接使用 `Stripe\StripeClient` 客户端，你可以在 `Cashier` 类上调用 `stripe` 方法。例如，你可以使用这个方法访问 `StripeClient` 实例，并从你的 Stripe 账户中检索价格列表：

```php
use Laravel\Cashier\Cashier;

$prices = Cashier::stripe()->prices->all();
```

## 测试

在测试使用 Cashier 的应用程序时，你可以模拟对 Stripe API 的实际 HTTP 请求；但是，这需要你部分重新实现 Cashier 的行为。因此，我们建议允许你的测试访问实际的 Stripe API。虽然这会更慢一些，但可以更有信心地确保你的应用程序按预期工作，任何慢速测试可以放在它们自己的 Pest / PHPUnit 测试组中。

在测试时，请记住 Cashier 本身已经有一个很好的测试套件，因此你应该专注于测试你自己应用程序的订阅和付款流程，而不是每个底层 Cashier 行为。

要开始，将你的 Stripe 测试密钥的**测试**版本添加到你的 `phpunit.xml` 文件中：

```xml
<env name="STRIPE_SECRET" value="sk_test_<your-key>"/>
```

现在，每当你在测试中与 Cashier 交互时，它将向你的 Stripe 测试环境发送实际的 API 请求。为方便起见，你应该在 Stripe 测试账户中预先填充订阅 / 价格，以便在测试中使用。

> **注意**  
> 为了测试各种计费场景，比如信用卡拒绝和失败，你可以使用 Stripe 提供的广泛范围的[测试卡号和令牌](https://stripe.com/docs/testing)。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/bi...](https://learnku.com/docs/laravel/11.x/billingmd/16715)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/bi...](https://learnku.com/docs/laravel/11.x/billingmd/16715)