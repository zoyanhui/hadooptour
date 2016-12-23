[Hive首页](hive:hive-index)

## 最佳的复制一个partitioned表的步骤：
1. 创建新的目标，跟旧表一样的schema. 如：
    create table new_xx like xx;
2. 使用 hadoop fs -cp 把旧表所有的分区文件，拷贝到目标表的文件夹。
3. 运行 MSCK REPAIR TABLE new_xx. 
这样就可以完成一个partition表的复制


## 应对Load Data时，分隔符在field中出现
对于TextFormat的hive表，当文本格式的数据，每列的分隔符是 逗号‘,'，而其中一列中的数据也包含逗号的时候，直接load会造成列的分割混乱。 这个时候， 可以使用escaped来解决这个问题：

1. create table 中指定  `ESCAPED BY`， 指定转义符，如下使用'\'作为转义符
```
create teable 
……
ROW FORMAT DELIMITED FIELDS TERMINATED BY "," ESCAPED BY '\\';  
……
```
对于已经存在的表，可以增加`escape.delim`：
```
ALTER TABLE XXXX   
set serde 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES ('escape.delim'='\\');
```
2. 文本文件中，对列中含有','的， 替换为 '\,'，使用转义。