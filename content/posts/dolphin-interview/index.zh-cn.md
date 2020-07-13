---
title: "海豚浏览器 Python 实习面试心得"
date: 2015-12-18 12:02:37 +0800
comments: true
weight: 5
draft: false
categories: ["Interview"]
tags: ["Interview", "Code"]
lightgallery: true
---
## 背景
12月11号下午4：30参加了海豚浏览器的 Python 后台开发实习生的电面，考官一开始先问了我一些 Python 基础的问题，接着问了些计网的经典面试题，最后考了2道算法题，然后开始扯皮一些之前做过的项目中的问题等，最后总结心得如下：
<!--more-->

## Python 的 List 能不能作为字典的 key 传入？
我回答的是不能，因为字典的 key 值必须是不变的，而 List 的值是可变的。

之后我上网查了下，更标准的说法是，Python Dict 的 key 值是 hashable 的，即

- 这个 key 值在其生命周期内是不变的。
- 并且可以和其他对象进行比较。

以下是官方对于 hashable 给出的解释：
>
> An object is hashable if it has a hash value which never changes during its lifetime (it needs a hash() method), and can be compared to other objects (it needs an eq() or cmp() method). Hashable objects which compare equal must have the same hash value.
>
> Hashability makes an object usable as a dictionary key and a set member, because these data structures use the hash value internally.
>
> All of Python’s immutable built-in objects are hashable, while no mutable containers (such as lists or dictionaries) are. Objects which are instances of user-defined classes are hashable by default; they all compare unequal (except with themselves), and their hash value is their id().

所以得出，Python 中所有不变的内奸对象都是 hashable 的，所有可变的容器(比如，list or dict)都不是 hashable 的，故不能作为字典的 key。

## Python 装饰器是什么，有什么看法？
正好之前我写了一篇深入理解 Python 装饰器的 [blog](http://thehackercat.me/blog/2015/12/07/python-decorator-learning/)

我就向他解释了下，装饰器是在不修改原先代码块的情况下，为其加上一些装饰。

然后我扯了一些装饰器所使用的 Python 语言的几个特性

- 闭包
- 把函数作为参数传递
- 装饰器的迭代

## Python 的 yield() 函数的看法？
我想起来之前有看过一篇 [blog](http://pyzh.readthedocs.org/en/latest/the-python-yield-keyword-explained.html) 正好讲过。

yield 函数是 Python 在进行迭代时，函数内部的代码并不立刻执行，而是返回一个 generator 对象，接着每次迭代时，再读取下一个元素。

这样的好处在于，不需要一次性读取全部对象，二是实时地读取生成数据，减少了内存的开支。

## 解释下 Django 的 MVC 模式，其中那一部分充当的是 controller 的部分？
我解释了下，其实 Django 是一个 MTV 模式的框架, MTV 三个部分如下，

- 模型( Model )，数据存取层：处理与数据相关的所有事务，即如何存取、如何验证有效性、包含哪些行为以及数据之间的关系等。
- 模板( Template )，表现层：处理与表现相关的决定，即如何在页面或其他类型文档中进行显示。
- 视图( View )，业务逻辑层：存取模型及调取恰当模板的相关逻辑。模型与模板之间的桥梁。

而其中作为 controller 的部分是 Django 的 URLconf。

它获取用户在地址栏中输入的 URL 并将其路由到 views 模块对应的各个函数，并调用他们。实现了相应的视图函数路由到相应界面的映射功能。

## Django 中的缓存用过吗？看法是？
正好之前在写一个 Django 练手的图书馆项目中试过 Django 的缓存机制，所以就以那个例子介绍了下。

Django 的缓存系统让开发者能够缓存某个视图的输出。这个缓存是无法在浏览器缓存中控制的，因为它并不包含在 http 头部内。

我用的是 Django 缓存系统的 memcached。 memcached 作为一个后台进程运行，并分配一个指定的内存量，它所实现的功能是提供一个添加、检索和删除缓存中任意数据的快速接口，所有的数据是直接存储在内存中的，所以没有用到数据库或者文件系统，减少了额外开销。

但是 memcached 有一个缺点是，它的缓存是完全存在内存中的，一旦服务器崩溃，辣么所有缓存的数据就丢失了。

其他的缓存机制偶没有用过，所以就没有谈。

## 用户在浏览器中输入一个网址，到 Django 后台捕捉到请求其中的过程？
这个我当时貌似讲偏题了，我说的是

用户输入一个网址后

1. 浏览器先检查缓存，如果有缓存，就从缓存中获得资源文件并加载，如果木有缓存，则执行下一步。
2. 进行 DNS 域名解析，将域名解析成 ip 地址。
3. 与 ip 地址对影的服务器进行 TCP 连接。
4. 接着经历 TCP 3次握手过程。
5. 一旦连接建立后，开始发送 Http 请求。
6. 服务器获得 Http 请求后，将该请求打包成 HttpRequest 对象。
7. 接着检查 Request 中是否需要 Django 中间件的方法，如果没有则执行下一步。
8. 判断 Request 中的各种信息，诸如 user_agent、GET/POST 等，并在 URLconf 中进行匹配路由到对应的 views 视图函数中。
9. 返回一个 Response 对象，并调用相应的 views 视图函数。
10. 最后返回一个 Http 相应，并加载页面。

## (数据结构与算法)获得两个列表的交集
我第一次写的是
``` python
def intset(L1,L2):
	L = []
	for i in L1:
		if i not in L2:
			L.append(i)
	return L
```
接着考官问我，这个时间复杂度是多少，很明显是O(n^2)，他又问我有没有更好的方法，

于是我写了第二种方法
``` python
def intset2(L1,L2):
	L = [set(L1)^set(L2)]
	return L
```
这样先把L1、L2列表中重复的元素删除了，接着再用异或符来取得他们的交集。

## (数据结构与算法)一个人一次可以爬3级或5级的台阶，请问他爬到第m层时，有n种解法，求解
这个我当时没写出来，我第一眼感觉是递归的题，后来室友告诉我是线性规划的题。之后我在 leetcode 上也看到了相应的解法，真是太蠢了我！

leetcode 解法如下：
``` python
class Solution:
    # @param {integer} n
    # @return {integer}
    def climbStairs(self, n):
        if n==1 or n==2:
            return n
        a=1;b=2;c=3
        for i in range(3,n+1):
            c=a+b;a=b;b=c
        return c
```

## 总结
以上就是我这次 Python 实习面试的大部分考题，面试完之后感觉自己基础还是不扎实，对于性能优化的理解还有缺陷，代码写得不够漂亮，算法方面很薄弱。故决定刷一下 Python 文档和 Leetcode。

而且这次面试感觉要黄，因为都一星期了，HR 还是木有给我打电话 T.T

不过，我有了其他的考虑了，心理也安定了许多。

现在，我只想去睡个十除以三的懒觉。
