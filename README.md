# Spark-Memory-optimization
大数据中最棘手的问题--数据倾斜，此时spark的性能会比期望差很多，数据倾斜调优，就是使用各种技术方案解决不同类型的数据倾斜问题，以保证Spark作业的性能。

数据倾斜发生的现象：

1.比如有1000个task，其中997个task都花了几分钟运行完了，最后3个task确花了几个小时才跑完，对应的key相应的数据量特比大。

2.正常执行中的spark作业，突然报OOM（内存溢出）的错误，观察异常栈，却是我们的代码出现问题。


数据倾斜-解决方案：

1.使用Hive ETL工具提前将Key在hive中聚合，或者预先和其他表进行join，治标不治本，数据倾斜发生在了hive etl中而不是spark生产中。我们将有些Spark作业的shuffle操作提前到了Hive ETL中，从而让Spark直接使用预处理的Hive中间表，尽可能地减少Spark的shuffle操作，大幅度提升了性能，将部分作业的性能提升了6倍以上（美团、点评）

2.过滤掉少数导致倾斜的Key，这些Key不参与计算了

3.提高shuffle操作的并行度：spark.sql.shuffle.partitions，该参数代表了shuffle read task的并行度，增加task的数量，这样每一个task运行的时间就更短了

4.两阶段聚合（局部聚合+全局聚合）

针对聚合类的shuffle：第一次局部聚合，首先将key值前面加上10以内的随机数，比如(hello, 1) (hello, 1) (hello, 1) (hello, 1)，就会变成(1_hello, 1) (1_hello, 1) (2_hello, 1) (2_hello, 1),接着对打上随机数后的数据，执行reduceByKey等聚合操作，进行局部聚合，那么局部聚合结果，就会变成了(1_hello, 2) (2_hello, 2)。然后将各个key的前缀给去掉，就会变成(hello,2)(hello,2)，再次进行全局聚合操作，就可以得到最终结果了，比如(hello, 4)。
