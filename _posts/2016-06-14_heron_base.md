---
type: post
layout: post
category: Heron
title: Heron实时流式计算
tagline: by Snail
tags:
  - Heron
---

##Heron简介

###Heron的由来

过去几年，Strom在Twitter上为实时分析的需求提供服务。由于Twitter的业务增长，当前的规模，以及经验。Twitter发现了Storm的局限性以及瓶颈，所以Twitter开发了Heron。


###关于Storm的瓶颈

####Worker的局限性

Strom的worker是一个复杂的设置，spout和bolt是用来执行我们的任务的task，多个task跑在executor上。而多个executor则组成一个worker，也就是一个jvm进程。一个supervisor节点上，可以跑多个worker进程。

TODO 插入图片

#####任务调度

线程被调度使用优先顺序和基于优先级的由JVM的调度算法。因为每个线程都运行几个任务，执行器实现另一个调度调用适当的任务的算法，基于传入的数据。这样的多个层次的调度和复杂的当任务是，相互作用往往会导致不确定性。


#####资源没隔离

另外，由于每个worker可以跑相同任务下的不同task，比如，一个负责数据源读入的spout的task，和一个负责数据写入的task在同一个worker中，worker没法控制，隔离资源给不同的task。也正是因为，这个分配不是固定的，假设我们任务，可能就没有这样的搭配的worker，那么我们就很难去重现这个问题，并且发现这个问题


#####日志混乱

许多的task的日志，是往同一个物理日志文件输出，甚至，当重启了任务，就会重新分配到不同的日志文件去输出，当在排查问题的时候，是个头疼的问题


#####资源分配浪费

对于资源分配，Strom假设每个worker是同样的，这个假设结果会导致低效的分配资源利用，导致过度配置。举例，有一个任务，需要3个spout和一个bolt和2个worker，而每个task需要的内存大小分别是5，10g。那么每个worker一共需要15g的内存，因为一个worker必须运行一个spout和bolt，而实际上，只需要25g的内存，但却分配了30g。


####Nimbus的问题

#####nimbus任务重

nimbus提供了调度，监控和jar分发，同时它也作为系统的指标报告组件和管理拓扑的计数器。因此，nimbus经常负载高，导致称为某个操作的瓶颈。

#####nimbus资源分配问题

nimbus不支持资源预留，提供了进程的粗力度隔离，所以，在同一台woker节点的不同拓扑任务，可以互相干扰，并且导致，问题排查困难。Heron试图在一台节点专用一个拓扑，但是这种方法会导致资源严重浪费（ps:个人觉得这个想法不该出现）。也试图通过Storm on Yarn，但发现，这也不能完全解决问题。


#####zookeeper瓶颈

nimbus使用zookeeper来管理worker和监测心跳，当集群大到一定程度后，zookeeper将会成为瓶颈。

#####单点故障

nimbus存在单点故障


####Strom缺少背压机制

Storm没有背压机制，Strom的机制是快速失败机制，如果接受者不能接受数据，那么发送者将tuple删除掉。这有2个缺点：

* 如果确认接受者异常，将会无限制的删除tuple，我们很难发现这个问题
* 上游做的工作将会丢失
* 系统的行为将会很难预测

在极端的情况下，这种设计导致的拓扑结构，而没有任何进展，同时消耗所有的资源。


####性能问题

在生产中，有几个不可预测的实例拓扑执行过程中的性能，从而导致元组失败，元组的回放，和执行滞后（数据到达率超过处理拓扑率)。最常见的导致这些性能下降的原因是：

* 随着tuple的发出，若在其中一个组件处理失败，当导致整个处理流失败，spout会重新发送该tuple，只是重新发送，并没有做什么有用的工作
* gc时间长，导致延迟及元祖故障率高
* 排队竞争，在某些情况下，有大量的竞争在传输队列，特别是当一个worker运行多个excutor。

##Heron Topology

###Topology简介

Topology是一个用来处理处理流数据的DAG图，里面的组件包括spouts和bolts，他们之间通过tuples来连接，如下图是一个简单的topology：

![图1](/Users/Alien/GitHub/alienxt.github.io/img/2016-06-14_heron_base/1.png)

Spouts负责发送tuple在topology里，而bolts负责处理这些tuple。如上图，S1发送tuple到B1和B2进行处理，B1发送tuple到B3和B4进行处理，B2则发送tuple给B4。

这只是一个简单的topology，我们还可以构建更为复杂的topology。

###生命周期

