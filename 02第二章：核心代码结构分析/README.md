# 第二章：核心代码结构分析

入口文件index.php的源码：

```php
<?php

/**
 * Laravel - A PHP Framework For Web Artisans
 *
 * @package  Laravel
 * @author   Taylor Otwell <taylor@laravel.com>
 */

define('LARAVEL_START', microtime(true));

/*
|--------------------------------------------------------------------------
| Register The Auto Loader
|--------------------------------------------------------------------------
|
| Composer provides a convenient, automatically generated class loader for
| our application. We just need to utilize it! We'll simply require it
| into the script here so that we don't have to worry about manual
| loading any of our classes later on. It feels great to relax.
|
*/

require __DIR__.'/../vendor/autoload.php'; // 第一阶段

/*
|--------------------------------------------------------------------------
| Turn On The Lights
|--------------------------------------------------------------------------
|
| We need to illuminate PHP development, so let us turn on the lights.
| This bootstraps the framework and gets it ready for use, then it
| will load up this application so that we can run it and send
| the responses back to the browser and delight our users.
|
*/

$app = require_once __DIR__.'/../bootstrap/app.php'; // 第二阶段

/*
|--------------------------------------------------------------------------
| Run The Application
|--------------------------------------------------------------------------
|
| Once we have the application, we can handle the incoming request
| through the kernel, and send the associated response back to
| the client's browser allowing them to enjoy the creative
| and wonderful application we have prepared for them.
|
*/

$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class); // 第三阶段
 
$response = $kernel->handle(  // 第四阶段
    $request = Illuminate\Http\Request::capture()
);

$response->send(); // 第五阶段

$kernel->terminate($request, $response);  // 第六阶段

```

> public/index.php

按照上面代码中的注释，我们简要说明一下6个阶段，每个阶段大致都做了什么：

1、实现项目第三方库文件的自动加载

2、引入框架app启动文件

这个阶段框架做了大量的准备工作，包括：

  - 2.1、实例化容器类
    - 2.1.1 设置基础目录路径
    - 2.1.2 注册基础绑定
    - 2.1.3 注册服务提供者
    - 2.1.4 注册核心别名类
  - 2.2、向容器中"装填"处理HTTP请求的核心类HttpKernel
  - 2.3、向容器中"装填"处理命令行的核心类ConsoleKernel
  - 2.4、向容器中"装填"处理异常的核心类

3、从app中得到容器中装填好的处理HTTP请求的类赋值给变量kernel

4、运行HttpKernel类的handle方法，处理接收到的HTTP请求，得到需要返回的对象

5、运行返回对象的send方法渲染页面

6、继续运行HttpKernel类的terminate方法，执行绑定在容器中可能要执行的其他动作

从下一章开始，我们将按照上面的次序从上到下为大家详细阐述每个阶段框架所做的工作。