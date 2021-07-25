kafka丢数据

分情况讲

对于broker：有落盘，但是是先写linux 的 page  cache 

极端情况下如果服务器挂了会丢，一般不会丢

生产者 考虑ack

0  不需要应答 

1 leader应答，leader没有同步给follower时挂了

-1 leader 应答，follower应答，效率低可靠性高  ，在 min -insync replica 参数大于1 考虑isr里是不是有同步的follower

消费者：

ss 手动维护 offset

flume事务

flink offset存在source状态，有ck，用kafka egle 监控可以看到 锯齿状的变化情况