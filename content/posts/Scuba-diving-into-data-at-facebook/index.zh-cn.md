---
title: "译《Scuba: Diving into data at Facebook》"
date: 2021-02-05T10:29:05+08:00
comments: true
weight: 5
draft: false
categories: ["Coding"]
lightgallery: true
---
# Scuba: Diving into Data at Facebook

原文链接: https://www.semanticscholar.org/paper/Scuba%3A-Diving-into-Data-at-Facebook-Abraham-Allen/0bce0735ef3ea01e02b73c98338727d519a71be4

## 概要

Facebook很重视性能监控。一些性能问题可能会影响超过10亿用户，所以我们跟踪了成千上万的服务器，每天上百PB的网络流量，每天上百个代码变更，以及许多其他指标。我们对时延性的要求是，在事件发生后的一分钟内，如一个客户电话请求，一个提交了bug报告，一份新的代码变更, 能及时更新到图表，并在监控系统中展示出来。

Scuba 是Facebook的数据管理系统，用于大部分的实时分析。Scuba 是一个快速的、可扩展的、分布式的、内存式的 Facebook 自研数据库。它目前吞吐率为每秒数百万行(事件)流入，并以同样的速度过期数据进行流出。Scuba 存储后端在数百台服务器上，每台服务器上的数据完全在内存中，每台服务器内存规格为 144 GB RAM。为了处理每个查询，Scuba 会从所有的数据中聚合服务器以处理每天处理近百万次查询。

在 Facebook, Scuba 广泛应用于交互式、临时性的分析查询，这些查询运行在在一秒内完成对实时数据的处理。



## 1. 介绍

在Facebook，无论是诊断性能回归还是衡量基础设施变化带来的影响，我们都希望用有效数据来衡量。Facebook基础设施团队依靠的是在实时数据监控以确保网站始终运行顺利。我们的需求包括能在非常短的延迟（通常在一分钟以内）从运行Facebook的网络服务器上查询到发生的事件。查询数据的灵活性和速度对于诊断出问题根因带来不少收益。识别问题的根本原因往往是由于各子系统之间复杂的依赖关系，很难在短时间内完成。然而，一旦出现了没有在几分钟或几小时未解决的问题。那 Facebook 的10亿用户就会变得不快乐，从而对 Facebook 整体发展不利。

为了解决上述问题， 起初，我们依靠的是预先汇总的图表和一套精心管理、手工编码的脚本，通过MySQL数据库的形式管理数据。到了2011年，这个解决方案变得过于僵化和缓慢。它无法跟上不断增长的数据吞吐率和查询效率。而 Facebook内部的其他查询系统，比如Hive[20]和Peregrine[13]，在数据被提供给查询之前，查询数据被写入HDFS，有很长的（典型的一天）延迟，而查询本身需要几分钟的时间来运行。

因此，我们建立了 Scuba，一个快速、可扩展、内存数据库。Scuba 是我们收集和分析来自各种系统的数据的方式的一个重大演变，正是这些系统保证了网站每天的正常运行。我们现在使用 Scuba 能对大多数实时的、结构化的人工数据进行分析。我们将在本文后面将 Scuba 与其他数据管理系统进行比较，但据我们了解，没有任何其他系统能像 Scuba 一样快速地吞吐数据并进行复杂的查询。

如今，Scuba 运行在数百台服务器上，每台服务器有144 GB内存，在一个无共享架构的集群中。它在内存中为1000多个表存储了大约70TB的压缩数据，通过在所有服务器上随机分配每个表来分配内存。Scuba 每秒可处理数百万行。由于 Scuba 是内存绑定的，它能以同样的速度过期掉老数据。为了限制数据量，Scuba 允许行指定一个可选的采样率，这表明 Scuba 只包含原始事件的一小部分（通常是100分之一到100万分之一）。这种采样对于像 Facebook 客户端请求这样每秒发生数百万次的事件是十分必要的。采样可以是统一的，也可以是基于一些关键数据，比如用户ID等。Scuba在计算汇总时，会对采样率进行聚合补偿。

除了提供一个 SQL 查询接口（对于一个 SQL 子集，包括分组和聚合，但不包括连接表），Scuba 提供了一个 GUI，可以生成时间序列图、饼图、列值分布，以及除文本选项之外的其他十几种数据的可视化。图1显示了一个时间序列图，在Scuba GUI中对页面流量进行了一周的比较。在后台，一个聚合树将每个查询分发到每个服务器，然后收集结果发回客户端。

