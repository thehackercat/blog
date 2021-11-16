---
title: 从 Facebook 故障开始探索 BGP 工具使用
date: 2021-11-16T11:46:09+08:00
comment: true
weight: 5
draft: false
categories: ["Coding"]
tags: ["Network", "Linux"]
lightgallery: true
---
## 背景

上个月，Facebook 发生了一次由 BGP 引起的大型故障，从[故障报告](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/) 中我确实看得云里雾里。所以近期阅读一些文章，期望深入学习。



这篇博文将会分享一些我如何学习 BGP 的过程及用来查询 BGP 信息的相关工具使用。因为我对 BGP 不是那么了解，故文章难免有不精确的部分，希望多多交流。



### BGP 探索之路

#### 如何发布BGP路由

之前我从没有开始接触 BGP 的原因之一是--我不知道如何在我的服务器上通过 BGP 发布路由。



对于大多数网络协议，如果你愿意，都可以折腾，自己实现协议。例如，你可以。

- 发行你自己的TLS证书

- 编写你自己的HTTP服务器

- 编写你自己的TCP实现

- 为你的域名编写你自己的权威DNS服务器

- 建立你自己的证书颁发机构



但对于BGP，我认为除非你拥有自己的 ASN，否则不能自己发布路由，尽管可能可以在你的家庭网络上实现BGP，但我希望它们运行在真正的互联网上。



于是我开始着手于如何在我的服务器上发布 BGP 路由。



