

# 版本

Flink 1.2.0 - 2017-02-06 异步io

Flink 1.6.0 - 2018-08-08 有状态流处理


Flink 1.7.0 - 2018-11-30

​	扩展流处理的范围对Scala 2.12的支持

Flink 1.8.0 - 2019-04-09 

​	将对模式演化的支持扩展到POJO，完成所有Flink内置序列化程序的升级以使用新的序列化兼容性抽象

Flink 1.9.0 - 2019-08-22

从任务失败中恢复批处理（DataSet，Table API和SQL）作业的时间大大减少了

Flink 1.10.0 - 2020-02-11 改进的内存管理和配置

# 资源配置

资源配置是首先要考虑的，最基本的

提交方式主要是perJob，资源的分配在提交Flink任务时指定

标准的提交脚本





最优并行度计算

压测流程

测试单个并行度的处理上限

并行度给9，测试单个并行度的处理上限，得到一个能力值

总QPS/能力值 = 参考并行度

不能单考虑QPS，因为单个并行度处理的任务可能数据量大也可能小

最好是根据峰值QPS压测，并行度*1.2倍为实际并行度，富留一些资源

Source段并行度的配置

source的并行度设置为kafka分区数，如果发生生产消息速度还是过快，考虑增加kafka分区，同步增加并行度

Transform端并行度的配置

keyby之前的算子

一般不做太重的操作，例如map，flatmap，fliter等处理较快的算子，并行度可以和source保持一致

keyby之后的算子

如果并发较大，建议设置并行度为 2 的整数次幂，例如：128、256、512；

小并发任务的并行度不一定需要设置成 2 的整数次幂；

大并发任务如果没有 KeyBy，并行度也无需设置为 2 的整数次幂；

sink端的并行度

Sink 端是数据流向下游的地方，可以根据 Sink 端的数据量及下游的服务抗压能力进行评估。如果Sink端是Kafka，可以设为Kafka对应Topic的分区数。

## RocksDB大状态调优

RocksDB基于LSM Tree 实现的，类似于HBase

写入的方式是先写内存，所以RocksDB的写效率较高

读数据时先查内存blockcache，内存没有查磁盘，性能瓶颈主要是对磁盘的读，所以如果处理性能不够仅仅需要横向扩展并行度可以提高整个job的吞吐量

几个参数

设置本地 RocksDB 多目录

flink-conf.yaml 

```xml
state.backend.rocksdb.localdir: /data1/flink/rocksdb,/data2/flink/rocksdb,/data3/flink/rocksdb
```

注意：不要配置单块磁盘的多个目录，务必将目录配置到多块不同的磁盘上，让多块磁盘来分担压力。

RocksDB的内存和slot托管内存挂钩

state.backend.incremental：开启增量检查点，默认false，改为true。

state.backend.rocksdb.predefined-options：SPINNING_DISK_OPTIMIZED_HIGH_MEM

设置为机械硬盘+内存模式，有条件上SSD，指定为FLASH_SSD_OPTIMIZED

从Flink1.10开始，Flink默认将RocksDB的内存大小配置为每个task slot的托管内存。

调试内存性能的问题主要是通过调整配置项taskmanager.memory.managed.size 或者 taskmanager.memory.managed.fraction以增加Flink的托管内存(即堆外内存)。

对于更细粒度的控制，应该首先通过设置 state.backend.rocksdb.memory.managed为false，禁用自动内存管理，然后调整如下配置项：

暂时略

CheckPoint设置

一般我们的 Checkpoint 时间间隔可以设置为分钟级别，例如 1 分钟、3 分钟，对于状态很大的任务每次 Checkpoint 访问 HDFS 比较耗时，可以设置为 5~10 分钟一次Checkpoint，并且调大两次 Checkpoint 之间的暂停间隔，例如设置两次Checkpoint 之间至少暂停 4或8 分钟。

RocksDB相关参数

略

使用 Flink ParameterTool 读取配置

在 Flink 中可以通过使用 ParameterTool 类读取配置，它可以读取环境变量、运行参数、配置文件。

ParameterTool 是可序列化的，所以你可以将它当作参数进行传递给算子的自定义函数类。

在 Flink 程序中可以直接使用 ParameterTool.fromArgs(args) 获取到所有的参数，也可以通过 parameterTool.get("username") 方法获取某个参数对应的值。

## 读取系统属性

ParameterTool 还⽀持通过 ParameterTool.fromSystemProperties() 方法读取系统属性。做个打印：

```java
ParameterTool parameterTool = ParameterTool.fromSystemProperties();
System.out.println(parameterTool.toMap().toString());
```

## 读取配置文件

```java
ParameterTool.fromPropertiesFile("/application.properties")
```

