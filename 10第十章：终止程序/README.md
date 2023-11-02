# 第十章：终止程序

现在，我们来到框架执行过程中的最后一个阶段（第六阶段），这个阶段对应下面这一行语句：

```php
$kernel->terminate($request, $response);
```

> public/index.php

显然terminate方法是属于类\Illuminate\Foundation\Http\Kernel的：

```php
/**
 * Call the terminate method on any terminable middleware.
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  \Illuminate\Http\Response  $response
 * @return void
 */
public function terminate($request, $response)
{
	$this->terminateMiddleware($request, $response);

	$this->app->terminate();
}
```

> vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php

继续追踪terminateMiddleware方法：

```php
 /**
   * Call the terminate method on any terminable middleware.
   *
   * @param  \Illuminate\Http\Request  $request
   * @param  \Illuminate\Http\Response  $response
   * @return void
   */
protected function terminateMiddleware($request, $response)
{
	$middlewares = $this->app->shouldSkipMiddleware() ? [] : array_merge(
		$this->gatherRouteMiddleware($request),
		$this->middleware
	);

	foreach ($middlewares as $middleware) {
		if (! is_string($middleware)) {
			continue;
		}

		[$name] = $this->parseMiddleware($middleware);

		$instance = $this->app->make($name);

		if (method_exists($instance, 'terminate')) {
			$instance->terminate($request, $response);
		}
	}
}
```

> vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php

这个方法体，是在处理所有中间件中的terminate方法，框架定义了规则：中间件中定义的terminate方法，会在这个阶段执行。

接下来继续看最后这行语句：

```php
$this->app->terminate();
```

> vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php

我们知道$this->app是指向应用唯一的容器实例，对应的类文件是：\Illuminate\Foundation\Application.php，因此很容易就能找到terminate方法：

```php
/**
 * Terminate the application.
 *
 * @return void
 */
public function terminate()
{
	foreach ($this->terminatingCallbacks as $terminating) {
		$this->call($terminating);
	}
}
```

> vendor/laravel/framework/src/Illuminate/Foundation/Application.php

这里是执行app容器身上绑定的terminatingCallback事件，而terminatingCallbacks中的内容是通过调用Application类中的terminating方法进行"装填"的。那什么时候框架会调用terminating方法呢，通过在phpstorm IDE中对">terminating("字符串进行全局查找，我们并没有发现哪里有主动执行这个方法，我们猜测这里是需要用户自己去主动调用的。

由于我们访问的默认主页这个URL对应的controller和相关的其他所有代码中都没有包含这部分处理，因此即使在terminate方法中主动加exit，也不会影响程序的执行：

```php
public function terminate()
{
	exit('ccc');
	foreach ($this->terminatingCallbacks as $terminating) {
		$this->call($terminating);
	}
}
```

> vendor/laravel/framework/src/Illuminate/Foundation/Application.php

程序并不会输出ccc，而是正常显示首页：

![](../images/test_11.png)