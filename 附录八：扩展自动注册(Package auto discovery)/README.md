# 附录八：扩展自动注册(Package auto discovery)

在进入探究 Laravel 包提供者与门面如何自动发现之前，让我们先粗浅剖析一下 PHP 中包的概念。

一个包就是一个在多个项目内可复用的代码片段，例如 包 spatie/laravel-analytics 可以让你在 laravel 项目内，用一种简易方式从谷歌统计（Google Analytics）中取回数据，该包被托管 在 GitHub 上，由 Spatie 进行维护，它们会持续发布，更新和修复该包 bug，如果你在项目当中使用该包，希望获取这些一旦发布的更新和修复，无须担心使用 Composer 从 Github 上 拷贝一份新代码即可。

> Composer 是一个 PHP 依赖管理工具。它允许你声明项目库依赖且管理（安装 / 更新）它们。 -- 详见官网 getcomposer.org

Laravel 自带 composer.json 文件，文件内的 require 或 require-dev 条目下，给出了你扩展应用功能需用到的包，执行 `composer update`:

```
{
   "require": {
       "spatie/laravel-analytics": "3.*",
   },
}
```

你也可以使用下面命令，达到同样的效果：

```
composer require spatie/laravel-analytics
```

Composer 所做的工作在于，拉取你所需版本包，下载到  vendor 目录，上述命令执行完毕， 包内所有类和文件被加载进项目，你就可以马上使用它们了，每次当你再次执行`composer update`，Composer将会重新获取（译者注 通常从 composer 仓库拉取）更新该包，并且自动更新位于你项目  vendor 目录下的文件。

在 Laravel 项目中使用某些 Laravel 包 需要以下额外几个步骤：

- 注册服务提供者
- 注册别名 / 门面
- 发布资源

如果你看过"Spatie 包安装说明"你会发现，在继续下一步这前，项目配置必须注册服务提供者和一个门面，是一个很好的习惯，这个步骤由 Taylor Otwell 定义，只是一个非必要条件， Dries Vints，且达到无论何时你决定引入一个新包或移除包，服务提供者和门面皆可被自动发现。
重温 Taylor 的新特性声明：在媒体上(https://medium.com/@taylorotwell/package-auto-discovery-in-laravel-5-5-ea9e3ab20518).

## 什么是服务提供者和门面？

> 服务提供者负责将事物绑定到 Laravel 的服务容器中，并通知 Laravel 在哪里加载包资源，例如视图，配置和本地化文件。-- laravel.com 文档

你可以在上面阅读有关服务提供者的更多信息 官方文档(https://learnku.com/docs/laravel/5.4/providers).

> 门面为应用程序服务容器中可用的类提供 "static" 接口 -- laravel.com 文档

你可以在上面阅读更多有关门面的信息 官方文档(https://learnku.com/docs/laravel/5.4/facades).

在查找和安装 / 更新不同的扩展包时，Composer 会触发你可以订阅的多个事件并运行你自己的一段代码甚至是命令行可执行文件，其中一个有趣的事件称为`post-autoload-dump`。 在 composer 生成项目中自动加载的最终类列表之后直接触发，此时 Laravel 已经可以访问所有类，并且应用程序已准备好使用加载到其中的所有包类。

Laravel在主composer.json文件中订阅此事件：

```
"scripts": {
   "post-autoload-dump": [
       "Illuminate\\Foundation\\ComposerScripts::postAutoloadDump",
       "@php artisan package:discover"
   ]
}
```

首先它调用`postAutoloadDump()`静态方法，此方法会清理缓存的服务或之前发现的包，另一个是它运行`package:discover`的artisan命令，这就是 Laravel 可以自动发现的秘密。


## 包自动发现

`Illuminate\Foundation\Console\PackageDiscoverCommand` 在 `Illuminate\Foundation\PackageManifest` 类中调用 `build()` 方法，该类是 Laravel 发现已安装包的地方。

PackageManifest 在应用程序引导程序的早期注册到容器中，完全来自 `Illuminate\Foundation\Application::registerBaseServiceProviders()`，此方法在创建 Laravel 应用程序的新实例后直接运行。

在`build()`方法中，Laravel 查找 `vendor/composer/installed.json` 文件，它由 composer 生成并保存一个完整的映射，其中包含 composer 安装的所有扩展包的 composer.json 文件内容， Laravel 映射该文件的内容并搜索包含`extra.laravel`部分的包：

```
"extra": {
   "laravel": {
       "providers": [
           "Barryvdh\\Debugbar\\ServiceProvider"
       ],
       "aliases": {
           "Debugbar": "Barryvdh\\Debugbar\\Facade"
       }
   }
}
```

它首先收集该部分的内容，然后查看主composer.json文件下的`extra.laravel.dont-discover`的内容，看看你是否决定不自动发现某些包或所有包：

```
"extra": {
   "laravel": {
       "dont-discover": [
           "barryvdh/laravel-debugbar"
       ]
   }
}
```

你可以在数组中添加 * 以指示 laravel 完全停止自动注册。

## 现在 Laravel 收集了有关扩展包的信息

是的，一旦获得所需要的信息，它将在 `bootstrap/cache/packages.php` 中编写一个 PHP 文件：

```
<?php return array (
 'barryvdh/laravel-debugbar' =>
 array (
   'providers' =>
   array (
     0 => 'Barryvdh\\Debugbar\\ServiceProvider',
   ),
   'aliases' =>
   array (
     'Debugbar' => 'Barryvdh\\Debugbar\\Facade',
   ),
 ),
);
```

## 包注册

Laravel 有两个 bootstrappers，在 HTTP 或控制台内核启动时使用：

- `\Illuminate\Foundation\Bootstrap\RegisterFacades`

- `\Illuminate\Foundation\Bootstrap\RegisterProviders`

第一个使用`Illuminate\Foundation\AliasLoader`将所有门面加载到应用程序中，现在 Laravel 将查看 packages.php 生成的文件并提取包中希望 Laravel 自动注册的所有别名并注册这些别名。 它使用`PackageManifest::aliases()`方法来收集这些信息。

```
// 在 RegisterFacades::bootstrap()

AliasLoader::getInstance(array_merge(
   $app->make('config')->get('app.aliases', []),
   $app->make(PackageManifest::class)->aliases()
))->register();
```

如你所见，从`config/app.php`文件加载的别名与从 PackageManifest 类加载的别名合并。
类似地，Laravel 在启动时注册服务提供者，`RegisterProviders`中的`bootstrapper 调用`Foundation\Application` 的`registerConfiguredProviders()` 方法，并且 Laravel 在这会收集所有应该自动注册的包提供者并注册它们。

```
$providers = Collection::make($this->config['app.providers'])
               ->partition(function ($provider) {
                   return Str::startsWith($provider, 'Illuminate\\');
               });

$providers->splice(1, 0, [$this->make(PackageManifest::class)->providers()]);
(new ProviderRepository($this, new Filesystem, $this->getCachedServicesPath()))
                   ->load($providers->collapse()->toArray());
```

在这里，我们在 Illuminate 服务提供者和可能在你的配置文件中的任何其他服务提供者之间注入自动发现的服务提供者，这样我们确保你可以通过在配置文件中重新注册它们来覆盖扩展包服务提供者，并且 Illuminate 组件将会在尝试加载任何其他服务提供者之前加载。

>原文作者：Summer
>转自链接：https://learnku.com/laravel/t/33459