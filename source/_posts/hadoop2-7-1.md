---
title: "Hadoop\_2.7.1\_完全分布式安装部署"
tags:
  - hadoop
id: 30
categories:
  - 大数据
date: 2015-10-27 08:01:51
---

## 1.准备工作
1.1 安装前，准备三台CENTOS 6.6系统的主机或虚机，并且关闭防火墙及selinux。<br/>
1.2 按如下表格配置IP地址，修改hosts文件及本机名。

|IP				  |主机名   |节点  |
|:-------------|:-------|:-----|
|192.168.199.21|hadoop21|Master|
|192.168.199.22|hadoop22|Slave1|
|192.168.199.23|hadoop23|Slave2|

![002nJwOegy6UlUhMq6xcd690](/uploads/2015/10/002nJwOegy6UlUhMq6xcd690.png)

同样方法修改Slave1，Slave2的IP地址，hosts文件及本机名。

## 2.安装ORACLE JDK

先卸载本机openJDK,使用rpm -qa|grep java查看，然后用rpm -e 卸载
从oracle网站找到最新JDK，我这选择了JDK8,可以从以下地址下载。

>http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

下载好以后解压，并移至/usr/java,如果没有可以mkdir命令建立。

```
#tar -xzvf jdk-8u51-linux-x64.gz
#mv jdk1.8.0_51 /usr/java
#vi /etc/profile
	export JAVA_HOME=/usr/java/jdk1.8.0_51
	export CLASSPATH=.:$JAVA_HOME/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
	export PATH=$PATH:$JAVA_HOME/bin
#source /etc/profile
```
## 3.安装配置
### 3.1 账号设置
在各节点分别建立Hadoop运行帐号grid,并设置密码

### 3.2 配置SSH免密码登陆。
在各节点分别以grid用户名生成两个密钥文件，一个是私钥`id_rsa`,另一个是公钥`id_rsa.pub`
	
	#ssh-keygen -t rsa -f ~/.ssh/id_rsa
    
 然后在hadoop21上
    
	#cp /home/grid/.ssh/id_rsa.pub /home/grid/.ssh/authorized_keys
	#scp hadoop22:/home/grid/.ssh/id_rsa.pub pubkeys22
	#scp hadoop23:/home/grid/.ssh/id_rsa.pub pubkeys23
	#cat pubkeys22 &gt;&gt; /home/grid/.ssh/authorized_keys
	#cat pubkeys23 &gt;&gt; /home/grid/.ssh/authorized_keys
	#rm pubkeys22
	#rm pubkeys23
  
 最后分发authorized_keys文件到各节点
    
    #scp /home/grid/.ssh/authorized_keys hadoop22:/home/grid/.ssh
    #scp /home/grid/.ssh/authorized_keys hadoop23:/home/grid/.ssh

### 3.3 下载hadoop
在Master机下载并解压Hadoop2.7.1（使用grid用户名)，找到最近的hadoop镜像，使用wget下载2.7.1
    
    #wget http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-2.7.1/hadoop-2.7.1.tar.gz
    
解压hadoop-2.7.1.tar.gz
	
	#tar -xzvf hadoop-2.7.1.tar.gz
	#cd hadoop-2.7.1
    
建立tmp,dfs,dfs/data,dfs/name

### 3.4 修改配置文件

修改hadoop-env.sh

    export JAVA_HOME=/usr/java/jdk1.8.0_51

![002nJwOegy6UlVcJVkK62690](/uploads/2015/10/002nJwOegy6UlVcJVkK62690.png)

![002nJwOegy6UlVhJ5xJ07690](/uploads/2015/10/002nJwOegy6UlVhJ5xJ07690.png)

![002nJwOegy6UlVcQwkOca690](/uploads/2015/10/002nJwOegy6UlVcQwkOca690.png)

![002nJwOegy6UlVcV90844690](/uploads/2015/10/002nJwOegy6UlVcV90844690.png)

![002nJwOegy6UlVcXuiM0c690](/uploads/2015/10/002nJwOegy6UlVcXuiM0c690.png)

### 3.5 配置文件分发
将配置文件分发至各Salve节点

    #scp -r /home/grid/hadoop-2.7.1 hadoop22:/home/grid
    #scp -r /home/grid/hadoop-2.7.1 hadoop23:/home/grid

### 3.6 格式化
将Master机格式化namenode

    #cd /home/grid/hadoop-2.7.1
    #./bin/hdfs namenode -format

## 4.启动
    #./sbin/start-dfs.sh
![002nJwOegy6UlVBlJbD4d690](/uploads/2015/10/002nJwOegy6UlVBlJbD4d690.png)
	
	#./sbin/start-yarn.sh
![002nJwOegy6UlVESzBLa1690](/uploads/2015/10/002nJwOegy6UlVESzBLa1690.png)

## 5.验证

Master机应该启动NameNode，SecondaryNameNode，ResourceManager;<br>
Slave机应该启动DataNode，NodeManager
![002nJwOegy6UlVIT0qf83690](/uploads/2015/10/002nJwOegy6UlVIT0qf83690.png)

![002nJwOegy6UlVIXVHm6c690](/uploads/2015/10/002nJwOegy6UlVIXVHm6c690.jpg)

![002nJwOegy6UlVJ1NcMc4690](/uploads/2015/10/002nJwOegy6UlVJ1NcMc4690.jpg)

