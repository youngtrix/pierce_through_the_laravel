# 附录十：Route之run方法

源文件路径：vendor\laravel\framework\src\Illuminate\Routing\Route.php

方法名：run

```php
/**
 * Run the route action and return the response.
 *
 * @return mixed
 */
public function run()
{
	$this->container = $this->container ?: new Container;

	try {
		if ($this->isControllerAction()) {
			return $this->runController();
		}

		return $this->runCallable();
	} catch (HttpResponseException $e) {
		return $e->getResponse();
	}
}
```

回到`Handle`类的`dispatchToRoute`方法:

```php
public function dispatchToRoute(Request $request)
{
	return $this->runRoute($request, $this->findRoute($request));
}
```

`findRoute`方法找到一条匹配的路由之后，接着就运行`Router`类的`runRoute`方法了：

```php
/**
 * Return the response for the given route.
 *
 * @param  \Illuminate\Http\Request  $request
 * @param  \Illuminate\Routing\Route  $route
 * @return \Illuminate\Http\Response|\Illuminate\Http\JsonResponse
 */
protected function runRoute(Request $request, Route $route)
{
	$request->setRouteResolver(function () use ($route) {
		return $route;
	});

	$this->events->dispatch(new Events\RouteMatched($route, $request));

	return $this->prepareResponse($request,
		$this->runRouteWithinStack($route, $request)
		);
}
```

Route类的`run`方法，就是`runRoute`中`$this->runRouteWithinStack()`语句调用的：

```php
protected function runRouteWithinStack(Route $route, Request $request)
{
	$shouldSkipMiddleware = $this->container->bound('middleware.disable') &&
					$this->container->make('middleware.disable') === true;

	$middleware = $shouldSkipMiddleware ? [] : $this->gatherRouteMiddleware($route);

	return (new Pipeline($this->container))
		->send($request)
		->through($middleware)
		->then(function ($request) use ($route) {
			return $this->prepareResponse(
				$request, $route->run()
				);
			});
}
```

现在我们来逐行分析这个方法中的语句：

第一行：`$this->container = $this->container ?: new Container;`很简单，这里就是对本类中的成员变量container进行赋值操作，这里之所以需要赋值，是因为在Route类的其他方法中使用到了这个`$this->container`对象的方法。

紧接着，程序就进入一个try catch结构中：

```php
try {
	if ($this->isControllerAction()) {
		return $this->runController();
	}

	return $this->runCallable();
} catch (HttpResponseException $e) {
	return $e->getResponse();
}
```

首先代码调用`isControllerAction`方法判断当前是否是Controller上的action：

```php
protected function isControllerAction()
{
    return is_string($this->action['uses']);
}
```

这个方法中的逻辑虽然简洁，但是这里涉及到`RouteCollection`类是怎样创建一个`Route`的，关于这一点读者可以回到【附录九】中，查看关于"路由注册"的部分。

接着如果`isControllerAction()`返回值为true，则执行`runController()`，然后返回其结果。否则执行`runCallable`并返回其结果。

runController：

```php
protected function runController()
{
	return $this->controllerDispatcher()->dispatch(
		$this, $this->getController(), $this->getControllerMethod()
	);
}
```

继续追踪`controllerDispatcher()`：

```php
public function controllerDispatcher()
{
	if ($this->container->bound(ControllerDispatcherContract::class)) {
		return $this->container->make(ControllerDispatcherContract::class);
	}

	return new ControllerDispatcher($this->container);
}
```

这个方法很明显返回的是一个`ControllerDispatcher`类，继续追踪`dispatch`方法：

```php
public function dispatch(Route $route, $controller, $method)
{
	$parameters = $this->resolveClassMethodDependencies(
		$route->parametersWithoutNulls(), $controller, $method
	);

	if (method_exists($controller, 'callAction')) {
		return $controller->callAction($method, $parameters);
	}

	return $controller->{$method}(...array_values($parameters));
}
```

到这里，我们最终发现，程序会去检测对应的controller类中是否包含`callAction`方法，是则直接执行controller类上的`callAction`方法，否则执行controller类上的`method`方法。回到调用`dispatch()`的语句中，我们会发现参数值$controller和$method是通过调用类中的`getController()`和`getControllerMethod()`方法得到的。

接下来我们来看另一个方法`runCallable`：

```php
/**
 * Run the route action and return the response.
 *
 * @return mixed
 */
protected function runCallable()
{
	$callable = $this->action['uses'];

	return $callable(...array_values($this->resolveMethodDependencies(
	    $this->parametersWithoutNulls(), new ReflectionFunction($this->action['uses'])
	)));
}
```

可以看到，这里直接取出当前对象中的成员变量`action`的uses键值赋值给$callable，然后直接执行$callable方法。

这种情况正是因为在路由文件中，支持下面这种定义：

```php
Route::get('/', function () {
    return view('welcome');
});
```

即某个路由直接对应到一个闭包函数上去。读者可以自行做vard_dump中断测试，在这种情况下，`route`对象中的各个成员变量究竟是什么。