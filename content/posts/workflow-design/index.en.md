---
title: 浅谈 Workflow 设计
date: 2017-12-03 22:30:07
comments: true
weight: 5
draft: false
categories: ["Code", "Python"]
tags: ["Python", "Workflow"]
lightgallery: true
---

## 浅谈 Workflow 设计

<div style="text-align: right">LexusLee</div>

### 背景

最近刚接触到 workflow 相关的东西，之前都没有造过这方面的轮子，所以看了一些框架总结了一下我认为的好的 Workflow 的设计应该是怎样的。

### 什么是 Workflow ?

Workflow 是一些可重复执行的事件按特定的顺序&路径组合成的事件流，这个组成的事件流通常是为了满足某一个流程较长的任务。

这些事件通常是不可再被细分，是具有原子性的。每个原子事件可能包含执行任务、文档或数据。这些事件按照提前声明好的规则组合起来就成了一个 Workflow .

e.g. ![workflow_example](http://media.tumblr.com/457d00b6561a83fbfdda280e58182620/tumblr_inline_mmuadskv9P1qz4rgp.png)

如上图所示，Workflow 类似软件工程中的流程图，指定了每个节点可能出现的路径分支，节点执行的事情以及节点的终结状态。

### 如何设计 Workflow ?

比较经典的 Workflow design pattern 应该满足以下几个元素：

- 路径覆盖
- 事件原子性
- 有效的状态迁移

#### 路径覆盖

路径覆盖是和节点状态相关的，通常的节点状态有如下几种：

- Start  —— 开始 jobs ，标明 workflow 起点
- Maybe —— 表示这个任务可能会执行，但不一定会执行，它的执行依赖于一定条件，比如上层节点的输出
- Likely —— 和 Maybe 节点类似，但是比它的优先级更高，是作为与 Maybe 节点共享父节点的默认路径节点
- Future —— 表示 workflow 执行体认为该路径一定会到达的节点，Future 节点的任务在不被 cancel 的情况下一定会执行
- Waiting —— 表示当前任务是个阻塞任务，还在执行中，需要等待执行完毕才能进入下个路径
- Ready —— 表示 Waiting 节点的任务已执行完，作为 Waiting 节点的 handler
- Complete —— 表示整个 workflow 的 Jobs 已经全部执行完毕，为终结节点
- Cancel —— 表示任务被明确终止了，在状态迁移过程中不作为最终状态

![path coverage](http://7xse6j.com1.z0.glb.clouddn.com/workflow%20state.png)

#### 事件原子性

`pass`
