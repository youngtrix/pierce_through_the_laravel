# 附录二：Container之instance方法

源文件路径：vendor/laravel/framework/src/Illuminate/Container/Container.php

方法名：instance

````php
/**
 * Register an existing instance as shared in the container.
 *
 * @param  string  $abstract
 * @param  mixed   $instance
 * @return mixed
 */
public function instance($abstract, $instance)
{
    $this->removeAbstractAlias($abstract);

    $isBound = $this->bound($abstract);

    unset($this->aliases[$abstract]);

    // We'll check to determine if this type has been bound before, and if it has
    // we will fire the rebound callbacks registered with the container and it
    // can be updated with consuming classes that have gotten resolved here.
    $this->instances[$abstract] = $instance;

    if ($isBound) {
        $this->rebound($abstract);
    }

    return $instance;
}
````

我们进入removeAbstractAlias方法内部，进行var_dump中断测试，如下：

```php
/**
 * Remove an alias from the contextual binding alias cache.
 *
 * @param  string  $searched
 * @return void
 */
protected function removeAbstractAlias($searched)
{
	if (! isset($this->aliases[$searched])) {
		var_dump($searched);exit;
	    return;
	}

	foreach ($this->abstractAliases as $abstract => $aliases) {
	    foreach ($aliases as $index => $alias) {
            if ($alias == $searched) {
                unset($this->abstractAliases[$abstract][$index]);
            }
	    }
	}
}
```

测试结果：

```php
string(4)"path"
```

继续做如下的var_dump中断测试：

```php
/**
 * Remove an alias from the contextual binding alias cache.
 *
 * @param  string  $searched
 * @return void
 */
protected function removeAbstractAlias($searched)
{
	if (! isset($this->aliases[$searched])) {
	    return;
	}

	foreach ($this->abstractAliases as $abstract => $aliases) {
		var_dump($searched);exit;
	    foreach ($aliases as $index => $alias) {
            if ($alias == $searched) {
                unset($this->abstractAliases[$abstract][$index]);
            }
	    }
	}
}
```

测试发现，首页仍然能正常访问。

上面两次中断测试，说明默认情况下，foreach循环体中的语句并不会被触发运行。这里我们能看到，这个方法的主要目的，就是检测abstractAliases成员变量中是否包含$searched这个键，如果包含则主动删除。

instance方法中剩余的代码，做的工作可以简述如下：

将$instance装填到容器的instances成员变量上，注意这个变量是一个数组，键值就是传递进来的$abstract。

之后判断$abstract是否被绑定过，如果是，则执行绑定回调事件。

最后返回$instance变量

```php
$this->instances[$abstract] = $instance;

if ($isBound) {
    $this->rebound($abstract);
}

return $instance;
```

最后我们简单总结一下：

1) instance方法执行的主要动作就是往容器对象的成员变量instances身上绑定新的键值对

2) 绑定之前会移除成员变量abstractAliases中的相应键值(如果有)

3) 如果成员变量aliases中有相应键值对也移除

4) 如果当前要绑定的这个键已经被绑定过，主动运行当前键上保存的绑定回调事件