虽然 Scuba 是为了支持性能分析而建立的，但它很快就成为了在其他时间敏感数据上执行探索性查询的首选系统。Facebook的许多团队都在使用Scuba。

- 移动开发团队使用 Scuba 来跟踪哪些用户在运行不同的移动设备、操作系统和Facebook应用的版本。
- 广告开发团队利用 Scuba 来监控人们对广告的印象、点击和收入的变化。当出现下降时，他们可以迅速将其缩小到特定的国家、广告类型或服务器集群，并确定根本问题。
- SRE 团队过使用 Scuba观察服务器错误。当发生峰值时，他们可以精确地指出是否是由于特定端点中的错误、特定数据中心或服务器集群中的服务，或数据中心的部分物理问题。
- 错误报告监测每小时运行数千次查询，以寻找Facebook用户报告的错误数量高峰，按几十个人口统计维度（位置、年龄、好友数等）进行分组。



![图一](http://lexus-blog.test.upcdn.net/Facebook-Scuba/361612433644_.pic_hd.jpg)

总的来说，用户会先提出高级别的汇总查询，以识别数据中的有趣现象，然后再深入挖掘（因此被称为Scuba），以找到感兴趣的基础数据点。在上述所有情况下，能够沿着多个维度对数据进行临时分解是至关重要的。

同时 Scuba 也是 Facebook 代码回归分析工具、bug 报告监控工具、实时帖子监控工具（例如，有多少 Facebook 帖子提到了电影 "Argo"?）以及许多其他工具的基础引擎。 Scuba 的主要特点是，即使在扫描数百GB的数据时，查询的执行时间也不到一秒钟，而且结果通常是实时的，超过一分钟前发生的事件。

在第 2 节中，我们描述了 Scuba 在 Facebook 具体落地的一些用例，包括性能监控、趋势发现和模式挖掘。第3节详细描述了 Scuba 的架构、存储和查询功能。在第 4 节中，我们提出了 Scuba 查询执行的简单分析模型。在第 5 节中，我们通过实验对 Scuba 进行了具体评估。我们用真实的数据和查询来研究它的加速和扩展特性。在第 6 节中，我们将 Scuba 与业界相关工作进行比较。在第 7 节中，我们以列出 Scuba 与其他大多数数据库系统的不同之处作为结论。通过上面这些差异使得 Scuba 更适合我们在Facebook的使用。



## 2. Scuba 使用场景

Scuba 目前存储了1000多个表。在本节中，我们将描述 Scuba 的几个代表性用例。

### 2.1 性能分析

Scuba 最原始也是最常见的用途是用于实时性能监控。Julie 负责监控 facebook.com 的性能。她会首先查看 Scuba 仪表板上的几十个图表，这些图表显示了服务器上的CPU负载；缓存请求、缓存命中和缓存丢失的次数；网络吞吐量；以及许多其他指标。这些图表比较了一周与一周前的性能差异，如图-1。每当她发现一个显著的性能差异时，她就会通过不同的列（通常包括堆栈痕迹）进行深入研究，细化查询，直到她能将差异锁定在一个特定的代码块上，并填写一份紧急错误报告。

Julie的仪表盘可以在不超过几秒钟的数据上运行预制查询。性能错误往往在引入后的几分钟到几小时内就能被及时发现（并修复！）-- 而且这些错误还在网站的一小部分上进行测试。这些性能图表中的峰值警报可以使她的初始监控自动化。在Facebook的所有服务器上实时记录和导入数据会太过昂贵；所以 Julie 的表格中包含了大约万分之一的事件样本（但不同的表格和事件类型会有所不同）。Scuba 会记录采样率并进行补偿。

### 2.2 趋势分析

Scuba 的另一个时间敏感型的使用场景是趋势发现。Eric 想寻找数据内容的趋势。他从用户帖子中提取词语集，并寻找词语频率随时间和许多维度的峰值：国家、年龄、性别等。和 Julie一样，Eric也在分析秒速时时彩数据。他为Facebook的传播热点建立了一个工具团队，以图表方式显示有多少帖子提到了当前的短语。例如，在奥斯卡颁奖典礼之前，这个工具就被用来实时查看过去一小时内有多少帖子包含了竞争电影的名字。与Julie不同的是，Eric通常会编写新的自定义查询，因为他在尝试新的趋势分析思路。他还会编写自定义的Javascript函数来计算数据的统计数据，比如整数列之间的共变性。

### 2.3 模式挖掘

产品分析和模式挖掘是 Scuba 的第三个用例。与Julie和Eric不同，Bob不是软件工程师。他是一名产品专家，他需要分析不同的 Facebook 用户如何应对网站或移动应用的变化。他根据不同的维度，如用户的位置和年龄、产品（设备、操作系统和构建）和错误报告中的关键词，寻找特定的模式，而不知道哪些维度可能很重要。鲍勃在不知道数据是如何被记录的，也不知道哪些列可能存在的情况下，查看许多表中的数据。他寻找他能找到的任何模式。他使用 Scuba 是为了以毫秒为单位运行滚动查询，而不是在 Hive 中需要几分钟。

## 3. Scuba 总览

在本节中，我们将提供 Scuba 的架构概述。如图2所示，Scuba 的存储引擎由许多独立的服务器组成，每个服务器被划分为称为 "叶子节点"（或叶子）的逻辑存储单元。cpu核心的数量决定了叶子的数量，目前每台服务器有8个。每个叶子包含了大部分表的数据分区，所有的查询都会进入所有的叶子节点，如下所述。现在我们介绍一下Scuba的数据模型。

### 3.1 数据模型

Scuba 为其用户提供了一个标准的数据模型。每个表都有包含四种可能类型的数据列的行。

1. 整数。整数用于聚合、比较和分组。时间戳也存储为整数。
2. 字符串：字符串用于比较和分组。字符串用于比较和分组。
3. 字符集：字符串集用于表示，比如说，Facebook帖子中的单词或特征集，Facebook帖子中的某个单词，或者是某一用户真实的功能集（如图搜索、新闻源重新设计等）。都使用字符串集来表示。
4. 字符串的矢量。字符串的Vectors是有序的，多用于堆栈痕迹，顺序对应堆栈级别。

请注意，不支持浮点数；在许多叶子上的浮点数上的聚合可能会导致太多准确度上的错误。作为替代，Scuba 建议用户选择他们关心的小数点位数，例如 5，并将 trunc(X ∗ 105) 存储为一个整数，而不是浮点 X。
由于 Scuba 捕获的数据是关于时间变化的现象，所以每一行都有一个强制性的时间戳。这些时间戳代表实际事件（客户端请求、错误报告、帖子等）的时间。任何表都可以有任意数量的一种或多种类型的列。所有的表都包含整数和字符串；只有一些表包含字符串的集合或向量。

### 3.2 数据结构

图3显示了 Scuba 对每种数据类型所使用的压缩方法。可以用N个字节-1位自然表示的整数，直接用N个字节编码。字典编码是指每个字符串在字典中存储一次，其索引在字典里（一个整数）存储在行中。字符串列可以被压缩或不压缩存储，这取决于有多少不同的值。对于压缩后的字符串列，每个索引使用表示最大索引所需的位数来存储。未压缩的列存储原始字符串及其长度。对于字符串集，索引被排序和delta编码，然后每个索引被Fibonacci编码。(斐波那契编码使用的是可变位数。)编码后的索引在行中连续存储。对于向量，有一个2字节的字符串计数和一个1字节大小的最大字典索引的位数。然后，每个索引使用大小位数连续存储在行中。所有的字典都是每个叶子的局部，每个列的字典都是独立的。与在每列中存储8个字节的整数和原始字符串相比，压缩数据使其体积减少了6倍以上（每个表的情况不同）。

Scuba 目前以行的顺序存储表，因为它最初的用例在每次查询中都会访问表的大部分列。鉴于 Scuba 的用例从那时起变得更加通用，我们现在正在探索面向列的存储布局。其他人[18，8]已经表明，列存储一般可以得到更好的压缩和更好的缓存定位。

Scuba 的数据模型有两个关键点与标准关系模型不同。首先，没有创建表声明类型；每当叶子第一次接收到数据时，就会在每个叶子节点上创建一个表。由于叶子在不同的时间接收数据，表可能只存在于一些叶子上，并且可能在每个叶子上有不同的模式。然而，尽管模式不同，Scuba 通过将任何缺失的列视为空值，向用户展示了一个单一的表映像。其次，表的行中的列可能被自动地填充；在一个表中有 2 或 3 个不同的行模式或一个列随着时间的推移改变其类型（通常是为了实现更好的压缩）是很常见的。这两种差异加在一起，让 Scuba 无需任何复杂的模式演化命令或工作流，就能让表适应用户的需求。这种适应性是 Scuba 的优势之一。

### 3.3 数据提取, 分发与生命周期

图2显示了数据导入 Scuba 的吞吐路径。Facebook 的代码库其中包含日志调用，用于将数据导入 Scuba。当事件发生时，这些调用会被执行，并且（在根据可选的采样率剔除部分数据之后）日志条目被写入Scribe。Scribe 是一个开源的分布式消息系统，用于收集、聚合和以低延迟交付大量日志数据。它由Facebook开发，并在Facebook广泛使用[5]。然后，一个尾随者进程订阅了为 Scuba 准备的主题，并通过 Scuba 的Thrift API将每一批新的行发送到 Scuba。(Thrift[7]是一个软件库，它为使用它定义的任何接口实现了跨语言的RPC通信)。这些传入的行完全描述了它们自己，包括它们的Schema。

对于每一批传入的行，Scuba 会随机选择两个叶子节点，并将该批行发送到有更多可用内存的叶子上。因此，每个表的行最终会被随机分区到集群中的所有叶子上。在任何表上都没有索引，尽管每个批次中的行在很短的时间内都有时间戳（然而，这些时间窗口可能在批次之间重叠，因为数据是在多个服务器上生成的）。

接收批处理的叶子节点将批处理文件的gzip压缩副本存储到磁盘上，以便持久化。然后它读取新行的数据，压缩每一列，并将行添加到内存中的表中。从一个事件发生到它被存储在内存中并可供用户查询的时间通常在一分钟之内。

![图二](http://lexus-blog.test.upcdn.net/Facebook-Scuba/371612446752_.pic_hd.jpg)

内存（不是CPU）是 Scuba 的紧缺资源。我们目前每隔2-3周增加新机器，以跟上表和数据的增长速度。由于Scuba的目的是为了分析今天的数据，同时可能会进行周与周之间的比较，所以我们以接收新数据的同样速度删除旧数据，以限制表的大小。数据被修剪的原因有两种。

- 数据生命周期：由时间戳决定的行是否太老了。
- 数据大小: 该表是否已经超过了空间限制，而这一行是：
  表中最古老的一个。

通常情况下, 大多数表的默认限制为30天1和100GB，尽管对于像fbflow这样保存网络流量数据的高容量表和记录广告点击和印象等创收数据的广告指标有更高的限制。每隔15分钟，一个cron作业就会驱逐超过其生命周期限制的数据。如果表超过了它的空间限制，表的最旧数据就会被驱逐，直到它低于其限制。

为了让一些数据保留的时间超过空间限制，Scuba 还提供了数据的子采样。在这种情况下，超过一定生命周期的的行的统一部分会被保留，其余部分会被删除。在未来，我们希望探索更复杂的抽样形式，如分层抽样，这中抽样可能会选择一组更有代表性的行。

### 3.4 查询模型

Scuba 提供了三种查询接口，如图2右图所示。

- 图1中所示的基于网络的界面允许用户发出基于表格的查询，并从12种可视化方式中选择一种，包括表格、时间序列图、饼图、叠加区域图等。图4、图5和图6显示了另外三种可视化方式。在不同的可视化界面之间的切换只需要几秒钟的时间。
- 支持命令行界面接受SQL的查询。
- 支持基于Thrift的API允许从PHP、C++、Java、Python、Javascript等语言的应用程序代码中进行查询。

SQL接口和GUI本身使用Thrift接口向 Scuba 的后台发送查询。脚本也可以使用SQL或Thrift接口发出查询。最后，我们提供了一种机制支持用Javascript编写的用户定义函数进行查询。

Scuba 查询具有以下SQL查询的表达能力。

```sql
SELECT column, column, ...,
  aggregate(column), aggregate(column), ...
FROM table
WHERE time >= min-timestamp
  AND time <= max-timestamp
 [AND condition ...]
GROUP BY column, column, ...
ORDER BY aggregate(column)
LIMIT number
```

Scuba 的 SQL 支持的聚合函数包括传统的计数、最小、最大、总和和平均函数，以及其他通用函数，如总/分钟、百分比和直方图。WHERE子句必须包含一个时间范围，LIMIT子句的默认值是100,000行，以避免分组时的内存问题和客户端的渲染问题。GROUP BY和ORDER BY子句完全是可选的。

任何与字符串的比较都可能包括一个正则表达式。字符集的条件是 set-column 包括 string-list 的 any/all/none，set-column 为空。对字符串向量的条件是 vector-column 包括 string-list 的 any/all/none/start/end/within。字符串列表中字符串的顺序很重要。

Scuba 中不支持 join 表查询。当需要合并来自多个来源的数据时，通常在将数据移植到 Scuba 之前进行连接，保证 Scuba 中的已经是连接过的数据。

![图3](http://lexus-blog.test.upcdn.net/Facebook-Scuba/381612447414_.pic_hd.jpg)

### 3.5 执行查询

**注: fanout 指数据扇出**

图7显示了 Scuba 如何执行用户查询的步骤分解。聚合器和叶子之间的所有通信都是通过Thrift进行的。具体查询步骤如下:

1. 客户端找到 Scuba 集群中的一个根聚合器并发送一个查询。Root Aggregator 收到查询，解析它，并验证它，以确保查询格式良好。
2. Root Aggregator 在集群中识别出另外四台机器，作为下一级的Intermediate Aggregator。这一步创建了一个五台机器的 fanouts（其他四台机器加上自己）。Root Aggregator用一个 sum 和一个 count 来替换任何平均函数（因此可以在最后进行聚合），并将查询发送给Intermediate Aggregators。
3. Intermediate Aggregator 再创建五个fanouts，并传播查询，直到每台机器上的（唯一）叶子聚合器收到查询。
4. Leaf Aggregator将查询发送到机器上的每个叶子节点服务器上并行处理。
5. Leaf Aggregator 从每个叶子节点服务器收集结果，并对它们进行聚合。聚合结果可以进行任何排序和限制约束，然而，对于它的限制，默认使用max(5 ∗ limit，100)，确保最后最高限制的所有组都能在树上一路传递。Leaf Aggregator还收集统计每个Leaf是否包含该表，它处理了多少行，以及有多少行满足条件并对结果有贡献。Leaf Aggrega-tor将其结果和统计数据返回给调用它的Intermediate Aggregator。
6. 每个Intermediate Aggregator将收到的部分结果进行整合，并将其传播到聚合树上。
7. Root Aggregator 计算最终结果，包括任何平均值和百分比，并应用所有预设的的排序和限制条件。
8. Root Aggregator将结果返回给等待的客户端，通常在几百毫秒内。

关于Leaf Server如何处理查询的几个细节很重要。首先，每个Leaf Server可能包含零个或多个表的分区，这取决于表的大小和表存在的时间。非常新的或非常小的表可能只存储在我们集群中数千个叶子中的几个或几百个叶子上。

其次，Leaf Server必须扫描表的每个分区中时间范围与查询时间范围重叠的行。

非重叠的分区被跳过。这些分区的时间范围是 Scuba 唯一的 "索引 "形式。

![图六](http://lexus-blog.test.upcdn.net/Facebook-Scuba/391612449236_.pic_hd.jpg)

![图七](http://lexus-blog.test.upcdn.net/Facebook-Scuba/401612449243_.pic_hd.jpg)

第三，Leaf服务器优化了每个字符串列谓词的正则表达式匹配。每当字符串列使用一个字典（大致对应于每当字符串值很可能在列中重复的时候），Leaf服务器就会维护一个每查询的缓存，其中包含对每个字符串值匹配表达式的结果。这个缓存是以字符串的字典索引为索引的。

第四，如果一个聚合器或叶子服务器在超时窗口内（如10毫秒）没有响应，它的结果就会从最终计算中省略。我们发现，在实践中，这种方法效果很好，因为在回答Scuba查询时涉及大量的样本。少量数据的缺失并不会对平均和百分位数计算产生不利影响，通过忽略一些叶子实现的低得多的响应时间弥补了缺失数据。此外，Scuba 维护并检查一个独立的服务，以获得每个查询的每个表的预期行数。然后，它使用这个计数来估计数据的部分，以确保数据的准确性。

 但数据准确性实际上是缺失的。如果分数是99.5%或更少，Scuba 会在GUI中打印一个警告。我们还在努力为那些需要100%准确结果的客户自动重复查询。

最后，每台物理机器上都运行着多个 Leaf 服务器和一个 Aggregator 服务器。每台机器都可以在聚合树的任何级别上提供聚合器服务器。

目前，聚合树的 fanouts 是五个。我们试验了 2,4,5,6 副本数的fanouts，经验上发现5个fanouts能产生最好的响应时间。独立地，Rosen等人[14]对聚合树的理论分析表明，响应时间最小的聚合树的fanouts是一个常数，与树中叶子节点的数量无关。他们的实证分析也表明，在他们的系统中，fanouts为5时导致响应时间最小。

![图八](http://lexus-blog.test.upcdn.net/Facebook-Scuba/411612449466_.pic.jpg)

## 4. Scuba 的性能分析模型

如上一节所述，Scuba使用聚合树来执行用户查询。在本节中，我们构建了一个简单的系统分析模型。通过这个模型帮助我们了解单个查询的性能特征。
图8描述了这个模型的参数。我们首先描述用于查询处理的聚合树的参数。聚合树中fanout F和N台机器的预期级别数L如公式1所示。

![公式1](http://lexus-blog.test.upcdn.net/Facebook-Scuba/421612449541_.pic.jpg)

大多数聚合器的实际 fanouts 量为 F 。然而，当 Scuba 集群中的机器数量 N 不是 F 的完美幂数时，最低一级的中间聚合器的 fanouts 量小于 F。等式 2 显示了聚合树中倒数第二层的fanouts。

![公式2](http://lexus-blog.test.upcdn.net/Facebook-Scuba/431612449622_.pic.jpg)

F L-1代表聚合树中L-1层的聚合器数量。因此，R代表在树的最后一级，从F L-1中间聚合器中的每一个到达所有N个叶子聚合器的fanouts。回想一下，每台机器都有一个Leaf Aggregator）。
现在，我们可以用等式3描述L层的fanouts。

![公式3](http://lexus-blog.test.upcdn.net/Facebook-Scuba/441612449709_.pic.jpg)

等式(3)中的最后一种情况描述了从叶子Aggregator到同一台机器上的叶子节点的fanoutD。
树中的每个Aggregator执行以下步骤。

1. 将查询分发到每个子Aggregator并等待结果。
2. 在收到每个子节点的结果时，将其纳入。当所有子代都有响应或查询超时时，停止等待。
3. 将合并的聚合结果和响应统计信息返回给调用者。

请注意，一旦查询被Aggregator转发给它的子代，子代就会并行进行。所以总的响应时间为在公式4中，用L级的聚合树计算的查询由TL表示。

![公式4](http://lexus-blog.test.upcdn.net/Facebook-Scuba/451612449829_.pic.jpg)

展开式(4)并结合式(3)可得式5。

![公式5](http://lexus-blog.test.upcdn.net/Facebook-Scuba/461612449901_.pic.jpg)

TL因此是任何查询的预测时间（在没有查询争用的情况下）。我们通过在每个查询中插入TA、TD、TG和TS的实际值，并将它们与我们的实验结果进行比较，来验证（并完善）这个模型。然而，我们并没有在这里提出比较。

## 5. 实验性评估

在本节中，我们介绍了在160台机器的测试集群上测量 Scuba 的响应时延和扩展性相关的实验结果。

### 5.1 实验环境搭建

我们测试集群中的机器是英特尔Xeon E5-2660 2.20 GHz机器，内存为144 GB[19]。有四个机架，每个机架有40台机器，用10G以太网连接。操作系统是CentOS 5.2版本。
在这些实验中，我们将集群中的机器数量从10台变化到160台。在每个实验中，每台机器有8个Leaf Nodes和1个Aggregator。每个Aggregator总是作为叶子Aggregator，并可以另外作为中间和根Aggregator。

### 5.2 实验数据和查询

实验的目的是为了隔离第4节中模型的各种参数，呈现最基础的 benchmark。因此，我们使用1个包含29小时价值的真实数据的表（从我们的生产集群复制）和2个非常简单的查询。表中的数据总量约为1.2TB。除非另有说明，每个叶子有1GB；每台机器有8个叶子，160台机器。(1 ∗ 8 ∗ 160 = 1280) 我们运行了两个不同的查询。

```sql
SELECT count(*), SUM(column1) as sum1,
   SUM(column2) as sum2
FROM mytable
WHERE time >= now()-3*3600
```

上面SQL中显示的第一个查询，在叶子上隔离扫描时间。它扫描了3个、6个或全部29个小时的数据（取决于常量），计算出3个聚合指标，并将这3个数字（和查询统计）传递到聚合树上（仅）。

```sql
SELECT count(*), sum(column1) as sum1,
   service,
   (time - now())/60*60 + now() as minute,
FROM mytable
WHERE time >= now()-3*3600
   and time <= now()
GROUP BY service, minute
ORDER BY sum1 DESC
LIMIT 1800
```

第二个查询比较复杂，尤其是在SQL中，如上图所示。该查询以 1 分钟的粒度生成过去 3、6 或 29 小时的时间序列图，为前 10 个服务中的每个服务绘制一条线。同样的查询在Scuba GUI中更容易执行，如图9所示。该查询为每个服务产生2个聚合指标，每分钟。1800=180分钟∗10个服务的限制。这个查询通过，并在聚合树的每一级聚合2个度量，每个度量都有1800点，其聚合时间TA是明显的。

![图9](https://i.loli.net/2021/02/04/CbWsdiM9pGhxYy6.png)



### 5.3 单机实验

第一组实验测试单个客户端的查询延迟。对于这些实验，我们使用29小时数据生命周期的版本的每个查询，并运行每个查询200次。我们绘制了平均响应时间，误差条表示最小和最大响应时间。

#### 5.3.1 加速查询

我们首先测量了分布在20台机器集群中的数据的单次查询的速度。我们将每个Leaf的数据量从1GB到8GB不等。因此，数据总量从160GB到整整1.2TB不等。
图10显示了结果。每个叶子扫描数据的时间与数据量成正比。然而，聚合成本与每个叶子的数据量无关，它是查询和集群大小的函数。在这个实验中，集群大小是恒定的。20台机器，扇出量为5，树上有3个层次（1个根聚合器，5个中间聚合器，20个叶聚合器）。扫描查询只经过聚合树上的一个点，所以聚合需要的时间可以忽略不计。时间序列查询需要在树的每一级聚合大量的点，所以需要更长的时间。

#### 5.3.2 扩展

接着，当我们将集群中的机器数量从10台变化到160台（每次将机器数量增加一倍）时，我们会衡量规模的扩大。每个Leaf有1GB的数据。
图11显示，扫描数据的时间（在每个Leaf上并行完成）是恒定的。聚合成本随着N的增长而呈对数增长。由于对于扫描查询来说，聚合成本可以忽略不计，所以随着机器数量的增加，其响应时间是恒定的。然而，时间序列查询需要在每个聚合器处聚合许多点，其响应时间随着聚合器的数量和聚合树的层数增加而增加。第4节中提出的模型对帮助我们理解这些扩展结果非常有用。

![图11](https://i.loli.net/2021/02/04/pkXOjiw4LsNPVUv.png)

### 5.4 多机实验

最后一个实验测试的是随着客户端数量从1个增加到32个，查询延迟和吞吐量。每个客户端连续发出200个查询，它们之间没有时间间隔。在这个实验中，我们使用160台机器，每个Leaf有1GB的数据。我们还使用了扫描和时间序列查询的3小时、6小时和29小时变体。3小时变体只需要扫描每个Leaf处大约10%的数据。

图12显示了当我们改变客户端数量时，每个查询的吞吐量。首先要注意的是，对于每个查询，吞吐量随着客户端数量的增加而上升，直到叶子上的CPU达到饱和。之后，吞吐量趋于平缓。对于所有的查询，在8个客户端之后，吞吐量是持平的。接下来的无表点是，在相同的时间范围内，扫描查询的吞吐量都比它们的时间序列查询对应的吞吐量高。这一点并不奇怪，因为扫描查询的速度更快。最后，对于给定的客户端数量，随着查询的时间范围（因此扫描的数据量）的增加，每种查询类型的吞吐量都会下降。

![图12](https://i.loli.net/2021/02/04/Wbjfy5txJT2GMkF.png)

![图13](https://i.loli.net/2021/02/04/hAoln82MYSLtiBC.png)

图13显示，查询的响应时间与客户端数量成正比增加，符合预期。

## 6.相关工作

与 Scuba 相关的工作可分为四类。

首先，有其他的系统用于临时性的实时分析。HyPer[11]也是将数据存储在内存中，但是是在一台巨型的、昂贵的机器上。另一方面，Scuba 使用相对便宜的商品计算机集群，并且通过增加更多的机器轻松扩展。HyPer也不使用压缩，因此对于同样数量的数据，它需要更多的内存。

Powerdrill[10]和Dremel[12]是Google的两个用于分析的数据管理系统。两者和 Scuba 一样，都是高度分布式的，而且扩展性很好。Dremel提供了比 Scuba 复杂得多的半结构化和稀疏数据模型；据报道，Powerdrill的速度比Dremel快得多，但仍需要30-40秒/查询。在这两个系统中，数据的主副本都住在磁盘上，所以存储的数据量不那么重要。

Splunk[6]也是一个导入日志数据、分析数据并以图表形式查看的系统。它的目的是针对在 "云 "中产生和分析的数据。我们没有关于它无论是导入还是查询数据的速度的数据。

其次，有多种系统使用压缩来重新减少内存和/或磁盘的占用。Westman等人[21]采用基于行的布局，发现字典在压缩率和CPU使用量之间提供了很好的权衡。C-Store[18]和Vertica、SAP Hana[15]、Dremel和Powerdrill都采取了基于列的方法，我们接下来想尝试一下，以获得更好的压缩效果。
第三，上述系统都不能用准确率来换取响应时间，Scuba 有意这么做。然而，BlinkDB[9]可以在有界时间内或有界精度的情况下，在采样数据集上产生结果。虽然BlinkDB在获取查询之前需要预先计算分层样本，但我们希望通过实验并优化其底层技术来更好地约束和报告 Scuba 的不准确度。
最后，Scuba 中的所有数据都有时间戳，很多分析都是基于时间的。在20世纪80年代末，Snodgrass 创建了 TQuel[16，17] 来推理数据的时间和间隔。我们应该重新审视这项工作，看看是否有 Scuba 应该加入的功能。

## 7. 总结

Scuba 与大多数数据库系统有多种不同，文章所描述的这些都集中在 Scuba 适合我们在 Facebook 的使用案例上。

- Scuba 驱逐数据的速度与它摄入数据的速度一样快，因为所有的表数据都存储在内存中，而内存是稀缺资源。驱逐是自动的，不过每个表保留多少数据的参数是可以调整的。
- Scuba 设计上期望表包含的是采样数据而不是原始数据，因为存储每个事件都会有太多的数据。采样率列会被特殊处理，查询结果包含一个原始计数和一个根据表中的采样率调整的计数。(可以有多个不同的采样率，每行可以有一个。)
- Scuba 的数据导入就是在代码中插入，并创建一个进程来监听这些事件。不需要任何模式声明；模式是从日志中推导出来的，它可以随着时间的推移而变化。

Scuba 并不打算成为一个完整的 SQL 数据库，它支持分组和聚合，但不支持连接或嵌套查询。它支持分组和聚合，但不支持连接或嵌套查询。它支持整数和字符串，但不支持浮动，尽管它也添加了字符串的集合和向量（但不支持类型的任意嵌套）。
我们想对Scuba进行一些修改。它目前是面向行的，尽管我们正在探索面向列的布局是否会更好。Scuba 可以使用本地警报；此外，支持用户在 Scuba 查询结果之上写警报。随着用户群的增长，Scuba 还需要继续扩展。

本文介绍的模型和实验是弄清它目前的扩展情况的第一步。
尽管如此，自两年前首次编写 Scuba 以来，Scuba 在 Facebook 的用户数、数据数和查询数都在飞速增长。Scuba 提供了导入和查询数据的灵活性和速度，这对 Facebook 的实时性能和数据分析至关重要。
