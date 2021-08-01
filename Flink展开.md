Flink版本

​    Flink 1.13.1 - 2021-05-28                                                 

​    Flink 1.13.0 - 2021-04-30                                                                                         

​    Flink 1.12.0 - 2020-12-08                                                                                  

​    Flink 1.11.0 - 2020-07-06                                                                                         

​    Flink 1.10.0 - 2020-02-11                                                                                  

​    Flink 1.9.0 - 2019-08-22                                                                                

​    Flink 1.8.0 - 2019-04-09                                                                                       

​    Flink 1.7.0 - 2018-11-30                                                                                    

​    Flink 1.6.0 - 2018-08-08     



# Flink背压机制？反压机制？

产生原因

当流系统中消息的处理速度小于消息的发送速度，就会导致消息积压。

如果系统能感知积压并处理就是有背压感知的系统。

Flink web UI 后台可以查看:

![img](https:////upload-images.jianshu.io/upload_images/13570404-019c16a39cfd87e5.png?imageMogr2/auto-orient/strip|imageView2/2/w/1013)



# Flink遇到的问题以及解决优化

## 问题1

生成checkpoint失败较多,

原因是某几个subtask的快照超时导致整体的checkpoint生成失败,

如果每天的任务变多的话也会产生较多这个问题

目前的 Checkpoint 算法在作业出现反压时，阻塞式的 Barrier 对齐反而会加剧作业的反压，甚至导致作业的不稳定。

首先， Chandy-Lamport 分布式快照的结束依赖于水印的流动，而反压则会限制 水印流动，导致快照的完成时间变长甚至超时。无论是哪种情况，都会导致 Checkpoint 的时间点落后于实际数据流较多。

这时作业进度是没有被持久化的，处于脆弱状态，如果作业出于异常被动重启或者被用户主动重启，作业会回滚丢失一定的进度。如果 Checkpoint 连续超时且没有很好的监控，回滚丢失的进度可能高达一天以上，对于实时业务这通常是不可接受的。

更糟糕的是，回滚后的作业落后的 Lag 更大，通常带来更大的反压，形成一个恶性循环。

### **优化1: rebalance分区改为rescale分区**

rebalance使用Round-ribon思想将数据均匀分配到各实例上。

dataStream.rebalance()

rescale与rebalance很像，也是将数据均匀分布到各下游各实例上，但它的传输开销更小，是就近发送给下游实例。

dataStream.rescale()

