[首页](index)

# HDFS 

## A. ha dfs 初始化和启动
1. 启动zookeeper集群
2. 在主Name结点上 格式化zookeeper上相应目录
  `hdfs zkfc -formatZK`
3. 格式化主NameNode, 格式化会格式化已存在的结点元数据
   `hdfs namenode -format`
4. 启动Journal Node集群
    `hadoop-daemon.sh start journalnode`
5. 启动主结点NameNode
    `hadoop-daemon.sh start namenode`
6. 格式化备NameNode
    `hdfs namenode -bootstrapStandby`
7. 启动备结点NameNode
    `hadoop-daemon.sh start namenode`
8. 两个NameNode上启动 zkfc
    `hadoop-daemon.sh start zkfc`
9. 启动所有结点的datanode
    `hadoop-daemon.sh start datanode`


## B. Balancer
 在线上的hadoop集群运维过程中，hadoop 的balance工具通常用于平衡hadoop集群中各datanode中的文件块分布，以避免出现部分datanode磁盘占用率高的问题（这问题也很有可能导致该节点CPU使用率较其他服务器高）
> The  tool moves  blocks from  highly utilized datanodes  to  poorly utilized datanodes
iteratively. In each iteration a datanode moves or receives no more than the lesser of 10G
bytes or the threshold fraction of its capacity. Each iteration runs no more than 20
minutes. At the end of each iteration, the balancer obtains updated datanodes information
from the namenode.

* 描述
-threshold 默认设置：10，参数取值范围：0-100，参数含义：判断集群是否平衡的目标参数，每一个 datanode 存储使用率和集群总存储使用率的差值都应该小于这个阀值 ，理论上，该参数设置的越小，整个集群就越平衡，但是在线上环境中，hadoop集群在进行balance时，还在并发的进行数据的写入和删除，所以有可能无法到达设定的平衡参数值。
dfs.balance.bandwidthPerSec  默认设置：1048576（1 M/S），参数含义：设置balance工具在运行中所能占用的带宽，设置的过大可能会造成mapred运行缓慢

* 脚本
```
hdfs balancer -threshold 5
或
start-balancer.sh
```
`start-balancer.sh [-threshold <threshold>]  # 启动 balancer`
`hdfs dfsadmin -setBalancerBandwidth <bandwidth in bytes per second>  # adjust the network bandwidth used by the balancer`

* 什么是balance
      rebalance的目的是为了使数据在集群中各节点的分布尽量均衡，那么，什么样的情况被认为是不均衡，又需要达到什么样的目标才算是完成了rebalance呢？

      简单来说，如果集群中没有“过载”或者“负载”的节点，则认为集群中的数据分布是均衡的，否则就是不均衡。所谓的“过载节点”是指存储使用率大于“平均存储使用率+允许偏差”的节点，“负载节点”是指存储使用率小于“平均存储使用率-允许偏差”的节点。这里又出现了几个概念，下面一一解释。

      什么是一个节点的存储使用率？它表示一个数据节点上已用空间占可用空间的百分比，所谓可用空间指的是分配给HDFS可使用的空间，并非是节点所在机器的全部硬盘空间。比如，一个数据节点，共有存储空间2T，分配给HDFS的空间为1T，已经用了600G，那么使用率就是600/1000=60%。

      将集群中各节点的存储使用率做个简单平均，就得到集群中节点的平均存储使用率。举例来说，假设有三个节点A,B,C，HDFS容量分别为2T,2T,1T,分别使用了50%，50%，10%，那么平均使用率是(50%+50%+10%)/3=36.7%，而不是(2*50%+2*50%+1*10%)/(2+2+1)=42%。

      允许偏差，是启动Rebalance功能的时候指定的一个阈值，也是一个百分比，如果没有指定则默认为是10%，表示允许单个节点的存储使用率与集群中各节点平均存储使用率之间有10%的偏差。

      Rebalance过程可以指定多次，每次可以指定不同的允许偏差值，以此来逐次渐进达到一个合理的数据均衡分布，同时又不至于使得Rebalance过程持续时间过长，影响集群的正常使用。

## C. Decommission & Recommission
* Decommision
    1.  配置 （在NameNode机器上）
```
     <property>
           <name>dfs.hosts.exclude</name>
           <value>/home/hadoop/env/conf/exclude-hosts</value>
     </property>
``` 
    或者 使用默认的 <HADOOP_CONF_DIR>/dfs.exclude 文件
    2. 在NameNode机器上， exclude-hosts中写入需要decommission的结点
        > On the NameNode host machine, edit the <HADOOP_CONF_DIR>/dfs.exclude
 file and add the list of DataNodes hostnames (separated by a newline character).
    3. 执行 
        > Update the NameNode with the new set of excluded DataNodes. On the NameNode host machine, execute the following command:
```
su <HDFS_USER> 
hdfs dfsadmin -refreshNodes
```  
    4. 在NameNode Web UI中check **Decommission In Progress** 。当结点状态都变成 **Decommissioned**，就可以shut down这些结点

    5. 如果集群配置了 dfs.include file 或者 在slaves文件中，把Decommissioned结点从其中删除，然后执行:
```
su <HDFS_USER> 
hdfs dfsadmin -refreshNodes
```  
      

# Yarn

## A. Web UI 任务时间
默认情况，显示的是UTC时间
* 修改：
查看 hadoop-2.6.3/share/hadoop/yarn/hadoop-yarn-common-2.6.3.jar!/webapps/static/yarn.dt.plugins.js
 脚本里面的 renderHadoopDate方法，修改Date格式化输出的方法。

```
- return new Date(parseInt(data)).toUTCString();
+ return new Date(parseInt(data)).toString();
```
修改后，重启yarn.


## B. 更改yarn fair schedule queue
* 修改fair-scheduler.xml
* yarn rmadim -refreshQueues

## C. 资源队列使用
* 配置
    - TEZ   (tez-site.xml)
```
<property>
    <name>tez.queue.name</name>
    <value>operations</value>
</property>
``` 

    - MR  (mapred-site.xml)
```
<property>
    <name>mapred.job.queue.name</name>
    <value>operations</value>
</property>
```

## D. Decommission
   1. 配置:
```
yarn.resourcemanager.nodes.exclude-path (yarn-site.xml)
或者
<HADOOP_CONF_DIR>/yarn.exclude
增加需要退伍的结点
```
    如果配置了 `<HADOOP_CONF_DIR>/yarn.include`， 把对应结点删除

   2. 执行:
```
su <YARN_USER>
yarn rmadmin -refreshNodes
```

### E. 修改yarn资源配置
   1. **yarn.scheduler.maximum-allocation-mb**:
```
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>10240</value>
    </property>
```
    修改后重启yarn




* 
持续更新中……