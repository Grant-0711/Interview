# 流的世界观



# 编程模型

分层api

process ，datastream，dataset，table api ，flink sql

编程模型 环境env streamExecutionEnvironment，source算子，transform算子，sink算子

# 部署模式

localcluster

yarn  

## 建议使用perjob  application隔离性更好

app和per的区别是解析代码的位置 per在客户端

app解析代码在jobmanager



mesos 

k8s 未来趋势，管理容器



# 时间语义

时间，处理，注入（数据插入的时间）

watermark六点

①衡量事件时间的进展

②解决乱序问题

③是一个特殊的时间戳，也是数据，来源于数据，要插入到数据流中

④单调不减的

⑤触发计算的，要结合窗口和计时器来使用

⑥flink认为事件时间小于水印的数据都已经处理过了，即便是迟到数据也使用侧输出流处理过了

如果涉及到多并行度的水印传递，默认传递最小的



如果上游有多个流，其中一个流的水印一直不更新，会影响下游水印的更新，这里是可以设置一个超时时间，过了超时时间将不参考该流的水印，举例使用kafka多分区



状态：大分类是管理状态和raw state

managed state 分为operator state 和 keyed state

ck：基于chandy lamport 分布式快照

具体过程

jm向source算子 triger ck

source task 在流中插入barrier n

source向下游传递 barrier n 

当下游收到所有的 barrier n 之后开始制作本次快照

完成之后向jm汇报，jm收到所有任务的汇报之后认为关于barrier n 的这一次ck完成，然后会向持久化存储中再存储一份ck的元数据



jvm自己的内存管理对于flink而言有很多问题，flink有自己的内存管理系统，源码里相关的内存模型有一个接口是flinkmemory，内存数据结构有一个类是memorysegment，最近还在看源码，还需要后续整理下

# ------------------------------

特性 高吞吐，底誓言，高性能的流处理引擎

有事件时间的窗口操作

支持严格一次性语义

支持各种窗口，分为时间和计数两大类，时间有滚动，华东，会话，全局需要自定义触发器关闭

计数有滚动，滑动





