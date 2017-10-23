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

5.将reduce join转为map join

不使用join算子进行连接操作，而使用Broadcast变量与map类算子实现join操作，进而完全规避掉shuffle类的操作，彻底避免数据倾斜的发生和出现。将较小RDD中的数据直接通过collect算子拉取到Driver端的内存中来，然后对其创建一个Broadcast变量；接着对另外一个RDD执行map类算子，在算子函数内，从Broadcast变量中获取较小RDD的全量数据，与当前RDD的每一条数据按照连接key进行比对，如果连接key相同的话，那么就将两个RDD的数据用你需要的方式连接起来。

6.采样倾斜key并分拆join操作（一个RDD少量key倾斜，另一个key数据量正常）

针对join类：对包含少数几个数据量过大的key的那个RDD，通过sample算子采样出一份样本来，然后统计一下每个key的数量，计算出来数据量最大的是哪几个key。 
然后将这几个key对应的数据从原来的RDD中拆分出来，形成一个单独的RDD，并给每个key都打上n以内的随机数作为前缀，而不会导致倾斜的大部分key形成另外一个RDD。 
接着将需要join的另一个RDD，也过滤出来那几个倾斜key对应的数据并形成一个单独的RDD，将每条数据膨胀成n条数据，这n条数据都按顺序附加一个0~n的前缀，不会导致倾斜的大部分key也形成另外一个RDD。 
再将附加了随机前缀的独立RDD与另一个膨胀n倍的独立RDD进行join，此时就可以将原先相同的key打散成n份，分散到多个task中去进行join了。 
而另外两个普通的RDD就照常join即可。 
最后将两次join的结果使用union算子合并起来即可，就是最终的join结果。

7.使用随机前缀和扩容RDD进行join(Rdd中大量的key导致数据倾斜)

该方案的实现思路基本和“采样倾斜key并分拆join操作”类似，首先查看RDD/Hive表中的数据分布情况，找到那个造成数据倾斜的RDD/Hive表，比如有多个key都对应了超过1万条数据。 
首先对有数据倾斜key的RDD的每条数据都打上一个n以内的随机前缀。 
同时对另外一个正常的RDD进行扩容，将每条数据都扩容成n条数据，扩容出来的每条数据都依次打上一个0~n的前缀。 
最后将两个处理后的RDD进行join即可。 
