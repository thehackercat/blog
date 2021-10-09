---
title: Kube-scheduler 调度模型与美团本地化改造
date: 2021-10-09T09:57:53+08:00
comment: true
weight: 5
draft: false
categories: ["Coding"]
tags: ["Kubernetes", "Linux"]
lightgallery: true
---
## 背景

今天主要分享下我在美团 Hulk 团队调度系统组期间学习到的 k8s scheduler 调度模型及美团对于调度器相关改造



### Hulk 团队介绍

Hulk 是美团负责容器云平台的团队, 我所在的组是调度系统组下面的 k8s 小组. 主要负责所有 k8s 相关组件开发维护.

![Hulk 组织架构图](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/2d00a5af-0eee-4522-8995-ddf2c22ff8cb/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20211009%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20211009T020410Z&X-Amz-Expires=86400&X-Amz-Signature=9eeb22d82330a778eb88931c1dda08c1f3c28d2591f49f453ef0407ea8a5ea83&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22)

### Kube-scheduler 调度模型介绍

*(本次介绍基于 k8s 1.16 版本)*

pod 是 k8s 工作负载的最小实体，也是 scheduler 调度的主要对象。

k8s 调度器所做的工作，即将 pod 绑定到指定节点上，为 pod 选择合适的节点。

Scheduler 会先 watch apiserver 获取其中待调度的 pod, 即 pod.spec.nodeName 字段为空的 pod 对象, 通过调用 pod informer 的 handler 并将该 pod 加入调度队列中。

默认情况下, k8s 调度队列是一个 PriorityQueue, 并且当某些集群信息发生改变时, 调度器会对调度队列的内容进行一些特殊操作. 在这儿不论述.

![pod 调度序列图](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/7c7bbabd-8d97-482e-9693-a44977b8350e/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20211009%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20211009T020949Z&X-Amz-Expires=86400&X-Amz-Signature=5a5f172dc3031b9d5f841eb6bdcbc0428e72648891a947422d7196bc77969287&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22)

一次调度过程在判断一个 Node是否可作为目标机器时，主要分为三个阶段：

- Predicates 预选阶段：硬性条件，过滤掉不满足条件的节点，这个过程称为 Predicates。这是固定先后顺序的一系列过滤条件，任何一个 Predicate 不符合则放弃该 Node。
- Prioritites 优选阶段：软性条件，对通过的节点按照优先级排序，称之为 Priorities。每一个 Priority 都是一个影响因素，都有一定的权重。
- Select 选定阶段：从优选列表中选择优先级最高的节点，称为 Select。选择的 Node 即为最终部署 Pod 的机器。

