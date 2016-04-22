---
type: post
layout: post
category: Storm
title: Strom实时流式计算
tagline: by Snail
tags:
  - Storm
---

整理了一下Storm的入门知识

<!--more-->

#Storm实时流式计算

##Strom介绍

###实时流式计算背景

实时处理

要说实时处理就得先提一下实时系统(Real-timeSystem)。所谓实时系统就是能在严格的时间限制内响应请求的系统。例如如果某系统能严格保证在10毫秒内处理来自网络的NASDAQ股票报价，那么这个系统就可以算作实时系统，至于系统是通过软件还是硬件或者通过怎样的设计达到的都不限。
实时处理与实时系统类似,但软件工业中似乎对实时二字没有什么明确的定义。例如许多人说实时交易，实际上是因为市场数据瞬息万变，决策经常在毫秒间。一个软实时(Soft Real-time)的例子是Amazon要求所有软件子系统在处理99%的请求时，都能在100-200毫秒内要么给出结果要么立刻失败。

流式处理

望文生义，流式处理就是指源源不断的数据流过系统时，系统能够不停地连续计算。所以流式处理没有什么严格的时间限制，数据从进入系统到出来结果可能是需要一段时间。

实时流式计算

实时流式处理就是实时处理和流式处理的结合，不仅有源源不断的数据流过系统，通过计算后，能在短时间内响应结果。

为什么需要实时计算？

伴随着信息科技日新月异的发展，信息呈现出爆发式的膨胀，人们获取信息的途径也更加多样、更加便捷，同时对于信息的时效性要求也越来越高。举一个例子，一个用户昨天在淘宝上买了一件衣服，今天呢，想在淘宝上买双鞋子，于是今天在淘宝上，发现淘宝推荐的都是衣服，裤子，对今天搜索的鞋子完全没有推荐。

###实现实时计算系统
* 低延迟   实时系统的延迟必须得低
* 高性能   性能不高就是浪费资源
* 分布式   数据量大，业务场景复杂，单机解决不了
* 可扩展   伴随着业务的发展，我们的数据量、计算量可能
会越来越大，所以希望这个系统是可扩展的
* 容错    这是分布式系统中通用问题，一个节点挂了不能影响应用

![](/img/2015-10-31-storm_base/post-bg-storm-02.jpg)

###Storm是什么？

Apache Storm是一个免费、开源的分布式实时计算系统。相对于Hadoop适用于批处理而言，Storm可以用于实时处理流式数据。Storm简单易用，支持多种编程语言。2014年Apache软件基金会宣布，Storm成为Apache顶级项目。

![](/img/2015-10-31-storm_base/post-bg-storm-03.png)

###为什么使用Storm？

* 简单的编程模型：类似于MapReduce降低了并行批处理的复杂度，Storm降低了并行实时流处理的复杂性
* 多语言支持：可以各种语言开发，默认支持Clojure，Java，Ruby，Python等
* 容错性： Storm的所有组件都是快速失败，并且无状态（Zookeeper）
* 分布式可水平扩展：master/slave结构，master管理worker，worker可扩展
* 可靠的消息机制：Storm保证每个消息至少能得到一次完整的处理，任务失败时，会将消息重发
* 本地模式：Strom有一个本地模式，可以在开发和测试中，完全模拟集群

###Storm的基本概念

* Nimbus：  守护进程，负责资源分配和任务调度
* Supervisor：守护进程，负责接受nimbus分配的任务，启动和停止属于自己管理的worker进程
* Worker：运行具体处理逻辑的进程
* Executor：worker中同一个spout/bolt task所属的物理线程
* Task：work中每一个spout/bolt的线程称为一个task，同一个spout/bolt可能会共享同一个Executor
* Topology：Storm的application
* Spout：在一个topology中产生源数据流的组件
* Bolt：在一个topology中处理业务逻辑的组件
* Tuple：一次消息传递的基本单元
* Stream grouping：消息流的分组，可随机，可根据key做分流

![](/img/2015-10-31-storm_base/post-bg-storm-04.jpg)


##Storm环境搭建

