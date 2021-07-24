数据倾斜

方案一

聚合原数据

避免shuffle过程

在进行spark job之前对某些热点数据的value进行聚合，例如对同一key的value进行特殊形式的拼接，或者对值进行预先聚合

增大或者缩小key的粒度



方案二

过滤导致倾斜的key

方案三提高shuffle操作中的reduce并行度



方案四 使用随机key实现双重聚合

对数据进行map操作，给不同的key加上随机前缀或者后缀，进行一次聚合，再对结果进行map，将key的前缀或者后缀删除，再进行聚合

方案五 将reduce join 转换为 map join （广播join）

方案六：sample采样得到倾斜的key，然后对该key进行单独的shuffle操作，因为spark中如果某个RDD只有一个key，在shuffle中会将key打散到不同的reduce'任务进行处理

之后再将该key的结果和别的数据得到的结果union到一起，sample一般和fraction算子一起用

方案七：采用随机数扩容单独进行join

通常是结合方案六，例如有两个rdd，进行join操作，其中数据量大的rdd中有几个key是数据量大的，我们可以用方案六中的sample+fraction操作找出这几个key单独过滤出来形成一个rdd，然后利用添加特殊前后缀的方式将key进行打散，然后把参与join的另一个rdd的相关key也进行相同操作，然后二者进行join，把结果和之前普通rdd的结果进行union

