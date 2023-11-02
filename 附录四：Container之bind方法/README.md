# 附录四：Container之bind方法

源文件路径：vendor\laravel\framework\src\Illuminate\Container\Container.php

方法名：bind

````php
/**
 * Register a binding with the container.
 *
 * @param  string  $abstract
 * @param  \Closure|string|null  $concrete
 * @param  bool  $shared
 * @return void
 */
public function bind($abstract, $concrete = null, $shared = false)
{
    $this->dropStaleInstances($abstract);

    // If no concrete type was given, we will simply set the concrete type to the
    // abstract type. After that, the concrete type to be registered as shared
    // without being forced to state their classes in both of the parameters.
    if (is_null($concrete)) {
        $concrete = $abstract;
    }

    // If the factory is not a Closure, it means it is just a class name which is
    // bound into this container to the abstract type and we will just wrap it
    // up inside its own Closure to give us more convenience when extending.
    if (! $concrete instanceof Closure) {
        $concrete = $this->getClosure($abstract, $concrete);
    }

    $this->bindings[$abstract] = compact('concrete', 'shared');

    // If the abstract type was already resolved in this container we'll fire the
    // rebound listener so that any objects which have already gotten resolved
    // can have their copy of the object updated via the listener callbacks.
    if ($this->resolved($abstract)) {
        $this->rebound($abstract);
    }
}
````

bind方法是Laravel实现绑定机制的核心方法之一，另外还有很多其他的方法：

1) bind把接口和实现类绑定，当make解析接口的时候创建其实现类的实例对象

2) singleton把接口和其实现类绑定，当第一次make解析的时候创建实例，后面都返回该实例不再创建

3) instance把接口和其实现类的实例绑定，直接绑定实例对象

4) 上下文绑定

5) 自动绑定

6) tag绑定

7) extends扩展绑定

本节我们仅聚焦bind方法的实现。

**参数**

1) 首先明确第一个参数$abstract，简单说就是id，可以当作是存入容器中的名字，它可以使一个字符串，一个类甚至是一个接口。

2) 第二个参数$concrete简单说就是真实的值，可以当作是一个真正存入容器的实体。他可以是一个实现类，实例或者一个闭包。

3) 第三个参数控制shared的值。

方法体：
1、绑定前先清空instances和aliases中存在的同名字的服务:

```php
protected function dropStaleInstances($abstract)
{
	unset($this->instances[$abstract], $this->aliases[$abstract]);
}
```

2、然后判断第二个参数$concrete是不是空，如果是空，则将abstract变量值直接赋值给$concrete

3、判断$concrete是否是一个闭包，不是则调用getClosure，返回一个闭包便于后面操作

我们看一下getClosure方法：

```php
protected function getClosure($abstract, $concrete)
{
	return function ($container, $parameters = []) use ($abstract, $concrete) {
		if ($abstract == $concrete) {
			return $container->build($concrete);
		}

		return $container->resolve(
			$concrete, $parameters, $raiseEvents = false
		);
	};
}
```

大家可以很清楚地看到，这里再次调用了build和resolve方法，并且将解析出来的对象当作闭包函数的返回值。

回到bind方法，上面 $concrete 得到一个闭包函数后，调用 compact 把 $concrete 和 $shard （第三个参数判断是否 shared）组成一个 key 分别为 concrete 和 shared 的数组，存入 binding 数组中，而 binding 数组的 key 是当前的抽象类：

```php
$this->bindings[$abstract] = compact('concrete', 'shared');
```

处理后的结构是这样的：

```php
$binding[$abstract] => [
    'concrete' => function($container,$parameters=[]),//getClosure()得到的
    'shared' => true/false,//shared的值是bind的第三个参数
]
```

最后这里，如果当前的抽象类曾经被解析过。那再次绑定的时候，我们要使用 rebound 函数触发 执行reboundCallbacks 数组中的回调函数：

```php
if ($this->resolved($abstract))
{
    $this->rebound($abstract);
}
```

如何判断当前的$abstract曾经被解析过，我们看下resolved函数，两个条件：

1、当前的resolved数组中是否存在$abstract

2、instances数组中是否存在对应的值，注意：在bind方法的第一句`$this->dropStaleInstances($abstract);`，那时我们已经清空了instances对应的abstract值，因此这里$abstract实际是找到的$abstract的别名。对应resolved方法开始的这几条语句。

```php
public function resolved($abstract)
{
    if ($this->isAlias($abstract)) {
        $abstract = $this->getAlias($abstract);
    }

    return isset($this->resolved[$abstract]) ||
isset($this->instances[$abstract]);
}
```



> 本节内容部分参考自LearnKu社区，以下为转载详情：
> 作者：HarveyNorman
> 链接：https://learnku.com/articles/41504

