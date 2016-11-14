[首页](index)

> 针对Hive的优化主要有以下几个方面：
  1. map
  2. reduce
  3. file format
  4. shuffle & sort
  5. job as whole
  6. job chain


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2716069-80b84e7d0f0cb9a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Hive map reduce 的过程如下：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2716069-9f1803da430d3e62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# Map 阶段优化
> Map 阶段的优化，主要是控制 map size的大小， map任务的数量
`Map task num = Total Input size / map size` 
`map size =  max{ 
                    ${mapred.min.split.size},       // 数据的最小分割单元大小， 可调整，默认1B
                    min(
                            ${dfs.block.size},           // hdfs 上数据块大小， 有hdfs配置决定
                            ${mapred.max.split.size})  // 数据最大分隔单元大小， 可调整，默认256MB
                }`
直接调整 `mapred.map.tasks` 这个参数是没有效果
根据map阶段的使用时间，来调整数据输入大小

* Map 阶段, 小文件对map任务数影响很大， 可以用以下参数合并小文件
> Map输入合并小文件对应参数：   
```
      - set mapred.max.split.size=256000000;     // 每个Map最大输入大小
      - set mapred.min.split.size.per.node=100000000;   // 一个节点上map最小分割，决定结点间是否合并
      - set mapred.min.split.size.per.rack=100000000;  // 一个机架下map最小分割，决定机架间是否合并
      - set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;   
        // 执行Map前进行小文件合并, 合并的大小由`mapred.max.split.size`参数决定
```

# Map与Reduce之间的优化(spill, copy, sort phase)
   1. map phase和reduce phase之间主要有3道工序
      - 首先要把map输出的结果进行排序后做成中间文件
      - 其次这个中间文件就能分发到各个reduce
      - 最后reduce端在执行reduce phase之前把收集到的排序子文件合并成一个排序文件
       > 需要强调的是，虽然这个部分可以调的参数挺多，但是大部分在一般情况下都是不要调整的，除非能精准的定位到这个部分有问题。

  ## Spill 与 Sort
     由于内存不够，局部排序的内容会写入磁盘文件，这个过程叫做spill.
      spill出来的文件再进行merge
    1. io.sort.mb 控制mapper buffer大小，影响spill的发生时机。 buffer满的时候，触发spill
    2. io.sort.factor 控制merge阶段合并的文件大小， 默认10(个文件)
      > 调整参数，由spill的时间和merge时间决定。io.sort.mb不能超过map的jvm heap size。
        Reduce端的merge也是一样可以用io.sort.factor。
        一般情况下这两个参数很少需要调整，除非很明确知道这个地方是瓶颈。比如如果map端的输出太大，    要么是每个map读入了很大的文件（比如不能split的大gz压缩文件），要么是计算逻辑导致输出膨胀了很多倍。

  ## Copy
