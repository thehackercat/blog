---
title: "绿盟 Web 后端实习面试心得"
date: 2015-12-23 21:37:16 +0800
comments: true
weight: 5
draft: false
categories: ["Interview"]
tags: ["Interview", "Code"]
lightgallery: true
---
## 背景
12月23号下午2：00参加了绿盟的 Web 后端开发实习生的面试。考官是个胖哥哥，也是科大的，人很温柔和蔼。先问了一些数据结构与算法的问题，接着问了计算机网络的一些基础问题，最后考察了下 Web 开发的一些知识。总得来说题目不难，但是自己也发挥不好，原来以为有了几次面经，但是在现场还是紧张得不行。 (真是给自己的心理素质跪了 ：P)
<!--more-->

## (数据结构与算法)图的遍历
我怕出错就写了5个结点的无向图，如下：
![undirected graph](http://thehackercat-hackercat.stor.sinaapp.com/tulun.jpg)

然后写了广度优先遍历：

1->2->3->4->5

深度优先遍历：

1->2->5->4->3

## (数据结构与算法)写个排序算法求列表中倒数第二大的元素
我用 Python 写了个冒泡排序来处理：

``` python
# 冒泡排序
def bubbleSort(L):
    for passnum in range(len(L)-1,0,-1):
        for i in range(passnum):
            if L[i] > L[i+1]:
				L[i],L[i+1] = L[i+1],L[i]
    return L[-2]
```

## (数据结构与算法)去重的优化算法
接着不造为什么就谈到了之前在海豚面试的时候对算法时间复杂度进行优化的问题，然后考官问了我一个去除一个列表中重复元素的算法。

``` python
# 去重
def induplicate(L):
	L1 = []
	return [L2.append(i) for i in L if not i in L2]
```
这样通过增加空间复杂度来降低时间复杂度

## Http 状态码
这个我当时说错了

我说的是

- 2 开头的是成功
- 3 开头的是需要等待
- 4 开头的通常是请求出错
- 5 开头的是服务器问题

后来回来查了下

- 3 开头的标识重定向
- 5 开头的表示服务不可用

## TCP 3次握手连接和4次握手断开连接的过程
这个不能更经典了。

就不详细列出了，可以参见这个[详解](http://blog.csdn.net/zhuying_linux/article/details/7449403)

## 设计一个产品参数配置页面布局
我本来打算多扯一些的，因为最近正好在看的[《写给大家看的设计书》](http://book.douban.com/subject/3323633/)，但是词穷了，就画了个抽屉菜单的布局,但是感觉还有很多交互设计的地方我欠考虑。

## Http 和 Https 的区别
这个我没答出来，我只知道 Https 是经过一定手段加密使得 Http 传输的数据包中一些明文数据变得"隐晦"，但是具体的实现方法不太清楚。

后来我看了一篇 [Blog](http://www.fenesky.com/blog/2014/07/19/how-https-works.html)，主要是用 SSL/TLS 来对数据包进行加密。

这样经过 SSL/TLS 协议加密后，当客户端收到服务器的 Https 请求后，会查询本机所支持的加密算法，并通过该算法来解密 Https 请求。
## 总结
这次面试总的来说题目相对简单。

面试官也教了我很多网安方面的知识，比如12306的签名协议和网关安全，虽然我是网络安全方面的小白，但是我觉得 Web 安全很炫酷。