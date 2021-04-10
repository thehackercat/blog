# Talk about Containerd




## 背景

最近看一些社区 issue 中正好接触到 containerd 源码, 早年大家基本都使用 docker/podman 这类容器上层包装, 包括我也是, 故对真正容器运行时的结构部分了解得不够深入, 最近 k8s 社区在 1.21 规划上计划从 kubelet 中移除 docker-shim 交互的逻辑, 大势上组件走上 containerd-shim, 故需要着手对容器运行时更深入了解.



带着问题作为引入,

容器底层还是对 Linux LXC 的交互, 那究竟 docker daemon 中是哪个组件来完成这一步的呢？



## 1、背景知识



首先我们需要了解 Docker Daemon 生产出容器的过程.



当前整个 Docker 调用链架构可以用下图来简单概括

![docker架构](https://miro.medium.com/max/1400/1*c3AiZFHuib7FUGyINzkEag.png)



从 Docker 1.11 之后，Docker Daemon 被分成了多个模块以适应 OCI 标准。拆分之后，结构分成了以下几个部分：

![]()



16年12月 Docker公司宣布将 [containerd](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2FContainerD.io%2F) 从Docker Engine中分离，并捐赠到开源社区独立发展和运营。



一个工业标准的容器运行时，注重简单、 健壮性、可移植性



Docker本身其实已经被剥离干净了，只剩下 Docker 自身作为 cli 的一些特色功能了，真正容器的管控都在 containerd 里实现。





在 17 年 docker重命名为moby, 从命名上逐渐脱离和 container 的关系, 而 Moby 更像是一个“乐高积木”，它包含了容器化的组件库 底层的构建、日志处理、卷管理、网络、镜像管理、containerd、SwarmKit等模块, 是个大杂烩.



19年2月containerd 正式从cncf社区毕业，成为继 [Kubernetes](https://kubernetes.io/), [Prometheus](https://prometheus.io/), [Envoy](https://envoyproxy.io/), and [CoreDNS](https://coredns.io/) 之后第五个毕业项目。





## 2、生态架构



![docker生态](http://cdn.zimug.com/Docker%E7%94%9F%E6%80%81%E7%B3%BB%E7%BB%9F%E5%85%A8%E8%A7%A3%E6%9E%90docker%E7%94%9F%E6%80%81%E7%B3%BB%E7%BB%9F.png)



单看 docker 生态圈是非常庞大的, 故今天我们聚焦在运行时 containerd 生态架构上, 周边生态如下:



- OCI
- CRI
- kubelet
- dockerd
  - docker.sock (/var/run/docker.sock)
- dockershim
  - dockershim.sock (/var/run/dockershim.sock)
- containerd-cri
  - containerd.sock (/var/run/docker/containerd/containerd.sock|/run/containerd/containerd.sock)
- containerd-shim
- Runc
- RunV
- Kata
- gVisor



containerd 在容器生态中的角色, 作为容器运行时的扛把子

![p](https://yqfile.alicdn.com/6ff06eb7bacf309994df662138197cbadfd49344.png)

## 3、containerd 架构

![img](http://lexus-blog.test.upcdn.net/containerd/821618039890_.pic_hd.jpg)

简单来说分为:

- containerd-shim
- runC
- LXC 调用封装



更细化为

![containerd-arch](https://images2018.cnblogs.com/blog/952033/201805/952033-20180520115610144-588472749.png)

grpc 模块向上层提供服务接口，metrics 则提供监控数据(cgroup 相关数据)，两者均向上层提供服务。containerd 包含一个守护进程，该进程通过本地 UNIX 套接字暴露 grpc 接口。

storage 部分负责镜像的存储、管理、拉取等metadata 管理容器及镜像的元数据，通过bootio存储在磁盘上task -- 管理容器的逻辑结构，与 low-level 交互event -- 对容器操作的事件，上层通过订阅可以知道发生了什么事情Runtimes -- low-level runtime（对接 runC）

#### containerd-shim

containerd-shim 是 containerd 的一个组件，主要是用于剥离 containerd 守护进程与容器进程。containerd 通过 shim 调用 runc 的包函数来启动容器.



#### 各个组件以插件的形式注册

![plugin registry](https://oscimg.oschina.net/oscnet/34d6fddfefcff0eb7fef5542709e9006c4a.jpg)



#### cri  /run/containerd/containerd.sock
- containers/tasks/event/snapshots/namespace/tasks/image    /run/containerd/containerd.sock
- 低内存环境   /run/containerd/containerd.sock.ttrpc
- debug /run/containerd/debug.sock   /debug/pprof
- metrics   metrics.sock   /v1/metrics
- bolt   元数据存储 和etcd底层使用相同结构





#### 直接启动containerd和moby启动方式

- 检测 /run/containerd/containerd.sock是否存在，判断是否启动containerd
- 用supervisor启动containerd，直接二进制调用



#### 容器的 namespace

![容器 namespace](https://oscimg.oschina.net/oscnet/50bb0c5ab1e69aa56caf4fca07a05db3974.jpg)

容器 ns 主要目的用于在 Linux 层面做 namespace 划分, 故拆分为一下3种主流 namespace.



目前最常见的 namespace type 主要分下面两种:

- io.kubernetes.cri.container-type

- io.kubernetes.docker.type





#### containerd-shim 容器生命周期管理


- 允许 runc 在创建&运行容器之后退出
- 用 shim 作为容器的父进程，而不是直接用 containerd 作为容器的父进程，是为了防止这种情况：当 containerd 挂掉的时候，shim 还在，因此可以保证容器打开的文件描述符不会被关掉
- 依靠 shim 来收集&报告容器的退出状态，这样就不需要 containerd 来wait 子进程



因此，使用 shim 的主要作用，就是将 containerd 和真实的容器（里的进程）解耦。



而第一点，为什么要允许 runc 退出呢？ 因为，Go 编译出来的二进制文件，默认是静态链接，因此，如果一个机器上起N个容器，那么就会占用M*N的内存，其中M是一个 runc 所消耗的内存。 但是出于上面描述的原因又不想直接让 containerd 来做容器的父进程，因此，就需要一个比 runc 占内存更小的东西来作父进程，也就是 shim。



shimv2 启动一个服务，复用





#### 与OCI组件的交互



runC 是标准化的产物，为了防止一家商业公司主导容器化标准，因此又成立了opencontainers 组织



二进制直接调用



更新 config.v2.json, hostconfig.json 文件



k8s runtime class





#### docker-init



UNIX系统中，1号进程是init进程，也是所有孤儿进程的父进程。而使用 docker 时，如果不加 --init 参数，容器中的1号进程 就是所给的ENTRYPOINT，例如下面例子中的 sh。而加上 --init 之后，1号进程就会是 [tini](https://github.com/krallin/tini)：

docker-init 作用
- 避免僵尸进程
- 默认信号处理

