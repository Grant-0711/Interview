# 版本相关

2019年1月阿里Blink开源

目前版本1.9

local cluster 8081

# Flink 中的核心概念和基础篇

## 简单介绍

Flink是一个分布式处理引擎，用于无界和有界流的有状态计算

提供了分布式，容错性和资源管理的核心功能

提供多层高抽象的API：

https://www.processon.com/diagraming/60d6fd14f346fb5e35b4b555

自下而上依次是状态流process，DStream和Dset，table，sql

提供具体领域的学习库：机器学习库和图计算库

## Flink特性

高吞吐，低时延，高性能流处理

有事件时间的窗口操作

有状态的严格一次性语义exactly-once

支持高灵活度的窗口，基于时间，count，session以及data-driven的窗口操作

​	全局窗口会指定同一key的数据进入同一个窗口，必须结合自定义触发器来使用，因为全局窗口没有进行数据聚合的结束点

​	自定义触发器

​		基于处理时间或者事件时间处理过一个元素之后, 注册一个定时器, 然后指定的时间执行.

```JAVA
Context和OnTimerContext所持有的TimerService对象拥有以下方法:
currentProcessingTime(): Long //返回当前处理时间
currentWatermark(): Long //返回当前watermark的时间戳
registerProcessingTimeTimer(timestamp: Long):Unit //会注册当前key的processing time的定时器。当processing time到达定时时间时，触发timer。
registerEventTimeTimer(timestamp: Long): Unit //会注册当前key的event time 定时器。当水位线大于等于定时器注册的时间时，触发定时器执行回调函数。
deleteProcessingTimeTimer(timestamp: Long): Unit //删除之前注册处理时间定时器。如果没有这个时间戳的定时器，则不执行。
deleteEventTimeTimer(timestamp: Long): Unit //删除之前注册的事件时间定时器，如果没有此时间戳的定时器，则不执行。
```


支持具有BackPressure功能的持续流模型

基于轻量级分布式快照（Snapshot）实现的容错

一个运行时同时支持batch on streaming 处理 和 streaming处理

Flink在jvm实现了自己的内存管理

支持迭代计算

支持程序自动优化：避免特定情况下shuffle，排序等昂贵操作，中间结果有必要进行缓存

## Flink vs Spark Streaming 

主要的是Flink是标准的实时处理引擎，基于事件驱动。而spark streaming 是微批次的

### 架构模型区别

SS的主要架构为 master，worker，driver，executor

Flink的是JM,TM和Slot

### 任务调度区别

SS连续生成微批次的数据，构建DAG（有向无环图）SS会依次创建DstreamGraph，JobGenerator，JobScheduler

Flink根据用户提交的代码生成StreamGraph，经过优化生成JobGraph，JG提交给JM，JM根据JG生成EG（ExecutionGraph），ExecutionGraph是Flink调度最核心的数据模型，JM根据EG对job进行调度

### 时间机制区别

SS只支持处理时间

Flink支持事件时间，处理时间和注入时间

#### Flink注入时间：

事件进入Flink的时间，即将每一个事件在**数据源算子**的处理时间作为事件时间的时间戳，并自动生成水位线，无法处理迟到数据和乱序事件

### 容错机制

对于SS任务，可以设置checkpoint，发生故障可以从上次checkpoint恢复，但是只能保证不丢数，可能会重复，不是严格一次性语义

## Flink的组件栈

https://www.processon.com/diagraming/60f0e20ce401fd4fe050cc41

## Flink集群规模

Flink支持多少节点的集群规模（基于Flink on Yarn）

基于生产环节的集群规模，节点和内存情况

Flink基础编程模型

```java
//source阶段
DataStream<String> lines =  env.addSource(new 
                                	FlinkKafkaConsumer<> (...));
//Transformation阶段
DataStream<String> events = lines.map((line) -> parse(line));
DataStream<Statistics> stats = events
		.keyby("id")
    	.timeWindow(Time.seconds(10))
    	.apply(new MyWindowAggFun());
//sink阶段
stats.addSink(new RollingSink(Path));
```

因此flink程序的基本构建是source，transformation和sink

执行时Flink程序映射到dataflows，由流和转换操作组成

## Flink集群有哪些角色？各自作用？

Flink在运行时主要有JM,TM和Client

JM等价于Master，整个集群协调者，负责接收job，协调检查点，故障恢复，管理TM

