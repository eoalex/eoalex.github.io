---
title: "Hadoop\_1.2.1\_完全分布式安装部署"
tags:
  - hadoop
id: 14
categories:
  - 大数据
date: 2015-10-26 16:50:15
---

## 1.安装前，准备三台CENTOS 6.6系统的主机或虚机,并且关闭防火墙及selinux.

## 2.按如下表格配置IP地址，修改hosts文件及本机名

<div>
<table border="1" cellspacing="1" cellpadding="1">
<tbody>
<tr>
<td>IP</td>
<td>主机名</td>
<td>节点</td>
</tr>
<tr>
<td>192.168.199.11</td>
<td>hadoop11</td>
<td>Master</td>
</tr>
<tr>
<td>102.168.199.12</td>
<td>hadoop12</td>
<td>Slave1</td>
</tr>
<tr>
<td>192.168.199.13</td>
<td>hadoop13</td>
<td>Slave2</td>
</tr>
</tbody>
</table>
</div>
修改master节点配置
[![002nJwOegy6UlNlUqCf5b&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlNlUqCf5b690.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlNlUqCf5b690.jpg)
[![002nJwOegy6UlNtbJ4j57&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlNtbJ4j57690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlNtbJ4j57690.png)
[![002nJwOegy6UlNCXNKyb2&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlNCXNKyb2690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlNCXNKyb2690.png)
同理修改Slave1，Slave2的IP地址，hosts文件及本机名。

## 3.安装ORACLE JDK

先卸载本机openJDK,使用rpm -qa|grep java查看，然后用rpm -e 卸载
从oracle网站找到最新JDK,我这选择了JDK8
http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
下载好以后解压，并移至/usr/java,如果没有可以mkdir 建立。

    #tar -xzvf jdk-8u51-linux-x64.gz
    #mv jdk1.8.0_51 /usr/java
    #vi /etc/profile
    export JAVA_HOME=/usr/java/jdk1.8.0_51
    export CLASSPATH=.:$JAVA_HOME/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    export PATH=$PATH:$JAVA_HOME/bin
    #source /etc/profile`</pre>

    ## 4.在各节点分别建立Hadoop运行帐号grid,并设置密码

    ## 5.配置SSH各节点免密码登陆。

    在各节点分别以grid用户名生成两个密钥文件，一个是私钥id_rsa,另一个是公钥id_rsa.pub
    [![002nJwOegy6UlO7nj194b&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlO7nj194b690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlO7nj194b690.png)
    [![002nJwOegy6UlMtCeGJ71&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlMtCeGJ71690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlMtCeGJ71690.png)
    在master机上把三个节点的公钥都添加至文件authorized_keys，然后分发至各节点。
    在hadoop11上
    <pre>`#cp /home/grid/.ssh/id_rsa.pub /home/grid/.ssh/authorized_keys
    #scp hadoop12:/home/grid/.ssh/id_rsa.pub pubkeys12
    #scp hadoop13:/home/grid/.ssh/id_rsa.pub pubkeys13
    #cat pubkeys12 &gt;&gt;  /home/grid/.ssh/authorized_keys
    #cat pubkeys13 &gt;&gt;  /home/grid/.ssh/authorized_keys
    #rm pubkeys12
    #rm pubkeys13
    `</pre>
    最后分发authorized_keys 到各节点
    <pre>`#scp /home/grid/.ssh/authorized_keys hadoop12:/home/grid/.ssh
    #scp /home/grid/.ssh/authorized_keys hadoop13:/home/grid/.ssh
    `</pre>

    ## 6.在Master机下载并解压Hadoop1.2.1（使用grid用户名)

    找到最近的hadoop镜像，使用wget下载1.2.1
    <pre>`#wget http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz`</pre>
    解压hadoop-1.2.1.tar.gz
    <pre>`#tar -xzvf hadoop-1.2.1.tar.gz
    #cd hadoop-1.2.1
    #mkdir tmp
    `</pre>

    ## 7.修改配置文件

    <pre>`#cd hadoop-1.2.1
    #cd conf`</pre>
    修改hadoop-env.sh
    <pre>`export JAVA_HOME=/usr/java/jdk1.8.0_51`</pre>
    修改core-site.xml
    [![002nJwOegy6UlSzzXzs1a&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlSzzXzs1a690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlSzzXzs1a690.png)
    修改hdfs-site.xml
    [![002nJwOegy6UlSBcLQ7cb&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlSBcLQ7cb690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlSBcLQ7cb690.png)
    修改mapred-site.xml
    [![002nJwOegy6UlSDlQIMf0&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlSDlQIMf0690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlSDlQIMf0690.png)
    修改masters,slaves
    [![002nJwOegy6UlRpkmdP41&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlRpkmdP41690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlRpkmdP41690.png)

    ## 8\. 分发至各Salve节点

    <pre>`#scp -r /home/grid/hadoop-1.2.1 hadoop12:/home/grid
    #scp -r /home/grid/hadoop-1.2.1 hadoop13:/home/grid
    `</pre>

    ## 9.Master机格式化namenode

    <pre>`#bin/hadoop namenode -format`</pre>
    [![002nJwOegy6UlRN6Cc101&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlRN6Cc101690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlRN6Cc101690.png)

    ## 10.启动Hadoop

    <pre>`#bin/start-all.sh

[![002nJwOegy6UlRSSYwo18&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlRSSYwo18690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlRSSYwo18690.png)

## 11.验证是否成功

Master机应该启动NameNode,SecondaryNameNode,JobTracker
Slave机应该启动DataNode,TaskTracker
[![002nJwOegy6UlRUCAcj7b&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlRUCAcj7b690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlRUCAcj7b690.png)