也就是shuffle，这个阶段把文件从map端copy到reduce端。
`mapred.reduce.slowstart.completed.maps` map完成多少的时候，启动copy。默认5%
`tasktracker.http.threads` 作为server端的map用于提供数据传输服务的线程
`mapred.reduce.parallel.copies` 作为client端的reduce同时从map端拉取数据的并行度（一次同时从多少个map拉数据）
`mapred.job.shuffle.input.buffer.percent` 控制shuffle buffer占reduce task heap size的大小，默认0.7(70%）
  > `tasktracker.http.threads` 与 `mapred.reduce.parallel.copies` 需要相互配合。
     shuffle阶段可能会出现的OOM。一般认为是内存分配不合理，GC无法及时释放内存导致。
    可以尝试调低`mapred.job.shuffle.input.buffer.percent` 或者增加 reduce 数量  `mapred.reduce.tasks`


2. Map输出合并
```
  - set hive.merge.mapfiles = true  // 在Map-only的任务结束时合并小文件
  - set hive.merge.mapredfiles = true  // 在Map-Reduce的任务结束时合并小文件
  - set hive.merge.size.per.task = 256*1000*1000 // 合并文件的大小
  - set hive.merge.smallfiles.avgsize=16000000
     // 当输出文件的平均大小小于该值时，启动一个独立的map-reduce任务进行文件merge
```

# Reduce 阶段优化
> reduce phase 优化
  1. 可以直接控制reduce的数量， `mapred.reduce.tasks` 参数
  2. 灵活配置，Hive自动计算reduce数量， 公式：
`num_reduce_tasks =  
    min {  
        ${hive.exec.reducers.max},           // 默认999， 上限
        ${input.size} / ${ hive.exec.reducers.bytes.per.reducer}  
}`
hive.exec.reducers.bytes.per.reducer // 默认1G

# 文件格式的优化
 Hive0.9版本有3种，textfile，sequencefile和rcfile。总体上来说，rcfile的压缩比例和查询时间稍好。
```
关于使用方法，可以在建表结构时可以指定格式，然后指定压缩插入：
create table rc_file_test( col int ) stored as rcfile;
set hive.exec.compress.output = true;
insert overwrite table rc_file_test select * from source_table;
```
```
也可以通过hive.default.fileformat
来设定输出格式，适用于create table as select的情况：
set hive.default.fileformat = SequenceFile;
set hive.exec.compress.output = true; 
/*对于sequencefile，有record和block两种压缩方式可选，block压缩比更高*/
set mapred.output.compression.type = BLOCK; 
create table seq_file_testas select * from source_table;
```

上面的文件格式转换，其实是由hive完成的（也就是插入动作）。但是也可以由外部直接导入纯文本启用压缩（lzo压缩），或者是由MapReduce Job生成的数据。
1. Lzo压缩
> lzo 文件支持split, 适合hdfs存储。 启用lzo压缩特别对于小规模集群还是很有用的，压缩比率大概能达到原始日志大小的1/4左右。
Hadoop原生是支持gzip和bzip2压缩， 压缩比率比lzo大，但是不支持split，所以速度很慢。
不过lzo不比gzip和bzip2，不是linux系统原生支持的，需要安装至少lzo，lzop 以及 hadoop lzo jar.
同时，配置hadoop里面 core-site.xml, mapred-site.xml
> 
core-site.xml
```
<property> 
        <name>io.compression.codecs</name>
        <value>org.apache.hadoop.io.compress.DefaultCodec,com.hadoop.compression.lzo.LzoCodec,com.hadoop.compression.lzo.LzopCodec,org.apache.hadoop.io.compress.GzipCodec,org.apache.hadoop.io.compress.BZip2Codec</value> 
</property> 
<property> 
        <name>io.compression.codec.lzo.class</name> 
        <value>com.hadoop.compression.lzo.LzoCodec</value> 
</property> 
```
>
使用lzo，lzop，gzip，bzip2压缩作为io压缩的编解码器，并指定lzo的类
mapred-site.xml
```
<property> 
        <name>mapred.compress.map.output</name> 
        <value>true</value> 
</property> 
<property> 
        <name>mapred.map.output.compression.codec</name> 
        <value>com.hadoop.compression.lzo.LzoCodec</value> 
</property> 
<property> 
        <name>mapred.child.java.opts</name> 
        <value>-Djava.library.path=/opt/hadoopgpl/native/Linux-amd64-64</value> 
</property> 
```
`map结果采用压缩输出，可以降低网络带宽的使用，并指定map输出所使用的lzo的类。以及指定编解码器所在位置。`
> ---------------------------------------------------------- 
创建lzo索引：
```
hadoop jar /opt/hadoopgpl/lib/hadoop-lzo.jar    \
com.hadoop.compression.lzo.LzoIndexer       \
/path/to/lzo/file/or/path
```
> ----------------------------------------------------------
在streaming中使用lzo：
```
hadoop jar /usr/share/hadoop/contrib/streaming/hadoop-streaming-1.0.3.jar \ 
-file map.py \ 
-file red.py \ 
-mapper map.py \ 
-reducer red.py \ 
-inputformat com.hadoop.mapred.DeprecatedLzoTextInputFormat \ 
-input /data/rawlog/test/20130325 -output /tmp/test_20130325
```
> ----------------------------------------------------------
以及在hive中指定压缩编解码器：
hadoop集群启用了压缩，就需要在Hive建表的时候指定压缩时所使用的编解码器，否则Hive无法正确读取数据。
Gzip和Bzip2由于是hadoop默认支持的，所以无需指定特殊的编解码器，只要指定Text类型即可。
```
CREATE EXTERNAL TABLE text_test_table( 
      id string,
      name string) 
      ROW FORMAT DELIMITED  
      FIELDS TERMINATED BY '\t'  
      STORED AS TEXTFILE  
      LOCATION 
            'hdfs://hadoopmaster:9000/data/dw/adpv/20130325' 
```
而LZO是外挂的第三方库，所以要指定输入和输出的编解码器。
```
CREATE EXTERNAL TABLE lzo_test_table( 
      id string,
      name string) 
      ROW FORMAT DELIMITED  
      FIELDS TERMINATED BY '\t'  
      STORED AS INPUTFORMAT  
      'com.hadoop.mapred.DeprecatedLzoTextInputFormat'  
      OUTPUTFORMAT  
      'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat' 
      LOCATION 
            'hdfs://hadoopmaster:9000/data/dw/adpv/20130325' 
```
> ----------------------------------------------------------
对于日志文件，可以本地lzop压缩好，然后推到hdfs。
另外，对hadoop的jobconf做一个指定。这样就可以做到，输入是lzo，输出也可以lzo。或者输入是text，输出是lzo。
```
-inputformat com.hadoop.mapred.DeprecatedLzoTextInputFormat  \
-jobconf mapred.output.compress=true  \
-jobconf mapred.output.compression.codec=com.hadoop.compression.lzo.LzopCodec 
```
最后对HDFS上的日志做Indexer（创建lzo索引），这样Hive或者MR，可以把大的lzo压缩文件，切分多个map计算（splittable）
>  **注意**
hive读取sequencefile，会忽略key，(比如，lzo文件就是key-value），直接读value，并且按照指定分隔符分隔value，获得字段值。
但是如果hive的数据来源是从mr生成的，那么写sequencefile的时候，key和value都是有意义的，key不能被忽略，而是应该当成第一个字段。为了解决这种不匹配的情况，有两种办法：
    - 一是要求凡是结果会给hive用的mr job输出value的时候带上key。但是这样的话对于开发是一个负担，读写数据的时候都要注意这个情况。
    - 二是把这个交给hive解决，写一个InputFormat包装一下，把value输出加上key即可。以下是核心代码，修改了RecordReader的next方法：
```
//注意：这里为了简化，假定了key和value都是Text类型，所以MR的输出的k/v都要是Text类型。
//这个简化还会造成数据为空时，出现org.apache.hadoop.io.BytesWritable cannot be cast to org.apache.hadoop.io.Text的错误，因为默认hive的sequencefile的key是一个空的ByteWritable。
public synchronized boolean next(K key, V value) throws IOException { 
          Text tKey = (Text) key; 
          Text tValue = (Text) value; 
          if (!super.next(innerKey, innerValue)) 
               return false;
          Text inner_key = (Text) innerKey; //在构造函数中用createKey()生成 
          Text inner_value = (Text) innerValue; //在构造函数中用createValue()生成
          tKey.set(inner_key);
          tValue.set(inner_key.toString() + '\t' + inner_value.toString()); // 分隔符注意自己定义
          return true;
}
```
2. Map 输入输出其他压缩 （ hadoop 2.x)
除了lzo, 常用的还有snappy压缩等。
比起zlib，snappy压缩大多数情况下都会更快，但是压缩后大小会大20%到100%。
snappy跟lzo同属于Lempel–Ziv 压缩算法家族，但是优于lzo的两点是：
  1）Snappy解压缩比 LZO 更快, 压缩速度相当, meaning the[total round-trip time is superior](https://github.com/ning/jvm-compressor-benchmark/wiki).
  2)  Snappy 是 BSD-licensed, 可以集成在 Hadoop。 LZO 是 GPL-licensed, 需要单独安装。
通常集群会使用lzo做map reduce的中间结果压缩，中间结果是用户不可见的，一般是mapper写入磁盘，并供reducer读取。中间结果对压缩友好，一是key有冗余，而是需要写入磁盘，压缩减少IO量。同时，lzo和snappy都不是CPU密集的压缩算法，不会造成map，reduce的CPU时间缺乏。 而且snappy的效率比lzo要高20%。
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2716069-92ba1a0c5799d7ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
需要注意的一点是，Snappy旨在与容器格式（例如序列文件或Avro数据文件）一起使用，而不是直接在纯文本上使用，因为后者不可拆分(not splittable)，无法处理并行使用MapReduce。 这不同于LZO，其中可以索引LZO压缩文件以确定分裂点，使得LZO文件可以在后续处理中被有效地处理。
如图，.snappy与.lzo都不是splittable的，lzo可以通过创建index文件弥补，snappy适合用在序列文件( Sequence Files)上，比如常用的在map阶段的输出，如下：
**使用**
*core-site.xml*:
`io.compression.codecs` 增加 `org.apache.hadoop.io.compress.SnappyCodec`
 *mapred-site.xml*:
```
<property>
        <name>mapred.compress.map.output</name>
        <value>true</value>
 </property>
<property>
        <name>mapred.map.output.compression.codec</name>
        <value>org.apache.hadoop.io.compress.SnappyCodec</value>
</property>
```

# Job整体优化
从job的整体上，看是否有可以优化的点，以下分别陈述几个：

  ## Job执行模式
hadoop map-reduce 三种模式，local、伪分布式、分布式。
分布式模式，需要启动分布式job，调度资源，启动时间比较长。对于很小的job可以采用local map-reduce 模式。参数如下：
```
`hive.exec.mode.local.auto`=true // 就可以自动开启local模式，同时需要以下两个参数配合
`hive.exec.mode.local.auto.tasks.max`  // 默认4
`hive.exec.mode.local.auto.inputbytes.max`  // 默认 128（M）
默认map task数少于4（参考文章上面提到的map任务的决定因素），并且总输入大小不超过128M，则自动开启local模式。
另外，简单的select语句，比如select limit 10， 这种取少量sample的方式，那么在hive0.10之后有专门的fetch task优化，使用参数hive.fetch.task.conversion。Hadoop 2.x 应该是默认开启的。
```

  ## JVM重用
正常情况下，MapReduce启动的JVM在完成一个task之后就退出了，但是如果任务花费时间很短，又要多次启动JVM的情况下（比如对很大数据量进行计数操作），JVM的启动时间就会变成一个比较大的overhead。在这种情况下，可以使用jvm重用的参数：
```
  set mapred.job.reuse.jvm.num.tasks=5;  // 一个jvm完成多少个task之后再退出
```
适合小任务较多的场景，对于大任务影响不大，而且考虑大任务gc的时间，启动新的jvm成本可以忽略。


  ## 索引
> Hive可以针对列建立索引，在以下情况需要使用索引：
1) 数据集很大
2) 查询时间大大超过预期
3) 需要快速查询
4) 当构建数据模型的时候
Hive的索引管理在独立的表中（类似于lookup table，而不是传统数据库的B-tree)，不影响数据表。另一个优势是索引同样可以被partition，决定于数据的大小。