TM是具体的执行计算的Worker，在上面执行一些Job的Task，单个TM管理其资源，如内存，磁盘，网络，启动时将资源状况向JM汇报

Client是Flink任务提交的客户端，会首先创建，对用户程序进行预处理，提交到集群，Client需要从用户配置中获取JM的地址，并建立JM的连接，将Job提交到JM

Flink 的 Slot的概念

TM是一个JVM进程，会以独立的线程来执行task或者多个subTask，为了控制一个TM能接受多少个task，Flink提出了Task slot的概念

TM会把自己节点上的资源分为不同的slot：固定大小的资源子集。这样避免了不同job的task相互竞争内存资源，但是主要的是slot只会做内存的隔离，没有cpu的隔离。

## Flink常用算子

Map：一对一，DataStream → DataStream

Filter：过滤掉指定条件的数据

KeyBy：按照指定的key进行分组

Reduce：用来进行结果汇总合并

Window：窗口函数，根据某些特性将每个key的数据进行分组（例如：在5s内到达的数据）

## Flink分区策略

分区策略是决定数据如何发向下游，目前有八种

GlobalPartitioner 数据会被分发到下游算子的第一个实例中进行处理。

ShufflePartitioner 数据会被随机分发到下游算子的每一个实例中进行处理。

RebalancePartitioner 数据会被循环发送到下游的每一个实例中进行处理。

RescalePartitioner 这种分区器会根据上下游算子的并行度，循环的方式输出到下游算子的每个实例。这里有点难以理解，假设上游并行度为2，编号为A和B。下游并行度为4，编号为1，2，3，4。那么A则把数据循环发送给1和2，B则把数据循环发送给3和4。假设上游并行度为4，编号为A，B，C，D。下游并行度为2，编号为1，2。那么A和B则把数据发送给1，C和D则把数据发送给2。

BroadcastPartitioner 广播分区会将上游数据输出到下游算子的每个实例中。适合于大数据集和小数据集做Jion的场景。

ForwardPartitioner ForwardPartitioner 用于将记录输出到下游本地的算子实例。它要求上下游算子并行度一样。简单的说，ForwardPartitioner用来做数据的控制台打印。

KeyGroupStreamPartitioner Hash分区器。会将数据按 Key 的 Hash 值输出到下游算子实例中。

CustomPartitionerWrapper 用户自定义分区器。需要用户自己实现Partitioner接口，来定义自己的分区逻辑。例如：

```java
static class CustomPartitioner implements Partitioner<String> {
      @Override
      public int partition(String key, int numPartitions) {
          switch (key){
              case "1":
                  return 1;
              case "2":
                  return 2;
              case "3":
                  return 3;
              default:
                  return 4;
```

## Flink并行度以及设置

Flink任务被分为多个并行任务来执行，每个并行的实例处理一部分数据。这些并行实例的数量被称为并行度。

可以在四个不同层面设置并行度：

操作算子层面

执行环境层面

客户端层面

系统层面

优先级自上而下依次降低

## slot和parallelism有什么区别

slot是指taskmanager的并发执行能力，假设我们将 taskmanager.numberOfTaskSlots 配置为3 那么每一个 
taskmanager 中分配3个 TaskSlot, 3个 taskmanager 一共有9个TaskSlot。

parallelism是指taskmanager实际使用的并发能力。假设我们把 parallelism.default 设置为1，那么9个 TaskSlot 只能用1个，有8个空闲。

## Flink重启策略

固定延迟重启策略（Fixed Delay Restart Strategy）

故障率重启策略（Failure Rate  Restart Strategy）

没有重启策略（no Restart Strategy）

FallBack重启策略（Fallback Restart Strategy）

## Flink分布式缓存

目的是将本地文件缓存到TM中，防止重复拉取

```scala
val env = ExecutionEnvironment.getEnvironment
// register a file from HDFS
env.registerCacheFile（"hdfs:///path/to/your/file", "hdfsFile"）
// register a local executable file (script, executable, ...)
env.registerCachedFile("file:///path/to/exec/file", "localExecFile", true)
 
// define your program and execute
...
val input：DataSet[String] = ...
val result:DataSet[Integer] = input.map(new MyMapper())
...
env.execute()
```

## Flink广播变量

当我们需要访问同一份数据。那么Flink中的广播变量就是为了解决这种情况。

