本文档最新版为 [10.x](https://learnku.com/docs/laravel/10.x)，旧版本可能放弃维护，推荐阅读最新版！

## Laravel Pint

+   [介绍](#introduction)
+   [安装](#installation)
+   [运行 Pint](#running-pint)
+   [配置 Pint](#configuring-pint)
    +   [预设](#presets)
    +   [规则](#rules)
    +   [排除文件 / 文件夹](#excluding-files-or-folders)
+   [持续集成](#continuous-integration)
    +   [GitHub Actions](#running-tests-on-github-actions)

## 介绍

[Laravel Pint](https://github.com/laravel/pint) 是面向极简主义者的 PHP 代码风格修复工具。Pint 建立在 PHP-CS-Fixer 之上，简化了确保代码风格保持清洁和一致的过程。

Pint 自动安装在所有新的 Laravel 应用程序中，因此您可以立即开始使用它。默认情况下，Pint 不需要任何配置，并且将通过遵循 Laravel 的主观编码风格来修复代码风格问题。

## 安装

Pint 已包含在 Laravel 框架的最新版本中，因此通常不需要安装。但是，对于旧应用程序，您可以通过 Composer 安装 Laravel Pint：

```shell
composer require laravel/pint --dev
```

## 运行 Pint

您可以通过调用项目的 `vendor/bin` 目录中的 `pint` 二进制文件来指示 Pint 修复代码风格问题：

您还可以在特定文件或目录上运行 Pint：

```shell
./vendor/bin/pint app/Models

./vendor/bin/pint app/Models/User.php
```

Pint 将显示更新的所有文件的详细列表。您可以通过在调用 Pint 时提供 `-v` 选项来查看有关 Pint 更改的更多详细信息：

如果您希望 Pint 仅检查代码样式错误而不实际更改文件，可以使用 `--test` 选项。如果发现任何代码风格错误，Pint 将返回非零退出代码：

如果您希望 Pint 仅修改根据 Git 有未提交更改的文件，您可以使用 `--dirty` 选项：

```shell
./vendor/bin/pint --dirty
```

如果您希望 Pint 修复任何具有代码风格错误的文件，但同时在修复任何错误时退出并返回非零退出代码，您可以使用 `--repair` 选项：

```shell
./vendor/bin/pint --repair
```

## 配置 Pint

如前所述，Pint 不需要任何配置。但是，如果您希望自定义预设、规则或检查的文件夹，您可以在项目的根目录中创建一个 `pint.json` 文件：

此外，如果您希望使用特定目录中的 `pint.json`，您可以在调用 Pint 时提供 `--config` 选项：

```shell
pint --config vendor/my-company/coding-style/pint.json
```

### 预设

预设定义了一组规则，可用于修复代码风格问题。默认情况下，Pint 使用 `laravel` 预设，通过遵循 Laravel 的主观编码风格来修复问题。但是，您可以通过向 Pint 提供 `--preset` 选项来指定不同的预设：

如果需要，您也可以在项目的 `pint.json` 文件中设置预设：

Pint 目前支持的预设有：`laravel`、`per`、`psr12` 和 `symfony`。

### 规则

规则是 Pint 将用于修复代码风格问题的样式指南。如上所述，预设是预定义的规则组，应该适用于大多数 PHP 项目，因此您通常不需要担心它们包含的各个规则。

然而，如果你愿意的话，你可以在你的 `pint.json` 文件中启用或禁用特定规则：

```json
{
    "preset": "laravel",
    "rules": {
        "simplified_null_return": true,
        "braces": false,
        "new_with_braces": {
            "anonymous_class": false,
            "named_class": false
        }
    }
}
```

Pint 是建立在 [PHP-CS-Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer) 之上的。因此，你可以使用它的任何规则来修复项目中的代码风格问题：[PHP-CS-Fixer Configurator](https://mlocati.github.io/php-cs-fixer-configurator)。

### 排除文件 / 文件夹

默认情况下，Pint 将检查项目中除了 `vendor` 目录中的所有 `.php` 文件。如果你希望排除更多文件夹，你可以使用 `exclude` 配置选项：

```json
{
    "exclude": [
        "my-specific/folder"
    ]
}
```

如果你希望排除所有包含特定名称模式的文件，你可以使用 `notName` 配置选项：

```json
{
    "notName": [
        "*-my-file.php"
    ]
}
```

如果你希望通过提供文件的确切路径来排除文件，你可以使用 `notPath` 配置选项：

```json
{
    "notPath": [
        "path/to/excluded-file.php"
    ]
}
```

## 持续集成

### GitHub Actions

要使用 Laravel Pint 自动化检查你的项目代码风格，你可以配置 [GitHub Actions](https://github.com/features/actions) 在代码推送到 GitHub 时运行 Pint。首先，请确保在 GitHub 的 **Settings > Actions > General > Workflow permissions** 中为 workflows 授予 “读取和写入权限”。然后，创建一个名为 `.github/workflows/lint.yml` 的文件，并包含以下内容：

```ini
name: Fix Code Style

on: [push]

jobs:
  lint:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: true
      matrix:
        php: [8.3]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          extensions: json, dom, curl, libxml, mbstring
          coverage: none

      - name: Install Pint
        run: composer global require laravel/pint

      - name: Run Pint
        run: pint

      - name: Commit linted files
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "Fixes coding style"
```

> 本译文仅用于学习和交流目的，转载请务必注明文章译者、出处、和本文链接  
> 我们的翻译工作遵照 [CC 协议](https://learnku.com/docs/guide/cc4.0/6589)，如果我们的工作有侵犯到您的权益，请及时联系我们。

* * *

> 原文地址：[https://learnku.com/docs/laravel/11.x/pi...](https://learnku.com/docs/laravel/11.x/pintmd/16726)
> 
> 译文地址：[https://learnku.com/docs/laravel/11.x/pi...](https://learnku.com/docs/laravel/11.x/pintmd/16726)