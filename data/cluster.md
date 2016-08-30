[首页](index)

## 添加DataNode
对于新添加的DataNode节点，需要启动datanode进程，从而将其添加入集群
1. 在新增的节点上，运行sbin/hadoop-daemon.sh start datanode即可
2. 然后在namenode通过hdfs dfsadmin -report查看集群情况
3. 最后还需要对hdfs负载设置均衡，因为默认的数据传输带宽比较低，可以设置为64M，即hdfs dfsadmin -setBalancerBandwidth 67108864即可
4. 默认balancer的threshold为10%，即各个节点与集群总的存储使用率相差不超过10%，我们可将其设置为5%
5. 然后启动Balancer，sbin/start-balancer.sh -threshold 5，等待集群自均衡完成即可

## 添加Nodemanager
由于Hadoop 2.X引入了YARN框架，所以对于每个计算节点都可以通过NodeManager进行管理，同理启动NodeManager进程后，即可将其加入集群
- 在新增节点，运行sbin/yarn-daemon.sh start nodemanager即可
在ResourceManager，通过yarn node -list查看集群情况

## 错误集
1. Journal Storage Directory (/path/of/journal) not formatted
    - Type 1:
      当你从异常信息中看到JournalNode not formatted，如果在异常中看到Journal节点都提示需要格式化JournalNode。这个时候如果是新的集群，可以重新格式化NameNode，同时JournalNode的目录也会被格式化
    - Type 2:
      如果只是其中几个Journal结点出现此异常，可以检查Journal结点相应的目录是否有权限。
      并且，从正常的Journal Node拷贝内容到异常的Journal结点
    - Type 3:
      如果是从普通的HDFS更新到HA HDFS，可以使用：
      **hdfs namenode -initializeSharedEdits**
      也就是你可以不用格式化NameNode就可以格式化你的JournalNode目录