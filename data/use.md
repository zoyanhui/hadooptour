[首页](index)

# Spark

## A. 小文件过多
* 解决方法：
        使用 SparkContext下newAPIHadoopFile完成数据输入，指定`org.apache.hadoop.mapreduce.lib.input.CombineTextInputFormat`


## B. 编程优化
   *Spark程序几点优化*

1. **repartition和coalesce**

      这两个方法都可以用在对数据的重新分区中，其中repartition是一个代价很大的操作，它会将所有的数据进行一次shuffle，然后重新分区。

      如果你仅仅只是想减少分区数，从而达到减少碎片任务或者碎片数据的目的。使用coalesce就可以实现，该操作默认不会进行shuffle。其实repartition只是coalesce的shuffle版本。

      一般我们会在filter算子过滤了大量数据后使用它。比如将 partition 数从1000减少到100。这可以减少碎片任务，降低启动task的开销。

      **note1**: 如果想查看当前rdd的分区数，在java/scala中可以使用rdd.partitions.size()，在python中使用rdd.getNumPartitions()。

      **note2**: 如果要增加分区数，只能使用repartition,或者把partition缩减为一个非常小的值，比如说“1”，也建议使用repartition。

2. **mapPartitions和foreachPartitions**

      适当使用mapPartitions和foreachPartitions代替map和foreach可以提高程序运行速度。这类操作一次会处理一个partition中的所有数据，而不是一条数据。

      mapPartition - 因为每次操作是针对partition的，那么操作中的很多对象和变量都将可以复用，比如说在方法中使用广播变量等。

      foreachPartition - 在和外部数据库交互操作时使用，比如 redis , mysql 等。通过该方法可以避免频繁的创建和销毁链接，每个partition使用一个数据库链接，对效率的提升还是非常明显的。

      **note**: 此类方法也存在缺陷，因为一次处理一个partition中的所有数据，在内存不足的时候，将会遇到OOM的问题。

3. **reduceByKey和aggregateByKey**

      使用reduceByKey/aggregateByKey代替groupByKey。

      reduceByKey/aggregateByKey会先在map端对本地数据按照用户定义的规则进行一次聚合，之后再将计算的结果进行shuffle，而groupByKey则会将所以的计算放在reduce阶段进行（全量数据在各个节点中进行了分发和传输）。所以前者的操作大量的减少shuffle的数据，减少了网络IO，提高运行效率。

4. **mapValues**

      针对k,v结构的rdd，mapValues直接对value进行操作，不对Key造成影响，可以减少不必要的分区操作。

5. **broadcast**

      Spark中广播变量有几个常见的用法。

      - 实现map-side join

      在需要join操作时，将较小的那份数据转化为普通的集合（数组）进行广播，然后在大数据集中使用小数据进行相应的查询操作，就可以实现map-side join的功能，避免了join操作的shuffle过程。在我之前的文章中对此用法有详细说明和过程图解。

      - 使用较大的外部变量

      如果存在较大的外部变量（外部变量可以理解为在driver中定义的变量），比如说字典数据等。在运算过程中，会将这个变量复制出多个副本，传输到每个task之中进行执行。如果这个变量的大小有100M或者更大，将会浪费大量的网络IO，同时，executor也会因此被占用大量的内存，造成频繁GC，甚至引发OOM。

      因此在这种情况下，我最好提前对该变量进行广播，它会被事先将副本分发到每个executor中，同一executor中的task则在执行时共享该变量。很大程度的减少了网络IO开销以及executor的内存使用。

6. **复用RDD**

      避免创建一些用处不大的中间RDD(比如从父RDD抽取了某几个字段形成新的RDD)，这样可以减少一些算子操作。

      对多次使用的RDD进行缓存操作，减少重复计算，在下文有说明。

7. **cache和persist**

      cache方法等价于persist(StorageLevel.MEMORY_ONLY)

      不要滥用缓存操作。缓存操作非常消耗内存，缓存前考虑好是否还可以对一些无关数据进行过滤。如果你的数据在接下来的操作中只使用一次，则不要进行缓存。

      如果需要复用RDD，则可以考虑使用缓存操作，将大幅度提高运行效率。缓存也分几个级别。

      - MEMORY_ONLY

      如果缓存的数据量不大或是内存足够，可以使用这种方式，该策略效率是最高的。但是如果内存不够，之前缓存的数据则会被清出内存。在spark1.6中，则会直接提示OOM。

      - MEMORY_AND_DISK

      优先将数据写入内存，如果内存不够则写入硬盘。较为稳妥的策略，但是如果不是很复杂的计算，可能重算的速度比从磁盘中读取还要快。

      - MEMORY_ONLY_SER

      会将RDD中的数据序列化后存入内存，占用更小的内存空间，减少GC频率，当然，取出数据时需要反序列化，同样会消耗资源。

      - MEMORY_AND_DISK_SER

      不再赘述。

      - DISK_ONLY

      该策略类似于checkPoint方法，把所有的数据存入了硬盘，再使用的时候从中读出。适用于数据量很大，重算代价也非常高的操作。

      - 各种_2结尾的存储策略

      实际上是对缓存的数据做了一个备份，代价非常高，一般不建议使用。



