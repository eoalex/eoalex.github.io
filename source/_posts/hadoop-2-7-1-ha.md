---
title: Hadoop 2.7.1 实现HA集群部署
tags:
  - HA
  - hadoop
  - zookeeper
id: 61
categories:
  - 大数据
date: 2015-10-27 11:14:52
---

## 1.0 简述

Hadoop 2.0中的HDFS增加了两个重大特性，HA和Federation，HA即为High Availability，用于解决NameNode单点故障问题，该特性通过热备的方式为主NameNode提供一个备用者，一旦主NameNode出现故障，可以迅速切换至备NameNode，从而实现不间断对外提供服务。Federation即为“联邦”，该特性允许一个HDFS集群中存在多个NameNode同时对外提供服务，这些NameNode分管一部分目录（水平切分），彼此之间相互隔离，但共享底层的DataNode存储资源。本文就讲如何安装Hadoop 2.7.1 实现HA集群。联邦的安装部署我们将在下次讲。

## 1.1 主机列表

<table width="553">
<tbody>
<tr>
<td width="109">IP</td>
<td width="156">NAME</td>
<td width="288">Function</td>
</tr>
<tr>
<td width="109">192.168.199.24</td>
<td width="156">hadoop24.hadoop.com</td>
<td width="288">主namenode, zookeeper,journalnode,zkfc, ResourceManager</td>
</tr>
<tr>
<td width="109">192.168.199.25</td>
<td width="156">hadoop25.hadoop.com</td>
<td width="288">备namenode,zookeepaer,journalnode,zkfc</td>
</tr>
<tr>
<td width="109">192.168.199.26</td>
<td width="156">hadoop26.hadoop.com</td>
<td width="288">datanode, ,zookeepaer,journalnode, NodeManager</td>
</tr>
<tr>
<td width="109">192.168.199.27</td>
<td width="156">hadoop27.hadoop.com</td>
<td width="288">Datanode, NodeManager</td>
</tr>
<tr>
<td width="109">192.168.199.28</td>
<td width="156">hadoop28.hadoop.com</td>
<td width="288">Datanode, NodeManager</td>
</tr>
</tbody>
</table>

