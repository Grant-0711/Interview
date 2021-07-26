版本20.1



特点

大多都是读请求

不支持频繁的写入，单次写入数量大，1000行以上，每秒2-3次写入

查询速度极快

采用列式存储

​	采用列式存储的好处是

​		对于聚合，计数，求和等操作可以对比行形式存储更优

​		当使用列式存储可以获得更高的压缩比例，也可以更大程度发挥内存的性能

clickhouse不支持窗口函数和相关子查询



采用类LSM tree结构，也就是log sort merge的缩写，导入数据是顺序写，然后有归并排序，之后有合并

有数据分区和线程并行的概念，数据被分区化，进一步依据index granuliarity（粒度）进行划分，这样单条查询就可以利用cpu的整个资源，这样极大程度的提升了查询的并行度，可以处理大数据量的查询，但是由于这种特征，不适合高qps的查询场景，也就是同时有多个查询语句的场景不适合



一些限制

没有完整的事务支持，不能低延迟，高频率的去修改数据，只能支持批次修改，但是符合gdpr，也符合实时场景的需求



特殊的数据类型，

整形数据有普通整形和正数整形数据，表示为Uint

没有布尔类型，用int8指定数值为0或者1

支持的日期时间有date

datetime datetime datetime64



常用引擎

tinylog

memeory

mergeTree

summingMergeTree

replacingMergeTree



高可用（复本表）

基于zookeeper，客户端有写入请求写入cka，会同时提交写入日志给zoo，zoo通知ckb有写入信息，然后ckb去cka复制指定数据

并不是所有的表都有副本，要指定引擎为replacingMergeTree，并且insert ，alter操作会自动复制，但是create table，drop，rename等只会在单服务器执行

实现方式是现在metrika。xml文件下修改 相关zookeeperl cluster 配置

然后在建表的时候使用replicaingmergertree 要指定存储位置和节点

高并发（分片集群）

把一个表的数据拆分成不同的片放在不同的节点，通过distributed 引擎再拼接起来使用

使用方法是先在每台节点按照ck，修改配置文件metrika.xml文件，在其中指定一个cluster



配置方面修改metrika.xml修改clickhouse-remote-server相关参数，指定不同分片和分片的副本，然后分发，修改对应的分片的索引

然后创建分布式表来使用

create table  xxx  on cluster xxx（设置的集群名）

（字段）

engine = distributed（集群名，表名，主键）

查看sql执行计划

是20.6之后才支持的，20.6之前需要在执行日志里面查看

语法是explain  可以指定一些参数例如header 查看每个步骤的header信息，指定ast查看语法树，description，action等



建表优化

时间字段用数值类型或者日期类型，不要用字符串，这样可以免去函数转换

null值建议存字段的默认值，因为官方支出nullable类型会拖累性能，因为需要额外创建一个文件来存储null的标记

写入和删除优化

避免小批量的增删操作，会产生小分区文件，后台merge压力大

一般建议每秒发起两三次写入，每次2万到5万条数据

写入太快会出现too many parts 和 memory limit的错误



改资源配置，主要在config。xml和users。xml中，基本上都在users。xml中

在官网查，ducument的operation，server里面

语法优化

如果是查count（） 不指定字段，系统会默认用system。table的total——row

如果查询的字段中有重复的会自动去重，hive如果有了重复字段不会处理，ck会认为是语法错误

谓词下推，就是提前过滤，在hive中的体现就是在map时就过滤，在ck中是having 后面的在where后面过滤

聚合计算外推

例如select  sum（sal*2）会推成sum（sal）*2

因为前者是查一行乘一次，后者是先sum再乘2，更快

聚合函数消除

就是如果对聚合字段值求聚合值例如max，min ，sum等没有意义，因为分组之后的聚合字段值都是相同的，这会情况会把聚合函数直接消除

删除重复的group by key

三元运算符优化  如果有 num = 1？ “1”：（num =2?"2","3"）

开启optimize——if——chain——to_multiff后会优化成

select  multiif（num = 1？1，num = 2？2，3）

查询优化

数据一致性

# 物化视图

实际就是可以对查询结果做一个持久化存储，查询可以是单张表的子集也可以是多表join之后的结果，相当于对结果做一个快照，语法是create materialize view 。。。as select 。。。

materializeMysql引擎

20.8版本之后ck可以伪装成mysql的从机去监控mysql的表，直接实现mysql的动态数据抓取，而不同经过canal，maxwell，cdc这样的工具

# 常见问题

## 分布式表某个节点创建失败

分布式建表的时候某个节点上没有建表但是client返回是正常的，查看日志有

other create executor at the same times

解决办法是直接重启改节点的ck服务

## 副本表的数据不一致

副本表的数据不一致，解决办法是在副本节点上创建一张本地表，表的结构从其他节点获取，然后该表会从其他节点同步数据

# 过期时间

使用mergeTree创建表时可以指定数据的过期时间，过期之后把值设置为数据字段值的默认值