我们可以把广播变量理解为是一个公共的共享变量，我们可以把一个dataset 数据集广播出去，然后不同的task在节点上都能够获取到，这个数据在每个节点上只会存在一份。

## Flink的窗口

Flink 支持两种划分窗口的方式，按照time和count。如果根据时间划分窗口，那么它就是一个time-window 如果根据数据划分窗口，那么它就是一个count-window。

flink支持窗口的两个重要属性（size和interval）

如果size=interval,那么就会形成tumbling-window(无重叠数据)  如果size>interval,那么就会形成sliding-window(有重叠数据) 如果size< interval,  那么这种窗口将会丢失数据。比如每5秒钟，统计过去3秒的通过路口汽车的数据，将会漏掉2秒钟的数据。

通过组合可以得出四种基本窗口：

- time-tumbling-window 无重叠数据的时间窗口，设置方式举例：timeWindow(Time.seconds(5))
- time-sliding-window 有重叠数据的时间窗口，设置方式举例：timeWindow(Time.seconds(5), Time.seconds(3))
- count-tumbling-window无重叠数据的数量窗口，设置方式举例：countWindow(5)
- count-sliding-window 有重叠数据的数量窗口，设置方式举例：countWindow(5,3)

## FLink状态存储

状态的存储、访问以及维护，由一个**可插入**的组件决定，这个组件就叫做**状态后端**（state backend）

**状态后端主要负责两件事：**

 本地(taskmanager)的状态管理

将检查点（checkpoint）状态写入远程存储

Flink提供了三种状态存储方式：

MemoryStateBackend（不稳定，几乎无状态的处理如ETL）

FsStateBackend(本地状态在TaskManager内存, Checkpoint时, 存储在文件系统(hdfs)中)

RocksDBStateBackend(存入本地的RocksDB数据库中,用于超大状态的作业, 例如天级的窗口聚合)

## Flink水印

为了处理 EventTime 窗口计算提出的一种机制, 本质上是一种时间戳。 一般来讲Watermark经常和Window一起被用来处理乱序事件。

## Flink Table & SQL

TableEnvironment是Table API 和 SQL集成的核心概念

主要用来：

​	在内部catalog中注册表

​		Catalog 就是元数据管理中心，其中元数据包括数据库、表、表结构等信息

​		Flink 的 Catalog 相关代码定义在 catalog.java 文件中，是一个 interface

​	注册外部catalog

​	执行sql查询

​	注册用户定义函数

​	将DataStream或者DataSet转换为表

​	持有ExecutionEnvironment或StreamExecutionEnvironment的引用

## Flink SQL原理以及如何解析

Flink SQL解析依赖于Apache calcite

基于此，一次完整的SQL解析过程如下：

- 用户使用对外提供Stream SQL的语法开发业务应用
- 用calcite对StreamSQL进行语法检验，语法检验通过后，转换成calcite的逻辑树节点；最终形成calcite的逻辑计划
- 采用Flink自定义的优化规则和calcite火山模型、启发式模型共同对逻辑树进行优化，生成最优的Flink物理计划
- 对物理计划采用janino codegen生成代码，生成用低阶API DataStream 描述的流应用，提交到Flink平台执行



# Flink 进阶篇

## Flink如何实现流批一体？

Flink中批处理是特殊的流处理，用流处理引擎支持DataSet API 和 DataStream API

## Flink如何做到高效数据交换？

在一个Job中，数据需要在不同的task中交换，交换是由TM负责，TM的网络组件首先从缓存buffer中收集records，然后发送

Records是累积一个批次发送，可以更好利用网络资源

## Fink容错

靠强大的CheckPoint机制和State机制。Checkpoint 负责定时制作分布式快照、对程序中的状态进行备份；State 用来存储计算过程中的中间状态。

### 状态一致性

状态一致性级别：
at-most-once(最多一次): 
at-least-once(至少一次): 
exactly-once(严格一次):

### 端到端的状态一致性

端到端的一致性保证，意味着结果的正确性贯穿了整个流处理应用的始终；每一个组件都保证了它自己的一致性，**整个端到端的一致性级别取决于所有组件中一致性最弱的组件**。

### Checkpoint原理

Flink的分布式快照是根据Chandy-Lamport算法量身定做的。简单来说就是持续创建分布式数据流及其状态的一致快照。

核心思想是在 input source 端插入 barrier，控制 barrier 的同步来实现 snapshot 的备份和 exactly-once 语义。

