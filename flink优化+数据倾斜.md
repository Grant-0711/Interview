rocksbd大状态的优化

rocksdb的写入是基于LSM tree的所以写请求效率比较高

所以使用rocksdb的性能瓶颈主要是读效率的提升，方法是可以给rocksdb的磁盘存储目录配置多目录，在flink-conf。xml中有相关配置项



数据倾斜情况的检测，是ui界面的subtask的receivced bytes 显示，如果有数据倾斜的话每个子任务的接收数据量差距会比较大





数据倾斜的处理

keyby之前和keyby之后

keyby之前发生数据倾斜，实际上是上游的算子的实例处理的数据量有差别，实际场景中例如消费kafka的数据有多个分区，不同分区的数据量不同的情况的话会有可能发生数据倾斜，解决办法是强制进行shuffle，使用shuffle，rebalance，rescale 算子



keyby之后一般是考虑两阶段聚合

先把key添加随机前缀，然后分组，开窗，聚合，然后数据获取窗口结束时间进行分组，先保证同一窗口的数据进入同一组，然后去除前缀，再次做keyby 聚合