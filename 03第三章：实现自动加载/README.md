# 第三章：实现自动加载
```php
<?php

/**
 * Laravel - A PHP Framework For Web Artisans
 *
 * @package  Laravel
 * @author   Taylor Otwell <taylor@laravel.com>
 */

define('LARAVEL_START', microtime(true));
...
... ...
```

第一行代码，只是简单定义了一个全局常量，并将其值设置为当前时间的时间戳 + 微秒数(小数点后4位)。这是方便在合适的地方计算代码运行时间的一个常量。值得注意的是，我们在框架内全局搜索"LARAVEL_START"关键词时，并没有任何发现，说明这行代码是框架预留的，在5.8.38版本中并没有实际的用处。


>mircotime函数如果不加任何参数，返回的是一个空格隔开的字符串，时间戳 + 微秒数）

接下来这行代码：

```php
require __DIR__.'/../vendor/autoload.php';
```

vendor目录实际上是第三方库的目录，使用composer来进行包管理。
因为包含了这个autoload.php文件，在之后的所有php代码中任何使用vendor目录中的类语句(这些包都是按照文件夹独立开来的)，直接引用类名即可，无需再显示包含类文件本身。

比如，我们很快会阅读的app.php（bootstrap/app.php）中，第14行 ~ 16行代码：

```php
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);
```

我们看到，这里直接实例化了一个Application对象，如果没有之前的包含autoload.php文件语句，这里肯定会报错。一个简单的验证方法是，注释`require __DIR__.'/../vendor/autoload.php`语句，直接刷新页面，报错：
```
Fatal error: Uncaught Error: Class 'Illuminate\Foundation\Application' not found in /home/vagrant/code/blog5/bootstrap/app.php:14 Stack trace: #0 /home/vagrant/code/blog5/public/index.php(38): require_once() #1 {main} thrown in /home/vagrant/code/blog5/bootstrap/app.php on line 14
```

继续追踪autoload.php中的代码，最终可以发现，composer使用了`spl_autoload_register`这个核心函数，来完成对PHP类的自动加载。

这里，我们不再对composer的包自动加载机制做过多的讲解，感兴趣的读者可以阅读下面这两篇文章。

**参考链接：**

- https://segmentfault.com/a/1190000014948542
- https://www.cnblogs.com/a609251438/p/12659934.html