## Flink 是如何保证Exactly-once语义的？

Flink通过实现两阶段提交和状态保存来实现端到端的一致性语义。 

分为以下几个步骤：

​	开始事务（beginTransaction）创建一个临时文件夹，来写把数据写入到这个文件夹

​	预提交（preCommit）将内存中缓存的数据写入文件并关闭

​	正式提交（commit）将之前写完的临时文件放入目标目录下。这代表着最终的数据会有一些延迟

​	丢弃（abort）丢弃临时文件

若失败发生在预提交成功后，正式提交前。可以根据状态来提交预提交的数据，也可删除预提交的数据。

## Flink 的 kafka 连接器有什么特别的地方？

Flink源码中有一个独立的connector模块，所有的其他connector都依赖于此模块，Flink 在1.9版本发布的全新kafka连接器，摒弃了之前连接不同版本的kafka集群需要依赖不同版本的connector这种做法，只需要依赖一个connector即可。

## Flink内存管理

Flink将对象都序列化到一个预分配的内存块上。

此外，大量使用了堆外内存，如果处理的数据超出内存限制，将会存到硬盘上。

Flink实现了自己的序列化框架

理论上Flink的内存管理分为三部分：

Network Buffers：TM创建时分配，存储网络数据，每个块32K，默认分配2048个，通过taskmanager.network.numberOfBuffers修改

Memory Manage Pool：大量Memory Segment 块，用于运行时算法，例如sort，join或者shuffle，内存的分配支持预分配和lazy load，默认懒加载

User Code，这部分是除了Memory Manager之外的内存用于User code和TaskManager本身的数据结构。

## Flink序列化

Java本身自带的序列化和反序列化的功能，但是辅助信息占用空间比较大，在序列化对象时记录了过多的类信息。

Apache Flink摒弃了Java原生的序列化方法，以独特的方式处理数据类型和序列化，包含自己的类型描述符，泛型类型提取和类型序列化框架。

TypeInformation 是所有类型描述符的基类。它揭示了该类型的一些基本属性，并且可以生成序列化器。TypeInformation 支持以下几种类型：

```java
BasicTypeInfo: //任意Java 基本类型或 String 类型

BasicArrayTypeInfo: //任意Java基本类型数组或 String 数组

WritableTypeInfo: //任意 Hadoop Writable 接口的实现类

TupleTypeInfo: //任意的 Flink Tuple 类型(支持Tuple1 to Tuple25)。Flink tuples 是固定长度固定类型的Java Tuple实现

CaseClassTypeInfo: //任意的 Scala CaseClass(包括 Scala tuples)

PojoTypeInfo: //任意的 POJO (Java or Scala)，例如，Java对象的所有成员变量，要么是 public 修饰符定义，要么有 getter/setter 方法

GenericTypeInfo: //任意无法匹配之前几种类型的类
```

针对前六种类型数据集，Flink皆可以自动生成对应的TypeSerializer，能非常高效地对数据集进行序列化和反序列化。

## Flink中的Window出现了数据倾斜解决办法？

window产生数据倾斜指的是数据在不同的窗口内堆积的数据量相差过多。本质上产生这种情况的原因是数据源头**发送的数据量速度不同导致**的。

出现这种情况一般通过两种方式来解决：

​	在数据进入窗口前做预聚合

​	重新设计窗口聚合的key

## Flink中在使用聚合函数 GroupBy、Distinct、KeyBy 等函数时出现数据热点该如何解决？

数据倾斜和数据热点是所有大数据框架绕不过去的问题

主要从三个方面入手：

### 业务上规避

某个key发生热点的话单独处理其数据

### key的设计

例如某个地区key热点，对其进行地区拆分

### 参数设置

Flink1.9性能优化有一项是升级了微批次处理（miniBatch）

​	缓存一定量的数据后触发处理，减少对state的访问，从而提升吞吐，减少数据输出

## Flink任务延迟高，想解决这个问题，你会如何入手？

在UI界面中查看子任务情况，可以看到Flink哪个算子和task出现了反压

### 主要的手段是资源调优和算子调优

对作业的operator并发数，CPU，堆内存等参数进行调优

算子调优比如并行度的设置，state设置，ck的设置

## Flink是如何处理反压

