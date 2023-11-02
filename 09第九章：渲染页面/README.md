# 第九章：渲染页面

这一章，我们实际上只需要分析一条语句：

```php
$response->send();
```

> public/index.php

我们知道，response实际上是\symfony\http-foundation\Response类的一个实例，因此要找到send方法很容易：

```php
/**
 * Sends HTTP headers and content.
 *
 * @return $this
 */
public function send()
{
	$this->sendHeaders();
	$this->sendContent();

	if (\function_exists('fastcgi_finish_request')) {
		fastcgi_finish_request();
	} elseif (!\in_array(\PHP_SAPI, ['cli', 'phpdbg'], true)) {
		static::closeOutputBuffers(0, true);
		flush();
	}

	return $this;
}
```

> vendor/symfony/http-foundation/Response.php

接下来，就是继续追踪sendHeaders方法和sendContents方法：

sendHeaders:

```php
/**
 * Sends HTTP headers.
 *
 * @return $this
 */
public function sendHeaders()
{
	// headers have already been sent by the developer
	if (headers_sent()) {
		return $this;
	}

	// headers
	foreach ($this->headers->allPreserveCaseWithoutCookies() as $name => $values) {
		$replace = 0 === strcasecmp($name, 'Content-Type');
		foreach ($values as $value) {
			header($name.': '.$value, $replace, $this->statusCode);
		}
	}

	// cookies
	foreach ($this->headers->getCookies() as $cookie) {
		header('Set-Cookie: '.$cookie, false, $this->statusCode);
	}

	// status
	header(sprintf('HTTP/%s %s %s', $this->version, $this->statusCode, $this->statusText), true, $this->statusCode);

	return $this;
}
```

> vendor/symfony/http-foundation/Response.php

这个方法里面的代码容易理解，就是设置HTTP的请求头，至于请求头中的信息来源，框架在前面程序的处理中已经"装填"好了。

sendContents:

```php
/**
 * Sends content for the current web response.
 *
 * @return $this
 */
public function sendContent()
{
	echo $this->content;

	return $this;
}
```

> vendor/symfony/http-foundation/Response.php

这个方法中的代码简单解释就是：输出当前对象的content成员变量值并返回当前对象。至于content变量的内容是什么时候赋值的，这和上一章我们讲的route类中的run方法相关。