* 索引类型
  - Compact Indexing
  > Compact indexing stores the pair of indexed column’s value and its blockid.
  - Bitmap Indexing (在hive0.8出行，通常以用于列值比较独特的场景（distinct values) )
  > Bitmap indexing stores the combination of indexed column value and list of rows as a bitmap. 
* 创建索引

```

CREATE INDEX index_name
 
ON TABLE table_name (columns,....)
 
AS 'org.apache.hadoop.hive.ql.index.compact.CompactIndexHandler'    // create a compact index
 
WITH DEFERRED REBUILD;   // This statement means we need to alter the index in later stages
```
这个语句会创建一个索引，但是要完成创建，还需要完成rebuild语句。通过添加一个或多个更改语句，会开启一个map-reudce 任务来完成索引创建。
```
ALTER INDEX index_nam on table_name REBUILD;  //  This ALTER statement will complete our REBUILDED index creation for the table.
```
* 查看表的索引

```
show formatted index on table_name;
```
* 创建bitmap索引         

```
CREATE INDEX index_name
 
ON TABLE table_name (age)
 
AS 'BITMAP'
 
WITH DEFERRED REBUILD;
 
ALTER INDEX index_name on table_name REBUILD;
```
* 删除索引

```
DROP INDEX IF EXISTS olympic_index ON olympic;
```
**Note:**
1. 同一个表，创建不同类型的index在相同的列上，先创建的index会被使用
2. 索引会加快query执行速度
3. 一个表能创建任意数量的索引
4. 使用什么索引，依赖于数据。有时候，bitmap索引要快，有时候compact索引要快
5. 索引应该创建在频繁操作的列上
6. 创建更多的索引，同样也会降低查询的性能
7. 应该基于数据的特点，创建一种类型索引。如果这个索引能加快查询执行
8. 在集群资源充足的情况下，很可能没有太大必要考虑索引


  ## join