Flink 内部是基于 producer-consumer 模型来进行消息传递的，Flink的反压设计也是基于这个模型。Flink 
使用了高效有界的分布式阻塞队列，就像 Java 通用的阻塞队列（BlockingQueue）一样。下游消费者消费变慢，上游就会受到阻塞。

### Flink的运行时如何在task之间传输数据缓冲区内的数据以及流数据如何自然地**两端降速**来应对背压

​	Flink运行时的构造部件是**operators**以及**streams**。每一个operator消费一个中间/过渡状态的流，对它们进行转换，然后生产一个新的流。描述这种机制最好的类比是：Flink使用有效的分布式阻塞队列来作为有界的缓冲区。如同Java里通用的阻塞队列跟处理线程进行连接一样，一旦队列达到容量上限，一个相对较慢的接受者将拖慢发送者。

​	在Flink中这些分布式的队列被认为是逻辑流，而它们的有界容量可以通过每一个生产、消费流管理的缓冲池获得。缓冲池是缓冲区的集合，它们都可以在被消费完之后循环利用。这个观点很好理解：你从池里获取一个缓冲区，填进数据，然后在数据被消费后，将该缓冲区返还回缓冲池，之后你还可以再次使用它。

​	这些缓冲池的大小在运行时能动态变化。在不同的发送者/接收者存在不同的处理速度的情况下，网络栈里的内存缓冲区的数量（等于队列的容量）决定了系统能够提供的缓冲区的数量。Flink保证总是有足够的缓冲区提供给应用程序，但处理的速度是由用户的程序以及可用内存的数量决定的。内存越多，意味着系统可以轻松应对一定的瞬时背压（short  periods，short  GC）。越少的内存意味着需要对背压进行更多的“即时”响应（意思是，如果内存少缓冲区就容易被填满，那么需要立即作出响应，消费走数据才能应对这个问题）。

我们来看两种场景：

- 本地传输：如果task1和task2都运行在同一个工作节点（TaskManager），缓冲区可以被直接共享给下一个task，一旦task  2消费了数据它会被回收。如果task 2比task 1慢，buffer会以比task 1填充的速度更慢的速度进行回收从而迫使task 1降速。
- 远程传输：如果task 1和task 2运行在不同的工作节点上。一旦缓冲区内的数据被发送出去(TCP Channel)，它就会被回收。在接收端，数据被拷贝到输入缓冲池的缓冲区中，如果没有缓冲区可用，从TCP连接中的数据读取动作将会被中断。输出端通常以`watermark`机制来保证不会有太多的数据在传输途中。如果有足够的数据已经进入可发送状态，会等到情况稳定到阈值以下才会进行发送。这可以保证没有太多的数据在路上。如果新的数据在消费端没有被消费（因为没有可用的缓冲区），这种情况会降低发送者发送数据的速度。

这个在固定大小的缓冲池之间的流示例，保证了Flink健壮的背压机制，从而使得task生产数据的速度跟消费的速度对等。

我们描述的这个方案可以从两个task之间的数据传输自然地扩展到更复杂的pipeline中，并保证背压在整个pipeline上扩散。

# 总结

Flink与持久化的source（例如kafka），能够为你提供即时的背压处理，而无需担心数据丢失。Flink不需要一个特殊的机制来处理背压，因为Flink中的数据传输相当于已经提供了应对背压的机制。因此，Flink所获得的最大吞吐量由其pipeline中最慢的部件决定。

​	

## Flink的反压和Strom有哪些不同？

Storm 是通过监控 Bolt 中的接收队列负载情况，如果超过高水位值就会将反压信息写到 Zookeeper ，Zookeeper 上的 watch 会通知该拓扑的所有 Worker 都进入反压状态，最后 Spout 停止发送 tuple。

Flink中的反压使用了高效有界的分布式阻塞队列，下游消费变慢会导致发送端阻塞。

二者最大的区别是Flink是逐级反压，而Storm是直接从源头降速。

##  Operator Chains（算子链）

为了更高效地分布式执行，Flink会尽可能地将operator的subtask链接（chain）在一起形成task。每个task在一个线程中执行。将operators链接成task是非常有效的优化：它能减少线程之间的切换，减少消息的序列化/反序列化，减少数据在缓冲区的交换，减少了延迟的同时提高整体的吞吐量。

##  Flink什么情况下才会把Operator chain在一起形成算子链？

