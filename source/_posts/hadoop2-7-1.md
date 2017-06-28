---
title: "Hadoop\_2.7.1\_完全分布式安装部署"
tags:
  - hadoop
id: 30
categories:
  - 大数据
date: 2015-10-27 08:01:51
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
<td>192.168.199.21</td>
<td>hadoop21</td>
<td>Master</td>
</tr>
<tr>
<td>102.168.199.22</td>
<td>hadoop22</td>
<td>Slave1</td>
</tr>
<tr>
<td>192.168.199.23</td>
<td>hadoop23</td>
<td>Slave2</td>
</tr>
</tbody>
</table>
</div>
[![002nJwOegy6UlUhMq6xcd&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlUhMq6xcd690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlUhMq6xcd690.png)

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
    #source /etc/profile
    `</pre>

    ## 4.在各节点分别建立Hadoop运行帐号grid,并设置密码

    ## 5.配置SSH免密码登陆。

    在各节点分别以grid用户名生成两个密钥文件，一个是私钥id_rsa,另一个是公钥id_rsa.pub
    <pre>`#ssh-keygen -t rsa -f ~/.ssh/id_rsa
    `</pre>
    然后在hadoop21上
    <pre>`#cp /home/grid/.ssh/id_rsa.pub /home/grid/.ssh/authorized_keys
    #scp hadoop22:/home/grid/.ssh/id_rsa.pub pubkeys22
    #scp hadoop23:/home/grid/.ssh/id_rsa.pub pubkeys23
    #cat pubkeys22 &gt;&gt; /home/grid/.ssh/authorized_keys
    #cat pubkeys23 &gt;&gt; /home/grid/.ssh/authorized_keys
    #rm pubkeys22
    #rm pubkeys23
    `</pre>
    最后分发authorized_keys 到各节点
    <pre>`scp /home/grid/.ssh/authorized_keys hadoop22:/home/grid/.ssh
    scp /home/grid/.ssh/authorized_keys hadoop23:/home/grid/.ssh
    `</pre>

    ## 6.在Master机下载并解压Hadoop2.7.1（使用grid用户名)

    找到最近的hadoop镜像，使用wget下载2.7.1
    <pre>`#wget http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-2.7.1/hadoop-2.7.1.tar.gz
    `</pre>
    解压hadoop-2.7.1.tar.gz
    <pre>`#tar -xzvf hadoop-2.7.1.tar.gz
    #cd hadoop-2.7.1
    `</pre>
    建立tmp,dfs,dfs/data,dfs/name

    ## 7.修改配置文件

    修改hadoop-env.sh
    <pre>`export JAVA_HOME=/usr/java/jdk1.8.0_51
    `</pre>
    [![002nJwOegy6UlVcJVkK62&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVcJVkK62690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVcJVkK62690.png)

    [![002nJwOegy6UlVhJ5xJ07&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVhJ5xJ07690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVhJ5xJ07690.png)

    [![002nJwOegy6UlVcQwkOca&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVcQwkOca690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVcQwkOca690.png)

    [![002nJwOegy6UlVcV90844&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVcV90844690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVcV90844690.png)

    [![002nJwOegy6UlVcXuiM0c&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVcXuiM0c690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVcXuiM0c690.png)

    ## 8\. 分发至各Salve节点

    <pre>`#scp -r /home/grid/hadoop-2.7.1 hadoop22:/home/grid
    #scp -r /home/grid/hadoop-2.7.1 hadoop23:/home/grid
    `</pre>

    ## 9.Master机格式化namenode

    <pre>`#cd /home/grid/hadoop-2.7.1
    #./bin/hdfs namenode -format

## 10.启动Hadoop

#./sbin/start-dfs.sh
[![002nJwOegy6UlVBlJbD4d&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVBlJbD4d690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVBlJbD4d690.png)
#./sbin/start-yarn.sh
[![002nJwOegy6UlVESzBLa1&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVESzBLa1690.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVESzBLa1690.png)

## 11.验证是否成功

Master机应该启动NameNode,SecondaryNameNode,ResourceManager
Slave机应该启动DataNode,NodeManager
![002nJwOegy6UlVIT0qf83&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVIT0qf83690.png)
[![002nJwOegy6UlVIXVHm6c&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVIXVHm6c690.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVIXVHm6c690.jpg)

[![002nJwOegy6UlVJ1NcMc4&amp;690](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVJ1NcMc4690.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/10/002nJwOegy6UlVJ1NcMc4690.jpg)