> Hive join实现有两种:
 1. map side join: 实现方式是replication join，把其中一个表复制到所有节点，这样另一个表在每个节点上面的分片就可以跟这个完整的表join了
 2. reduce side join: 实现方式是repartition join，把两份数据按照join key进行hash重分布，让每个节点处理hash值相同的join key数据，也就是做局部的join。

* **Map join**
在hive 0.11之前，使用map join的配置方法有两种：
一种直接在sql中写hint，语法是/*+MAPJOIN (tbl)*/，其中tbl就是你想要做replication的表。 
e.g.:
```
select /*+mapjoin(a) */count(*)from map_join_test ajoin map_join_test b on a.id = b.id;
```
另一种方法是设置 `hive.auto.convert.join = true`，这样hive会自动判断当前的join操作是否合适做map join，主要是找join的两个表中有没有小表。至于多大的表算小表，则是由`hive.smalltable.filesize`决定，默认25MB。
>Before release 0.11, a MAPJOIN could be invoked either through an optimizer hint:
```
select /*+ MAPJOIN(time_dim) */ count(*) from
store_sales join time_dim on (ss_sold_time_sk = t_time_sk)
```
or via auto join conversion:
```
set hive.auto.convert.join=true;
select count(*) from
store_sales join time_dim on (ss_sold_time_sk = t_time_sk)
```
The default value for hive.auto.convert.join was false in Hive 0.10.0.  Hive 0.11.0 changed the default to true ([HIVE-3297](https://issues.apache.org/jira/browse/HIVE-3297)). Note that hive-default.xml.template incorrectly gives the default as false in Hive 0.11.0 through 0.13.1.

MAPJOINs 把小表加载到内存中，建议内存的hashmap，然后跟大表进行穿透式key匹配。 主要实现如下的分工方式:
   - Local work:
1. read records via standard table scan (including filters and projections) from source on local machine
2. build hashtable in memory
3. write hashtable to local disk
4. upload hashtable to dfs
5. add hashtable to distributed cache

   - Map task:
1. read hashtable from local disk (distributed cache) into memory
2. match records' keys against hashtable
3. combine matches and write to output

   - No reduce task:
    
从Hive 0.11开始，map join 链会得到优化（`hive.auto.convert.join=true` 或者 map join hint），e.g.:
```
select /*+ MAPJOIN(time_dim, date_dim) */ count(*) from
store_sales 
join time_dim on (ss_sold_time_sk = t_time_sk) 
join date_dim on (ss_sold_date_sk = d_date_sk)
where t_hour = 8 and d_year = 2002
```
map joins优化，会尝试合并更多Map Joins。像上例一样，两个表的小维度匹配部分，需要同时满足内存要求。合并会指数的降低query的执行时间，如例中降低两次读取和写入（在job之间通过HDFS通信）为一次读取。

配置参数：
  - `hive.auto.convert.join`： 是否允许Hive优化普通join为map join，决定与输入的size. (Hive 0.11 default true)
  - `hive.auto.convert.join.noconditionaltask`: 是否允许Hive优化普通join为map join，决定与输入的size. 如果这个参数生效，在n-way join中n-1个表或者分区的总大小，小于`hive.auto.convert.join.noconditionaltask.size`,  则会直接转化为一个map join (there is no conditional task). ( default true, Hive 0.11 中添加）
  - `hive.auto.convert.join.noconditionaltask.size`, 如果`hive.auto.convert.join.noconditionaltask`没有开启，这个参数不会生效，如果开启，n-way join中n-1个表或者分区的总大小小于这个值，则这个join会直接转化为一个map join. (there is no conditional task). (默认 10MB).

**Note:**
同样的表，即使inner join能转化为map join，outer join也不能转化为map join，因为map join只能允许一个table(或分区）是streamed。对于full outer join是不能转化为map join的。 对于left 或者right outer join，只有除了需要streamed的表以外的表，能满足size配置。
> Outer joins offer more challenges. Since a map-join operator can only stream one table, the streamed table needs to be the one from which all of the rows are required. For the left outer join, this is the table on the left side of the join; for the right outer join, the table on the right side, etc. This means that even though an inner join can be converted to a map-join, an outer join cannot be converted. An outer join can only be converted if the table(s) apart from the one that needs to be streamed can be fit in the size configuration. A full outer join cannot be converted to a map-join at all since both tables need to be streamed.


* **Sort-Merge-Bucket (SMB) Map join**
>  sort-merge-bucket(SMB) join，是定义了sort和bucket表所使用的join方式。这种join方式，通过简单的合并已经排好序的表，来让这个join操作比普通的map join快。这就是map join优化。

参数：
```
set hive.auto.convert.sortmerge.join=true;
set hive.optimize.bucketmapjoin = true;
set hive.optimize.bucketmapjoin.sortedmerge = true;
```
`hive.optimize.bucketmapjoin= true` 来控制hive 执行bucket map join 了, 需要注意的是你的小表的*number_buckets* 必须是大表的倍数. 无论多少个表进行连接这个条件都必须满足.(其实如果都按照2的指数倍来分bucket)。这样数据就会按照join key做hash bucket。小表依然复制到所有节点，map join的时候，小表的每一组bucket加载成hashtable，与对应的一个大表bucket做局部join，这样每次只需要加载部分hashtable就可以了。
`set hive.optimize.bucketmapjoin.sortedmerge = true`，来控制对join key上都有序的表的join优化。同时，需要设置input format，`set hive.input.format = org.apache.hadoop.hive.ql.io.BucketizedHiveInputFormat`

* **Left Semi Join**
旧版Hive 中没有in/exist 这样的子句，所以需要将这种类型的子句转成left semi join。 left semi join 是只传递表的join key给map 阶段 , 如果key 足够小还是执行map join, 如果不是则还是common join。


   ## 数据倾斜
数据倾斜可能会发生在group过程和join过程。
* Group过程
Group过程的数据倾斜，`set hive.map.aggr=true` (默认开启)，在map端完成聚合，来优化倾斜。也就是在mapper内部，做部分的聚合，来输出更少的行，减少需要排序和分发到reducer的数据量。
Hive在尝试做此优化，不过会判断aggregation的效果，如果不能节省足够的内存，就会退回标准map过程。也就是在处理了100000 行（`hive.groupby.mapaggr.checkinterval` 控制)后，检查内存中的hash map的项，如果超过50%(`hive.map.aggr.hash.min.reduction` 控制），则认为聚合会被终止。
Hive同样会估计hash map中每一项所需要的内存，当使用的内存超过了mapper可用内存的50%（ `hive.map.aggr.hash.percentmemory` 控制），则会把flush此hash map到reducers。然而这是对行数和每行大小的估计，所以如果实际值过高，可能导致还没有flush就out of memory了。
当出现这种OOM时，可用减少`hive.map.aggr.hash.percentmemory`， 但是这个对内存增长与行数无关的数据来说，不一定是有效的。这个时候，可以使用关闭以下方法，
    1. map端聚合`set hive.map.aggr=false`，
    2. 给mapper分配更多的内存
    2. 重构query查询。利用子查询等方法，优化查询语句，e.g.:
        ```
        select count(distinct v) from tbl
        改写成
        select count(1) from (select v from tbl group by v) t.
        ```
Group过程倾斜，还可以开启`hive.groupby.skewindata=true`来改善，这个是让key随机分发到reducer，而不是同样的key分发到同一个reducer，然后reduce做聚合，做完之后再做一轮map-reduce。这个是把上面提到的map端聚合放到了reduce端，增加了reducer新的开销，大多数情况效果并不好。

* Join过程
    1. `map join`可以解决大表join小表时候的数据倾斜
    2. `skew join`是hive中对数据倾斜的一个解决方案，`set hive.optimize.skewjoin = true;`
        > 根据`hive.skewjoin.key`（默认100000）设置的数量hive可以知道超过这个值的key就是特殊key值。对于特殊的key，reduce过程直接跳过，最后再启用新的map-reduce过程来处理。
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2716069-a89009a3e062fe4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
业务数据本身的倾斜，可以从业务数据特点本身出发，通过设置reduce数量等方式，来避免倾斜


   ## Top N 问题
> order by col limit n. hive默认的order by实现只会用1个reduce做全局排序，这在数据量大的时候job运行效率非常低。hive在0.12版本引入了parallel order by，也就是通过sampling的方式实现并行（即基于TotalOrderPartitioner）。具体开关参数是`hive.optimize.sampling.orderby`。但是如果使用这个参数还是很可能碰到问题的：
   - 首先如果order by字段本身取值范围过少，会造成Split points are out of order错误。这是因为，假设job中reduce数量为r的话，那么TotalOrderPartitioner需要order by字段的取值至少要有r - 1个。那么这样一来还需要关心reduce数量，增加了开发负担，而且如果把reduce数量设的很小，优化的效果就不太明显了。
   - 其次，设置这个参数还可能造成聚会函数出错，[这个问题](https://issues.apache.org/jira/browse/HIVE-12165)只在比较新的hive版本中解决了。

`sort by col limit n` 可以解决top N问题，sort by保证每个reduce内数据有序，这样就等于是做并行排序。而limit n则保证每个reduce的输出记录数只是n（reducer内top N）。等局部top n完成之后，再起一轮job，用1个reduce做全局top n，由于数据量大大减少单个reducer也能快速完成。

# SQL整体优化
   ## Job间并行
> 对于子查询和union等情况，可以并行的执行job

`set hive.exec.parallel=true`, 默认的并行度为8（hive.exec.parallel. thread.number 控制），表示最多可以8个job并行，注意这个值的大小，避免占用太多资源。

   ## 减少Job数
通过优化查询语句（SQL）来实现减少job数目（子查询数目）的目的。


   ## 参考
1. [数据仓库中的sql性能优化（hive篇）](http://sunyi514.github.io/2013/09/01/%E6%95%B0%E6%8D%AE%E4%BB%93%E5%BA%93%E4%B8%AD%E7%9A%84sql%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%EF%BC%88hive%E7%AF%87%EF%BC%89/)
2. [Apache Hive](https://cwiki.apache.org/confluence/display/Hive/Home)
3. [Indexing in Hive](https://acadgild.com/blog/indexing-in-hive/)
4. [Map-side aggregations in Apache Hive](http://dev.bizo.com/2013/02/map-side-aggregations-in-apache-hive.html)