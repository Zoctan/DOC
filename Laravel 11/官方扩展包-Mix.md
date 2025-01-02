本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Mix

+   [介绍](#introduction)

## 介绍

[Laravel Mix](https://github.com/laravel-mix/laravel-mix) 是一个由 [Laracasts](https://laracasts.com/) 的创始人 Jeffrey Way 开发的包，它提供了一个流畅的 API，用于定义 [webpack](https://webpack.js.org/) 构建步骤，以在 Laravel 应用程序中使用多种常见的 CSS 和 JavaScript 预处理器。

换句话说，Mix 让编译和压缩应用程序的 CSS 和 JavaScript 文件变得轻而易举。您可以通过简单的方法链接，流畅地定义资源文件管道。例如：

```js
mix.js('resources/js/app.js', 'public/js')
    .postCss('resources/css/app.css', 'public/css');
```

如果您曾经因为开始使用 Webpack 和资源编译而感到困惑和不知所措，那么你会喜欢上 Laravel Mix 。然而，在开发应用程序时，并不需要强制使用它；你可以自由选择任何资源文件管道工具，甚至不使用任何工具。

> 注意  
> 在新安装 Laravel 中，Vite 已经取代了 Laravel Mix 。如果需要查看 Mix 的文档，请访问 [Laravel Mix 官方](https://laravel-mix.com/) 网站。如果你想切换到 Vite，请阅读我们的 [Vite 迁移指南](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-laravel-mix-to-vite).

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/mi...](https://learnku.com/docs/laravel/11.x/mixmd/16722)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/mi...](https://learnku.com/docs/laravel/11.x/mixmd/16722)