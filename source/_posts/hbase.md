---
title: Hbase完全分布式安装
tags:
  - Hbase
id: 103
categories:
  - 大数据
date: 2015-11-06 11:17:16
---

环境：CENTOS6.6, Hadoop 1.2.1, zookeeper 3.4.6, jdk1.8.0_51
<table>
<tbody>
<tr>
<td width="181">IP</td>
<td width="189">HOST NAME</td>
<td width="172"></td>
</tr>
<tr>
<td width="181">192.168.199.11</td>
<td width="189">Hadoop11</td>
<td width="172">Namenode</td>
</tr>
<tr>
<td width="181">192.168.199.12</td>
<td width="189">Hadoop12</td>
<td width="172">datanode</td>
</tr>
<tr>
<td width="181">192.168.199.13</td>
<td width="189">Hadoop13</td>
<td width="172">datanode</td>
</tr>
</tbody>
</table>
&nbsp;

1.安装配置Habse

1.1从镜像站点下载HBase

Hadoop 1.2.1目前对应的Hbase版本是0.98.15

[![2015-10-30_20-26-35](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_20-26-35.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_20-26-35.png)

1.2 修改/etc/profile

export HBASE_HOME=/home/grid/hbase-0.98.15-hadoop1

export PATH=$PATH:$HBASE_HOME/bin

#source /etc/profile

复制/etc/profile到hadoop12和hadoop13，并在hadoop12、hadoop13上执行

#source /etc/profile

&nbsp;

1.3修改文件$HBASE_HOME/conf/hbase-env.sh

export JAVA_HOME=/usr/java/jdk1.8.0_51

Zookeepr我们用自己安装的，所以改为false

export HBASE_MANAGES_ZK=false

&nbsp;

1.4修改文件$HBASE_HOME/conf/hbase-site.xml

&lt;configuration&gt;

&lt;property&gt;

&lt;name&gt;hbase.rootdir&lt;/name&gt;

&lt;value&gt;hdfs://hadoop11:9000/hbase&lt;/value&gt;

&lt;/property&gt;

&lt;property&gt;

&lt;name&gt;hbase.cluster.distributed&lt;/name&gt;

&lt;value&gt;true&lt;/value&gt;

&lt;/property&gt;

&lt;property&gt;

&lt;name&gt;hbase.zookeeper.quorum&lt;/name&gt;

&lt;value&gt;hadoop11,hadoop12,hadoop13&lt;/value&gt;

&lt;/property&gt;

&lt;/configuration&gt;

1.5 修改regionservers

[![2015-10-30_22-15-02](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-15-02.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-15-02.png)

1.6复制hbase 至其他两节点

#scp -r hbase-0.98.15-hadoop1 grid@hadoop12:/home/grid

#scp -r hbase-0.98.15-hadoop1 grid@hadoop13:/home/grid

1.7启动集群

首先在hadoop11上启动hadoop

#start-all.sh

[![2015-10-30_22-23-54](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-23-54.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-23-54.png)

然后启动zookeeper，分别在三台机器上执行。

#~/zookeeper-3.4.6/bin/zkServer.sh start

[![2015-10-30_22-23-33](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-23-33.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-23-33.png)

最后在hadoop11上启动hbase集群

#start-hbase.sh

[![2015-10-30_21-55-48](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_21-55-48.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_21-55-48.png)

至此Hbase集群安装成功。

2.验证测试hbase

2.1 jps

运行jps命令，我们看到hbase服务起来了。

[![2015-10-30_21-56-49](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_21-56-49.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_21-56-49.png)

2.2 Hbase shell操作

[![2015-10-30_22-10-03](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-10-03.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-10-03.png)

[![2015-10-30_22-10-20](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-10-20.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-10-20.png)

2.3 web监控

[![2015-10-30_22-31-27](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-30_22-31-27.png)](http:/blog.yaodataking.com/wp-content/uploads/2015/11/2015-10-30_22-31-27.png)