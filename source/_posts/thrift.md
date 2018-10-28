---
title: 'Thrift 入门:安装及示列'
tags:
  - Thrift
id: 229
categories:
  - 大数据
date: 2015-11-22 22:00:42
---

## 1.简介

Thrift是一个软件框架，用来进行可扩展且跨语言的服务的开发。它结合了功能强大的软件堆栈和代码生成引擎，以构建在 C++, Java, Python, PHP, Ruby, Erlang, Perl, Haskell, C#, Cocoa, JavaScript, Node.js, Smalltalk, and OCaml 这些编程语言间无缝结合的、高效的服务。

## 2.环境

环境：CENTOS6.6, Hadoop 1.2.1, zookeeper 3.4.6, jdk1.8.0_51，Hbase 0.98.15


|IP|Host Name|Function|
|:--|:--|:--|
|192.168.199.11|Hadoop11|namenode,zookeeper,hbase|
|192.168.199.12|Hadoop12|datanode,zookeeper,hbase|
|192.168.199.13|Hadoop13|datanode,zookeeper,hbase|

## 3.安装

### 3.1 安装开发工具
	
	#yum -y groupinstall "Development Tools"
![2015-11-22_18-34-31](/uploads/2015/11/2015-11-22_18-34-31.jpg)

### 3.2 更新autoconf

	#wget http://ftp.gnu.org/gnu/autoconf/autoconf-2.69.tar.gz
![2015-11-22_18-45-14](/uploads/2015/11/2015-11-22_18-45-14.jpg)
	
	#tar xvf autoconf-2.69.tar.gz
	#cd autoconf-2.69
	#./configure --prefix=/usr
	#make
	#make install
![2015-11-22_18-47-06](/uploads/2015/11/2015-11-22_18-47-06.jpg)
	
### 3.3 更新automake
	
	#cd ..
	#wget http://ftp.gnu.org/gnu/automake/automake-1.14.tar.gz
![2015-11-22_18-52-21](/uploads/2015/11/2015-11-22_18-52-21.jpg)

	#tar xvf automake-1.14.tar.gz
	#cd automake-1.14
	#./configure --prefix=/usr
	#make
	#make install
![2015-11-22_18-53-41](/uploads/2015/11/2015-11-22_18-53-41.jpg)

### 3.4 更新bison
	
	#cd ..
	#wget http://ftp.gnu.org/gnu/bison/bison-2.5.1.tar.gz
![2015-11-22_18-56-13](/uploads/2015/11/2015-11-22_18-56-13.jpg)

	#tar xvf bison-2.5.1.tar.gz
	#cd bison-2.5.1
	#./configure --prefix=/usr
	#make
	#make install
![2015-11-22_18-57-54](/uploads/2015/11/2015-11-22_18-57-54.jpg)

### 3.5 安装C++ lib
	
	#cd ..
	#yum -y install libevent-devel zlib-devel openssl-devel
![2015-11-22_18-59-06](/uploads/2015/11/2015-11-22_18-59-06.jpg)

### 3.6 安装boost 1.53.0
	
	#wget http://sourceforge.net/projects/boost/files/boost/1.53.0/boost_1_53_0.tar.gz
![2015-11-22_19-07-20](/uploads/2015/11/2015-11-22_19-07-20.jpg)
	
	#tar xvf boost_1_53_0.tar.gz
	#cd boost_1_53_0
	#./bootstrap.sh
	#./b2 install

### 3.7 安装Thrift
	
	#git clone https://git-wip-us.apache.org/repos/asf/thrift.git
![2015-11-22_19-28-54](/uploads/2015/11/2015-11-22_19-28-54.jpg)

	#cd thrift
	#git checkout -b thrift-0.9.1 0.9.1
	#./bootstrap.sh
	#./configure
	#make
	#make install
![2015-11-22_20-27-10](/uploads/2015/11/2015-11-22_20-27-10.jpg)

## 4.示例

### 4.1 Hbase的Thrift接口

1)启动Hadoop、Zookeeper、HBase 
(关于HBase的分布式安装参见另一篇博客[Hbase完全分布式安装](/2015/11/hbase完全分布式安装/))
在Hadoop11启动
	
	#start-all.sh
在Hadoop11,12,13分别启动
	
	#~/zookeeper-3.4.6/bin/zkServer.sh start
在Hadoop11启动
	
	#start-hbase.sh
2)启动Hbase thrift接口
	
	#hbase thrift -p 9090 start
![2015-11-22_21-05-16](/uploads/2015/11/2015-11-22_21-05-16.jpg)

3)生成Hbase.py
下载源码
	
	#wget http://mirrors.aliyun.com/apache/hbase/0.98.16/hbase-0.98.16-src.tar.gz
	#thrift -gen py /home/grid/hbase-0.98.16/hbase-thrift/src/main/resources/org/apache/hadoop/hbase/thrift/Hbase.thrift
	#cp -r /home/grid/gen-py/hbase/ /usr/lib64/python2.6/site-packages/

![2015-11-22_20-55-05](/uploads/2015/11/2015-11-22_20-55-05.jpg)

4)Python代码测试

```Python
#!/usr/bin/python
#coding=utf-8
import sys
sys.path.append('/usr/lib64/python2.6/site-packages/')
from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol
from hbase import Hbase
from hbase.ttypes import *
transport = TSocket.TSocket('192.168.199.11', 9090)
transport = TTransport.TBufferedTransport(transport)
protocol = TBinaryProtocol.TBinaryProtocol(transport)
client = Hbase.Client(protocol)
transport.open()
print(client.getTableNames())
```

执行这段代码，我们看到Python代码正确获取了Hbase的表名
![2015-11-22_21-26-25](/uploads/2015/11/2015-11-22_21-26-25.jpg)