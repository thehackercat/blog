# 在 Ubuntu 下搭建 uWSGI + nginx + Django

### 背景 :
公司要求用 Django 做些项目，之前按网上教程搭环境的时候就遇到很多问题，感觉有些教程都是有误的，今天用 uWSGI 开多线程的时候服务器报了 HTTP 500 的错( Internal Server Error )，然后就一直连不上去了。所以按[官网的教程](http://uwsgi-docs.readthedocs.org/en/latest/tutorials/Django_and_nginx.html)重新配置一遍，把出现的问题记录下来。
<!--more-->

### 准备工作
#### 概况
一个 Web 服务器能加载 ( Html , images , CSS 等静态文件)，但是它不能直接跑 Django 应用 (对于动态的请求无法处理) ，它需要某些工具来支持 Django 应用的运行，从而使 服务器能够接受客户端的请求，处理，并返回请求。s

这时，我们就需要一个服务器网关接口 -- WSGI ! WSGI 是一种Web服务器网关接口。它是一个 Web 服务器（如 nginx）与应用服务器（如 uWSGI 服务器）通信的一种规范

而 uWSGI 是一个Web服务器，它实现了 WSGI 协议、 uwsgi 、 http 等协议。

nginx 中 ```HttpUwsgiModule``` 的作用是与 uWSGI 服务器进行数据交换。

所以这套配置的实现原理是将 nginx 作为服务器最前端，它将接收 Web 的所有请求，统一管理请求。nginx 把所有静态请求自己来处理（这是 Nginx 的强项）。然后，Nginx 将所有非静态请求通过 uwsgi 传递给 Django ，由 Django 来进行处理，从而完成一次 Web 请求。

配置 uWSGI + nginx + Django 即实现以下4个链接，在下文，我们会一步步进行链接。

```
the web client <-> the web server <-> the socket <-> uwsgi <-> Django
```

#### Python
Ubuntu 14.04 自带了 Python2.7.6

你也可以通过

```
sudo apt-get install python2.7 python2.7-dev
```

来安装最新版本的 Python2.7.11
#### Python-pip
pip 是 Python 的包管理工具，建议 Python 的包都用  pip 进行管理。

通过以下命令安装  pip  :

```
sudo apt-get install python-pip
```

#### Django
通过以下命令安装 Django 并创建一个新的项目，然后进入到项目根目录 :

```
sudo pip install Django
django-admin.py startproject mysite
cd mysite
```

#### 关于域名和端口
在这篇 blog 中，我们把调试域名定为 127.0.0.1，你可以用自己的域名或本机 ip 地址来替代它。

并且，我们用 8000 端口作为 web 调试地址端口，这个端口与大部分 web 服务器的端口不重叠，当然你也可以自行修改调试地址的端口。

### 安装配置 uWSGI
#### 安装

```
sudo pip install uwsgi
```

用 ```pip``` 安装 uwsgi 最为方便，因为如果你用 ```apt-get install``` 来安装 uwsgi 的话，你需要在 Python 搜索路径中添加入 uwsgi 模块。

#### 测试 uwsgi
在刚刚的 mysite 目录下新建一个 Python 文件 ```test.py``` :

``` python
# test.py
def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return [b"Hello World"] # python3
	#return ["Hello World"] # python2
```


接着运行 uWSGI 命令:

```
uwsgi --http :8000 --wsgi-file test.py
```

注意：在 ```http``` 与 ```:8000``` 之间有一个空格！

参数含义：

- ```http :8000```：使用 http 协议，8000端口
- ```wsgi-file test.py``` : 加载指定文件 test.py

接着在浏览器中输入以下 url :

```
http://127.0.0.1:8000
```

如果出现了 'Hello World!' 那说明 uWSGI 安装成功，以下链接是成功的 :

```
the web client <-> uWSGI <-> Django
```

#### 测试 Django 项目
现在我们用 uWSGI 来跑 Django 网站试试。

首先进入 Django 项目根目录，即之前的 /mysite/ 运行

```
python manage.py runserver 127.0.0.1:8888
```

在浏览器中访问该 url ，如果出现如下界面则说明你的 mysite 项目是可运行的 :

![django_test](http://thehackercat-hackercat.stor.sinaapp.com/django_test1.png)

接着运行 uWSGI :

```
uwsgi --http :8000 --module mysite.wsgi
```

- ```module mysite.wsgi``` : 读取特定的 wsgi 模块

如果出现同样的界面，说明你的 uWSGI 已经可以搭载你的 Django 应用惹，所以以下的链接是成功的 :

```
the web client <-> uWSGI <-> Django
```

### 安装配置 nginx
#### 安装 nginx

```
sudo apt-get install nginx
sudo /etc/init.d/nginx start    # 开启 nginx 服务
```

现在到浏览器中输入 ```http://127.0.0.1``` ，如果你看到以下信息 : “Welcome to nginx!”那么说明 nginx 服务器运行成功，以下链接成功 :

```
the web client <-> the web server
```

#### 配置 nginx
首先，你需要一个 uwsgi_params 文件。

- 将 uwsgi_params 文件拷贝到项目文件夹下(即 /mysite/ )。uwsgi_params文件在/etc/nginx/目录下，也可以从这个[页面下载](https://github.com/nginx/nginx/blob/master/conf/uwsgi_params)
- 在项目文件夹下创建文件 mysite_nginx.conf ,填入并修改下面内容：

```
# mysite_nginx.conf

# the upstream component nginx needs to connect to
upstream django {
    # server unix:///path/to/your/mysite/mysite.sock; # for a file socket
    server 127.0.0.1:8001; # for a web port socket (we'll use this first)
}

# configuration of the server
server {
    # the port your site will be served on
    listen      8000;
    # the domain name it will serve for
    server_name 127.0.0.1; # substitute your machine's IP address or FQDN
    charset     utf-8;

    # max upload size
    client_max_body_size 75M;   # adjust to taste

    # Django media
    location /media  {
        alias /path/to/your/mysite/media;  # your Django project's media files - amend as required
    }

    location /static {
        alias /path/to/your/mysite/static; # your Django project's static files - amend as required
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        include     /path/to/your/mysite/uwsgi_params; # the uwsgi_params file you installed
    }
}
```

在终端中进入之前的 /mysite/ 项目文件夹，输入 ``pwd``` ，复制下该路径，将 mysite_nginx.conf 中的 /path/to/your/mysite 全部替换掉。

在/etc/nginx/sites-enabled目录下创建本文件的连接，使nginx能够使用它 :

```
sudo ln -s ~/path/to/your/mysite/mysite_nginx.conf /etc/nginx/sites-enabled/
```

注意：记得检查 /etc/nginx/sites-enabled/ 下的软链接是否成功，因为之前我就遇到了路径出错的问题。

#### 部署静态文件
在运行 nginx 前，你还需要把 Django 的所有静态文件全部整理到之前的 static 文件夹里，在 /mysite/mysite/settings.py 中添加以下内容 :

```
STATIC_ROOT = os.path.join(BASE_DIR, "static/")
```

接着运行

```
python manage.py collectstatic
```

现在你发现 Django 所有的静态文件都被整理到了 /mysite/static/ 文件夹里了。

#### 测试 nginx
首先重启 ngxin 服务 :

```
sudo /etc/init.d/nginx restart
```

在 /mysite/mysite/media/ 文件夹中添加一个 ```media.png``` 文件。

在浏览器中打开 :```http://127.0.0.1:8000/media/media.png```

如果显示出了图片，说明 nginx 服务已经正确运行惹。

注意在从浏览器中请求图片信息时，在 uwsgi 里是没有输出信息的，而请求一个其他的动态网页时，则会输出类似
```
[pid: 1952|app: 0|req: 3/3] 127.0.0.1 () {36 vars in 599 bytes} [Wed Mar 18 08:43:27 2015] GET /time/ => generated 63 bytes in 1 msecs (HTTP/1.1 200) 2 headers in 88 bytes (1 switches on core 0)
```
这样的信息。

也就是缩，当你在浏览器中请求 media.png 时， nginx 会检查这个地址 /media/ ，接着它会在 mysite_nginx.conf 文件中看到这段代码:

```
location /media  {
        alias /home/thehackercat/Dev/mysite/mysite/media;  # your Django project's media files - amend as required
```

它会直接从这个路径下去寻找图片，找到了就显示粗来，没找着就报 404 错误。

#### nginx and uWSGI and test.py
现在进入 /mysite/ 文件夹 输入以下命令 ：

```
uwsgi --socket :8001 --wsgi-file test.py
```

在浏览器中访问 ```http://127.0.0.1:8000 ``` 如果出现 'Hello World!' 则说明以下链接是成功的 :

```
the web client <-> the web server <-> the socket <-> uWSGI <-> Python
```

### 用 UNIX socket 取代 TCP port
在 mysite/ 文件夹下创建一个新文件 mysite.sock （空文本文档即可）。

然后对 ```mysite_nginx.conf``` 做以下修改 :

```
server unix:///path/to/your/mysite/mysite.sock; # for a file socket
# server 127.0.0.1:8001; # for a web port socket (we'll use this first)
```

运行 :

```
sudo /etc/init.d/nginx restart
uwsgi --socket mysite.sock --wsgi-file test.py
```

打开 ```http://127.0.0.1:8000``` 结果报错了，出现了这个错误

```
[crit] 4133#0: *1 connect() to unix:/home/thehackercat/Dev/mysite/mysite.sock failed (13: Permission denied) while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "uwsgi://unix:/home/thehackercat/Dev/mysite/mysite.sock:", host: "127.0.0.1:8000"
```

发现原来是权限的问题，于是在命令中加入这一段 :

```
uwsgi --socket mysite.sock --wsgi-file test.py --chmod-socket=666
```

然后就成功了！

#### 用 uswgi 和 nginx 跑 Django 应用
如果上面一切都运行正常，则输入下面命令可以跑 Django 应用 :

```
uwsgi --socket mysite.sock --module mysite.wsgi --chmod-socket=666
```

### 配置 uWSGI 便捷开启服务器
如果每次都按上述命令来跑 Django 应用实在麻烦，所以使用 .ini 文件来简化工作，便捷开启服务器，方法如下 :

在 /mysite/ 文件夹下创建文件 mysite_uwsgi.ini ，并填写修改下面内容 :

```
mysite_uwsgi.ini file
[uwsgi]

# Django-related settings
# the base directory (full path)
chdir = /home/thehackercat/Dev/mysite
# Django's wsgi file
module = mysite.wsgi
# the virtualenv (full path) 如果你没有装 virtualenv 就把下面这行用注释掉
home = /usr/bin/virtualenv

# process-related settings
# master
master = true
# maximum number of worker processes
processes = 10
# the socket (use the full path to be safe
socket = /home/thehackercat/Dev/mysite/mysite.sock
# ... with appropriate permissions - may be needed
chmod-socket = 666
# clear environment on exit
vacuum = true
```

现在，只要运行以下命令，就可以跑 Django 应用了 :

```
uwsgi --ini mysite_uwsgi.ini
```

到这里，如果你在浏览器中访问``` http://127.0.0.1:8000 ``` 可以看到正常的 Django 页面，则说明 uWSGI + nginx + Django 配置成功！

#### 参考文档
1. [nginx与Django不可不说的秘密](http://www.jianshu.com/p/32dbe2537b78)
2. [Setting up Django and your web server with uWSGI and nginx](http://uwsgi-docs.readthedocs.org/en/latest/tutorials/Django_and_nginx.html)

有些同学一定会被网上各种教程的 Django 目录结构搞得头大，其实这个目录是可自定义的，下面是我的目录结构 :

![django_menu](http://thehackercat-hackercat.stor.sinaapp.com/django_test4.png)