## 1.2 下载Zookeeper并安装配置

    #tar -xzvf zookeeper-3.4.6.tar.gz
    #cd zookeeper-3.4.6/conf
    #cp zoo_sample.cfg zoo.cfg
    #vi zoo.cfg
    dataDir=/home/grid/zookeeper-3.4.6/data
    server.1=hadoop24.hadoop.com:2888:3888
    server.2=hadoop25.hadoop.com:2888:3888
    server.3=hadoop26.hadoop.com:2888:3888
    #cd ..
    #mkdir data
    #cd ..`</pre>
    生成分发脚本，并执行
    <pre>`#cat zookeeper-3.4.6/conf/zoo.cfg | grep server |sed 's/[^*]*=//g' |sed 's/:[^*]*//g' |awk '{print "scp -rp ./zookeeper-3.4.6 grid@"$1":/home/grid"}' | sed '1d' &gt; zoocopy
    #./zoocopy
    `</pre>
    [![20151027001](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027001.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027001.png)

    在hadoop24上
    <pre>`#echo "1" &gt; ~/zookeeper-3.4.6/data/myid`</pre>
    在hadoop25
    <pre>`#echo "2" &gt; ~/zookeeper-3.4.6/data/myid`</pre>
    在hadoop26
    <pre>`#echo "3" &gt; ~/zookeeper-3.4.6/data/myid`</pre>
    在每台上执行启动
    <pre>`#~/zookeeper-3.4.6/bin/zkServer.sh start`</pre>
    [![20151027002](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027002.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027002.png)

    查看状态
    <pre>`#~/zookeeper-3.4.6/bin/zkServer.sh status`</pre>
    [![20151027003](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027003.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027003.png)

    可以看到一台leader，两台follower

    ## 1.3 安装配置hadoop 2.7.1

    <pre>`#tar -xzvf hadoop-2.7.1.tar.gz
    #cd hadoop-2.7.1
    #mkdir name data tmp journal
    #cd etc/Hadoop
    `</pre>
    设置JAVA_HOME路径
    <pre>`#vi hadoop-env.sh
    export JAVA_HOME=/usr/java/jdk1.8.0_51
    #vi yarn-env.sh
    export JAVA_HOME=/usr/java/jdk1.8.0_51`</pre>
    编辑slaves文件
    <pre>`#vi slaves
    hadoop26.hadoop.com
    hadoop27.hadoop.com
    hadoop28.hadoop.com
    `</pre>
    <pre>`#vi core-site.xml
    &lt;configuration&gt;
    &lt;property&gt;
    &lt;name&gt;fs.defaultFS&lt;/name&gt;
    &lt;value&gt;hdfs://mycluster&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;hadoop.tmp.dir&lt;/name&gt;
    &lt;value&gt;file:/home/grid/hadoop-2.7.1/tmp&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;io.file.buffer.size&lt;/name&gt;
    &lt;value&gt;131702&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;hadoop.proxyuser.hadoop.hosts&lt;/name&gt;
    &lt;value&gt;*&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;hadoop.proxyuser.hadoop.groups&lt;/name&gt;
    &lt;value&gt;*&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;ha.zookeeper.quorum&lt;/name&gt;
    &lt;value&gt;192.168.199.24:2181,192.168.199.25:2181,192.168.199.26:2181&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;ha.zookeeper.session-timeout.ms&lt;/name&gt;
    &lt;value&gt;1000&lt;/value&gt;
    &lt;/property&gt;
    &lt;/configuration&gt;

    #vi mapred-site.xml
    &lt;configuration&gt;
    &lt;property&gt;
    &lt;name&gt;mapreduce.framework.name&lt;/name&gt;
    &lt;value&gt;yarn&lt;/value&gt;
    &lt;/property&gt;
    &lt;/configuration&gt;

    #vi yarn-site.xml
    &lt;configuration&gt;
    &lt;property&gt;
    &lt;name&gt;yarn.nodemanager.aux-services&lt;/name&gt;
    &lt;value&gt;mapreduce_shuffle&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;yarn.nodemanager.auxservices.mapreduce.shuffle.class&lt;/name&gt;
    &lt;value&gt;org.apache.hadoop.mapred.ShuffleHandler&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;yarn.resourcemanager.address&lt;/name&gt;
    &lt;value&gt;hadoop24.hadoop.com:8032&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;yarn.resourcemanager.scheduler.address&lt;/name&gt;
    &lt;value&gt;hadoop24.hadoop.com:8030&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;yarn.resourcemanager.resource-tracker.address&lt;/name&gt;
    &lt;value&gt;hadoop24.hadoop.com:8031&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;yarn.resourcemanager.admin.address&lt;/name&gt;
    &lt;value&gt;hadoop24.hadoop.com:8033&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;yarn.resourcemanager.webapp.address&lt;/name&gt;
    &lt;value&gt;hadoop24.hadoop.com:8088&lt;/value&gt;
    &lt;/property&gt;
    &lt;/configuration&gt;

    #vi hdfs-site.xml
    &lt;configuration&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.namenode.name.dir&lt;/name&gt;
    &lt;value&gt;file:/home/grid/hadoop-2.7.1/name&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.datanode.data.dir&lt;/name&gt;
    &lt;value&gt;file:/home/grid/hadoop-2.7.1/data&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.replication&lt;/name&gt;
    &lt;value&gt;3&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.webhdfs.enabled&lt;/name&gt;
    &lt;value&gt;true&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.permissions&lt;/name&gt;
    &lt;value&gt;false&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.permissions.enabled&lt;/name&gt;
    &lt;value&gt;false&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.nameservices&lt;/name&gt;
    &lt;value&gt;mycluster&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.ha.namenodes.mycluster&lt;/name&gt;
    &lt;value&gt;nn1,nn2&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.namenode.rpc-address.mycluster.nn1&lt;/name&gt;
    &lt;value&gt;hadoop24.hadoop.com:9000&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.namenode.rpc-address.mycluster.nn2&lt;/name&gt;
    &lt;value&gt;hadoop25.hadoop.com:9000&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.namenode.servicerpc-address.mycluster.nn1&lt;/name&gt;
    &lt;value&gt;hadoop24.hadoop.com:53310&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.namenode.servicerpc-address.mycluster.nn2&lt;/name&gt;
    &lt;value&gt;hadoop25.hadoop.com:53310&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.namenode.http-address.mycluster.nn1&lt;/name&gt;
    &lt;value&gt;hadoop24.hadoop.com:50070&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.namenode.http-address.mycluster.nn2&lt;/name&gt;
    &lt;value&gt;hadoop25.hadoop.com:50070&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.namenode.shared.edits.dir&lt;/name&gt;
    &lt;value&gt;qjournal://hadoop24.hadoop.com:8485;hadoop25.hadoop.com:8485;hadoop26.hadoop.com:8485/mycluster&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.client.failover.proxy.provider.mycluster&lt;/name&gt;
    &lt;value&gt;org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.ha.fencing.methods&lt;/name&gt;
    &lt;value&gt;sshfence&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.ha.fencing.ssh.private-key-files&lt;/name&gt;
    &lt;value&gt;/home/grid/.ssh/id_rsa&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.ha.fencing.ssh.connect-timeout&lt;/name&gt;
    &lt;value&gt;30000&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.journalnode.edits.dir&lt;/name&gt;
    &lt;value&gt;/home/grid/hadoop-2.7.1/journal&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.ha.automatic-failover.enabled&lt;/name&gt;
    &lt;value&gt;true&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;ha.failover-controller.cli-check.rpc-timeout.ms&lt;/name&gt;
    &lt;value&gt;60000&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;ipc.client.connect.timeout&lt;/name&gt;
    &lt;value&gt;60000&lt;/value&gt;
    &lt;/property&gt;
    &lt;property&gt;
    &lt;name&gt;dfs.image.transfer.bandwidthPerSec&lt;/name&gt;
    &lt;value&gt;4194304&lt;/value&gt;
    &lt;/property&gt;
    &lt;/configuration&gt;
    `</pre>
    生成分发脚本
    <pre>`#cat zoocopy|sed 's/zookeeper-3.4.6/hadoop-2.7.1/g' &gt; tmphadoop
    #cat ./hadoop-2.7.1/etc/hadoop/slaves |awk '{print "scp -rp ./hadoop-2.7.1 grid@"$1":/home/grid"}' &gt;&gt; tmphadoop
    #sort -n tmphadoop | uniq &gt; hadoopcopy
    #rm -r tmphadoop
    #cat hadoopcopy
    #./hadoopcopy
    `</pre>
    [![20151027004](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027004.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027004.png)

    创建znode
    <pre>`#~/zookeeper-3.4.6/bin/zkServer.sh start
    #~/hadoop-2.7.1/bin/hdfs zkfc -formatZK
    `</pre>
    验证
    <pre>`#~/zookeeper-3.4.6/bin/zkCli.sh`</pre>
    [![20151027005](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027005.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027005.png)

    在hadoop24,hadoop25,hadoop26启动journalnode
    <pre>`#~/hadoop-2.7.1/sbin/hadoop-daemon.sh start journalnode`</pre>
    [![20151027006](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027006.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027006.png)

    在主namenode节点(Hadoop24.hadoop.com)上格式化Namenode
    <pre>`#~/hadoop-2.7.1/bin/hadoop namenode –format`</pre>
    [![20151027007](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027007.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027007.png)

    在主namenode节点(Hadoop24.hadoop.com)启动namenode进程
    <pre>`#~/hadoop-2.7.1/sbin/hadoop-daemon.sh start namenode`</pre>
    [![20151027008](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027008.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027008.png)

    在备namenode节点(Hadoop25.hadoop.com)同步元数据
    <pre>`#~/hadoop-2.7.1/bin/hdfs namenode -bootstrapStandby`</pre>
    启动备NameNode节点(Hadoop25.hadoop.com)
    <pre>`#~/hadoop-2.7.1/sbin/hadoop-daemon.sh start namenode`</pre>
    在两个namenode节点都执行以下命令来配置自动故障转移：

    在NameNode节点上安装和运行ZKFC
    <pre>`#~/hadoop-2.7.1/sbin/hadoop-daemon.sh start zkfc`</pre>
    [![20151027008](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027008.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027008.png)

    [![20151027009](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027009.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027009.png)

    启动datanode(hadoop26,hadoop27,hadoop28)
    <pre>`#~/hadoop-2.7.1/sbin/hadoop-daemons.sh start datanode`</pre>
    [![20151027010](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027010.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027010.png)

    启动yarn
    <pre>`#~/hadoop-2.7.1/sbin/start-yarn.sh`</pre>
    [![20151027011](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027011.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027011.png)

    主namenode节点上active

    [![20151027012](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027012.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027012.png)

    备namenode节点上standby

    [![20151027013](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027013.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027013.png)

    ## 1.4 测试主备namenode节点是否工作

    在主namenode机器上通过jps命令查找到namenode的进程号，然后通过kill -9的方式杀掉进程，观察另一个namenode节点是否会从状态standby变成active状态

    [![20151027015](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027015.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027015.png)

    Kill进程后,主节点不能访问

    [![20151027016](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027016.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027016.png)

    备节点已变为active

    [![20151027017](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027017.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027017.png)

    测试备节点能否继续工作

    在备节点上测试
    <pre>`#~/hadoop-2.7.1/bin/hadoop fs -ls /
    #~/hadoop-2.7.1/bin/hadoop fs -put /home/grid/test.txt /
    #~/hadoop-2.7.1/bin/hadoop fs -cat /test.txt
    #~/hadoop-2.7.1/bin/hadoop jar ~/hadoop-2.7.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar wordcount /test.txt /out
    #~/hadoop-2.7.1/bin/hadoop fs -ls /out
    #~/hadoop-2.7.1/bin/hadoop fs -cat /out/part-r-00000

[![20151027018](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027018.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/20151027018.png)

至此在Hadoop2.7.1集群上成功实现HA