本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Precognition

+   [介绍](#introduction)
+   [实时验证](#live-validation)
    +   [使用 Vue](#using-vue)
    +   [使用 Vue 和 Inertia](#using-vue-and-inertia)
    +   [使用 React](#using-react)
    +   [使用 React 和 Inertia](#using-react-and-inertia)
    +   [使用 Alpine 和 Blade](#using-alpine)
    +   [配置 Axios](#configuring-axios)
+   [自定义验证规则](#customizing-validation-rules)
+   [处理文件上传](#handling-file-uploads)
+   [管理副作用](#managing-side-effects)
+   [测试](#testing)

## 介绍

Laravel Precognition 允许您预测未来的 HTTP 请求结果。Precognition 的主要用例之一是为您的前端 JavaScript 应用提供 “实时” 验证，而无需复制应用的后端验证规则。Precognition 特别搭配 Laravel 基于 Inertia 的[入门套件](https://learnku.com/docs/laravel/11.x/starter-kits)使用效果更佳。

当 Laravel 收到 “预知请求” 时，它将执行所有路由的中间件并解析路由的控制器依赖项，包括验证[表单请求](https://learnku.com/docs/laravel/11.x/validation#form-request-validation) - 但不会实际执行路由的控制器方法。

## 实时验证

### 使用 Vue

使用 Laravel Precognition，您可以为用户提供实时验证体验，而无需在前端 Vue 应用中复制验证规则。为了说明它的工作原理，让我们在应用程序中构建一个用于创建新用户的表单。

首先，要为路由启用 Precognition，应在路由定义中添加 `HandlePrecognitiveRequests` 中间件。您还应创建一个[表单请求](https://learnku.com/docs/laravel/11.x/validation#form-request-validation)来存放路由的验证规则：

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下来，你应该通过 NPM 安装用于 Vue 的 Laravel Precognition 前端辅助工具：

```shell
npm install laravel-precognition-vue
```

安装了 Laravel Precognition 包后，你现在可以使用 Precognition 的 `useForm` 函数创建一个表单对象，提供 HTTP 方法（`post`）、目标 URL（`/users`）和初始表单数据。

然后，为了启用实时验证，在每个输入的 `change` 事件上调用表单的 `validate` 方法，提供输入的名称：

```vue
<script setup>
import { useForm } from 'laravel-precognition-vue';

const form = useForm('post', '/users', {
    name: '',
    email: '',
});

const submit = () => form.submit();
</script>

<template>
    <form @submit.prevent="submit">
        <label for="name">姓名</label>
        <input
            id="name"
            v-model="form.name"
            @change="form.validate('name')"
        />
        <div v-if="form.invalid('name')">
            {{ form.errors.name }}
        </div>

        <label for="email">邮箱</label>
        <input
            id="email"
            type="email"
            v-model="form.email"
            @change="form.validate('email')"
        />
        <div v-if="form.invalid('email')">
            {{ form.errors.email }}
        </div>

        <button :disabled="form.processing">
            创建用户
        </button>
    </form>
</template>
```

现在，当用户填写表单时，Precognition 将提供由路由表单请求中的验证规则驱动的实时验证输出。当表单的输入发生更改时，将发送一个经过防抖处理的 “预知” 验证请求到你的 Laravel 应用程序。你可以通过调用表单的 `setValidationTimeout` 函数来配置防抖超时时间：

```js
form.setValidationTimeout(3000);
```

在验证请求进行中时，表单的 `validating` 属性将为 `true`：

```html
<div v-if="form.validating">
    验证中...
</div>
```

在验证请求或表单提交时返回的任何验证错误将自动填充表单的 `errors` 对象：

```html
<div v-if="form.invalid('email')">
    {{ form.errors.email }}
</div>
```

你可以使用表单的 `hasErrors` 属性确定表单是否存在任何错误：

```html
<div v-if="form.hasErrors">
    <!-- ... -->
</div>
```

你还可以通过将输入的名称传递给表单的 `valid` 和 `invalid` 函数来确定输入是否通过或未通过验证：

```html
<span v-if="form.valid('email')">
    ✅
</span>

<span v-else-if="form.invalid('email')">
    ❌
</span>
```

> \[!WARNING\]  
> 仅当输入发生更改并接收到验证响应时，表单输入才会显示为有效或无效。

如果你正在使用 Precognition 验证表单的部分输入，手动清除错误可能很有用。你可以使用表单的 `forgetError` 函数来实现：

```html
<input
    id="avatar"
    type="file"
    @change="(e) => {
        form.avatar = e.target.files[0]

        form.forgetError('avatar')
    }"
>
```

当然，你也可以在对表单提交的响应中执行代码。表单的 `submit` 函数返回一个 Axios 请求 promise。这提供了一个方便的方式来访问响应数据，成功提交后重置表单输入，或处理请求失败的情况：

```js
const submit = () => form.submit()
    .then(response => {
        form.reset();

        alert('用户已创建。');
    })
    .catch(error => {
        alert('发生错误。');
    });
```

你可以通过检查表单的 `processing` 属性来确定表单提交请求是否正在进行中：

```html
<button :disabled="form.processing">
    提交
</button>
```

### 使用 Vue 和 Inertia

> **注意**  
> 如果你想在使用 Vue 和 Inertia 开发 Laravel 应用程序时快速入门，请考虑使用我们的[入门套件](https://learnku.com/docs/laravel/11.x/starter-kits)之一。Laravel 的入门套件为你的新 Laravel 应用程序提供了后端和前端身份验证脚手架。

在使用 Vue 和 Inertia 之前，请务必查阅我们关于[在 Vue 中使用 Precognition](#using-vue) 的通用文档。在使用 Vue 和 Inertia 时，你需要通过 NPM 安装兼容 Inertia 的 Precognition 库：

```shell
npm install laravel-precognition-vue-inertia
```

安装完成后，Precognition 的 `useForm` 函数将返回一个经过增强的 Inertia [表单助手](https://inertiajs.com/forms#form-helper)，具有上述讨论的验证功能。

表单助手的 `submit` 方法已经简化，无需指定 HTTP 方法或 URL。相反，你可以将 Inertia 的[访问选项](https://inertiajs.com/manual-visits)作为第一个且唯一的参数传递。此外，`submit` 方法不像上面的 Vue 示例中返回一个 Promise。相反，在传递给 `submit` 方法的访问选项中，你可以提供任何 Inertia 支持的[事件回调](https://inertiajs.com/manual-visits#event-callbacks)：

```vue
<script setup>
import { useForm } from 'laravel-precognition-vue-inertia';

const form = useForm('post', '/users', {
    name: '',
    email: '',
});

const submit = () => form.submit({
    preserveScroll: true,
    onSuccess: () => form.reset(),
});
</script>
```

### 使用 React

使用 Laravel Precognition，你可以为用户提供实时验证体验，而无需在前端 React 应用程序中重复验证规则。为了说明它是如何工作的，让我们构建一个用于在应用程序中创建新用户的表单。

首先，为了在路由上启用 Precognition，应该将 `HandlePrecognitiveRequests` 中间件添加到路由定义中。你还应该创建一个[表单请求](https://learnku.com/docs/laravel/11.x/validation#form-request-validation)来存放路由的验证规则：

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下来，你应该通过 NPM 安装用于 React 的 Laravel Precognition 前端辅助工具：

```shell
npm install laravel-precognition-react
```

安装了 Laravel Precognition 包之后，你现在可以使用 Precognition 的 `useForm` 函数创建一个表单对象，提供 HTTP 方法（`post`）、目标 URL（`/users`）和初始表单数据。

为了启用实时验证，你应该监听每个输入的 `change` 和 `blur` 事件。在 `change` 事件处理程序中，你应该使用 `setData` 函数设置表单数据，传递输入的名称和新值。然后，在 `blur` 事件处理程序中调用表单的 `validate` 方法，提供输入的名称：

```js
import { useForm } from 'laravel-precognition-react';

export default function Form() {
    const form = useForm('post', '/users', {
        name: '',
        email: '',
    });

    const submit = (e) => {
        e.preventDefault();

        form.submit();
    };

    return (
        <form onSubmit={submit}>
            <label for="name">Name</label>
            <input
                id="name"
                value={form.data.name}
                onChange={(e) => form.setData('name', e.target.value)}
                onBlur={() => form.validate('name')}
            />
            {form.invalid('name') && <div>{form.errors.name}</div>}

            <label for="email">Email</label>
            <input
                id="email"
                value={form.data.email}
                onChange={(e) => form.setData('email', e.target.value)}
                onBlur={() => form.validate('email')}
            />
            {form.invalid('email') && <div>{form.errors.email}</div>}

            <button disabled={form.processing}>
                创建用户
            </button>
        </form>
    );
};
```

现在，当用户填写表单时，Precognition 将根据路由的表单请求中的验证规则提供实时验证输出。当表单输入发生更改时，将发送一个经过节流处理的 “先知” 验证请求到你的 Laravel 应用程序。你可以通过调用表单的 `setValidationTimeout` 函数来配置节流超时时间：

```js
form.setValidationTimeout(3000);
```

当验证请求正在进行时，表单的 `validating` 属性将为 `true`：

```js
{form.validating && <div>验证中...</div>}
```

在验证请求或表单提交期间返回的任何验证错误将自动填充表单的 `errors` 对象：

```js
{form.invalid('email') && <div>{form.errors.email}</div>}
```

你可以使用表单的 `hasErrors` 属性确定表单是否有任何错误：

```js
{form.hasErrors && <div><!-- ... --></div>}
```

你还可以通过分别将输入的名称传递给表单的 `valid` 和 `invalid` 函数来确定输入是否通过或未通过验证：

```js
{form.valid('email') && <span>✅</span>}

{form.invalid('email') && <span>❌</span>}
```

> **警告**  
> 只有在输入发生更改并接收到验证响应后，表单输入才会显示为有效或无效。

如果你正在使用 Precognition 验证表单的一部分输入，手动清除错误可能会很有用。你可以使用表单的 `forgetError` 函数来实现：

```js
<input
    id="avatar"
    type="file"
    onChange={(e) => 
        form.setData('avatar', e.target.value);

        form.forgetError('avatar');
    }
>
```

当然，你还可以根据表单提交的响应执行代码。表单的 `submit` 函数返回一个 Axios 请求 Promise。这提供了一种方便的方式来访问响应有效负载，在成功的表单提交时重置表单输入，或处理失败的请求：

```js
const submit = (e) => {
    e.preventDefault();

    form.submit()
        .then(response => {
            form.reset();

            alert('用户已创建。');
        })
        .catch(error => {
            alert('发生错误。');
        });
};
```

你可以通过检查表单的 `processing` 属性来确定表单提交请求是否正在进行中：

```html
<button disabled={form.processing}>
    提交
</button>
```

### 使用 React 和 Inertia

> **注意**  
> 如果你希望在使用 React 和 Inertia 开发 Laravel 应用程序时快速入门，请考虑使用我们的[入门套件](https://learnku.com/docs/laravel/11.x/starter-kits)之一。Laravel 的入门套件为你的新 Laravel 应用程序提供了后端和前端身份验证脚手架。

在使用 React 和 Inertia 时，请确保先阅读我们关于[在 React 中使用 Precognition](#using-react) 的一般文档。在使用 React 和 Inertia 时，你需要通过 NPM 安装与 Inertia 兼容的 Precognition 库：

```shell
npm install laravel-precognition-react-inertia
```

安装完成后，Precognition 的 `useForm` 函数将返回一个已增强了验证功能的 Inertia [表单助手](https://inertiajs.com/forms#form-helper)。

表单助手的 `submit` 方法已经简化，无需指定 HTTP 方法或 URL。相反，你可以将 Inertia 的[访问选项](https://inertiajs.com/manual-visits)作为第一个且唯一的参数传递。此外，与上述 React 示例中返回 Promise 不同，`submit` 方法不返回 Promise。相反，你可以在传递给 `submit` 方法的访问选项中提供任何受支持的 Inertia [事件回调](https://inertiajs.com/manual-visits#event-callbacks)：

```js
import { useForm } from 'laravel-precognition-react-inertia';

const form = useForm('post', '/users', {
    name: '',
    email: '',
});

const submit = (e) => {
    e.preventDefault();

    form.submit({
        preserveScroll: true,
        onSuccess: () => form.reset(),
    });
};
```

### 使用 Alpine 和 Blade

使用 Laravel Precognition，你可以为用户提供实时验证体验，而无需在前端 Alpine 应用程序中重复验证规则。为了说明它是如何工作的，让我们构建一个用于在应用程序中创建新用户的表单。

首先，要为路由启用 Precognition，应将 `HandlePrecognitiveRequests` 中间件添加到路由定义中。你还应创建一个[表单请求](https://learnku.com/docs/laravel/11.x/validation#form-request-validation)来存放路由的验证规则：

```php
use App\Http\Requests\CreateUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (CreateUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

接下来，你应该通过 NPM 安装用于 Alpine 的 Laravel Precognition 前端助手：

```shell
npm install laravel-precognition-alpine
```

然后，在你的 `resources/js/app.js` 文件中注册 Alpine 中的 Precognition 插件：

```js
import Alpine from 'alpinejs';
import Precognition from 'laravel-precognition-alpine';

window.Alpine = Alpine;

Alpine.plugin(Precognition);
Alpine.start();
```

安装并注册了 Laravel Precognition 包之后，你现在可以使用 Precognition 的 `$form` "魔术" 来创建一个表单对象，提供 HTTP 方法 (`post`)、目标 URL (`/users`) 和初始表单数据。

为了启用实时验证，你应将表单数据绑定到相关输入，并监听每个输入的 `change` 事件。在 `change` 事件处理程序中，你应调用表单的 `validate` 方法，提供输入的名称：

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '',
        email: '',
    }),
}">
    @csrf
    <label for="name">姓名</label>
    <input
        id="name"
        name="name"
        x-model="form.name"
        @change="form.validate('name')"
    />
    <template x-if="form.invalid('name')">
        <div x-text="form.errors.name"></div>
    </template>

    <label for="email">邮箱</label>
    <input
        id="email"
        name="email"
        x-model="form.email"
        @change="form.validate('email')"
    />
    <template x-if="form.invalid('email')">
        <div x-text="form.errors.email"></div>
    </template>

    <button :disabled="form.processing">
        创建用户
    </button>
</form>
```

当用户填写表单时，Precognition 将提供由路由表单请求中的验证规则支持的实时验证输出。当表单输入发生更改时，将发送一个经过防抖处理的 “先知” 验证请求到你的 Laravel 应用程序。你可以通过调用表单的 `setValidationTimeout` 函数来配置防抖超时时间：

```js
form.setValidationTimeout(3000);
```

在进行验证请求时，表单的 `validating` 属性将为 `true`：

```html
<template x-if="form.validating">
    <div>正在验证...</div>
</template>
```

在验证请求或表单提交期间返回的任何验证错误将自动填充表单的 `errors` 对象：

```html
<template x-if="form.invalid('email')">
    <div x-text="form.errors.email"></div>
</template>
```

你可以使用表单的 `hasErrors` 属性确定表单是否存在任何错误：

```html
<template x-if="form.hasErrors">
    <div><!-- ... --></div>
</template>
```

你还可以通过分别将输入的名称传递给表单的 `valid` 和 `invalid` 函数来确定输入是否通过或未通过验证：

```html
<template x-if="form.valid('email')">
    <span>✅</span>
</template>

<template x-if="form.invalid('email')">
    <span>❌</span>
</template>
```

> \[!WARNING\]  
> 表单输入只有在更改后并接收到验证响应后才会显示为有效或无效。

你可以通过检查表单的 `processing` 属性来确定表单提交请求是否正在进行中：

```html
<button :disabled="form.processing">
    提交
</button>
```

#### 重新填充旧表单数据

在上述讨论的用户创建示例中，我们使用 Precognition 执行实时验证；然而，我们执行传统的服务器端表单提交来提交表单。因此，表单应填充任何来自服务器端表单提交的 “旧” 输入和验证错误：

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '{{ old('name') }}',
        email: '{{ old('email') }}',
    }).setErrors({{ Js::from($errors->messages()) }}),
}">
```

或者，如果你希望通过 XHR 提交表单，可以使用表单的 `submit` 函数，该函数返回一个 Axios 请求 promise：

```html
<form 
    x-data="{
        form: $form('post', '/register', {
            name: '',
            email: '',
        }),
        submit() {
            this.form.submit()
                .then(response => {
                    form.reset();

                    alert('用户已创建。')
                })
                .catch(error => {
                    alert('发生错误。');
                });
        },
    }"
    @submit.prevent="submit"
>
```

### 配置 Axios

Precognition 验证库使用 [Axios](https://github.com/axios/axios) HTTP 客户端向你的应用程序后端发送请求。如果需要，可以自定义 Axios 实例以满足你的应用程序要求。例如，在使用 `laravel-precognition-vue` 库时，你可以在应用程序的 `resources/js/app.js` 文件中为每个传出请求添加额外的请求标头：

```js
import { client } from 'laravel-precognition-vue';

client.axios().defaults.headers.common['Authorization'] = authToken;
```

或者，如果你已经为应用程序配置了 Axios 实例，可以告诉 Precognition 使用该实例：

```js
import Axios from 'axios';
import { client } from 'laravel-precognition-vue';

window.axios = Axios.create()
window.axios.defaults.headers.common['Authorization'] = authToken;

client.use(window.axios)
```

> **警告**  
> Inertia 风格的 Precognition 库仅会对验证请求使用配置的 Axios 实例。表单提交将始终由 Inertia 发送。

## 自定义验证规则

可以通过使用请求的 `isPrecognitive` 方法来自定义在先知请求期间执行的验证规则。

例如，在用户创建表单上，我们可能只希望在最终表单提交时验证密码是否 “未被破解”。对于先知验证请求，我们只需验证密码是否必需且至少有 8 个字符。使用 `isPrecognitive` 方法，我们可以自定义表单请求中定义的规则：

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rules\Password;

class StoreUserRequest extends FormRequest
{
    /**
     * 获取适用于请求的验证规则。
     *
     * @return array
     */
    protected function rules()
    {
        return [
            'password' => [
                'required',
                $this->isPrecognitive()
                    ? Password::min(8)
                    : Password::min(8)->uncompromised(),
            ],
            // ...
        ];
    }
}
```

## 处理文件上传

默认情况下，Laravel Precognition 在验证请求期间不会上传或验证文件。这样可以确保大文件不会被多次不必要地上传。

由于这种行为，你应该确保你的应用程序[自定义相应表单请求的验证规则](#customizing-validation-rules)以指定该字段仅在完整表单提交时为必需：

```php
/**
 * 获取适用于请求的验证规则。
 *
 * @return array
 */
protected function rules()
{
    return [
        'avatar' => [
            ...$this->isPrecognitive() ? [] : ['required'],
            'image',
            'mimes:jpg,png',
            'dimensions:ratio=3/2',
        ],
        // ...
    ];
}
```

如果你希望在每个验证请求中包含文件，可以在客户端表单实例上调用 `validateFiles` 函数：

## 管理副作用

当向路由添加 `HandlePrecognitiveRequests` 中间件时，你应该考虑在先知请求期间应跳过的其他中间件中是否存在任何副作用。

例如，你可能有一个中间件，用于增加每个用户与你的应用程序的总「交互」次数，但你可能不希望将 Precognition 请求计为交互。为实现此目的，在增加交互计数之前，我们可以在检查请求的 `isPrecognitive` 方法：

```php
<?php

namespace App\Http\Middleware;

use App\Facades\Interaction;
use Closure;
use Illuminate\Http\Request;

class InteractionMiddleware
{
    /**
     * 处理传入请求。
     */
    public function handle(Request $request, Closure $next): mixed
    {
        if (! $request->isPrecognitive()) {
            Interaction::incrementFor($request->user());
        }

        return $next($request);
    }
}
```

## 测试

如果你希望在测试中进行先知请求，Laravel 的 `TestCase` 包括一个 `withPrecognition` 助手，该助手将添加 `Precognition` 请求头。

此外，如果你希望断言先知请求成功，例如，没有返回任何验证错误，你可以在响应上使用 `assertSuccessfulPrecognition` 方法：

```php
it('使用先知验证注册表单', function () {
    $response = $this->withPrecognition()
        ->post('/register', [
            'name' => 'Taylor Otwell',
        ]);

    $response->assertSuccessfulPrecognition();

    expect(User::count())->toBe(0);
});
```

```php
public function test_it_validates_registration_form_with_precognition()
{
    $response = $this->withPrecognition()
        ->post('/register', [
            'name' => 'Taylor Otwell',
        ]);

    $response->assertSuccessfulPrecognition();
    $this->assertSame(0, User::count());
}
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/pr...](https://learnku.com/docs/laravel/11.x/precognitionmd/16727)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/pr...](https://learnku.com/docs/laravel/11.x/precognitionmd/16727)