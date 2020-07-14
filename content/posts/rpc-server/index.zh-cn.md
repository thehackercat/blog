---
layout: post
title: "Twisted+gevent 异步+协程服务器开发"
date: 2016-07-04 04:01:33 +0800
comments: true
weight: 5
draft: false
categories: ["Coding"]
tags: ["Python", "Rpc", "Twisted"]
lightgallery: true
---

## 背景

最近接触到用 Twisted 来写个 RPC 服务器，对高并发、性能和大量长连接时的稳定性方面有要求，所以应该在 Twisted 的基础上再造些轮子，最后考虑用 Twisted + gevent 来实现 「异步+协程」的部分。

<!--more-->

分别简要介绍下 Twisted 和 gevent。

## Twisted
Twisted是用 Python 实现的基于事件驱动的异步的网络引擎框架。它封装了大部分主流的网络协议(传输层或应用层)，如 TCP、UDP、SSL/TLS、HTTP、IMAP、SSH、IRC以及FTP等，在这我主要会用到 TCP 协议。

使用 Twisted 的好处在于，它是以事件驱动编程实现的，所以提供了事件注册的回调函数的接口，每次接受到请求，获得了事件通知，就调用事件所注册的回调函数( Node.js 程序员可能比较熟悉)。这让我不必去操心服务器事件驱动的编写。

并且，在网络引擎方面，有心跳包和粘包的三方库，非常完善。

然而，Twisted 有一个缺陷，它的异步有点问题，单个连接建立后是一个进程，在进程里用多线程实现并发，但多个连接建立后仍然会出现同步阻塞的情况，所以这就要引入 gevent 来填充其性能上的缺陷。

## gevent
gevent 是一种基于协程的 Python 网络库，它用到 greenlet 提供的，封装了 libevent 事件循环的高层同步API。

如果你不知道什么是协程，那么可以简单这么理解：

协程就是由程序员自己编码实现调度的多线程。

而 gevent 对 greenlet 协程进行了封装，同时 gevent 提供了看上去非常像传统的基于线程模型编程的接口，但是在隐藏在下面做的是异步 I/O ，所以它以同步的编码实现了异步的功能。

##开搞
### Step 1 完成基础框架
首先由于我要编写一个 RPC 服务器(使用 TCP 协议)，所以需要先实现一个 TCP 服务器。

``` python
# server.py
from twisted.internet.protocol import ServerFactory, ProcessProtocol
from twisted.protocols.basic import LineReceiver
from twisted.internet import reactor

PORT = 5354

class CmdProtocol(LineReceiver):
    client_ip = ''

    # 连接建立接口
    def connectionMade(self):
        # 获得连接对端 ip
        self.client_ip = self.transport.getPeer().host
        print("Client connection from %s" % self.client_ip)

    # 连接断开接口
    def connectionLost(self, reason):
        print('Lost client connection. Reason: %s' % reason)

    # 数据接收接口
    def dataReceived(self, data):
        print('Cmd received from %s : %s' % (self.client_ip, data))

class RPCFactory(ServerFactory):
    # 使用 CmdProtocol 与客户端通信
    protocol = CmdProtocol

# 启动服务器
if __name__ == "__main__":
    reactor.listenTCP(PORT, RPCFactory())
    reactor.run()
```
Twisted 提供3个非常基础的接口使程序员进行重写:

- connectionMade() 连接建立后执行操作
- connectionLost() 连接断开后执行操作
- dataReceived() 接收到数据后触发操作

这3个接口通常来说是必须的，以此基础上进行完善，可以看到我只是先输出了友好信息。

这样简单完成了一个 TCP 服务器，可以看出 Twisted 网络引擎的架构如下：

1. 先由程序员来制定一个或多个协议(该协议可以继承各种底层网络协议)。
2. 接着指定唯一一个工厂，这个工厂必须声明使用的协议对象。
3. 使用 reactor 选择监听模式、监听工厂和端口，开启服务器。

### Step 2 完善基础框架
显然，这个 TCP 服务器基础框架显得有些单薄，我首先想到的是需要进行多客户端的控制及 ip 记录，故应有个队列来实时更新连接入服务器的 ip。

并且，最近有好几部电影在豆瓣我标记了，我想和高圆圆一起去看，所以不能一直盯着屏幕来观察反馈，所以需要一个日志系统来记录反馈信息。

故增加一个 log.py 日志系统文件：

```python
# log.py
import os
import logging
import logging.handlers
from twisted.python import log

#当前执行文件所在地址
CURRENT_PATH = os.getcwd()
#日志文件路径
LOG_FILE = CURRENT_PATH + '/rpcserver.log'
# 全局日志模块
gl_logger = None

class log(Protocol):
    def init_log():
    global gl_logger
    try:
        os.makedirs(os.path.dirname(LOG_FILE))
    except:
        pass
    # 实例化handler
    handler = logging.handlers.RotatingFileHandler(LOG_FILE, maxBytes=1024 * 1024, backupCount=1)
    fmt = '[%(asctime)s][%(levelname)s][%(filename)s:%(lineno)d:%(funcName)s] - %(message)s'
     # 实例化formatter
    formatter = logging.Formatter(fmt)
    # 为handler添加formatter
    handler.setFormatter(formatter)
    # 获取名为rpcserver的logger
    gl_logger = logging.getLogger('rpcserver')
    # 为logger添加handler
    loggergl_logger.addHandler(handler)
    handlergl_logger.setLevel(logging.DEBUG)

    gl_logger.info("----------------------------------")
```

