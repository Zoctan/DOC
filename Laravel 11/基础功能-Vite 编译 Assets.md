本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## 资源绑定 (Vite)

+   [介绍](#introduction)
+   [安装和配置](#installation)
    +   [安装 Node](#installing-node)
    +   [安装 Vite 和 Laravel 扩展](#installing-vite-and-laravel-plugin)
    +   [配置 Vite](#configuring-vite)
    +   [加载你的脚本和样式](#loading-your-scripts-and-styles)
+   [运行 Vite](#running-vite)
+   [使用 JavaScript](#working-with-scripts)
    +   [别名](#aliases)
    +   [Vue](#vue)
    +   [React](#react)
    +   [Inertia](#inertia)
    +   [URL 处理](#url-processing)
+   [使用样式表](#working-with-stylesheets)
+   [使用 Blade 和路由](#working-with-blade-and-routes)
    +   [使用 Vite 处理静态资源](#blade-processing-static-assets)
    +   [保存后刷新](#blade-refreshing-on-save)
    +   [别名](#blade-aliases)
+   [自定义基础 URL](#custom-base-urls)
+   [环境变量](#environment-variables)
+   [在测试中禁用 Vite](#disabling-vite-in-tests)
+   [服务端渲染 (SSR)](#ssr)
+   [脚本和样式标记属性](#script-and-style-attributes)
    +   [内容安全策略（CSP）随机数](#content-security-policy-csp-nonce)
    +   [子资源完整性（SRI）](#subresource-integrity-sri)
    +   [任意属性](#arbitrary-attributes)
+   [高级定制](#advanced-customization)
    +   [更正开发服务器 URL](#correcting-dev-server-urls)

## 介绍

[Vite](https://vitejs.dev/) 是一款现代前端构建工具，提供极快的开发环境并将你的代码捆绑到生产准备的资源中。在使用 Laravel 构建应用程序时，通常会使用 Vite 将你的应用程序的 CSS 和 JavaScript 文件绑定到生产环境的资源中。

Laravel 通过提供官方插件和 Blade 指令，与 Vite 完美集成，以加载你的资源进行开发和生产。

> **注意**  
> 你正在运行 Laravel Mix 吗？在新的 Laravel 安装中，Vite 已经取代了 Laravel Mix 。有关 Mix 的文档，请访问 [Laravel Mix](https://laravel-mix.com/) 网站。如果你想切换到 Vite，请参阅我们的 [迁移指南](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-laravel-mix-to-vite)。

#### 选择 Vite 还是 Laravel Mix

在转向 Vite 之前，新的 Laravel 应用程序在打包资源时通常使用 [Mix](https://laravel-mix.com/)，它由 [webpack](https://webpack.js.org/) 支持。Vite 专注于在构建丰富的 JavaScript 应用程序时提供更快、更高效的开发体验。如果你正在开发单页面应用程序（SPA），包括使用 Inertia 工具开发的应用程序，则 Vite 是完美选择。

Vite 也适用于具有 JavaScript 「sprinkles」的传统服务器端渲染应用程序，包括使用 [Livewire](https://livewire.laravel.com/) 的应用程序。但是，它缺少 Laravel Mix 支持的某些功能，例如将没有直接在 JavaScript 应用程序中引用的任意资源复制到构建中的能力。

#### 切换回 Mix

如果你使用我们的 Vite 脚手架创建了一个新的 Laravel 应用程序，但需要切换回 Laravel Mix 和 webpack，那么也没有问题。请参阅我们的[从 Vite 切换到 Mix 的官方指南](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-vite-to-laravel-mix)。

## 安装和设置

> **注意**  
> 以下文档讨论如何手动安装和配置 Laravel Vite 插件。但是，Laravel 的[起始套件](https://learnku.com/docs/laravel/11.x/starter-kitsmd)已经包含了所有的脚手架，并且是使用 Laravel 和 Vite 开始最快的方式。

### 安装 Node

在运行 Vite 和 Laravel 插件之前，你必须确保已安装 Node.js（16+）和 NPM：

你可以通过[官方 Node 网站](https://nodejs.org/en/download/)的简单图形安装程序轻松安装最新版本的 Node 和 NPM。或者，如果你使用的是 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sailmd)，可以通过 Sail 调用 Node 和 NPM：

```sh
./vendor/bin/sail node -v
./vendor/bin/sail npm -v
```

### 安装 Vite 和 Laravel 插件

在 Laravel 的全新安装中，你会在应用程序目录结构的根目录下找到一个 `package.json` 文件。默认的 `package.json` 文件已经包含了你开始使用 Vite 和 Laravel 插件所需的一切。你可以通过 NPM 安装应用程序的前端依赖：

### 配置 Vite

Vite 通过项目根目录中的 `vite.config.js` 文件进行配置。你可以根据自己的需要自定义此文件，也可以安装任何其他插件，例如 `@vitejs/plugin-vue` 或 `@vitejs/plugin-react`。

Laravel Vite 插件需要你指定应用程序的入口点。这些入口点可以是 JavaScript 或 CSS 文件，并包括预处理语言，例如 TypeScript、JSX、TSX 和 Sass。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel([
            'resources/css/app.css',
            'resources/js/app.js',
        ]),
    ],
});
```

如果你正在构建一个单页应用程序，包括使用 Inertia 构建的应用程序，则最好不要使用 CSS 入口点：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel([
            'resources/css/app.css', // [移除]
            'resources/js/app.js',
        ]),
    ],
});
```

相反，你应该通过 JavaScript 导入你的 CSS。通常，这将在应用程序的 `resources/js/app.js` 文件中完成：

```js
import './bootstrap';
import '../css/app.css'; // [添加]
```

Laravel 插件还支持多个入口点和高级配置选项，例如 [SSR 入口点](#ssr)。

#### 使用安全的开发服务器 r

如果你的本地开发 Web 服务器通过 HTTPS 提供应用程序服务，则可能会遇到连接到 Vite 开发服务器的问题。

如果你正在使用 [Laravel Herd](https://herd.laravel.com/) 并且已经保护了您的网站，或者您正在使用 [Laravel Valet](https://learnku.com/docs/laravel/11.x/valetmd) 并且已经对你的应用程序运行了 [secure 命令](https://learnku.com/docs/laravel/11.x/valetmd#securing-sites) 命令，Laravel Vite 插件将自动检测并使用生成的 TLS 证书。

如果你使用与应用程序目录名不匹配的主机来保护网站，您可以在应用程序的 `vite.config.js` 文件中手动指定主机。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            detectTls: 'my-app.test', // [添加]
        }),
    ],
});
```

当使用其他 Web 服务器时，你应生成一个受信任的证书并手动配置 Vite 使用生成的证书：

```js
// ...
import fs from 'fs'; // [添加]

const host = 'my-app.test'; // [添加]

export default defineConfig({
    // ...
    server: { // [添加]
        host, // [添加]
        hmr: { host }, // [添加]
        https: { // [添加]
            key: fs.readFileSync(`/path/to/${host}.key`), // [添加]
            cert: fs.readFileSync(`/path/to/${host}.crt`), // [添加]
        }, // [添加]
    }, // [添加]
});
```

如果你无法为系统生成可信证书，则可以安装并配置 [`@vitejs/plugin-basic-ssl` plugin](https://github.com/vitejs/vite-plugin-basic-ssl) 插件。使用不受信任的证书时，你需要通过在运行 `npm run dev` 命令时按照控制台中的 「 Local 」 链接来接受 Vite 开发服务器的证书警告。

#### 在 WSL2 上使用 Sail 运行开发服务器

在 Windows 子系统 Linux 2 (WSL2) 上使用 [Laravel Sail](https://learnku.com/docs/laravel/11.x/sailmd) 运行 Vite 开发服务器时，你应该在 `vite.config.js` 文件中添加以下配置以确保浏览器能够与开发服务器通信：

```js
// ...

export default defineConfig({
    // ...
    server: {
        // [添加开始]
        hmr: {
            host: 'localhost',
        },
    }, // [添加结束]
});
```

如果你在开发服务器运行时文件更改没有在浏览器中反映出来，你可能还需要配置 Vite 的 [`server.watch.usePolling` 选项](https://vitejs.dev/config/server-options.html#server-watch)。

### 加载你的脚本和样式

配置了 Vite 入口点后，你只需要在应用程序根模板的 `<head>` 中添加一个 `@vite()` Blade 指令引用它们即可：

```blade
<!doctype html>
<head>
    {{-- ... --}}

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

如果你通过 JavaScript 导入你的 CSS 文件，你只需要包含 JavaScript 的入口点：

```blade
<!doctype html>
<head>
    {{-- ... --}}

    @vite('resources/js/app.js')
</head>
```

`@vite` 指令会自动检测 Vite 开发服务器并注入 Vite 客户端以启用热模块替换。在构建模式下，该指令将加载已编译和版本化的资源，包括任何导入的 CSS 文件。

如果需要，在调用 `@vite` 指令时，你还可以指定已编译资源的构建路径：

```blade
<!doctype html>
<head>
    {{-- Given build path is relative to public path. --}}

    @vite('resources/js/app.js', 'vendor/courier/build')
</head>
```

#### 内联资源

有时可能需要直接包含资源的原始内容，而不是链接到资源的版本化 URL 。例如，当将 HTML 内容传递给 PDF 生成器时，你可能需要直接将资源内容嵌入到页面中。你可以使用 `Vite` 门面提供的 `content` 方法输出 Vite 资源的内容：

```blade
@use('Illuminate\Support\Facades\Vite')

<!doctype html>
<head>
    {{-- ... --}}

    <style>
        {!! Vite::content('resources/css/app.css') !!}
    </style>
    <script>
        {!! Vite::content('resources/js/app.js') !!}
    </script>
</head>
```

## 运行 Vite

你可以通过两种方式运行 Vite。你可以通过 `dev` 命令运行开发服务器，在本地开发时非常有用。开发服务器会自动检测文件的更改，并立即在任何打开的浏览器窗口中反映这些更改。

或者，运行 `build` 命令将版本化并打包应用程序的资源，并准备好部署到生产环境：

```shell
# 运行 Vite 开发服务器...
npm run dev

# 构建并为生产环境版本化资源...
npm run build
```

如果你在 WSL2 上的 [Sail](https://learnku.com/docs/laravel/11.x/sailmd) 中运行开发服务器，你可能需要一些[额外的配置](#configuring-hmr-in-sail-on-wsl2)选项。

## 使用 JavaScript

### 别名

默认情况下，Laravel 插件提供一个常用的别名，以帮助你快速开始并方便地导入你的应用程序的资源：

```js
{
    '@' => '/resources/js'
}
```

你可以通过添加自己的别名到 `vite.config.js` 配置文件中，覆盖 `'@'` 别名：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel(['resources/ts/app.tsx']),
    ],
    resolve: {
        alias: {
            '@': '/resources/ts',
        },
    },
});
```

### Vue

如果你想使用 [Vue](https://vuejs.org/) 框架构建前端，那么你还需要安装 `@vitejs/plugin-vue` 插件：

```sh
npm install --save-dev @vitejs/plugin-vue
```

然后你可以在 `vite.config.js` 配置文件中包含该插件。 Vue 插件与 Laravel 一起使用时，你需要一些额外的设置：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.js']),
        vue({
            template: {
                transformAssetUrls: {
                    // Vue 插件会在引用单文件组件中的资源 URL 时
                    // 将其重写指向 Laravel Web 服务器
                    // 将此设置为 `null` 可以让 Laravel 插件
                    // 将资源 URL 重写为指向
                    // Vite 服务器
                    base: null,

                    // Vue 插件会解析绝对 URL
                    // 并将其视为磁盘上的绝对路径
                    // 将此设置为 `false` 将保留绝对 URL 不变
                    // 以便它们可以按预期引用 public 目录中的资源
                    includeAbsolute: false,
                },
            },
        }),
    ],
});
```

> **注意**  
> Laravel 的 [启步套件](https://learnku.com/docs/laravel/11.x/starter-kitsmd) 已经包含了适当的 Laravel、Vue 和 Vite 配置。你可以查看 [Laravel Breeze](https://learnku.com/docs/laravel/11.x/starter-kitsmd#breeze-and-inertia)，这是使用 Laravel、Vue 和 Vite 快速入门的最快方法。

### React

如果你想使用 [React](https://reactjs.org/) 框架构建前端，那么你还需要安装 `@vitejs/plugin-react` 插件：

```sh
npm install --save-dev @vitejs/plugin-react
```

然后你可以在 `vite.config.js` 配置文件中包含该插件：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.jsx']),
        react(),
    ],
});
```

当使用 Vite 和 React 时，你将需要确保任何包含 JSX 的文件都有一个 `.jsx` 和 `.tsx` 扩展，记住更新入口文件，如果需要 [如上所示](#configuring-vite)。你还需要在现有的 @vite 指令旁边包含额外的 @viteReactRefresh Blade 指令。

你还需要在现有的 `@vite` 指令旁边包含额外的 `@viteReactRefresh` Blade 指令。

```blade
@viteReactRefresh
@vite('resources/js/app.jsx')
```

`@viteReactRefresh` 指令必须在 `@vite` 指令之前调用 。

> **注意**  
> Laravel 的 [起步套件](https://learnku.com/docs/laravel/11.x/starter-kitsmd) 已经包含了正确的 Laravel、React 和 Vite 配置。查看 [Laravel Breeze](https://learnku.com/docs/laravel/11.x/starter-kitsmd#breeze-and-inertia) 以了解使用 Laravel、React 和 Vite 的最快入门方式。

### Inertia

Laravel Vite 插件提供了一个方便的 `resolvePageComponent` 函数，帮助你解决 Inertia 页面组件。以下是使用 Vue 3 的助手示例；然而，你也可以在其他框架中使用该函数，例如 React：

```js
import { createApp, h } from 'vue';
import { createInertiaApp } from '@inertiajs/vue3';
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers';

createInertiaApp({
  resolve: (name) => resolvePageComponent(`./Pages/${name}.vue`, import.meta.glob('./Pages/**/*.vue')),
  setup({ el, App, props, plugin }) {
    return createApp({ render: () => h(App, props) })
      .use(plugin)
      .mount(el)
  },
});
```

> **注意**  
> Laravel 的 [起步套件](https://learnku.com/docs/laravel/11.x/starter-kitsmd) 已经包含了正确的 Laravel、Inertia 和 Vite 配置。查看 [Laravel Breeze](https://learnku.com/docs/laravel/11.x/starter-kitsmd#breeze-and-inertia) 以了解使用 Laravel、Inertia 和 Vite 的最快入门方式。

### URL 处理

当使用 Vite 并在你的应用程序的 HTML，CSS 和 JS 中引用资源时，有几件事情需要考虑。首先，如果你使用绝对路径引用资源，Vite 将不会在构建中包含资源；因此，你需要确认资源在你的公共目录中是可用的。

在引用相对路径的资源时，你应该记住这些路径是相对于它们被引用的文件的路径。通过相对路径引用的所有资源都将被 Vite 重写、版本化和打包。

参考以下项目结构：

```nothing
public/
  taylor.png
resources/
  js/
    Pages/
      Welcome.vue
  images/
    abigail.png
```

以下示例演示了 Vite 如何处理相对路径和绝对 URL：

```html
<!-- 这个资源不被 Vite 处理，不会被包含在构建中 -->
<img src="/taylor.png">

<!-- 这个资源将由 Vite 重写、版本化和打包 -->
<img src="../../images/abigail.png">
```

## 使用样式表

你可以在 [Vite 文档](https://vitejs.dev/guide/features.html#css) 中了解有关 Vite 的 CSS 支持更多的信息。如果你使用 PostCSS 插件，如 [Tailwind](https://tailwindcss.com/)，你可以在项目的根目录中创建一个 postcss.config.js 文件，Vite 会自动应用它：

```js
export default {
    plugins: {
        tailwindcss: {},
        autoprefixer: {},
    },
};
```

> **注意**  
> Laravel 的 [启步套件](https://learnku.com/docs/laravel/11.x/starter-kitsmd) 已经包含了适当的 Tailwind、PostCSS 和 Vite 配置。你可以查看 Laravel Breeze，这是使用 Laravel、Vue 和 Vite 快速入门的最快方法

Laravel 的 [入门套件](https://learnku.com/docs/laravel/11.x/starter-kitsmd) 已经包含了正确的 Tailwind、PostCSS 和 Vite 配置。或者，如果你想在不使用我们的入门套件的情况下使用 Tailwind 和 Laravel，可以查看 [Tailwind 的 Laravel 安装指南](https://tailwindcss.com/docs/guides/laravel)。

## 使用 Blade 和 路由

### 通过 Vite 处理静态资源

在你的 JavaScript 或 CSS 中引用资源时，Vite 会自动处理和版本化它们。此外，在构建基于 Blade 的应用程序时，Vite 还可以处理和版本化你仅在 Blade 模板中引用的静态资源。

然而，要实现这一点，你需要通过将静态资源导入到应用程序的入口点来让 Vite 了解你的资源。例如，如果你想处理和版本化存储在 `resources/images` 中的所有图像和存储在 `resources/fonts` 中的所有字体，你应该在应用程序的 `resources/js/app.js` 入口点中添加以下内容：

```js
import.meta.glob([
  '../images/**',
  '../fonts/**',
]);
```

这些资源将在运行 `npm run build` 时由 Vite 处理。然后，你可以在 Blade 模板中使用 `Vite::asset` 方法引用这些资源，该方法将返回给定资源的版本化 URL：

```blade
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```

### 保存刷新

当你的应用程序使用传统的服务器端渲染 Blade 构建时，Vite 可以通过在你的应用程序中更改视图文件时自动刷新浏览器来提高你的开发工作流程。要开始，你可以简单地将 `refresh` 选项指定为 `true`。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: true,
        }),
    ],
});
```

当 `refresh` 选项为 `true` 时，保存以下目录中的文件将在你运行 `npm run dev` 时触发浏览器进行全面的页面刷新：

+   `app/View/Components/**`
+   `lang/**`
+   `resources/lang/**`
+   `resources/views/**`
+   `routes/**`

监听 `routes/**` 目录对于在应用程序前端中利用 [Ziggy](https://github.com/tighten/ziggy) 生成路由链接非常有用。

如果这些默认路径不符合你的需求，你可以指定自己的路径列表：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: ['resources/views/**'],
        }),
    ],
});
```

在后台，Laravel Vite 插件使用了 [vite-plugin-full-reload](https://github.com/ElMassimo/vite-plugin-full-reload) 包，该包提供了一些高级配置选项，可微调此功能的行为。如果你需要这种级别的自定义，可以提供一个 `config` 定义：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: [{
                paths: ['path/to/watch/**'],
                config: { delay: 300 }
            }],
        }),
    ],
});
```

### 别名

在 JavaScript 应用程序中 [创建别名](#aliases) 来引用常用目录是很常见的。但是，你也可以通过在 `Illuminate\Support\Facades\Vite` 类上使用 `macro` 方法来创建在 Blade 中使用的别名。通常，「宏」 应在 [服务提供商](https://learnku.com/docs/laravel/11.x/providersmd) 的 `boot` 方法中定义：

```php
/**
 * 引导任何应用程序服务。
 */
public function boot(): void
{
    Vite::macro('image', fn (string $asset) => $this->asset("resources/images/{$asset}"));
}
```

定义了宏之后，可以在模板中调用它。例如，我们可以使用上面定义的 `image` 宏来引用位于 `resources/images/logo.png` 的资源：

```blade
<img src="{{ Vite::image('logo.png') }}" alt="Laravel Logo">
```

## 自定义 base URL

如果你的 Vite 编译的资产部署到与应用程序不同的域（例如通过 CDN），必须在应用程序的 `.env` 文件中指定 `ASSET_URL` 环境变量：

```env
ASSET_URL=https://cdn.example.com
```

配置完资源 URL 后，所有重写到你的资源的 URL 都将以配置值为前缀：

```nothing
https://cdn.example.com/build/assets/app.9dce8d17.js
```

请记住，[绝对路径的 URL 不会被 Vite 重新编写](#url-processing)，因此它们不会被添加前缀。

## 环境变量

你可以在应用程序的 `.env` 文件中以 `VITE_` 为前缀注入环境变量以在 JavaScript 中使用：

```env
VITE_SENTRY_DSN_PUBLIC=http://example.com
```

你可以通过 `import.meta.env` 对象访问注入的环境变量：

```js
import.meta.env.VITE_SENTRY_DSN_PUBLIC
```

## 在测试中禁用 Vite

Laravel 的 Vite 集成将在运行测试时尝试解析你的资产，这需要你运行 Vite 开发服务器或构建你的资产。

如果你希望在测试中模拟 Vite，你可以调用 `withoutVite` 方法，该方法对任何扩展 Laravel 的 `TestCase` 类的测试都可用：

```php
//译者注：Pest 示例
test('without vite example', function () {
    $this->withoutVite();

    // ...
});
```

```php
//译者注：PHPUnit 示例
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_without_vite_example(): void
    {
        $this->withoutVite();

        // ...
    }
}
```

如果你想在所有测试中禁用 Vite，可以在基本的 `TestCase` 类上的 `setUp` 方法中调用 `withoutVite` 方法：

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function setUp(): void// [tl! add:start]
    {
        parent::setUp();

        $this->withoutVite();
    }// [tl! add:end]
}
```

## 服务器端渲染 (SSR)

Laravel Vite 插件使得使用 Vite 进行服务器端渲染变得轻而易举。要开始，请在 `resources/js/ssr.js` 创建一个 SSR 入口点，并通过向 Laravel 插件传递一个配置选项来指定入口点：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            ssr: 'resources/js/ssr.js',
        }),
    ],
});
```

为确保不遗漏重建 SSR 入口点，我们建议增加应用程序的 `package.json` 中的 「build」 脚本来创建 SSR 构建：

```json
"scripts": {
     "dev": "vite",
     "build": "vite build" // [tl! remove]
     "build": "vite build && vite build --ssr" // [tl! add]
}
```

然后，要构建和启动 SSR 服务器，你可以运行以下命令：

```sh
npm run build
node bootstrap/ssr/ssr.js
```

如果你正在使用 [Inertia 的 SSR](https://inertiajs.com/server-side-rendering), 你可以使用 `inertia:start-ssr` Artisan 命令来启动 SSR 服务器。

```sh
php artisan inertia:start-ssr
```

> **技巧**  
> Laravel 的 [起步套件](https://learnku.com/docs/laravel/11.x/starter-kitsmd) 已经包含了适当的 Laravel、Inertia SSR 和 Vite 配置。查看 [Laravel Breeze](https://learnku.com/docs/laravel/11.x/starter-kitsmd#breeze-and-inertia) ，以了解使用 Laravel、Inertia SSR 和 Vite 的最快入门方式。

## Script & Style 标签的属性

### 内容安全策略（CSP）随机数

如果你希望在你的脚本和样式标签中包含 [`nonce 属性`](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce)，作为你的 [内容安全策略](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的一部分，你可以使用自定义 [中间件](https://learnku.com/docs/laravel/11.x/middlewaremd) 中的 `useCspNonce` 方法生成或指定一个 nonce：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Vite;
use Symfony\Component\HttpFoundation\Response;

class AddContentSecurityPolicyHeaders
{
    /**
     * 处理传入请求。
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        Vite::useCspNonce();

        return $next($request)->withHeaders([
            'Content-Security-Policy' => "script-src 'nonce-".Vite::cspNonce()."'",
        ]);
    }
}
```

调用了 `useCspNonce` 方法后，Laravel 将自动在所有生成的脚本和样式标签上包含 `nonce` 属性。

如果你需要在其他地方指定 nonce，包括 Laravel 的 [起步套件](https://learnku.com/docs/laravel/11.x/starter-kitsmd) 中带有的 [Ziggy `@route` 指令](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy) 指令，你可以使用 `cspNonce` 方法来检索它：

```blade
@routes(nonce: Vite::cspNonce())
```

如果你已经有了一个 nonce，想要告诉 Laravel 使用它，你可以通过将 nonce 传递给 `useCspNonce` 方法来实现：

```php
Vite::useCspNonce($nonce);
```

### 子资源完整性 (SRI)

如果你的 Vite 清单包含了资源的 `完整性` 哈希值，Laravel 将自动在它生成的任何脚本和样式标签上添加 `integrity` 属性，以强制执行[子资源完整性](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)。默认情况下，Vite 的清单中不包括 `完整性` 哈希值，但你可以通过安装 [`vite-plugin-manifest-sri`](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPM 插件来启用它。

```shell
npm install --save-dev vite-plugin-manifest-sri
```

然后，在你的 `vite.config.js` 文件中启用此插件：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import manifestSRI from 'vite-plugin-manifest-sri';// [tl! add]

export default defineConfig({
    plugins: [
        laravel({
            // ...
        }),
        manifestSRI(),// [tl! add]
    ],
});
```

如果需要，你也可以自定义清单中的完整性哈希键：

```php
use Illuminate\Support\Facades\Vite;

Vite::useIntegrityKey('custom-integrity-key');
```

如果你想完全禁用这个自动检测，你可以将 `false` 传递给 `useIntegrityKey` 方法：

```php
Vite::useIntegrityKey(false);
```

调用 `useCspNonce` 方法后，Laravel 将自动在所有生成的脚本和样式标签中包含 `nonce` 属性。

如果你需要在其他地方指定 `nonce`，包括 Laravel 的 [起步套件](https://learnku.com/docs/laravel/11.x/starter-kitsmd) 中包含的 [Ziggy `@route` directive](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy) ，你可以使用 `cspNonce` 方法来获取它：

```blade
@routes(nonce: Vite::cspNonce())
```

如果你已经有了一个想要指示 Laravel 使用的 `nonce` ，你可以将该 `nonce` 传递给 `useCspNonce` 方法：

```php
Vite::useCspNonce($nonce);
```

### 子资源完整性（SRI）

如果你的 Vite 清单中包含资源的 `integrity` 哈希值，Laravel 将自动在生成的任何脚本和样式标签中添加 `integrity` 属性，以强制执行 [子资源完整性](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) 。默认情况下，Vite 不会在其清单中包含 `integrity` 哈希值，但你可以通过安装 [`vite-plugin-manifest-sri`](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPM 插件来启用它：

```shell
npm install --save-dev vite-plugin-manifest-sri
```

然后你可以在 `vite.config.js` 文件中启用这个插件：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import manifestSRI from 'vite-plugin-manifest-sri';// [这里添加]

export default defineConfig({
    plugins: [
        laravel({
            // ...
        }),
        manifestSRI(),// [这里添加]
    ],
});
```

如果有需要，你还可以自定义能找到 `integrity` 哈希值的清单键：

```php
use Illuminate\Support\Facades\Vite;

Vite::useIntegrityKey('custom-integrity-key');
```

如果你想完全禁用这个自动检测功能，你可以将 `false` 传递给 `useIntegrityKey` 方法：

```php
Vite::useIntegrityKey(false);
```

### 任意属性

如果你需要在脚本和样式标签中包含其他属性，例如 [`data-turbo-track`](https://turbo.hotwired.dev/handbook/drive#reloading-when-assets-change) 属性，你可以通过 `useScriptTagAttributes` 和 `useStyleTagAttributes` 方法指定它们。通常，这些方法应从一个[服务提供者](https://learnku.com/docs/laravel/11.x/providersmd)中调用：

```php
use Illuminate\Support\Facades\Vite;

Vite::useScriptTagAttributes([
    'data-turbo-track' => 'reload', // 为属性指定一个值...
    'async' => true, // 在不使用值的情况下指定属性...
    'integrity' => false, // 排除一个将被包含的属性...
]);

Vite::useStyleTagAttributes([
    'data-turbo-track' => 'reload',
]);
```

如果你需要有条件地添加属性，你可以传递一个回调函数，它将接收到资产源路径、它的 URL、它的清单块和整个清单：

```php
use Illuminate\Support\Facades\Vite;

Vite::useScriptTagAttributes(fn (string $src, string $url, array|null $chunk, array|null $manifest) => [
    'data-turbo-track' => $src === 'resources/js/app.js' ? 'reload' : false,
]);

Vite::useStyleTagAttributes(fn (string $src, string $url, array|null $chunk, array|null $manifest) => [
    'data-turbo-track' => $chunk && $chunk['isEntry'] ? 'reload' : false,
]);
```

> **警告**  
> 在 Vite 开发服务器运行时，`$chunk` 和 `$manifest` 参数将为 `null`。

## 高级定制

Laravel 的 Vite 插件默认使用了一些合理的约定，这些约定应该适用于大多数应用程序；然而，有时你可能需要自定义 Vite 的行为。为了启用额外的自定义选项，我们提供了以下方法和选项，这些可以用来替代 `@vite` Blade 指令：

```blade
<!doctype html>
<head>
    {{-- ... --}}

    {{
        Vite::useHotFile(storage_path('vite.hot')) // 自定义 「hot」 文件...
            ->useBuildDirectory('bundle') // 自定义构建目录...
            ->useManifestFilename('assets.json') // 自定义清单文件名...
            ->withEntryPoints(['resources/js/app.js']) // 自定义清单文件名...
            ->createAssetPathsUsing(function (string $path, ?bool $secure) { // 为构建的资源定制后端路径生成...
                return "https://cdn.example.com/{$path}";
            })
    }}
</head>
```

在 `vite.config.js` 文件中，你应该指定相同的配置：

```js
import { defineConfig } from 'vite'
import laravel from 'laravel-vite-plugin'

export default defineConfig({
  plugins: [
    laravel({
      hotFile: 'storage/vite.hot', // 自定义 "hot" 文件...
      buildDirectory: 'bundle', // 自定义构建目录...
      input: ['resources/js/app.js'] // 指定入口点...
    })
  ],
  build: {
    manifest: 'assets.json' // 自定义清单文件名...
  }
})
```

### 纠正开发服务器 URL

Vite 生态系统内的一些插件假设以正斜杠开头的 URL 总是指向 Vite 开发服务器。然而，由于 Laravel 集成的性质，情况并非如此。

例如，当 Vite 提供你的资源时，`vite-imagetools` 插件会输出如下 URL：

```html
<img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520" />
```

`vite-imagetools` 插件期待输出的 URL 会被 Vite 拦截，然后插件可以处理所有以 `/@imagetools` 开始的 URL。如果你使用的插件期待这种行为，你将需要手动纠正 URL。你可以在 `vite.config.js` 文件中使用 `transformOnServe` 选项来实现这一点。

在这个特定的例子中，我们将在生成的代码中将 `/@imagetools` 开头的所有 URL 替换为开发服务器的 URL：

```js
import { defineConfig } from 'vite'
import laravel from 'laravel-vite-plugin'
import { imagetools } from 'vite-imagetools'

export default defineConfig({
  plugins: [
    laravel({
      // ...
      transformOnServe: (code, devServerUrl) => code.replaceAll('/@imagetools', devServerUrl + '/@imagetools')
    }),
    imagetools()
  ]
})
```

现在，当 Vite 提供资源时，它将输出指向 Vite 开发服务器的 URL：

```html
- <img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520" /><!-- [tl! remove] -->
+ <img src="http://[::1]:5173/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520" /><!-- [tl! add] -->
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/vi...](https://learnku.com/docs/laravel/11.x/vitemd/16665)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/vi...](https://learnku.com/docs/laravel/11.x/vitemd/16665)