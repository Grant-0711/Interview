版本20.1

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

物化视图

materializeMysql引擎

常见问题