首先我们来谈谈 BGP 的一些术语。这部分我会简单的介绍，因为我对工具更感兴趣，而且网上有很多关于 BGP 的高水平资料（比如 [cloudflare](https://blog.cloudflare.com/october-2021-facebook-outage/) 的这篇帖子）。



#### 什么是 AS（autonomous system）？

我们需要了解的第一件术语是 AS。每一个 AS 都是什么呢？



参考这篇[维基百科](https://zh.wikipedia.org/wiki/%E8%87%AA%E6%B2%BB%E7%B3%BB%E7%BB%9F), 他们有以下特征，

1. 由一个组织拥有的（通常是一个大型组织，如你的ISP，政府，大学，Facebook，等等）

2. 他们能控制一组特定的 IP 地址（例如，我的 ISP 的 AS 包括 247,808 个IP地址）。

3. 他们有一个固定标识的数字码（如1403）, 我们称为 ASN。



下面是我简单做一些 AS 相关的实验观察。

1. 一些大型的科技公司其实并没有自己的 AS。例如，我在 BGPView 上查看了 Patreon，就我所知，他们没有自己的 AS -- 他们的主站（patreon.com，104.16.6.49）在 Cloudflare 的 AS 中。

2. 一个 AS 可以包括许多国家的 IP。Facebook 的 AS（AS32934）肯定有新加坡、加拿大、中国、美国和其他国家的IP地址。

3. 似乎 IP 地址可以在一个以上的 AS 中。例如，如果我查找 209.216.230.240，它有 2 个 ASN 与之相关-- AS6130 和 AS21581。显然，当这种情况发生时，可能会有路由规则就会有对应的优先算法--所以到该IP的数据包会被路由到 AS21581。



#### 什么是 BGP 路由呢

互联网上有很多的路由设备。例如，我的 ISP 有所在云厂商提供的路由策略。

当我向我的 ISP 发送一个数据包时（例如通过运行 ping 129.134.30.0），我的 ISP 的路由器需要弄清楚如何将我的数据包实际发送到 IP 地址 129.134.30.0

路由器计算的方法是它有一个路由表 -- 它有一堆IP范围的列表（比如129.134.30.0/23），以及它知道的通往该子网的路由。

下面是一个 129.134.30.0/23 的真实路由的例子：

```shell
  11670 32934
    206.108.35.2 from 206.108.35.254 (206.108.35.254)
      Origin IGP, metric 0, valid, external
      Community: 3856:55000
      Last update: Mon Oct  4 21:17:33 2021
```

这条路由说明了通往 129.134.30.0 的一条路径是通过机器 206.108.35.2 ，这是在其本地网络上。所以路由器接下来可能会把我的 ping 包发送到 206.108.35.2，然后 206.108.35.2 会知道如何把它送到目标地址。

其中开头的两个数字（11670 32934）是ASN。



#### 所以什么是 BGP 呢

我对 BGP 的理解比较模糊，但它是一个公司用来公布 BGP 路由的协议。

如上个月发生在 Facebook 的故障是，他们发布了 BGP 广播，撤销了他们所有的 BGP 路由，所以世界上每一个路由器都删除了所有与 Facebook 有关的路由，所以没有流量可以到达那里。

好了，现在我们已经涵盖了一些基本的术语，让我们来谈谈可以用来查看自治系统和 BGP 的工具吧!

#### 工具1：用 BGPView 查看你的 ISP 的 AS

为了使 AS 这个东西不那么抽象，我通常使用一个叫 [BGPView](https://bgpview.io/) 的工具来查看一个真实的 AS。

我的 ISP 拥有 [AS4808](https://bgpview.io/asn/4808)。[这里](https://bgpview.io/asn/4808#prefixes-v4)是我的 ISP 拥有的 IP 地址。

![bgpview-asn](https://blog-1251445337.cos.ap-chengdu.myqcloud.com/bgp%E5%B7%A5%E5%85%B7%E6%8E%A2%E7%B4%A2/bgpview-01.png)

如果我查找我的计算机的公网 IPv4 地址，可以发现它是我的 ISP 拥有的 IP 地址之一 -- 它在 [111.192.0.0/12](https://bgpview.io/prefix/111.192.0.0/12) 块中。

BGPView 还提供了我的 ISP 与其他 AS 的连接情况的图表

![bgp连接图标](https://blog-1251445337.cos.ap-chengdu.myqcloud.com/bgp%E5%B7%A5%E5%85%B7%E6%8E%A2%E7%B4%A2/bgpview-02.png)

#### 工具2:  `traceroute -A` 和 `mtr -z`

既然我们对 AS 有所了解。那么让我们看看我的网络请求会从哪些 AS 中穿过

`traceroute` 和 `mtr` 都能可以告诉你每个 IP 的 ASN 。

具体用法是` traceroute -A` 和 `mtr -z`，分别。

让我们看看我用 mtr 在访问 baidu.com 的路上经过了哪些 AS

```shell
$ sudo mtr -z baidu.com
                                            My traceroute  [v0.94]
Bong (10.255.2.107) -> baidu.com                                                     2021-11-16T13:12:47+0800
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                                                     Packets               Pings
 Host                                                              Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. AS???    10.255.0.1                                             0.0%    86    2.3  50.2   1.7 925.0 169.2
 2. AS4808   121.69.9.97                                            1.2%    86    3.8  47.7   3.4 876.4 158.8
 3. AS4808   218.241.128.173                                        1.2%    86    3.3  54.0   3.0 996.9 180.1
 4. AS4808   218.241.254.185                                        0.0%    86    5.0  59.5   2.5 998.4 196.0
 5. AS4808   218.241.254.197                                        0.0%    86    3.6  55.7   3.4 974.3 184.4
 6. AS4808   218.241.254.249                                        0.0%    86    9.0  51.7   4.4 921.5 171.7
 7. AS4808   218.241.244.17                                        58.1%    86   14.2  45.7   4.9 874.7 151.3
 8. AS4808   61.149.212.201                                        62.4%    86  1003. 105.0   5.4 1003. 244.9
 9. AS4808   202.96.13.249                                          4.7%    86    5.6  66.5   5.1 1010. 201.5
10. AS4808   124.65.194.77                                         12.9%    85   11.0  33.7   5.7 647.0 105.9
11. AS4837   219.158.13.78                                          0.0%    85    6.1  58.5   6.1 916.7 172.3
12. AS4837   219.158.44.114                                        89.3%    85   28.4  43.1   6.3 310.7 100.6
13. AS4134   202.97.18.125                                         72.6%    85    8.4  53.8   6.3 1026. 212.0
14. AS23724  36.110.246.126                                        82.1%    85    9.4  87.2   7.3 982.4 252.9
15. (waiting for reply)
16. (waiting for reply)
17. (waiting for reply)
18. (waiting for reply)
19. (waiting for reply)
20. (waiting for reply)
21. AS23724  220.181.38.148                                         0.0%    85    9.6  59.0   7.9 983.3 182.6
```

虽然看得眼花缭乱，但看起来网络数据包直接从我的 ISP 的AS（4808）到 Baidu 的 AS（23724），中间有一个 "互联网交换"。

*我不确定互联网交换是什么，但我知道它是互联网的一个极其重要的部分。不过这部分暂时不在文章中介绍了。我最好的猜测是，它是互联网中实现 "对等 "的部分-- 就像 IX 是一个有巨大的交换机的机房，里面有无限的带宽，一堆不同的公司把他们的计算机放在那里，这样他们就可以互相发送数据包。*



#### 工具3：PCH looking glass

PCH（"packet clearing house"）是运行大量互联网交换点的组织。"looking glass" 似乎是一个通用术语，指的是让你从另一个人的电脑上运行网络命令的终端。有一些 looking glass 不支持 BGP，而我只对那些能显示 BGP 路由信息的 looking glass 工具感兴趣，本文只重点介绍这一部分。

这里是常用的 PCH looking glass 地址：https://www.pch.net/tools/looking_glass/。

我按下图随便选了一个选项进行 bgp 查询

![pch-looking-glass](https://blog-1251445337.cos.ap-chengdu.myqcloud.com/bgp%E5%B7%A5%E5%85%B7%E6%8E%A2%E7%B4%A2/pch-looking-glass-01.png)

**Step1："show ip bgp summary"**

下面是输出结果。由于太长我删减了一些不重要内容。

```shell
From cache (number of seconds old (max 600)): 339


 IPv4 Unicast Summary:
 BGP router identifier 74.80.118.4, local AS number 3856 vrf-id 0
 BGP table version 3612368
 RIB entries 546990, using 96 MiB of memory
 Peers 150, using 3064 KiB of memory
 Peer groups 8, using 512 bytes of memory

 Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
 206.108.34.3    4       5769   18732   18160        0    0    0 01w5d14h          808
 ...
 206.220.231.55  4       3856   16515 1824368        0    0    0 06:52:52            0
 74.80.118.1     4         42   18189   18159        0    0    0 01w5d14h           49

 Total number of neighbors 150
```

上述结果我的理解是，TORIX 会直接连接到我的 ISP, 并探测当前网络环境我的邻居 AS 共有哪些。

**Step 2: “show ip bgp 129.134.30.0”**

下面是挑选 129.134.30.0（Facebook 的一个IP地址）的 "show ip bgp "的输出。

```shell
BGP routing table entry for 129.134.30.0/23
Paths: (4 available, best #3, table default)
  Advertised to non peer-group peers:
  206.220.231.55
  11670 32934
    206.108.35.3 from 206.108.35.253 (206.108.35.253)
      Origin IGP, metric 0, valid, external
      Community: 3856:55000
      Last update: Mon Nov 15 11:46:18 2021

  11670 32934
    206.108.35.2 from 206.108.35.254 (206.108.35.254)
      Origin IGP, metric 0, valid, external
      Community: 3856:55000
      Last update: Sat Nov  6 11:55:40 2021

  32934
    206.108.35.2 from 206.108.35.2 (157.240.58.182)
      Origin IGP, metric 0, valid, external, multipath, best (Older Path)
      Community: 3856:55000
      Last update: Wed Nov  3 14:52:59 2021

  32934
    206.108.35.3 from 206.108.35.3 (157.240.58.225)
      Origin IGP, metric 0, valid, external, multipath
      Community: 3856:55000
      Last update: Wed Nov  3 14:52:59 2021
```

这似乎是在说，从该互联网交换中心到 Facebook 有 4 条路由路径。

魁北克互联网交换中心似乎不感知 Facebook 该 ip 地址

我也试过从魁北克互联网交换中心QIX做同样的事。但 QIX 似乎对 Facebook 一无所知 -- 当我输入 129.134.30.0 时，它提示"Network not in table"。

所以我想这就是为什么我必须要从 TORIX 访问 Facebook。



#### 更多的 BGP looking glasses

除以上之外，这里还有一些 BGP looking glasses 的网站，可以从其他角度给你提供类似的信息。他们似乎都支持相同的 show ip bgp 语法，也能有一定的参考性。

- http://www.routeviews.org/routeviews/index.php/collectors/

- http://www.routeservers.org/

- https://lg.he.net/



下面是一个与这个名单上的一个服务器进行会话的例子：route-views.routeviews.org。这次我是通过 telnet 连接的，而不是通过web表单，但输出的格式看起来是一样的。

```shell
$ telnet route-views.routeviews.org

route-views>show ip bgp 31.13.80.36

BGP routing table entry for 31.13.80.0/24, version 1053404087
Paths: (23 available, best #2, table default)
  Not advertised to any peer
  Refresh Epoch 1
  3267 1299 32934
    194.85.40.15 from 194.85.40.15 (185.141.126.1)
      Origin IGP, metric 0, localpref 100, valid, external
      path 7FE0C3340190 RPKI State valid
      rx pathid: 0, tx pathid: 0
  Refresh Epoch 1
  6939 32934
    64.71.137.241 from 64.71.137.241 (216.218.252.164)
      Origin IGP, localpref 100, valid, external, best
      path 7FE135DB6500 RPKI State valid
      rx pathid: 0, tx pathid: 0x0
  Refresh Epoch 1
  701 174 32934
    137.39.3.55 from 137.39.3.55 (137.39.3.55)
      Origin IGP, localpref 100, valid, external
      path 7FE1604D3AF0 RPKI State valid
      rx pathid: 0, tx pathid: 0
  Refresh Epoch 1
  20912 3257 1299 32934
    212.66.96.126 from 212.66.96.126 (212.66.96.126)
      Origin IGP, localpref 100, valid, external
      Community: 3257:8095 3257:30622 3257:50001 3257:53900 3257:53904 20912:65004
      path 7FE1195AF140 RPKI State valid
      rx pathid: 0, tx pathid: 0
  Refresh Epoch 1
  7660 2516 1299 32934
    203.181.248.168 from 203.181.248.168 (203.181.248.168)
      Origin IGP, localpref 100, valid, external
      Community: 2516:1030 7660:9001
      path 7FE0D195E7D0 RPKI State valid
      rx pathid: 0, tx pathid: 0

```



#### 工具4: BGPlay

到目前为止，所有其他的工具都只是向我们展示了 Facebook 路由的当前状态，其中一切正常，但这第四个工具让我们看到了这个Facebook BGP 互联网故障的历史记录。

这是一个 GUI 工具，所以我将贴上一堆屏幕截图。

这个工具下载地址在 https://stat.ripe.net/special/bgplay 。我输入了 IP 地址 129.134.30.12（Facebook的IP之一）进行验证。

首先，让我们看看一切出错之前的状态。我在10月4日13:11:28点击了时间线，得到了这个结果。

![bgplay-before](https://jvns.ca/images/bgplay-before.png)

最初发现这非常难以看懂。发生了什么事？但后来有人在 Twitter 上指出，下一个要看的地方是点击 Facebook 故障发生后的时间线（10月4日18点38分）

![bgplay-after](https://jvns.ca/images/bgplay-after.png)

很明显，这张图有问题 -- 所有的 BGP 路由都消失了！可以看到各个 AS 之间已经没有了连线！

上面的文字显示了最后一条 Facebook BGP 路由的消失时间。

```shell
Type: W > withdrawal Involving: 129.134.30.0/24
Short description: The route 50869, 25091, 32934 has been withdrawn.
Date and time: 2021-10-04 16:02:33 Collected by: 20-91.206.53.12
```

如果我再点击 "快进 "按钮，就能看到 BGP 路由开始回来了，这是 FB 工程师开始进行故障恢复了。

![bgplay-return](https://jvns.ca/images/bgplay-return.png)

可以看到，第一条恢复的是 BGP 路由规则是 `137409 32934`。

但我不认为这实际上是第一个恢复的--在同一秒内（在2021-10-04 21:00:40）有很多路由规则同时在恢复，我认为 BGPlay 内部的排序是随机的。

如果我再次点击 "快进 "按钮，越来越多的路由开始回来，路由开始恢复正常

可以看到这个 GUI 工具还是十分有趣的，可以直观地带我们探索 Facebook 网络故障的过程。



### 总结

至此，所有工具的介绍基本结束。也许你会看到一堆 BGP 相关术语和知识，或者一些工具的参考，如果你想（作为一个业余爱好者）实际发布 BGP 路由，这里有一些链接提供参考

- [a guide to getting your own ASN](https://labs.ripe.net/author/samir_jafferali/build-your-own-anycast-network-in-nine-steps/)
- [dn42](https://dn42.eu/Home) seems to have a playground for BGP (it’s not on the public internet, but it does have other people on it which seems more fun than just doing BGP by yourself at home)

我想还有很多BGP工具（比如PCH有一堆路由数据的每日快照，都是十分有趣的工具），但这篇文章已经很长了，而且我确实也没有逐个了解过这些工具。

我对我作为一个没有专业网络知识的工程师可以得到这么多关于BGP的信息感到惊讶，我一直认为它是一个 "秘密的网络向导 "的东西，但显然有各种公共机器，任何人都可以直接telnet到那里，用来查看路由表! 还有各种资料！

希望以后能更多的接触到 BGP 相关知识，再和你们做分享！
