本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## 入门套件

+   [简介](#introduction)
+   [Laravel Breeze](#laravel-breeze)
    +   [安装](#laravel-breeze-installation)
    +   [Breeze 和 Blade](#breeze-and-blade)
    +   [Breeze 和 Livewire](#breeze-and-livewire)
    +   [Breeze 和 React / Vue](#breeze-and-inertia)
    +   [Breeze 和 Next.js/ API](#breeze-and-next)
+   [Laravel Jetstream](#laravel-jetstream)

## 简介

为了帮助您快速开始构建新的 Laravel 应用程序，我们很高兴提供认证和应用程序入门套件。这些套件会自动为您的应用程序生成所需的路由、控制器和视图，以便注册和验证应用程序的用户。

虽然欢迎您使用这些入门套件，但并非强制要求。您可以自由地从头开始构建自己的应用程序，只需安装一个全新的 Laravel。无论哪种方式，我们相信您都会构建出优秀的应用程序！

## Laravel Breeze

[Laravel Breeze](https://github.com/laravel/breeze) 是 Laravel 的一个最小化、简单的实现，包括登录、注册、密码重置、电子邮件验证和密码确认等所有 [认证功能](https://learnku.com/docs/laravel/11.x/authenticationmd)。此外，Breeze 还包含一个简单的 「个人资料」页面，用户可以在该页面更新他们的姓名、电子邮件地址和密码。

Laravel Breeze 的默认视图层由简单的 [Blade 模板](https://learnku.com/docs/laravel/11.x/blademd) 和 [Tailwind CSS](https://tailwindcss.com/) 样式化组成。此外，Breeze 提供了基于 [Livewire](https://livewire.laravel.com/) 或 [Inertia](https://inertiajs.com/) 的脚手架选项，可以选择在基于 Inertia 的脚手架中使用 Vue 或 React。

![breeze-register.png](https://laravel.com/img/docs/breeze-register.png)

#### Laravel Bootcamp

如果您是 Laravel 的新手，请随时参加 [Laravel Bootcamp](https://bootcamp.laravel.com/)。Laravel Bootcamp 将指导您使用 Breeze 构建您的第一个 Laravel 应用程序。这是一个了解 Laravel 和 Breeze 提供的所有功能的绝佳方式。

### 安装

首先，你应该[创建一个新的 Laravel 应用程序](https://learnku.com/docs/laravel/11.x/installationmd)。如果你使用 [Laravel 安装器](https://learnku.com/docs/laravel/11.x/installationmd#creating-a-laravel-project) 创建应用程序，将在安装过程中提示你安装 Laravel Breeze。否则，你将需要按照下面的手动安装说明进行操作。

如果你已经创建了一个没有入门套件的新 Laravel 应用程序，你可以使用 Composer 手动安装 Laravel Breeze：

```shell
composer require laravel/breeze --dev
```

在 Composer 安装了 Laravel Breeze 包之后，你应该运行 `breeze:install` Artisan 命令。此命令将认证视图、路由、控制器和其他资源发布到你的应用程序。Laravel Breeze 将所有代码发布到你的应用程序中，以便你完全控制并查看其功能和实现。

`breeze:install` 命令将提示你选择首选的前端堆栈和测试框架：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

### Breeze 和 Blade

Breeze 的默认「堆栈」是 Blade 堆栈，它使用简单的 [Blade 模板](https://learnku.com/docs/laravel/11.x/blademd) 来渲染应用程序的前端。可以通过调用 `breeze:install` 命令并选择 Blade 前端堆栈来安装 Blade 堆栈。在安装 Breeze 的脚手架之后，你还应该编译应用程序的前端资源：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

接下来，你可以在 Web 浏览器中导航到应用程序的 `/login` 或 `/register` URL。Breeze 的所有路由都定义在 `routes/auth.php` 文件中。

> **注意**  
> 要了解有关编译应用程序 CSS 和 JavaScript 的更多信息，请查看 Laravel 的 [Vite 文档](https://learnku.com/docs/laravel/11.x/vitemd#running-vite)。

### Breeze 和 Livewire

Laravel Breeze 提供了 [Livewire](https://livewire.laravel.com/) 的脚手架支持。Livewire 允许你使用纯 PHP 构建动态、响应式的前端 UI，对于那些主要使用 Blade 模板且希望寻找比 Vue 和 React 等 JavaScript 驱动的 SPA 框架更简单的替代方案的团队来说，Livewire 是一个理想的选择。

要使用 Livewire 栈，你可以在执行 `breeze:install` Artisan 命令时选择 Livewire 前端栈。完成 Breeze 的脚手架安装后，运行以下命令进行数据库迁移：

```shell
php artisan breeze:install

php artisan migrate
```

### Breeze 和 React / Vue

Laravel Breeze 还通过 [Inertia](https://inertiajs.com/) 提供了 React 和 Vue 的脚手架。Inertia 允许你结合 Laravel 的服务器端路由和控制器，构建现代的单页面 React 或 Vue 应用。

Inertia 使你能够利用 React 和 Vue 的前端功能，同时保留 Laravel 的后端高效和 [Vite](https://vitejs.dev/) 的快速编译。在执行 `breeze:install` 命令时，你可以选择 Vue 或 React 前端栈，并且安装程序会提示是否支持 [Inertia SSR](https://inertiajs.com/server-side-rendering) 或 TypeScript。安装完 Breeze 的脚手架后，还需要编译前端资产：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

然后，你可以在浏览器中访问 `/login` 或 `/register` URL。所有 Breeze 的路由定义在 `routes/auth.php` 文件中。

### Breeze and Next.js / API

Laravel Breeze 还可以构建一个身份验证 API ，该 API 可以对现代 JavaScript 应用程序，例如由 [Next](https://nextjs.org/)， [Nuxt](https://nuxt.com/) 等驱动的应用进行身份验证。要开始，请在执行 `breeze:install` Artisan 命令时指定 `api` 堆栈作为所需的堆栈：

```shell
php artisan breeze:install

php artisan migrate
```

在安装期间，Breeze 将在应用程序的 `.env` 文件中添加 `FRONTEND_URL` 环境变量。该 URL 应该是你的 JavaScript 应用程序的 URL。在本地开发期间，这通常是 `http://localhost:3000` 。另外，你应该确保 `APP_URL` 设置为 `http://localhost:8000` ，这是 `serve` Artisan 命令使用的默认 URL。

#### Next.js 参考实现

最后，你可以将此后端与你选择的前端配对。Breeze 前端的 Next 参考实现在 [在 GitHub 上提供](https://github.com/laravel/breeze-next)。此前端由 Laravel 维护，并包含与 Breeze 提供的传统 Blade 和 Inertia 堆栈相同的用户界面。

## Laravel Jetstream

虽然 Laravel Breeze 为构建 Laravel 应用程序提供了简单和最小的起点，但 Jetstream 通过更强大的功能和附加的前端技术栈增强了该功能。 **对于全新接触 Laravel 的用户，我们建议使用 Laravel Breeze 学习一段时间后再尝试 Laravel Jetstream。**

Jetstream 为 Laravel 提供了美观的应用程序脚手架，并包括登录、注册、电子邮件验证、双因素身份验证、会话管理、通过 Laravel Sanctum 支持的 API 以及可选的团队管理。Jetstream 使用 [Tailwind CSS](https://tailwindcss.com/) 设计，并提供你选择使用 [Livewire](https://livewire.laravel.com/) 或 [Inertia](https://inertiajs.com/) 驱动的前端脚手架。

有关安装 Laravel Jetstream 的完整文档，请参阅 [Jetstream 官方文档](https://jetstream.laravel.com/)。

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/st...](https://learnku.com/docs/laravel/11.x/starter-kitsmd/16651)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/st...](https://learnku.com/docs/laravel/11.x/starter-kitsmd/16651)