![调度流程图](https://p0.meituan.net/travelcube/ef026e009a90602b853af323f58af5c3266174.jpg)

### 预选阶段

Scheduler 起 n 个 goroutine 并发对整个集群的所有 nodes 进行 predicates 串行运算过滤规则，直到最终过滤出符合条件的 nodes 列表

### 打分(优选)阶段

优选阶段，也叫打分阶段，采用 0-10 分加权平均制, 每个优选算法有一定的权重, 优选算法*权重后的积分则为优选阶段最终评分, 0 分为最低分, 10 分为最高分.

最终会选取打分最高的节点作为调度分配节点.

#### 打分规则

最常用为 LeastRequestedPriority 打分规则. 可以简单总结为:

即, 选择出宿主机空闲资源(cpu/memory)最多的节点。

同时还有如 BalancedResourceAllocation 打分规则, 其标准是判断调度完成后，所有节点里各种资源分配最均衡的那个节点. 随着社区 scheduler 不断演进, 也不断增加更细粒度的打分算法.

节点打分算法我们可以抽象为以下公式:

![节点打分算法](https://static001.infoq.cn/resource/image/cc/1a/ccf8e76258ee68f4fa3085caa881111a.png)



## 美团优化

由于早期 kube-scheduler 版本在大规模集群下存在一定的调度时延问题, 未解决该问题, 美团也做了定制化改造.

### 预选中断机制

通过深入分析调度过程可以发现，调度器在预选阶段即使已经知道当前 Node不符合某个过滤条件仍然会继续判断后续的过滤条件是否符合。试想如果有上万台 Node节点，这些判断逻辑会浪费很多计算时间，这也是调度器性能低下的一个重要因素。

为此，我们提出了“预选失败中断机制”，即一旦某个预选条件不满足，那么该 Node即被立即放弃，后面的预选条件不再做判断计算，从而大大减少了计算量，调度性能也大大提升。如下图所示：

![改造1](https://p1.meituan.net/travelcube/f0fbc86f4397e6b51a6546a53669cfbc181723.jpg)

### 局部最优解

当前调度策略中，每次调度调度器都会遍历集群中所有的Node，以便找出最优的节点，这在调度领域称之为BestFit算法。但是在生产环境中，我们是选取最优Node还是次优Node，其实并没有特别大的区别和影响，有时候我们还是会避免选取最优的Node（例如我们集群为了解决新上线机器后频繁在该机器上创建应用的问题，就将最优解随机化）。换句话说，找出局部最优解就能满足需求。

假设集群一共1000个Node，一次调度过程PodA，这其中有700个Node都能通过Predicates（预选阶段），那么我们就会把所有的Node遍历并找出这700个Node，然后经过得分排序找出最优的Node节点NodeX。但是采用局部最优算法，即我们认为只要能找出N个Node，并在这N个Node中选择得分最高的Node即能满足需求，比如默认找出100个可以通过Predicates（预选阶段）的Node即可，最优解就在这100个Node中选择。当然全局最优解NodeX也可能不在这100个Node中，但是我们在这100个Node中选择最优的NodeY也能满足要求。最好的情况是遍历100个Node就找出这100个Node，也可能遍历了200个或者300个Node等等，这样我们可以大大减少计算时间，同时也不会对我们的调度结果产生太大的影响。

**方案1：**

每次调度只对 K 个 node

![调度改造2.1](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/cc70e943-b5e8-470d-8d2f-e35b54dfed42/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20211009%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20211009T021735Z&X-Amz-Expires=86400&X-Amz-Signature=13c77713f44a98fa842c66af74906eb857e90e8da1b0d3de41c6e1cde64e0573&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22)

**方案2：**

每次调度对全量 node, 一旦某一个打分超过水位线, 则终止其他 node 打分

![调度改造1.2](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/a46f7af3-bd50-402d-ad84-a0ebc6fb41a9/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20211009%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20211009T021802Z&X-Amz-Expires=86400&X-Amz-Signature=96bee6946eb515e5554813590f64f34b88aaba57bbff6d4c03401b57a8b5e42f&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22)

### PriorityQueue 改造

PriorityQueue 其实内部细分为多个 Queue 数据结构.

- ActiveQ 待调度队列
- BackOffQ Bind失败队列
- UnscheduledQ 调度失败队列
- 其余与本次改造无关的队列(此处不一一列举)...

当某一时刻出现大量调度失败的高优先级 bad pods 时，由于重调度组件和 priorityQueue 机制，会导致这些 bad pods 重新回到 activeQ 队列头, 导致低优先级的正常 pod 无法调度.

![调度 bad case](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/620dec26-9db1-42b1-aa15-221f3923cfc0/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20211009%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20211009T021859Z&X-Amz-Expires=86400&X-Amz-Signature=3a6a04d2c8be2dbd707d9db382a210ff00fc7d2ee8eb711b85b6c5f778272567&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22)

**降级方案:**

当 ActiveQ 出现大量积压时, 且存在 Unscheduled pods 时 PriorityQueue 降级为 FIFO Queue, 确保 bad pods 在下一个调度周期进入到队列尾, 保证 good pod 能被正常调度

