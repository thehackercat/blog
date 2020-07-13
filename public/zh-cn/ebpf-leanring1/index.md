# eBPF learning 01

## (译) eBPF Tracing 简明教程与示例

### 背景

| 原文链接 | http://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html |
| -------- | ------------------------------------------------------------ |
| 原文作者 | Brendan Gregg                                                |
| 出版时间 | 01 Jan 2019                                                  |
| 翻译时间 | 11 Jan 2020                                                  |

之前开 2019 ShangHai KubeConf 听了几场 eBPF 的分享, 最近才开始深入研究, 故打算把研究 Linux performance 的大佬 Brendan Gregg 的文章[《Learn eBPF Tracing: Tutorial and Examples》](http://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html) 作为 eBPF 入门翻译一遍, 希望对其他非英语母语的开发者有帮助。

### 译文

在 2019 年的 Linux Plumber's 大会上至少有 24 场关于 eBPF 的讲座, eBPF 迅速地成为了炙手可热的技术。所以也许你也计划在新的一年里开始学习 eBPF!

如今 eBPF 在一些主流技术中都有其身影, 如虚拟内核指令集(VKIS), 由于其继承自大名鼎鼎的  Berkeley Packet Filter (BPF) , 所以它有较多的用途，如: 网络性能分析, 防火墙, 安全追踪, 设备驱动等。这些基于 eBPF 的比如一些性能追踪的产品， 在网上也不乏教程好文档。说到「追踪」，我们常认为是一些性能分析和一些提供更多观察点的工具。你可能也已经用过其中的一些工具，如 **tcpdump**, **strace** 和其他的一些 tracer

在这篇文章里我会介绍 eBPF 追踪, 并针对初学者, 稍有经验的人以及一些老手在内容上做些简单划分便于阅读。总的来说分为以下内容:

- 初学者: 运行 `bcc` 工具
- 稍有经验: 开发 `bpftrace` 工具
- 老手: 开发 `bcc` 工具, 为 `bcc&bpftrace` 提 PR

### 初学者

#### 1. 什么是 eBPF, bcc, bpftrace 和 iovisor?

eBPF 对于 Linux 而言相当于 JavaScript 对于 HTML 的作用, 即相比于一个静态的 HTML 网站, JavaScript 能声明一些小的程序用于监听一些运行在浏览器虚拟内核上的一些用户事件如鼠标的点击等。所以有了 eBPF, 相比于一个固定的黑盒的 LInux 内核, 你也可以在用户态编写一些简单的脚本来获取一些内核事件如磁盘 I/O 等, 同样的, 这些事件也是运行在 Linux 安全内核虚拟机里。事实上呢, eBPF 比纯 JavaScript 更高级些, 它更相当于一个运行着 JavaScript 的 v8 虚拟机，且 eBPF 如今也已是 Linux 内核的一部分。

直接在 eBPF 中编写代码是非常难得, 就像你要直接写些 v8 虚拟机二进制代码一样。实际上也没啥人直接写 v8 代码, 大家都是直接写更友好的 JavaScript 或者一个基于 JavaScript 的上层框架(jQuery, Angular,  React 等)。

这件事在 eBPF 也一样, 人们不会直接写内核代码, 而是通过 eBPF 框架编写一些上层代码。拿性能追踪来说，主要用的是 `bcc` 和 `bpftrace` 。这两个工具并不在 内核代码中, 它们独立于 GitHub 上另一个 Linux 基金会的项目叫 `iovisor`

#### 2. eBPF 性能追踪的简单例子

先介绍一个基于 eBPF 的工具, `tcplife`

`tcplife` 可以展示出完整生命周期的 TCP sessions, 并且可以看到进程 id (PID), 进程命令名 (COMM), 发送和接受的字节数以及 TCP 连接的持续时间 (单位是 ms), 如图:

![image-20200112151502737](https://blog-1251445337.cos.ap-chengdu.myqcloud.com/ebpf-5.png)

需要注意的是 eBPF 并没有开放给用户可以通过旧的内核技术来重写 TCP 生命周期的入口，不过假如哪天开放了, 这样的入口也会带来一些性能上的额外开销,一些内核安全问题等等,所以我们永远不会使用这样的工具。eBPF 给工具带来的特性应该是可实践的, 便捷的，安全的。拿 `tcplife` 举个例子, 它并不会追踪每一个包, 这样会带来很多额外的开销，相反的，eBPF 只会追踪 TCP 生命周期的事件, 这些时间并不会这么频繁, 从而给系统带来的开销会低得多, 我们也可以从容地让这个工具 7x24 小时地在生产环境运行。

#### 3. 怎么使用 eBPF 呢?

对于新手来说, 可以说是从 `bcc` 开始使用, 具体教程可以参照[这篇文章](https://github.com/iovisor/bcc/blob/master/INSTALL.md)

在 Ubuntu 系统上, 可以简单地这么安装

![image-20200112154032819](https://blog-1251445337.cos.ap-chengdu.myqcloud.com/ebpf-4.png)

我呢是通过运行 `opensnoop` 来测试 `bcc` 是否 work 的, 所以如果你也这么搞过, 那么恭喜你, 你也用过 eBPF 啦！

如今一些公司如 Netflix 和 Facebook 已经默认在所有服务器上安装了 `bcc` 所以也许你也可以开始这么搞。

#### 4. 新手教程

如果你想直接学习 `bcc` 那么也可以看看[这篇教程](https://github.com/iovisor/bcc/blob/master/docs/tutorial.md), 这是一个对于新手来学习 eBPF 性能追踪极好的出发点。

作为新手呢, 你不需要一开始就写些 eBPF 代码。`bcc` 有接近 70 多个衍生工具你都可以直接使用。这篇教程也会带你逐步过一遍以下 11 个趴(part):

- execsnoop
- opensnoop
- ext4slower (or btrfs*, xfs*, zfs*)
- biolatency
- biosnoop
- cachestat
- tcpconnect
- tcpaccept
- tcpretrans
- runqlat
- profile

一旦上述内容你都尝试过后, 你可能就会有了一个全局性的概念:

![image-20200112155332639](https://blog-1251445337.cos.ap-chengdu.myqcloud.com/ebpf-3.png)

网上已经有过很多较为完整的 man pages 和示例来介绍这十一个趴(part)了。在官方 repo (bcc/tools) 中的示例文件 `*_example.txt` 展示了上述概念的具体释义，比如 [biolatency_example.txt](https://github.com/iovisor/bcc/blob/master/tools/biolatency_example.txt). 中你可以了解到如何用 bcc 来追踪 block I/O 时延, repo 中也有很多我写的示例文件, man pages 和具体工具，满打满算估摸着有 50 多篇 blog 了吧

不过目前还欠缺的是在生产环境中的一些示例。因为这些 blog 绝大多数是我在 eBPF 刚诞生不久，我们并没在生产环境中运行而只在测试实例上跑，所以大部分示例都是在一定场景下构造的。不久后也许我们会再新加些生产环境的示例，而这部分也需要大家的帮助，比如当你使用一个 eBPF 工具解决了某些生产环境问题时, 可以考虑留些现场和截图，并且提些 PR 加到示例文件中。

###稍有经验

这一章节的假设读者应该都已经使用过 `bcc` 及其他各种相关工具了, 如果你对二次开发 eBPF 工具感兴趣的话, 那么最佳的入手点就在于先尝试从 `bcc` 切换到 `bpftrace` , 这是一个更高级更易入门的语言。但 `bpftrace` 也有不足之处, 它不像 `bcc` 那样支持自定化, 所以大概率你可能最终使用完仍然会切换回 `bcc`

想要深入了解 `bpftrace` 的话可以参考[这篇文章](https://github.com/iovisor/bpftrace/blob/master/INSTALL.md) , 这是一个相对较新的项目, 所以在我写这篇文章时可能这个工具并没有被成熟的引入各个 Linux 包管理系统，不过在未来，应该可能通过类似 `apt-get install bpftrace` 或类似的方式来安装, 目前只能通过编译源码包安装。

#### 1. bpftrace 教程

我也写了另一个教程会教你如何通过单行命令玩转 bpftrace

- [bpftrace One-Liners Tutorial](https://github.com/iovisor/bpftrace/blob/master/docs/tutorial_one_liners.md)

这个教程分为 12 章节来逐步叫你学习 `bpftrace` ， 一个简单示例如下:

![image-20200112172025509](https://blog-1251445337.cos.ap-chengdu.myqcloud.com/ebpf-2.png)

这句命令会找出所有打开系统调用追踪点的 PID 和文件路径

#### 2. bpftrace 参考指南

想了解更多有关 bpftrace 的话，可以参考我的这篇文章

- [bpftrace Reference Guide](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md)

你可以从中了解到有关 bpftrace 语法，探针和一些内置组件。

#### 3. bpftrace 示例

bpftrace repo 中有近 20 个工具，你可以简单参考

- [bpftrace Tools](https://github.com/iovisor/bpftrace/tree/master/tools)

比如其中用于追踪 block I/O 时延的脚本

![image-20200112172931029](https://blog-1251445337.cos.ap-chengdu.myqcloud.com/ebpf-1.png)

如同 `bcc`, 这些工具也有详尽的 man pages 和示例, 比如 [biolatency_example.txt](https://github.com/iovisor/bpftrace/blob/master/tools/biolatency_example.txt)

### 老鸟

#### 1. 开发 bcc

这儿是我整理的一些关于开发及二次开发 `bcc` 的相关文档:

- [bcc Python Developer Tutorial](https://github.com/iovisor/bcc/blob/master/docs/tutorial_bcc_python_developer.md)
- [bcc Reference Guide](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md)

在 `bcc/tools/` 目录下有很多 `*.py` 文件可供学习, 这些 py 文件大约可以分成两部分:

- BPF 内核 C 代码的 Python 实现(或 lua, C++ 实现)
- Python 的一些 BPF 用户态工具

开发 bcc 工具确实是需要成熟的技术的，因为这可能会涉及到一些细碎的内核问题或应用底层问题

#### 2. 提贡献

我们非常欢迎以下贡献

- 提交 bcc issues
- 提交 bpftrace issues

对于 bpftrace, 有个 [bpftrace Internals Development Guide](https://github.com/iovisor/bpftrace/blob/master/docs/internals_development.md) 教程, 如果你正在写些 llvm IR 的代码, 那么这部分的开发对你是个挺大的挑战......

当然, 提贡献这部分也可以涉及内核 eBPF (aka BPF) 的引擎。

如果你浏览过 `bcc` 和 `bpftrace` 的 issues 的话, 你会看到很多优化相关的请求，诸如  [bpftrace kernel tag](https://github.com/iovisor/bpftrace/issues?q=is%3Aissue+is%3Aopen+label%3Akernel) 等等。此外，持续关注 [netdev](https://www.spinics.net/lists/netdev/)  邮件列表来料及到最新的内核 BPF 开发也是需要做的，这部分通常有些会在下一个 release 中 merge 到 Linux 主分支上。

此外，除了编写代码，你也可以写些测试，包构建，博客或者演讲分享来帮助我们。

### 总结

eBPF 确实做了很多事情。在这篇文章里我只是简单介绍了如何学习 eBPF 性能分析和追踪，总的来说是以下内容

- 初学者: 把 `bcc` 工具跑起来
- 稍有经验: 开发 `bpftrace` 工具
- 老鸟: 开发 `bcc` 工具, 对 `bcc` & `bpftrace` 做贡献

同时在这个页面 **[eBPF Tracing Tools](http://www.brendangregg.com/ebpf.html)**  有这篇文章更详细的内容, 其中涉及细节较多, 感兴趣的话可以仔细阅读。

祝你好运！