两个operator chain在一起的的条件：

    上下游的并行度一致
    
    下游节点的入度为1 （也就是说下游节点没有来自其他节点的输入）
    
    上下游节点都在同一个 slot group 中（下面会解释 slot group）
    
    下游节点的 chain 策略为 ALWAYS（可以与上下游链接，map、flatmap、filter等默认是ALWAYS）
    
    上游节点的 chain 策略为 ALWAYS 或 HEAD（只能与下游链接，不能与上游链接，Source默认是HEAD）
    
    两个节点间数据分区方式是 forward（参考理解数据流的分区）
    
    用户没有禁用 chain
## 说说Flink1.9的新特性？

- 支持hive读写，支持UDF
- Flink SQL TopN和GroupBy等优化
- Checkpoint跟savepoint针对实际业务场景做了优化
- Flink state查询

## 消费kafka数据的时候，如何处理脏数据？

可以在处理前加一个fliter算子，将不符合规则的数据过滤出去。



# Flink 源码篇

## Flink Job的提交流程 用户提交的Flink Job会被转化成一个DAG

分别是：StreamGraph，JobGraph和ExecutionGraph

Flink中JM,TM和Client是基于Akka工具包的，是通过消息驱动

整个Flink的job提交还包含ActorSystem的创建，JM的启动，TM的启动

## Flink所谓"三层图"结构是哪几个"图"？

一个Flink的DAG生成图大致三个过程：

​	StreamGraph，最接近代码逻辑层面的计算拓扑图，按照用户代码执行顺序向env添加StreamTransformation构成流式图

​	JobGraph从SGraph生成，将可以串联合并的节点合并，设置节点的边，安排资源共享slot槽位和放置相关联的节点，上传所需要的的文件，设置ck

相当于经过部分初始化和处理的任务图

​	ExecutionGraph由JobGraph转换而来，包含了任务执行具体的内容，是贴近底层实现的执行图

## JobManger在集群中扮演了什么角色？

JobManager负责整个Flink集群的任务调度以及资源管理，从客户端获取应用，根据TM上TaskSlot的使用情况，为提交的应用分配相应的TaskSlot资源并命令TM启动从客户端获取的应用。

JobManager 相当于整个集群的 Master 节点，且整个集群有且只有一个活跃的 JobManager ，负责整个集群的任务管理和资源管理。

JobManager 和 TaskManager 之间通过 Actor System 进行通信，获取任务执行的情况并通过 Actor System 将应用的任务执行情况发送给客户端。

同时在任务执行的过程中，Flink JobManager 会触发 Checkpoint  操作，每个 TaskManager 节点 收到 Checkpoint 触发指令后，完成 Checkpoint 操作，所有的  Checkpoint 协调过程都是在 Fink JobManager 中完成。

当任务完成后，Flink 会将任务执行的信息反馈给客户端，并且释放掉 TaskManager 中的资源以供下一次提交任务使用。

## JobManger在集群启动过程中起到什么作用？

JobManager的职责主要是接收Flink作业，调度Task，收集作业状态和管理TaskManager。它包含一个Actor，并且做如下操作：

- RegisterTaskManager: 它由想要注册到JobManager的TaskManager发送。注册成功会通过AcknowledgeRegistration消息进行Ack。
- SubmitJob: 由提交作业到系统的Client发送。提交的信息是JobGraph形式的作业描述信息。
- CancelJob: 请求取消指定id的作业。成功会返回CancellationSuccess，否则返回CancellationFailure。
- UpdateTaskExecutionState: 由TaskManager发送，用来更新执行节点(ExecutionVertex)的状态。成功则返回true，否则返回false。
- RequestNextInputSplit: TaskManager上的Task请求下一个输入split，成功则返回NextInputSplit，否则返回null。
- JobStatusChanged： 它意味着作业的状态(RUNNING, CANCELING, FINISHED,等)发生变化。这个消息由ExecutionGraph发送。

## TaskManager在集群中扮演了什么角色？

TaskManager 相当于整个集群的 Slave 节点，负责具体的任务执行和对应任务在每个节点上的资源申请和管理。

客户端通过将编写好的 Flink 应用编译打包，提交到 JobManager，然后 JobManager 会根据已注册在 JobManager 中 TaskManager 的资源情况，将任务分配给有资源的 TaskManager节点，然后启动并运行任务。

