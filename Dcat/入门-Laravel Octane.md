[Laravel Octane](https://github.com/laravel/octane) 是一个基于 `Swoole/RoadRunner` 驱动的可以提升 `Laravel` 框架性能的项目，安装后可以大幅提升 `Laravel` 项目的性能。

`Dcat Admin` 从 `v2.0.23-beta` 版本起兼容了 `Laravel Octane` 环境，只需在配置文件 `config/octane.php` 中加入如下配置即可：

```php

    'listeners' => [
        ...,

        RequestReceived::class => [
            ...Octane::prepareApplicationForNextOperation(),
            ...Octane::prepareApplicationForNextRequest(),

            // 开启对 Dcat Admin 的支持
            Dcat\Admin\Octane\Listeners\FlushAdminState::class,
        ],

        ...
    ],    
```