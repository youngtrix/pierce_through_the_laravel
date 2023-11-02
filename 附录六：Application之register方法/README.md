# 附录六：Application之register方法

源文件路径：vendor\laravel\framework\src\Illuminate\Foundation\Application.php

方法名：register

```php
/**
 * Register a service provider with the application.
 *
 * @param  \Illuminate\Support\ServiceProvider|string  $provider
 * @param  bool   $force
 * @return \Illuminate\Support\ServiceProvider
 */
public function register($provider, $force = false)
{
	if (($registered = $this->getProvider($provider)) && ! $force) {
		return $registered;
	}

	// If the given "provider" is a string, we will resolve it, passing in the
	// application instance automatically for the developer. This is simply
	// a more convenient way of specifying your service provider classes.
	if (is_string($provider)) {
		$provider = $this->resolveProvider($provider);
	}

	$provider->register();

	// If there are bindings / singletons set as properties on the provider we
	// will spin through them and register them with the application, which
	// serves as a convenience layer while registering a lot of bindings.
	if (property_exists($provider, 'bindings')) {
		foreach ($provider->bindings as $key => $value) {
			$this->bind($key, $value);
		}
	}

	if (property_exists($provider, 'singletons')) {
		foreach ($provider->singletons as $key => $value) {
			$this->singleton($key, $value);
		}
	}

	$this->markAsRegistered($provider);

	// If the application has already booted, we will call this boot method on
	// the provider class so it has an opportunity to do its boot logic and
	// will be ready for any usage by this developer's application logic.
	if ($this->isBooted()) {
		$this->bootProvider($provider);
	}

	return $provider;
}
```

第一步：

```php
if (($registered = $this->getProvider($provider)) && ! $force) {
    return $registered;
}
```

这里主要就是调用`getProvider($provider)`，作用就是检查前面是否已经加载了我们需要的provider，如果是，则直接返回这个provider，不用再加载。如果不是，返回null。

getProvider：

```php
public function getProvider($provider)
{
	return array_values($this->getProviders($provider))[0] ?? null;
}
```

> array_values函数：返回数组的value值不包含key

getProviders：

```php
public function getProviders($provider)
{
	$name = is_string($provider) ? $provider : get_class($provider);

	return Arr::where($this->serviceProviders, function ($value) use ($name) {
		return $value instanceof $name;
	});
}
```

这里分两步：

- a) 如果参数$provider是字符串，直接赋值给变量$name，如果不是，使用get_class获取该参数$provider所属类的路径，返回

- b) 调用工具类Arr的where方法

```php
public static function where($array, callable $callback)
{
	return array_filter($array, $callback, ARRAY_FILTER_USE_BOTH);
}
```

这里简单解释就是去$this->serviceProviders中找匹配$name值的项，以数组形式返回。

继续看后面的代码：

```php
if (is_string($provider)) {
	$provider = $this->resolveProvider($provider);
}
```

如果$provider是字符串，则执行if中的代码，我们继续看resolveProvider方法：

```php
public function resolveProvider($provider)
{
	return new $provider($this);
}
```

很简单，直接new一个provider对象并返回。

这里我们需要注意new对象同时，传入了$this变量值作为参数。我们猜测，这个$this一定会在类的构造函数中使用到，于是我们去找一个具体的provider类型的类，比如RoutingServiceProvider，看看情况是不是这样。

然而通过查看RoutingServiceProvider类的源码，我们并没有找到这个类的构造行数__contruct()，既然这样我们只能"向上"继续查找，注意到RoutingServiceProvider类继承了父类ServiceProvider，我们直接跳转到ServiceProvider，终于在这个类中看到了它的构造函数：

```php
public function __construct($app)
{
	$this->app = $app;
}
```

