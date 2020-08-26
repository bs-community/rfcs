- 仓库：blessing-skin-server
- 日期: 2020-08-26
- 作者：[Pig Fang](https://github.com/g-plane)
- RFC PR: [#0005](https://github.com/bs-community/rfcs/pull/5)
- 实现 PR: 

# 应用结构更改

## 概要

对 Blessing Skin Server 的整体结构进行改动，大体上类似于 Flarum 的结构。

## 初衷

### 插件依赖解析

目前的插件系统的插件依赖解析是由 Blessing Skin 自己实现的，语义化检查部分则通过 `composer/semver` 库来进行。而目前 Composer 作为 PHP 的包管理器，已经开发出成熟、稳定的依赖解析算法。如果能使用 Composer，不仅避免重复造轮子，还可以沿用成熟的代码。

### 统一化的应用升级

Blessing Skin 的核心以及插件将通过执行 `composer update` 命令来进行，从而实现一条命令升级所有功能，并可以避免部分用户单独升级插件或单独升级核心而带来的不一致问题。

### 免去自建市场源、更新源

由于所有代码（包括插件）将托管在 Packagist，而国内有多个 Packagist 镜像，因此用户可以使用这些镜像来加速核心的升级、插件的安装和升级。

开发版用户在安装 Blessing Skin 的依赖时速度慢的问题也将被解决。

### 独立的配置文件

用户可以随意修改所有的配置文件。尤其是开发版用户，不再需要担心版本控制的问题。

## 详细设计

### 整体目录结构

对用户而言，不再需要 `app` 目录、`resources` 目录，开发版用户也不再需要 **直接** 使用版本控制来升级核心。因为整个应用目录都不再包含源代码（包括前端和后端的代码，以及视图和国际化文本等），所以每个站点可以有各自不同的应用目录。

用户现在需要在应用的根目录下放置一个 `composer.json` 文件，也还可以放置一个可选的 `composer.lock` 文件（通常由 Composer 自动生成）。

### 核心代码的发布

所有的后端代码将被提取成一个 Composer 包发布在 Packagist 上。

普通用户可以在 `composer.json` 的 `require` 字段中指定 Blessing Skin 的版本，例如指定 `^5.0.0` 则表示使用 v5 版本。此时，假设 v6 发布了，如果用户不希望进行升级，则可以保持版本不变（即使是执行 `composer update` 命令，也会依照 `composer.json` 中的版本约束来进行）；如果想升级，则改成 `^6.0.0` 即可。

开发版用户可以直接指定为 `dev-dev`，每次执行 `composer update` 都会从 GitHub 拉取代码。（Composer 会在内部调用 Git）

该 Composer 包的名称将为 `blessing/core`。

鉴于核心代码库的复杂情况，不考虑在 Packagist 上直接把 `blessing/core` 包与 `blessing-skin-server` 仓库关联在一起，而是在每次 push 代码时，利用 GitHub Actions 将代码做适当处理后 push 到另外一个仓库，并将该仓库与 `blessing/core` 包关联。仓库名可以为 `bs-community/core`。

### 命名空间

后端代码将作为一个 Composer 包，因此原来的以 `App` 为开头的命名空间已经不再适合。对于核心代码，将采用 `Blessing\Core` 作为新的命名空间。

然而更改命名空间将会对插件产生破坏性变动。例如目前所有插件在使用 ORM 模型时，都是使用 `App\Models` 下的类。因此需要进行渐进式迁移。

首先，对于插件开发文档中未被提到的类（例如 `App\Services\Translations` 下的类），最快可以在 v5 的后续版本中更改其命名空间并提取到 `blessing/core` 包中。因为这部分模块不属于对外公开使用的功能（即使插件可以使用），插件不应该依赖于这部分功能；同时非公开模块的任意变动，不需要遵循语义化版本规则。

其次，对于文档提到的类（如 `App\Models` 下的所有模型）以及某些可能被广泛使用的类（如 `App\Http\Controllers\Controller`），在 v5 的后续开发中可以将这部分代码放入 `blessing/core` 包中，同时先不从核心中移除这些类，而是设置没有任何代码的 placeholder 并从 `Blessing\Core` 中继承这些类。

以 User 模型为例，我们可以在 `blessing/core` 包中编写这样的一个类：

```php
namespace Blessing\Core\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    // 这里的实现与原来的 User 一样
}
```

然后在 Blessing Skin 的 `app` 目录中的 `Models` 目录编写这样的一个类：

```php
namespace App\Models;

use Blessing\Core\Models\User as CoreUser;

class User extends CoreUser {}
```

如此一来，插件在迁移的过程中，就可以渐进式地修改每个类。

而到了 v6，则可以将这些以 `App` 开头的命名空间的类提取到一个单独的 Composer 包中（如 `blessing/compat`），方便有需要的插件作者引入这个包。而核心不再使用这些类。

特别地，考虑移除 `App\Http\Controllers\Controller`，改为使用 `Illuminate\Routing\Controller`，但仍遵循上述的迁移策略。

### 插件系统

#### 目前的插件系统

短期内保留现有的插件系统，但不再提供插件市场。

#### 插件管理

插件管理页面将继续保留，没有改动的计划。

#### 新的插件系统

所有的插件将通过 Composer 来安装，安装后需要在「插件管理」页面手动启用。

由于所有的插件都通过 Composer 来安装，因此 Blessing Skin 不再需要自己来加载插件用到的依赖——Composer 在 dump-autoload 阶段已做好相关工作。

在新的插件系统中，加载所有插件不再需要以读取每个插件的目录的方式来进行，而是读取 `vendor/composer/installed.json` 文件，这个文件包含了所有 Composer 包的信息。

也因为如此，插件的相关信息（例如配置页面入口、图标等）将写在各个插件的 `composer.json` 的 `extra` 字段的 `bs-plugin` 字段中（所有信息都记录在这个字段，不再需要额外的 `enchants` 字段）。另外，`name` 字段以及 `author` 字段可以直接根据 `composer.json` 的 Schema 来写；而 `version` 字段是不需要的，Composer 会自动分析出每个 Composer 包的版本。

#### 官方插件

目前所有的官方插件的源代码都在一个仓库中，而 Packagist 要求每个 Composer 包单独对应一个仓库。因此计划单独设立一个 GitHub organization，用于保存每个插件。当发布插件时，利用 GitHub Actions 向该插件对应的仓库中推送代码并打 Git tag。

Laravel 及 SocialiteProviders 有类似做法。

### 前端

每次将代码 push 到 `bs-community/blessing-skin-server` 仓库时，使用 GitHub Actions 对前端代码进行构建，然后再将后端代码一起 push 到 `bs-community/core` 仓库中。

开发版用户将不再需要自己构建前端文件。

### 视图与国际化文本

视图与国际化文本将被发布到 Packagist，但在使用时可能需要加上命名空间。详见「未能解决的问题」一节。

## 缺点

### 对插件生态的影响

首先，更改命名空间将会导致所有插件必须做出相应的更改。尽管此 RFC 中给出了渐进式的迁移方案，但还是需要插件开发者来跟进。当然，一款插件如果仍然在被维护，那么它应该是可以继续发展的；而且 Blessing Skin 目前的插件系统已经有语义化版本检查，可以确保不相兼容的版本不会被错误地运行。

第二，每个插件都必须是一个 Composer 包。（注意：Composer 包不需要非得开源并发布到 Packagist，它可以是本地私有的包）这意味着创建和开发一个插件将可能比以前复杂：需要正确配置每个插件自己的 `composer.json` 然后在应用根目录下的 `composer.json` 中 `repositories` 字段进行相应的配置以便读取到在本地开发的插件。

### 对开发版用户的影响

需要采取一定的迁移策略帮助开发版用户过渡到新的应用结构和模式。

### 插件的安装与删除

由于使用了 Composer，插件的安装与删除将通过 Composer 来进行，这意味着不能像以前那样通过在网页 UI 上通过鼠标点击来安装或移除插件，而是要用命令行工具。

## 备选方案

无。

## FAQ

待补充。

## 未能解决的问题

### 视图与国际化文本的命名空间问题

对于一个 Laravel 应用，Laravel 会默认从应用的 `resources` 目录下读取并加载文件。而在此 RFC 中，这些文件都将被作为一个 Composer 包发布到 Packagist，而这种做法可能导致引用视图和国际化文件时需要加上命名空间。具体是否需要还需要作进一步的研究。

### 帮助开发版用户进行平滑迁移

如何帮助开发版用户平滑地迁移到新的结构是一个还需要研究的问题。

## 相关内容

- [Flarum 核心](https://github.com/flarum/core)
- [Flarum 应用模板](https://github.com/flarum/flarum)

## 其它

无。