一旦我们已经构建了Heron集群，我们可以用Heron's CLI tool来管理整个topology的生命周期：

* submit	已经提交到集群，但还未开始处理数据，但已经准备好被激活
* activate	激活topology后，该topology将会按照我们构建的结构开始处理数据流
* restart	重启topology，比如我们更新结构，配置等
* deactivate 停用topolgy，一旦停用后，还会运行在集群中，只是已经不开始处理数据流了
* kill	集群中将不会运行改topology，你需要重新submit

###Spouts

Spout是一个流的源头，在拓扑中负责发送tuple。例如，我们可以从Kestrel队列，Kafka队列等读取消息，并且发送到其他bolt

###Blouts

Bolts负责从从其他bolt或者spout中接收数据，然后进行转换，处理，计算，多流的分组，聚合，持久化等操作。

###逻辑计划

一个拓扑的逻辑计划类似于数据的查询计划，如图1，就是一个拓扑结构的示例图。

###物理计划

一个物理计划于它的逻辑计划相关，但与一个物理计划映射拓扑的实际执行逻辑有很大区别，包括运行每spout和bolt的机器，逻辑计划只是一个粗略的视觉表现


##Heron的结构

Heron直接继承与Storm，但从结构来讲，他明显不同与Storm，但是他完全兼容Strom。

###Heron vs Strom

Heron直接继承与Storm，同时有两个设计的目标：

* 克服风暴的性能，可靠性和其他的缺点，通过更换风暴的线程为基础的计算模型与基于过程的模型
* 保持与风暴的数据模型和拓扑结构的完全兼容

###Heron设计的目标

* 隔离	拓扑结构应该是以过程为基础的，而不是基于线程的，每个进程都应该在隔离的过程中进行简单的调试、分析和故障排除
* 资源限制	拓扑结构应该只使用它们最初分配的和永远不会超过这些界限的资源。这使得Heron运行在共享的基础设施里安全
* 兼容性	兼容Strom，使得Strom原来的程序可以直接部署在Heron，避免学习成本
* 背压机制	在一个分布式的系统中，无法保证所有组件运行速度相同，Heron的背压机制可以使组件自我调整滞后的情况
* 高性能	许多Heron的设计，以及增强的配置，可以使的有更高的吞吐量，以及更低的延迟
* 语义保证	Heron提供了最多一次，至少一次处理语义的支持
* 效率	Heron的设计目标是使用最小的资源

###Heron的组件

* Topology Master
* Container
* Stream Manager
* Heron Instance
* Metrics Manager
* Heron Tracker

####Topology Master

Topology Master (TM)管理一个Topology的整个生命周期，从它提交到最终被杀死。当一个topology被部署上集群后，会创建一个TM，以及多个Container。同时，TM创建了一个短暂的Zookeeper节点，来确保一个Topology中只有一个TM。同时TM还会创建一个拓扑的物理计划

![图2](/Users/Alien/GitHub/alienxt.github.io/img/2016-06-14_heron_base/2.png)


####Container

每个topology由多个容器组成，其中每个容器里一个Stream Manager（SM），一个Metrics Manager（MM），还有多个Heron Instance，SM负责与MM通信，与确保能形成一个完整的图


####Stream Manager

Stream Manager（SM）管理组件之间的路由拓扑结构，负责本地的消息以及网络中的消息之间的传递，如下图：

![图3](/Users/Alien/GitHub/alienxt.github.io/img/2016-06-14_heron_base/3.png)


下面这张图表达出背压机制的设计：

![图4](/Users/Alien/GitHub/alienxt.github.io/img/2016-06-14_heron_base/4.png)

如上图，B3负责从S1接收消息，B3运行比其他缓慢。结果就是B3拒绝从C，D中获取消息，这可能导致整个拓扑的吞吐量崩溃。而一旦B3开始正常运转，集群中的SM会互相通知，拓扑路由也会回复正常


####Heron Instance

一个Heron instance是一个spout或bolt的task运行环境，目前Heron只支持java，所以是一个java进程


####Metrics Manager

每个topology会启动一个Metrics Manager (MM)，它用来收集和上报在container中的组件的指标


####Heron CLI

Heron有一个客户端工具，叫“heron”，它用来管理拓扑结构，生命周期

####Heron Tracker

Heron Tracker是一个集中式管理集群信息，拓扑结构等信息的工具，并提供了Json Rest API供外部调用

####Heron UI

Heron提供了UI便于我们看集群信息，拓扑信息，甚至可以看到拓扑的物理和逻辑计划