并在 server.py 中添加如下代码：
(添加多连接控制，把 print 替换为 log.msg 来打印日志)

``` python
# server.py
from twisted.internet.protocol import ServerFactory, ProcessProtocol
from twisted.protocols.basic import LineReceiver
from twisted.internet import reactor
from twisted.python import log

PORT = 5354

class CmdProtocol(LineReceiver):
    client_ip = ''

    # 连接建立接口
    def connectionMade(self):
        # 获得连接对端 ip
        self.client_ip = self.transport.getPeer().host
        log.msg("Client connection from %s" % self.client_ip)

        # 进行多连接控制
        if len(self.factory.clients) >= self.factory.clients_max:
            log.msg("Too many connections. Disconnect!")
            self.client_ip = None
            self.transport.loseConnection()
        else:
            self.factory.clients.append(self.client_ip)

    # 连接断开接口
    def connectionLost(self, reason):
        log.msg('Lost client connection. Reason: %s' % reason)
        if self.client_ip:
            self.factory.clients.remove(self.client_ip)

    # 数据接收接口
    def dataReceived(self, data):
        log.msg('Cmd received from %s : %s' % (self.client_ip, data))

class RPCFactory(ServerFactory):
    # 使用 CmdProtocol 与客户端通信
    protocol = CmdProtocol

    # 设置最大连接数
    def __init__(self, clients_max=10):
        self.clients_max = clients_max
        self.clients = []

# 启动服务器
if __name__ == "__main__":
    log.startLogging(sys.stdout)
    reactor.listenTCP(PORT, RPCFactory())
    reactor.run()
```

### Step 3 增加 rpc 实例
既然是 rpc 服务器，辣么接下来就要实现一个简单的远程命令调用，既然之前写了日志模块，那就写一个对应的远程日志查看调用吧！

对了，写到这里，已经是 02：53 了，我不知道为什么开始胡思乱想起来。

我想大概是因为越是无端的，越是会心念着...

![飘渺心事](https://img1.doubanio.com/view/status/median/public/e818b8c2b37c179.jpg)

嗷，跑题了...

远程调用呢， Twisted 提供了一个敲好用的子进程父类 ```ProcessProtocol```

这个类提供了2个接口:

- outReceived 用来接收和外发数据
- processEnded 进程结束回调

于是，我在 server.py 中加入以下代码:

``` python
# 打印日志
class TailProtocol(ProcessProtocol):
    def __init__(self, write_callback):
        self.write = write_callback

    def outReceived(self, data):
        self.write("Begin logger\n")
        data = [line for line in data.split('\n') if not line.startswith('==')]
        for d in data:
            self.write(d + '\n')
        self.write("End logger\n")

     def processEnded(self, reason):
        if reason.value.exitCode != 0:
            log.msg(reason)
```
循环读取日志文件中每一行并输出信息。

接着在 CmdProtocol 类中加入以下函数:

``` python
# 根据 cmd 执行相应操作
def processCmd(self, line):
      if line.startwith('getlog'):
          tailProtocol = TailProtocol(self.transport.write)
          # 打印rpcserver.log日志
          reactor.spawnProcess(tailProtocol, '/usr/bin/tail', args=['/usr/bin/tail', '-10', '/var/log/rpcserver.log'])
```

通过获取远程发送来的命令 「getlog」 触发了以下事件 tailProtocol ，并调用 TailProtocol 类中的回调函数 outReceived 来循环读取日志文件中每一行并输出日志信息，返回给客户端。

同理，其余 RPC 远程调用实例也可类似的编写。

注意，这里使用了 Twisted 自带的 ```spawnProcess()``` 来处理事件回调，并新建一个线程来执行函数，这就是单个连接中并发的实现。

### Step 4 加入 gevent 协程部分

首先我考虑的是使用一个队列来储存每次接收到事件触发的钩子后，把钩子接收的参数存入队列中，再用 gevent 的协程来进行任务的分发。

直接上代码：

``` python
# server.py
import geventfrom gevent.queue
import Queue

# 任务队列
tasks = Queue()

class CmdProtocol(LineReceiver):
      def worker(self, target):
          while not tasks.empty():
            task = tasks.get()
            log.msg('User %s got task %s' % (target, task))
            self.processCmd(task)
            gevent.sleep(0)

      def dispatch(self, data):
          tasks.put_nowait(data)

      def dataReceived(self, data):
          log.msg('Cmd received from %s : %s' % (self.client_ip, data))
          gevent.spawn(self.dispatch, data).join()
          gevent.spawn(self.worker, self.client_ip)
```

首先， gevent 的队列 Queue 有两个主要的方法 ```get()``` 和 ```put()``` 来对队列中的元素进行读和写。```put_nowait()``` 相当于 ```put()``` 的无阻塞模式。

在 ```dispatch()``` 中，我把每个收到的 data 的 trigger 放入任务队列中，使其进入等待分发的状态。

接着，协程会执行下一步 ```worker()```从任务队列中取出相应的 trigger ，传入 ```processCmd``` 中触发回调，执行相应的函数。

执行完后，协程会回到上一步 ```dispatch()``` 接着再到 ```worker()```  这样交替轮循，直到任务列表里的任务全部执行完为止，这个过程中，各个任务执行是独立的，不会造成阻塞，吊！

 ## 欧勒！
就酱，我们撸出了一个高性能的、协程的、异步的 RPC 服务器！

![rpcserver](http://7xse6j.com1.z0.glb.clouddn.com/twisted.png)
