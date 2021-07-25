# web端口是16010



删除表要先disable



查看表scan 指定startrow 和stoprow

在hbase存维度数据时用了盐表

regionserver 节点

region

每个regionserver里面有region  ，可以有多个region，region是真正存数据的组件

如果一个表都在一个region 里面读写速度是慢的

建表时默认是一个region，随着数据的增加会自动分裂region

分裂方式：单个region超过10g分裂

新版本：分裂数据每次增加 2*128乘以region个数的三次方

这种自动分裂会有数据迁移的情况

# 所以hbase提出了预分区

在建表的时候预测分区，根据rowkey来划分region，这样就不用分裂，没有分裂就不用迁移

# 方式是直接指定或者利用Phoenix

也就是盐表，一般是支持3-256

每一个region负责一部分数据

## 有盐表之后rowkey的设计添加hash前缀

以此来决定数据进入哪个region