### C. SparkSql优化配置

* Caching Data In Memory
> Spark SQL can cache tables using an in-memory columnar format by calling `spark.cacheTable("tableName")` or `dataFrame.cache()`. Then Spark SQL will scan only required columns and will automatically tune compression to minimize memory usage and GC pressure. You can call `spark.uncacheTable("tableName")` to remove the table from memory.
Configuration of in-memory caching can be done using the setConf method on SparkSession or by running SET key=value commands using SQL.

    - spark.sql.inMemoryColumnarStorage.compressed
    default: true
    > When set to true Spark SQL will automatically select a compression codec for each column based on statistics of the data.

    - spark.sql.inMemoryColumnarStorage.batchSize
    default: 10000
    > Controls the size of batches for columnar caching. Larger batch sizes can improve memory utilization and compression, but risk OOMs when caching data.

* spark.sql.files.maxPartitionBytes  
default: 134217728 (128 MB)
> The maximum number of bytes to pack into a single partition when reading files.

* spark.sql.files.openCostInBytes
default: 4194304 (4 MB)
> The estimated cost to open a file, measured by the number of bytes could be scanned in the same time. This is used when putting multiple files into a partition. It is better to over estimated, then the partitions with small files will be faster than partitions with bigger files (which is scheduled first).

* spark.sql.autoBroadcastJoinThreshold
default: 10485760 (10 MB)
> Configures the maximum size in bytes for a table that will be broadcast to all worker nodes when performing a join. By setting this value to -1 broadcasting can be disabled. Note that currently statistics are only supported for Hive Metastore tables where the command ANALYZE TABLE <tableName> COMPUTE STATISTICS noscan
 has been run.

* spark.sql.shuffle.partitions
default: 200
> Configures the number of partitions to use when shuffling data for joins or aggregations.
  - 自动决定join和groupby时reducer的数量。 如果使用默认200配置，可能出现，sparksql的reduce task总是200个的情况，导致insert into hive的文件数量也是200个，容易造成小文件过多
  -  如果partitions数目过少，容易出现shuffle内存超限。org.apache.spark.shuffle.MetadataFetchFailedException: Missing an output location for shuffle


### D. Spark Sql unsupported (Spark 1.6.2)
* Tables with buckets: bucket is the hash partitioning within a Hive table partition. Spark SQL doesn’t support buckets yet.
* UNION type
* Unique join
* Column statistics collecting: Spark SQL does not piggyback scans to collect column statistics at the moment and only supports populating the sizeInBytes field of the hive metastore
* File format for CLI: For results showing back to the CLI, Spark SQL only supports TextOutputFormat.
* Hadoop archive
* Block level bitmap indexes and virtual columns (used to build indexes)
* Automatically determine the number of reducers for joins and groupbys: Currently in Spark SQL, you need to control the degree of parallelism post-shuffle using “SET spark.sql.shuffle.partitions=[num_tasks];”.
* Meta-data only query: For queries that can be answered by using only meta data, Spark SQL still launches tasks to compute the result.
* Skew data flag: Spark SQL does not follow the skew data flags in Hive.
* STREAMTABLE hint in join: Spark SQL does not follow the STREAMTABLE hint.
* Merge multiple small files for query results: if the result output contains multiple small files, Hive can optionally merge the small files into fewer large files to avoid overflowing the HDFS metadata. Spark SQL does not support that.


### E.spark executor memory
spark.storage.memoryFraction   
 - this would allow cache use more of allocated memory
大多数情况下，cache memory可以松动，在保证运行内存和shuffle内存的情况下，满足cache的要求。否则，可以采用 内存+硬盘的 缓存方式，来解决内存不够分配缓存的情况。

spark.shuffle.memoryFraction=0.5
 - this would allow shuffle use more of allocated memory

spark.yarn.executor.memoryOverhead=1024
 - this is set in MB. Yarn kills executors when its memory usage is larger then (executor-memory + executor.memoryOverhead)

**Little more info**
- If you get shuffle not found exception.
In case of org.apache.spark.shuffle.MetadataFetchFailedException: Missing an output location for shuffle
 you should increase spark.shuffle.memoryFraction, for example to 0.5.

- Most common reason for Yarn killing off my executors was memory usage beyond what it expected. To avoid that you increase spark.yarn.executor.memoryOverhead, I've set it to 1024, even if my executors use only 2-3G of memory.