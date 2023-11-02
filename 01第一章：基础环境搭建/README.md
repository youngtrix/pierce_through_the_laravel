# 第一章：基础环境搭建

分析代码前，有必要主动告知大家本文分析的Laravel框架是哪个版本。以防有读者在核对代码时，发现作者引用的代码和其本地的代码版本不一致，由此引发不必要的误会。

对于框架版本不一致的情况，强烈建议您在本地安装一个和作者分析的版本一致的Laravel应用。作者相信通读这个系列的文章后，对于新版本框架的代码，您可以依葫芦画瓢，依靠自己的力量去理解新旧框架之间的不同部分。

下面是本文安装的Laravel框架版本和php版本：
## Laravel
```
$ php artisan
Laravel Framework 5.8.38

Usage:
  command [options] [arguments]
...
... ...
```

> 如果读者本地的Laravel版本不是5.8.38，建议以git方式下载本书，同时切换到相应的分支(除main分支外，其他分支名称均包含Laravel版本号)，目前主分支main对应的Laravel版本号为：5.8.38。

## PHP
```
vagrant@homestead:~$ php7.2 --version
PHP 7.2.34-24+ubuntu20.04.1+deb.sury.org+1 (cli) (built: Aug 26 2021 15:55:49) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.2.34-24+ubuntu20.04.1+deb.sury.org+1, Copyright (c) 1999-2018, by Zend Technologies
```
> 笔者本机上采用的运行环境为Homestead, Homestead依赖Virtualbox虚拟机需要前置安装Vagrant、Virtualbox等软件。

## composer安装
composer安装方法：`composer create-project --prefer-dist laravel/laravel blog "5.8.*"`

> 上述命令中的blog，实际上是我们安装完laravel框架后项目的文件夹名称

## .env配置
通常，当我们使用composer执行完上述命令后，blog应用并不是立即可用的，你还需要做一些必要的配置：

源文件路径：blog/.env

```
APP_NAME=Blog
APP_ENV=local
APP_KEY=base64:fvgDEFOa3vK7Hk8lvUwN8Pat0w/u7o16IJ4j+lj3g1k=
APP_DEBUG=true
APP_URL=http://dev.blog.z

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=192.168.10.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=123456

BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_CONNECTION=database
QUEUE_DRIVER=database
SESSION_DRIVER=file
#SESSION_DRIVER=database
#SESSION_DRIVER=redis//使用redis
#SESSION_CONNECTION=session//使用redis
SESSION_LIFETIME=120

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
REDIS_DB=0

MAIL_DRIVER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=

PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_APP_CLUSTER=mt1

MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"

```

>  注意：默认composer安装laravel框架完成后，根目录下只有一个".env.example"文件，你需要自己生成一个.env文件，并且修改其中的配置值。上面这个示例，是作者本机应用的一个配置。关于Laravel如何配置，您可以阅读Laravel5.8手册获取相关知识，此处不再赘述。
>
>  由于我们分析的代码不涉及读取数据库的任何操作，上述配置仍可继续简化（将"DB_"为前缀的配置项全部删除或注释）

## nginx站点配置

```nginx
server {
        listen    80;
        server_name  dev.blog.z;
        root "/home/www/blog/public/";
        index  index.htm index.html index.php;

        location / {
             try_files $uri $uri/ /index.php?$query_string;
        }
                
        location ~ \.php(.*)$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include  fastcgi_params;
        }               
}
```

> 注意：在这个配置实例中，我们给web网站定义的域名是"dev.blog.z"（和之前.env文件中的APP_URL保持一致），这个域名您当然可以随意更改。同时您需要注意的是root对应的配置值：/home/www/blog/public/，这个路径一定要对应您本地安装blog应用后文件夹所处的实际目录。



默认情况下完成上述配置后，访问的主页应该像下面这样：

![](../images/test_11.png)

【图1.2】

## 关于Nginx重定向

通常，基于MVC模式的框架都是走统一的单一入口文件模式，而在web服务器中, 要实现这一点也很简单，利用web服务器的URL重定向功能，比如nginx提供的try files指令就能实现。我们来看一个Laravel站点的配置文件(nginx.conf)：

```nginx
server {
        listen    80;
        server_name  dev.blog.z;
        root "/home/www/www/blog/public/";
        index  index.htm index.html index.php;

        location / {
             try_files $uri $uri/ /index.php?$query_string;
        }
                
        location ~ \.php(.*)$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include  fastcgi_params;
        }               
}
```

现在假设用户在浏览器中输入:http://dev.blog.z/test/index
根据上面这个配置, nginx会进行怎样的处理呢？

- 首先nginx读取路径中的uri部分：/test/index，try files指令的意思是优先读取相对应目录下的文件，读取不到则丢弃该请求，并重新发起一个子请求。
- 由于/home/www/www/blog/public/下没有test/index文件夹，因此服务器会发起一个内部子请求到/home/www/www/blog/public/index.php
- 由于try files指令中index.php后面携带了?$query_string，因此请求发送到index.php后，在服务端PHP通过`$_SERVER['REQUEST_URI']`语句获取到的值为/test/index

由此, 我们可以猜想Laravel框架在路由解析这一块使用的是分析`$_SERVER['REQUEST_URI']`结果, 和ThinkPHP3中分析path_info或者Phalcon中直接将uri放在_url=后作为重定向地址不同。

## 思考

1) 如果public下面有test/index/index.htm文件存在, 浏览器直接访问http://dev.blog.z/test/index 会发生什么？
2) 关于本文末尾提到的路由解析猜想，如何验证？