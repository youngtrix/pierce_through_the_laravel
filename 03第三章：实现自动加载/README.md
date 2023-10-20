## 第三章：实现自动加载
```php
<?php
use Illuminate\Contracts\Http\Kernel;
use Illuminate\Http\Request;
...
... ...
```
前面这两行代码，是php中的命名空间引入，这样做的目的是为后面引用相关类做"准备"。读者阅读后面的代码就能看到，在后面的代码中确实有引用到Kernel类和Request类。
接下来这一行代码：
```
define('LARAVEL_START', microtime(true));
```
这里是简单定义了一个全局常量，并将其值设置为当前时间的时间戳 + 微秒数(小数点后4位)。这是方便在合适的地方计算代码运行时间的一个常量。值得注意的是，我们在框架内全局搜索"LARAVEL_START"关键词时，除了常量的定义外并没有发现有其他地方引用了这个常量值，说明这行代码是框架预留的，在10.27.0版本中并没有实际的用处。

>mircotime函数如果不加任何参数，返回的是一个空格隔开的字符串，时间戳 + 微秒数）

继续看后面的代码：
```php
if (file_exists($maintenance = __DIR__.'/../storage/framework/maintenance.php')) {
    require $maintenance;
}
```
这是框架在检测应用是否处于维护模式，若处于维护模式则执行if中的语句。 接下来这行代码：
```
require __DIR__.'/../vendor/autoload.php';
```
vendor目录实际上是第三方库的目录，使用composer来进行包管理。 因为包含了这个autoload.php文件，在之后的所有php代码中任何使用vendor目录中的类语句(这些包都是按照文件夹独立开来的)，直接引用类名即可，无需再显示包含类文件本身。

比如，我们很快会阅读的app.php（bootstrap/app.php）中，第14行 ~ 16行代码：

```php
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);
```

我们看到，这里直接实例化了一个Application对象，如果没有之前的包含autoload.php文件语句，这里肯定会报错。一个简单的验证方法是，注释`require __DIR__.'/../vendor/autoload.php`语句，直接刷新页面，报错：
```
( ! )Fatal error: Uncaught Error: Class 'Illuminate\Foundation\Application' not found in /home/vagrant/code/blog/bootstrap/app.php:14 
( ! )Error: Class "Illuminate\Foundation\Application" not found in /home/vagrant/code/blog/bootstrap/app.php on line 14
...
... ...
```

继续追踪autoload.php中的代码，最终可以发现，composer使用了`spl_autoload_register`这个核心函数，来完成对PHP类的自动加载。

这里，我们不再对composer的包自动加载机制做过多的讲解，感兴趣的读者可以阅读下面这两篇文章。

**参考链接：**

- https://segmentfault.com/a/1190000014948542
- https://www.cnblogs.com/a609251438/p/12659934.html