[*集群安装*](http://storm.apache.org/documentation/Setting-up-a-Storm-cluster.html)

##Storm开发流程

###Storm应用程序编写
1. Topology 编写
2. Spout编写
3. Bolts编写
4. 本地模式和集群模式

####构建ExclamationTopology (官网demo)

```java
TopologyBuilder builder = new TopologyBuilder();        
builder.setSpout("words", new TestWordSpout(), 10);        
builder.setBolt("exclaim1", new ExclamationBolt(), 3)
        .shuffleGrouping("words");
builder.setBolt("exclaim2", new ExclamationBolt(), 2)
        .shuffleGrouping("exclaim1");
```

该topology包含一个名字为“words”的spout（数据源），设置了10个的并行度，随机往名字为exclaim1的bolt（处理节点）喷发数据，设置了3个的并行度，最后又有exclaim1，这里设置的并行度为2，喷发tuple到exclaim2上

####编写TestWordSpout

* 实现IRichSpout接口
* 继承BaseRichSpout（建议使用）
	* 不需要显式的ack，不需要显示做锚（讲消息机制的时候详细说明）
	* 不需要实现所有接口的方法

BaseRichSpout API

open()：当一个Task被初始化的时候会调用此open方法。一般都会在此方法中对发送Tuple的对象SpoutOutputCollector和配置对象TopologyContext初始化。


```java
public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {   
          _collector = collector;  
} 
```

declareOutputFields()：此方法用于声明当前Spout的Tuple发送流的域名字，用来声明当前Spout的Tuple的流向，可多个，可一个。

```java
public void declareOutputFields(OutputFieldsDeclarer declarer) {  
        declarer.declare(new Fields("word"));  
}
```


getComponentConfiguration()：此方法定义在BaseComponent类内，用于声明针对当前组件的特殊的Configuration配置。

```java
public Map<String, Object> getComponentConfiguration() {  
         if (!_isDistributed) {  
           Map<String, Object> ret = new HashMap<String, Object>();  
           ret.put(Config.TOPOLOGY_MAX_TASK_PARALLELISM, 3);  
           return ret;  
         } else {  
             return null;  
         }  
 }
```


nextTuple()：这是Spout类中最重要的一个方法。发射一个Tuple到Topology都是通过这个方法来实现的。

```java
public void nextTuple() {
  Utils.sleep(100);
  final String[] words = new String[] {"nathan", "mike", "jackson", "golda", "bertels"};
  final Random rand = new Random();
  final String word = words[rand.nextInt(words.length)];
  _collector.emit(new Values(word));
}
```



  



####编写ExclamationBolt 

* 实现IRichBolt接口
* 继承BaseRichBolt（建议使用）
	* 不需要显式的ack，不需要显示做锚（讲消息机制的时候详细说明）
	* 不需要实现所有接口的方法

BaseRichBolt API

Bolt类接收由Spout或者其他上游Bolt类发来的Tuple，对其进行处理

prepare()：此方法和Spout中的open方法类似，为Bolt提供了OutputCollector，用来从Bolt中发送Tuple。

```java
public void prepare(Map conf, TopologyContext context, OutputCollector collector) {         
         _collector = collector;  
}
```


declareOutputFields()：用于声明当前Bolt发送的Tuple中包含的字段，和Spout中类似。

```java
public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
}
```


getComponentConfiguration()：和Spout类一样，在Bolt中也可以有getComponentConfiguration方法。


```java
public Map<String, Object> getComponentConfiguration() {  
         Map<String, Object> conf = new HashMap<String, Object>();  
         conf.put(Config.TOPOLOGY_TICK_TUPLE_FREQ_SECS, emitFrequencyInSeconds);  
         return conf;  
}
```


execute()：这是Bolt中最关键的一个方法，对于Tuple的处理都可以放到此方法中进行。具体的发送也是通过emit方法来完成的。

emit有一个参数：此唯一的参数是发送到下游Bolt的Tuple，此时，由上游发来的旧的Tuple在此隔断，新的Tuple和旧的Tuple不再属于同一棵Tuple树。新的Tuple另起一个新的Tuple树。


```java
public void execute(Tuple tuple) {  
      _collector.emit(new Values(tuple.getString(0) + "!!!"));  
} 
```


emit有两个参数：第一个参数是旧的Tuple的输入流，第二个参数是发往下游Bolt的新的Tuple流。此时，新的Tuple和旧的Tuple是仍然属于同一棵Tuple树，即如果下游的Bolt处理Tuple失败。
则会向上传递到当前Bolt，当前Bolt根据旧的Tuple流继续往上游传递，申请重发失败的Tuple。保证Tuple处理的可靠性。


```java
public void execute(Tuple tuple) {  
      _collector.emit(tuple, new Values(tuple.getString(0) + "!!!"));  
      _collector.ack(tuple);  
}
```



####本地模式和集群模式

本地模式

```java
StormSubmitter.submitTopologyWithProgressBar(TOPOLOGY_NAME, conf, builder.createTopology());
```


集群模式

```java
LocalCluster cluster = new LocalCluster();
cluster.submitTopology(TOPOLOGY_NAME, conf, builder.createTopology());
```


####注意事项

* 每个组件(Spout或者Bolt)的构造方法和declareOutputFields方法都只被调用一次。
* open方法、prepare方法的调用是多次的。入口函数中设定的setSpout或者setBolt里的并行度参数指的是executor的数目，即负责运行组件中的task的线程的数目，此数目是多少，上述的两个方法就会被调用多少次，在每个executor运行的时候调用一次。相当于一个线程的构造方法。
* nextTuple方法、execute方法是一直被运行的，nextTuple方法不断的发射Tuple，Bolt的execute不断的接收Tuple进行处理。只有这样不断地运行，才会产生无界的Tuple流，体现实时性。
* 在提交了一个topology之后，Storm就会创建spout/bolt实例并进行序列化。之后，将序列化的component发送给所有的任务所在的机器(即Supervisor节点)，在每一个任务上反序列化component。
* Spout和Bolt之间、Bolt和Bolt之间的通信，是通过zeroMQ的消息队列实现的。


##Strom stream分组策略

Storm里面有7种类型的stream grouping，你也可以通过实现CustomStreamGrouping接口来实现自定义流分组 

1. Shuffle Grouping 
随机分组，随机派发stream里面的tuple，保证每个bolt task接收到的tuple数目大致相同。 

2. Fields Grouping 
按字段分组，比如，按"user-id"这个字段来分组，那么具有同样"user-id"的 tuple 会被分到相同的Bolt里的一个task， 而不同的"user-id"则可能会被分配到不同的task。 

3. All Grouping 
广播发送，对亍每一个tuple，所有的bolts都会收到 

4. Global Grouping 
全局分组，整个stream被分配到storm中的一个bolt的其中一个task。再具体一点就是分配给id值最低的那个task。 

5. None Grouping 
不分组，这个分组的意思是说stream不关心到底怎样分组。目前这种分组和Shuffle grouping是一样的效果， 有一点不同的是storm会把使用none grouping的这个bolt放到这个bolt的订阅者同一个线程里面去执行（如果可能的话）。 

6. Direct Grouping 
指向型分组， 这是一种比较特别的分组方法，用这种分组意味着消息（tuple）的发送者指定由消息接收者的哪个task处理这个消息。只有被声明为 Direct Stream 的消息流可以声明这种分组方法。而且这种消息tuple必须使用 emitDirect 方法来发射。消息处理者可以通过 TopologyContext 来获取处理它的消息的task的id (OutputCollector.emit方法也会返回task的id)  

7. Local or shuffle grouping 
本地或随机分组。如果目标bolt有一个或者多个task与源bolt的task在同一个工作进程中，tuple将会被随机发送给这些同进程中的tasks。否则，和普通的Shuffle Grouping行为一致。 

##Storm的消息机制

###什么样的消息算是被完全处理？

```java
TopologyBuilder builder = new TopologyBuilder();
builder.setSpout("sentences", new KestrelSpout("kestrel.backtype.com",22133,"sentence_queue",  new StringScheme()));
builder.setBolt("split", new SplitSentence(), 10) .shuffleGrouping("sentences");
builder.setBolt("count", new WordCount(), 20).fieldsGrouping("split", new Fields("word"));
```


上面的topology从Kestrel queue读取句子，将这些句子划分成词组，然后按照前面划分词组时统计的每个词的次数发送每个词。离开spout的某个tuple可能会触发创建很多基于它的tuples：句子中每个单词都会对应一个tuple，同时每个单词的次数也会对应一个tuple。如果处理发生错误或者时间超时就认为消息处理失败，可以通过Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS参数配置超时的时间，默认是30秒

tuple被emit出去后, 可能会被多级bolt处理, 并且bolt也有可能由该tuple生成多组tuples, 
所以情况还是比较复杂的,最终由一个tuple trigger(触发)的所有tuples会形成一个树或DAG(有向无环图)只有当tuple tree上的所有节点都被成功处理的时候, storm才认为该tuple被fully processed，如果tuple tree上任一节点失败或者超时, 都被看作该tuple fail, 失败的tuple会被重发    

![](/img/2015-10-31-storm_base/post-bg-storm-05.png)

###如果消息被完全处理或未完全处理，会发生什么？

```java
public interface ISpout extends Serializable {
	void open(Map conf, TopologyContext context, SpoutOutputCollector collector);
	void close();
	void nextTuple();
	void ack(Object msgId);
	void fail(Object msgId);
}
```

首先，Storm会通过Spout的nextTuple方法从Spout创建或获取一个tuple。在open方法中，Spout使用此方法提供的SpoutOutputCollector去发射一个tuple到输出streams中去。当发射一个tuple时，Spout会提供一个“message id”，用来后面区分不同的tuple。例如， KestrelSpout从kestrel队列中读取消息，然后在发射时会将Kestrel为消息提供的id作为“message id”。发射一条消息到SpoutOutputCollector，如下所示：_collector.emit(newValues("field1", "field2", 3), msgId);

然后，这个tuple会发送到消费bolts，同时Storm会跟踪已被创建的消息树状图。如果Storm检测到一个tuple已被“fully processed”， Storm将会原始的Spout task（即发射这个tuple的Spout）上调用ack方法，参数msgId就是这个Spout提供给Storm的“message id”。类似的，如果这个tuple超时了， Storm会在原始的Spouttask上调用fail方法。注意，一个tuple只能被创建它的Spouttask进行acked或者failed。


###Storm的可靠性API

Storm本身通过什么机制来判断tuple是否成功被fully processed?

1. 如何知道tuple tree的结构?    
2. 如何知道tuple tree上每个节点的运行情况, success或fail?

答案很简单, 你必须告诉它, 如何告诉它? 
	 
1. 对于tuple tree的结构, 需要知道每个tuple由哪些tuple产生, 即tree节点间的link tree节点间的link称为anchoring. 当每次emit新tuple的时候, 必须显式的通过API建立anchoring(锚)_collector.emit(tuple, new Values(word)); emit的第一个参数是tuple, 这就是用于建anchoring，当然你也可以直接调用unanchoring的emit版本, 如果不需要保证reliable的话, 这样效率会比较高_collector.emit(new Values(word));	

2. 对于tuple tree上每个节点的运行情况, 你需要在每个bolt的逻辑处理完后, 显式的调用OutputCollector的ack和fail来汇报_collector.ack(tuple)，Storm为了保证reliable, 必然是要牺牲效率的, 此处Storm会在task memory里面去记录你汇报的tuple tree的结构和运行情况.而只有当某tuple节点被ack或fail后才会被从内存中删除, 所以如果你总是不去ack或fail, 那么会导致task的out of memory，如果继承的是BasicBolt的话，不需要显示的建立锚和显示的ack，或者fail



##Strom并行度

###Storm并行度讲解

Strom程序的执行是由多个supervisor共同执行的。supervisor运行的是topology中的spout/bolt task，task是storm中进行计算的最小的运行单位，表示是spout或者bolt的运行实例。
程序执行的最大粒度的运行单位是进程，刚才说的task也是需要有进程来运行它的，在supervisor中，运行task的进程称为worker，Supervisor节点上可以运行非常多的worker进程，一般在一个进程中是可以启动多个线程的，所以我们可以在worker中运行多个线程，这些线程称为executor，在executor中运行task。

这表示是一个worker进程，其实就是一个jvm虚拟机进程，在这个work进程里面有多个executor线程，每个executor线程会运行一个或多个task实例。一个task是最终完成数据处理的实体单元。(默认情况下一个executor运行一个task，可修改)

worker(进程)>executor(线程)>task(实例) 

![](/img/2015-10-31-storm_base/post-bg-storm-06.png)

* worker
	* 1个worker进程执行的是1个topology的子集，一个work只为一个topology服务。1个worker进程会启动1个或多个executor线程来执行1topology的component(spout或bolt)。因此，1个运行中的topology就是由集群中多台物理机上的多个worker进程组成的。

* executor
	* executor是1个被worker进程启动的单独线程。每个executor只会运行1个topology的1个component(spout或bolt)的task，task可以是1个或多个，storm默认是1个component只生成1个task，executor线程里会在每次循环里顺序调用所有task实例。

* task	
	* task是最终运行spout或bolt中代码的单元，topology启动后，1个component(spout或bolt)的task数目是固定不变的，但该component使用的executor线程数可以动态调整






