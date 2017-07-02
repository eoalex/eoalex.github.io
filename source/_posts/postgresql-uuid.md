---
title: postgresql使用UUID功能
tags:
  - postgresql
  - uuid
id: 405
categories:
  - 数据库
date: 2016-01-01 16:51:06
---

## 1. UUID介绍

### 1.1 什么是UUID?

UUID是Universally Unique Identifier的缩写，它是在一定的范围内（从特定的名字空间到全球）唯一的机器生成的标识符。UUID具有以下涵义：
1.1.1	经由一定的算法机器生成
为了保证UUID的唯一性，规范定义了包括网卡MAC地址、时间戳、名字空间（Namespace）、随机或伪随机数、时序等元素，以及从这些元素生成UUID的算法。UUID的复杂特性在保证了其唯一性的同时，意味着只能由计算机生成。
1.1.2	非人工指定，非人工识别
UUID是不能人工指定的，除非你冒着UUID重复的风险。UUID的复杂性决定了“一般人“不能直接从一个UUID知道哪个对象和它关联。
1.1.3	在特定的范围内重复的可能性极小
UUID的生成规范定义的算法主要目的就是要保证其唯一性。但这个唯一性是有限的，只在特定的范围内才能得到保证，这和UUID的类型有关,UUID是16字节128位长的数字，通常以36字节的字符串表示，示例如下：3F2504E0-4F89-11D3-9A0C-0305E82C3301其中的字母是16进制表示，大小写无关。GUID（Globally Unique Identifier）是UUID的别名；但在实际应用中，GUID通常是指微软实现的UUID。

### 1.2 UUID的版本

UUID具有多个版本，每个版本的算法不同，应用范围也不同。首先是一个特例－－Nil UUID－－通常我们不会用到它，它是由全为0的数字组成，如下：00000000-0000-0000-0000-000000000000
1.2.1	*UUID版本1：基于时间的UUID*
基于时间的UUID通过计算当前时间戳、随机数和机器MAC地址得到。由于在算法中使用了MAC地址，这个版本的UUID可以保证在全球范围的唯一性。但与此同时，使用MAC地址会带来安全性问题，这就是这个版本UUID受到批评的地方。如果应用只是在局域网中使用，也可以使用退化的算法，以IP地址来代替MAC地址－－Java的UUID往往是这样实现的（当然也考虑了获取MAC的难度）。
1.2.2	*UUID版本2：DCE安全的UUID*
DCE（Distributed Computing Environment）安全的UUID和基于时间的UUID算法相同，但会把时间戳的前4位置换为POSIX的UID或GID。这个版本的UUID在实际中较少用到。
1.2.3	*UUID版本3：基于名字的UUID（MD5）*
基于名字的UUID通过计算名字和名字空间的MD5散列值得到。这个版本的UUID保证了：相同名字空间中不同名字生成的UUID的唯一性；不同名字空间中的UUID的唯一性；相同名字空间中相同名字的UUID重复生成是相同的。
1.2.4	*UUID版本4：随机UUID*
根据随机数，或者伪随机数生成UUID。这种UUID产生重复的概率是可以计算出来的，但随机的东西就像是买彩票：你指望它发财是不可能的，但狗屎运通常会在不经意中到来。
1.2.5 *UUID版本5：基于名字的UUID（SHA1）*
和版本3的UUID算法类似，只是散列值计算使用SHA1（Secure Hash Algorithm 1）算法。

## 2. Postgresql的UUID实现

默认安装的Postgresql是不带UUID函数的，但是我们可以用源代码编译安装它。这里我们假定postgresql原来已经安装，我们要增加UUID功能，如果是全新安装postgresql，安装时加上相应的选项即可。

### 2.1 安装UUID库

	#yum install uuid-devel uuid
![2016-01-01_13-20-46](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-01_13-20-46.jpg)

	#rpm -ql uuid
![2016-01-01_16-17-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-01_16-17-13.jpg)

我们可以知道UUID库安装在/usr/lib64,在后面安装postgresql动态库uuid-ossp.so时会用到

### 2.2	编译postgresql的uuid动态库

假定postgresql原来安装在/usr/local/pgsql目录。
进入postgres源代码根目录(如何下载postgres源代码,可参见我的另一篇博文[Postgresql搭建eclipse开发环境](/2015/11/28/centos-6-5-postgresql-eclipse/)
	
	#./configure --prefix=/usr/local/pgsql --with-ossp-uuid --with-libraries=/usr/lib64
进入uuid-ossp编译
	
	#cd contrib/uuid-ossp/
	#make
	#make install
![2016-01-01_15-18-59](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-01_15-18-59.jpg)

查看动态库是否安装成功至/usr/local/pgsql
	
	#ll /usr/local/pgsql/lib/uuid*
	#ll /usr/local/pgsql/share/extension/
![2016-01-01_15-20-41](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-01_15-20-41.jpg)

	#chown postgres:postgres -R /usr/local/pgsql

### 2.3	安装UUID函数

启动postgres
	
	#pg_ctl -D /usr/local/pgsql/data -l /usr/local/pgsql/data/log start
	#psql
使用下面sql语句安装UUID函数
	
	create extension "uuid-ossp";
	select extname,extowner,extnamespace,extrelocatable,extversion from pg_extension;

![2016-01-01_15-37-46](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-01_15-37-46.jpg)

至此UUID函数成功安装。

## 3.测试

我们分别测试uuid的不同版本

	select uuid_generate_v1();
	select uuid_generate_v1mc();

![2016-01-01_15-36-41](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-01_15-36-41.jpg)

	select uuid_generate_v3(uuid_ns_url(), 'http://blog.yaodataking.com');
	select uuid_generate_v4();
	select uuid_generate_v5(uuid_ns_url(), 'http://blog.yaodataking.com');

![2016-01-01_15-37-02](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-01_15-37-02.jpg)

关于postgresql uuid的更多信息，我们可以参见官网说明
http://www.postgresql.org/docs/current/static/uuid-ossp.html