这样就能解释通了，在执行register方法实例化所有"ServiceProvider类型类"的时候，把当前所有对象共享的容器也""挂载"进去。因为容器需要知道哪些ServiceProvider类是已经加载过的，哪些是没有加载的。而容器是联结所有对象的纽带，因此操作共享容器app，就是最好的方式。

> ServiceProvider类型类：是指所有继承了ServiceProvider基类的类，实际上在Laravel框架中要编写一个新的Provider类，默认都需要继承ServiceProvider基类，因为只有这样Laravel框架在执行过程中才能正确解析出这个Provider类。

接下来这一句，是非常关键的一句：

```php
$provider->register();
```

执行ServiceProvider类型类上的`register`方法，仍以RouteServiceProvider为例：

```php
public function register()
{
	$this->registerRouter();
	$this->registerUrlGenerator();
	$this->registerRedirector();
	$this->registerPsrRequest();
	$this->registerPsrResponse();
	$this->registerResponseFactory();
	$this->registerControllerDispatcher();
}
```

我们看到，这里就是RouteServiceProvider类的核心逻辑了。

由于本节我们重点关注的是Application类中的register方法，这里我们不再对RouteServiceProvider这个类中的register方法中的所有子方法展开讲解。

继续往下看后面的代码：

```php
if (property_exists($provider, 'bindings')) {
	foreach ($provider->bindings as $key => $value) {
		$this->bind($key, $value);
	}
}

if (property_exists($provider, 'singletons')) {
	foreach ($provider->singletons as $key => $value) {
		$this->singleton($key, $value);
	}
}
```

这里，我们能分析出来，代码是在检测上一步实例化出来的类(赋值给$provider)中是否包含bindings和singletons成员，如果有包含，就将bindings和singletons成员中的包含的内容解析出来，调用Application类的bind和singleton方法。

然而Application类本身并没有bind和singleton方法，那这两个方法在哪呢？

Application类继承了Container类，所以这两个方法实际就是容器类中的方法。在【附录四】中，我们已经给大家讲解过，bind和singleton正是容器执行绑定的两个常用方式。

继续看后面的代码：

```php
$this->markAsRegistered($provider);
	
if ($this->isBooted()) {
	$this->bootProvider($provider);
}

return $provider;
```

`markAsRegistered`方法，是把这个已经注册完成的provider对象进行保存(存入serviceProvider数组中)。后面再有需求的时候不需要再次进行注册，这里正好和前面我们讲的第一步相"呼应"。除了这个动作之外，还把对应的类名存储到loadedProviders数组中，以备后用。

```php
protected function markAsRegistered($provider)
{
	$this->serviceProviders[] = $provider;

	$this->loadedProviders[get_class($provider)] = true;
}
```

之后，判断当前application这个容器对象是都已经启动过了，简单说，就是application的boot方法是否已经执行过了。如果执行过了(即容器已经启动)，则继续调用`bootProvider`方法。

这里要怎么理解呢？

我们可以这么分析：既然程序执行`register`方法时，已经走到了这里，那就是说当前这个provider是肯定没有注册过的。那么首次注册这个provider，就需要去触发这个provider对象上的boot方法执行。

bootProvider：

```php
protected function bootProvider(ServiceProvider $provider)
{
	if (method_exists($provider, 'boot')) {
		return $this->call([$provider, 'boot']);
	}
}
```

这个强制执行provider对象身上boot方法的规则，大家可以理解为它就是Laravel处理ServiceProvider类的方式。实际上框架提供的文档中，也会对此有明确的说明。

除了在容器主动执行`register`方法时会触发执行$provider身上定义的boot方法外，在应用启动阶段也会遍历容器定义的所有serviceProviders，然后触发执行这些类身上的bootProvider方法，关于这一点，大家可以在Application类中的boot方法中，找到相同功能的代码：

```php
public function boot()
{
    ...
    array_walk($this->serviceProviders, function ($p) {
        $this->bootProvider($p);
    });
    ...
}
```

最后一句很好理解，直接返回$provider对象。