## 注册全局参数

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); env.getConfig().setGlobalJobParameters(ParameterTool.fromArgs(args));
```

压测方式

在kafka中积压数据，之后开启Flink任务，出现反压就是处理瓶颈

# 反压处理

反压就是系统检测被阻塞的operator，然后自适应的降低源头或者上游的数据发送频率

Flink任务运行在多个节点，数据从上游到下游需要网络传输，反压时要降低上游速率也需要网络传输。

Flink网络流量控制细节

反压以及定位

不能简单通过BufferPool监控

Flink通过运行中任务采样来确定反压，如果一个task因为反压导致速度降低，会卡在向LocalBufferPool 申请内存块上

```java
java.lang.Object.wait(Native Method)
o.a.f.[...].LocalBufferPool.requestBuffer(LocalBufferPool.java:163) o.a.f.[...].LocalBufferPool.requestBufferBlocking(LocalBufferPool.java:133) [...]
```

只有当web页面切换到job的backpressure页面时才会触发反压监控

默认JM触发100次stack trace 每次间隔50ms来确定反压

0.01 表示在 100 个采样中只有 1 个被卡在LocalBufferPool.requestBufferBlocking()

## 利用 Flink Web UI 定位产生反压的位置

在 Flink Web UI 中有 BackPressure 的页面，通过该页面可以查看任务中 subtask 的反压状态，如下两图所示，分别展示了状态是 OK 和 HIGH 的场景。

排查的时候，先把operator chain禁用，方便定位。

## 利用Metrics定位反压位置

查看UI metrics页面

当某个 Task 吞吐量下降时，基于 Credit 的反压机制，上游不会给该 Task 发送数据，所以该 Task 不会频繁卡在向 Buffer Pool 去申请 Buffer。反压监控实现原理就是监控 Task 是否卡在申请 buffer 这一步，所以遇到瓶颈的 Task 对应的反压⻚⾯必然会显示 OK，即表示没有受到反压。

如果该 Task 吞吐量下降，造成该Task 上游的 Task 出现反压时，必然会存在：该 Task 对应的 InputChannel 变满，已经申请不到可用的Buffer 空间。如果该 Task 的 InputChannel 还能申请到可用 Buffer，那么上游就可以给该 Task 发送数据，上游 Task 也就不会被反压了，所以说遇到瓶颈且导致上游 Task 受到反压的 Task 对应的 InputChannel 必然是满的（这⾥不考虑⽹络遇到瓶颈的情况）。从这个思路出发，可以对该 Task 的 InputChannel 的使用情况进行监控，如果 InputChannel 使用率 100%，那么该 Task 就是我们要找的反压源。Flink 1.9 及以上版本inPoolUsage 表示 inputFloatingBuffersUsage 和inputExclusiveBuffersUsage 的总和。

## 反压的原因及处理

先检查基本原因，然后再深入研究更复杂的原因，最后找出导致瓶颈的原因。下面列出从最基本到比较复杂的一些反压潜在原因。
注意：反压可能是暂时的，可能是由于负载高峰、CheckPoint 或作业重启引起的数据积压而导致反压。如果反压是暂时的，应该忽略它。另外，请记住，断断续续的反压会影响我们分析和解决问题。

## 系统资源

检查涉及服务器基本资源的使用情况，如CPU、网络或磁盘I/O，目前 Flink 任务使用最主要的还是内存和 CPU 资源，本地磁盘、依赖的外部存储资源以及网卡资源一般都不会是瓶颈。如果某些资源被充分利用或大量使用，可以借助分析工具，分析性能瓶颈（JVM Profiler+ FlameGraph生成火焰图）。
如何生成火焰图：http://www.54tianzhisheng.cn/2020/10/05/flink-jvm-profiler/
如何读懂火焰图：https://zhuanlan.zhihu.com/p/29952444
针对特定的资源调优Flink
通过增加并行度或增加集群中的服务器数量来横向扩展
减少瓶颈算子上游的并行度，从而减少瓶颈算子接收的数据量（不建议，可能造成整个Job数据延迟增大）



垃圾收集（GC）
长时间GC暂停会导致性能问题。可以通过打印调试GC日志（通过-XX:+PrintGCDetails）或使用某些内存或 GC 分析器（GCViewer工具）来验证是否处于这种情况。

```xml
-Denv.java.opts="-XX:+PrintGCDetails -XX:+PrintGCDateStamps"
```

最重要的指标是Full GC

CPU/线程瓶颈
有时，一个或几个线程导致 CPU 瓶颈，而整个机器的CPU使用率仍然相对较低，则可能无法看到 CPU 瓶颈。例如，48核的服务器上，单个 CPU 瓶颈的线程仅占用 2％的 CPU 使用率，就算单个线程发生了 CPU 瓶颈，我们也看不出来。可以考虑使用2.2.1提到的分析工具，它们可以显示每个线程的 CPU 使用情况来识别热线程。
线程竞争
与上⾯的 CPU/线程瓶颈问题类似，subtask 可能会因为共享资源上高负载线程的竞争而成为瓶颈。同样，可以考虑使用2.2.1提到的分析工具，考虑在用户代码中查找同步开销、锁竞争，尽管避免在用户代码中添加同步。
负载不平衡
如果瓶颈是由数据倾斜引起的，可以尝试通过将数据分区的 key 进行加盐或通过实现本地预聚合来减轻数据倾斜的影响。（关于数据倾斜的详细解决方案，会在下一章节详细讨论）
外部依赖
如果发现我们的 Source 端数据读取性能比较低或者 Sink 端写入性能较差，需要检查第三方组件是否遇到瓶颈。例如，Kafka 集群是否需要扩容，Kafka 连接器是否并行度较低，HBase 的 rowkey 是否遇到热点问题。关于第三方组件的性能问题，需要结合具体的组件来分析。



## 盐表

# 数据倾斜

# Kafka source调优

# Flink SQL优化