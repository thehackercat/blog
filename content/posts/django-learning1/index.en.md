---
weight: 5
title: " Django 学习笔记1-- URLconf "
date: 2015-11-14 20:12:37 +0800
comments: true
draft: false
categories: ["Code", "Python"]
tags: ["Python", "Django"]
lightgallery: true
---

![PRAY FOR PARIS ](https://scontent-nrt1-1.xx.fbcdn.net/hphotos-xfp1/t31.0-8/12186716_1082564598443763_5619412981167558277_o.jpg)

今天好像巴黎有点乱，希望明天太阳还会照常升起。

## 简介
Django 是一个由 Python 编写、开源并采用经典的 [MVC](https://msdn.microsoft.com/en-us/library/ff649643.aspx) 设计模式的 Web Full Stack 应用框架。

在 Django 中，控制器接受用户输入的部分由框架自行处理，所以 Django 里关注更多在模型( Model )、模板( Template )和视图( Views )，称为 MTV 模式。他们各自的职责如下：
<!--more-->
 - 模型( Model )，数据存取层：处理与数据相关的所有事务，即如何存取、如何验证有效性、包含哪些行为以及数据之间的关系等。
 - 模板( Template )，表现层：处理与表现相关的决定，即如何在页面或其他类型文档中进行显示。
 - 视图( View )，业务逻辑层：存取模型及调取恰当模板的相关逻辑。模型与模板之间的桥梁。

而 Django 的编译方式比较特别，他的 MVC 控制器部分由 URLconf 来实现。

## URLconf
当我在 Django 中编写完视图要想将其路由要页面上时，我发现了 Django 的 URLconf 路由机制，他实现了为相应的视图函数路由到相应界面的映射功能，也就是说，当用户访问了 ```http://127.0.0.1:8000/hello/``` 时， Django 调用了视图 views.py 中的 hello () 函数。

``` python
from django.conf.urls import include, url
from mysite.views import hello,current_datetime,hours_ahead,letter

urlpatterns = [
url(r'^hello/$', hello),
url(r'^time/$', current_datetime),
url(r'^time/plus/(\d{1,2})/	$',hours_ahead),
]
```

可以看出， URLconf 的路由是通过正则表达式来匹配一个完整的 hello 的 URL ，这样的话就可以保证 诸如 /hello/foo/ 等 URL 不会被匹配到。
为了更深入了解 URLconf 路由的机制，我找到了类似的 [tornado](https://github.com/tornadoweb/tornado) 框架来对比。

注意到在其中 web.py 文件中的第2964行开始的如下代码：

``` python
application = tornado.web.Application([
	(r"/", MainHandler),
])
http_server = tornado.httpserver.HTTPServer(application)
http_server.listen(options.port)
tornado.ioloop.IOLoop.current( ).start( )
```

可以看出 torando 现把一个路由表作为一个参数，传给 Application 类的构造函数，接着创建了一个实例，然后再把这个实例传递给 http_server 。那么当客户端发起``` get / ```请求的时候, http server 接收到这个请求，在路由表中匹配 url pattern ，最后交给 MainHandler 去处理。

这个机制跟 Django 的 URLconf 是类似的，都是通过在 pattern 中匹配好对应的 url 接着传给处理器来负责从路由表中检索并路由。

这种方法**松耦合**了 http server 层和 web application 层，从而让开发者可以专注于 web 应用的逻辑层，很好！  ：D

## Django 如何处理请求

所以了解过了 Django 的 URLconf 机制后，我开始思考他是如何处理请求的。

我开启服务器后在地址栏中输入 ``` http://127.0.0.1:8000/time/plus/20/ ```

![timeplus20](http://thehackercat-hackercat.stor.sinaapp.com/urlconf1.png)

然后花现处理路线如下：

1. 进来的请求转入 /time/plus/20/ .

2. Django 通过在 ROOT_URLCONF 配置来决定根 URLconf .

3. Django 在 URLconf 中的所有 URL 模式中，查找第一个匹配 /time/plus/20/ 的条目。

4. 如果找到匹配，将调用相应的视图函数

5. 如果没找到匹配，则返回相应的 Http 状态码 (如图)

6. 视图函数返回一个HttpResponse

7. Django 转换 HttpResponse 为一个适合的 HTTP response ，以 Web page 显示出来

![http_request](http://thehackercat-hackercat.stor.sinaapp.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202015-11-14%20%E4%B8%8B%E5%8D%889.27.45.png)