TaskManager 从 JobManager 接收需要部署的任务，然后使用 Slot 资源启动 Task，建立数据接入的网络连接，接收数据并开始数据处理。同时 TaskManager 之间的数据交互都是通过数据流的方式进行的。

可以看出，Flink 的任务运行其实是采用多线程的方式，这和 MapReduce 多  JVM 进行的方式有很大的区别，Flink 能够极大提高 CPU 使用效率，在多个任务和 Task 之间通过 TaskSlot  方式共享系统资源，每个 TaskManager 中通过管理多个 TaskSlot 资源池进行对资源进行有效管理。

## TaskManager在集群启动过程中起到什么作用？

TaskManager的启动流程较为简单： 
启动类：org.apache.flink.runtime.taskmanager.TaskManager 核心启动方法 ： 
selectNetworkInterfaceAndRunTaskManager 
启动后直接向JobManager注册自己，注册完成后，进行部分模块的初始化。

## Flink 计算资源的调度是如何实现的？

TaskManager中最细粒度的资源是Task slot，代表了一个固定大小的资源子集，每个TaskManager会将其所占有的资源平分给它的slot。

通过调整 task slot 的数量，用户可以定义task之间是如何相互隔离的。每个  TaskManager 有一个slot，也就意味着每个task运行在独立的 JVM 中。每个 TaskManager  有多个slot的话，也就是说多个task运行在同一个JVM中。

而在同一个JVM进程中的task，可以共享TCP连接（基于多路复用）和心跳消息，可以减少数据的网络传输，也能共享一些数据结构，一定程度上减少了每个task的消耗。   每个slot可以接受单个task，也可以接受多个连续task组成的pipeline

## 简述Flink的数据抽象及数据交换过程？

Flink  为了避免JVM的固有缺陷例如java对象存储密度低，FGC影响吞吐和响应等，实现了自主管理内存。MemorySegment就是Flink的内存抽象。默认情况下，一个MemorySegment可以被看做是一个32kb大的内存块的抽象。这块内存既可以是JVM里的一个byte[]，也可以是堆外内存（DirectByteBuffer）。

在MemorySegment这个抽象之上，Flink在数据从operator内的数据对象在向TaskManager上转移，预备被发给下个节点的过程中，使用的抽象或者说内存对象是Buffer。

对接从Java对象转为Buffer的中间对象是另一个抽象StreamRecord。

## Flink 中的分布式快照机制是如何实现的？

Flink的容错机制的核心部分是制作分布式数据流和操作算子状态的一致性快照。 这些快照充当一致性checkpoint，系统可以在发生故障时回滚。
Flink用于制作这些快照的机制在“分布式数据流的轻量级异步快照”中进行了描述。 
它受到分布式快照的标准Chandy-Lamport算法的启发，专门针对Flink的执行模型而定制。

barriers在数据流源处被注入并行数据流中。快照n的barriers被插入的位置（我们称之为Sn）是快照所包含的数据在数据源中最大位置。例如，在Apache  Kafka中，此位置将是分区中最后一条记录的偏移量。 将该位置Sn报告给checkpoint协调器（Flink的JobManager）。

然后barriers向下游流动。当一个中间操作算子从其所有输入流中收到快照n的barriers时，它会为快照n发出barriers进入其所有输出流中。  一旦sink操作算子（流式DAG的末端）从其所有输入流接收到barriers  n，它就向checkpoint协调器确认快照n完成。在所有sink确认快照后，意味快照着已完成。

一旦完成快照n，job将永远不再向数据源请求Sn之前的记录，因为此时这些记录（及其后续记录）将已经通过整个数据流拓扑，也即是已经被处理结束。

## Chandy-Lamport

## FlinkSQL的是如何实现的？

构建抽象语法树的事情交给了 Calcite 去做。SQL query 会经过  Calcite 解析器转变成 SQL 节点树，通过验证后构建成 Calcite 的抽象语法树（也就是图中的 Logical  Plan）。另一边，Table API 上的调用会构建成 Table API 的抽象语法树，并通过 Calcite 提供的 RelBuilder  转变成 Calcite 的抽象语法树。然后依次被转换成逻辑执行计划和物理执行计划。

在提交任务后会分发到各个 TaskManager 中运行，在运行时会使用 Janino 编译器编译代码后运行。

 Janino

运行时嵌入的编译器，比如作为表达式求职的翻译器或者JSP的服务端页面引擎

还可以用于静态代码分析或者进行修改