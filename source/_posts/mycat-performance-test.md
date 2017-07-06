---
title: 基于Docker容器的Mycat性能测试案例
tags:
  - Docker
  - Mycat
  - Mysql
id: 756
categories:
  - 数据库
date: 2016-02-25 22:53:16
---
## 1. 简介
本文我们将基于Docker容器的Mycat做性能测试。测试要求是：
创建一个 person表，主键为Id，hash方式分片，主键自增（采用数据库方式），当自增的step分别为10,100,1万的三种情况下，对此表做性能测试。
person表结构如下
Id，主键，Mycat自增主键
name,字符串，16字节最长
school,毕业学校，数字，1-1000范围，是学校编号
age，年龄，18-60
addr,地址，32字节，建议为 gz-tianhe（城市-地区两级 枚举的仿真数据）
zcode,邮编，
birth，生日，为日期类型， 1980到2010年之间随机的日期
score，得分，0-100分
测试环境我们这里采用一台虚机，三个Docker容器，虚机上运行测试工具，一个Docker容器运行Myat Server，二个Docker容器运行MySQL
测试工具我们采用Mycat标准测试工具。
## 2. 准备工作
### 2.1 启动容器
	
	#docker start mysqlsrv101
	#docker start mysqlsrv102
	#docker start mycat
关于这三个容器的安装详见我的另一篇博文[基于Mycat的MySQL主从读写分离及自动切换的docker实现](／2016/01/17/mycat-mysql-docker-sample1/)。

### 2.2 测试工具下载
我们从Mycat的github[下载](https://github.com/MyCATApache/Mycat-download),目前最新版1.5
![2016-02-25_22-00-04](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_22-00-04.jpg)

### 2.3 定义表
person表按要求定义如下

    create table person 
    (
    Id bigint not null primary key AUTO_INCREMENT,
    name varchar(16) ,
    school int,
    age int,
    addr varchar(32),
    zcode int,
    birth date,
    score int
    ) AUTO_INCREMENT = 1;
mycat_sequence表定义如下，mycat_sequence表是用来存放全局序列号的

    CREATE TABLE mycat_sequence(
    name VARCHAR(50) NOT NULL,
    current_value INT NOT NULL,
    increment INT NOT NULL DEFAULT 100, 
    PRIMARY KEY(name)
    ) ENGINE=InnoDB;
## 3. 配置文件修改
在mycat容器conf目录下
### 3.1 修改schema.xml
修改schema.xml,增加person表，自动增长设为true，增加mycat_sequence表为存放全局序列号.
![2016-02-25_22-41-21](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_22-41-21.jpg)
### 3.2 修改rule.xml
修改rule.xml ，这里我们对addr使用一致性hash算法分区。
![2016-02-25_22-42-59](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_22-42-59.jpg)

![2016-02-25_22-43-40](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_22-43-40.jpg)
### 3.3 修改server.xml
修改server.xml,在system下添加,1表示使用数据库方式，0表示本地文件方式。这里我们采用数据库方式。
![2016-02-25_22-45-42](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_22-45-42.jpg)
### 3.4 修改 sequnence_db_conf.properties
修改sequence_db_conf.properties,添加要给序号的表, 并指定分区。

    #sequence stored in datanode
    PERSON=dn1

### 3.5 建立SQL语句
在测试工具bin目录下，建立insert_person.sql文件

    total=100000
    sql=insert into person(Id,name,school,age,addr,zcode,birth,score) values(next value for MYCATSEQ_PERSON,'${char([a-f,0-9]8:16)}','${int(1-1000)}','${int(18-60)}','${enum(gz-tianhe,sh-huangpu,sz-baoan)}','${int(100000-90000)}','${date(yyyyMMdd-[1980-2010]y)}','${int(0-100)}')
    
![2016-02-25_22-18-19](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_22-18-19.jpg)

以上工作完成，我们进入mycat 9066管理端，做 reload @@config_all;使刚才配置生效。
![2016-02-24_16-19-35](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-24_16-19-35.jpg)
## 4. 其他设置
### 4.1 创建表
进入mycat 8066 端口，执行2.3 步骤语句创建person表及mycat_sequence表.
### 4.2 创建函数
进入mysql101客户端,执行创建三个函数。
    
    DROP FUNCTION IF EXISTS `mycat_seq_currval`;
    DELIMITER ;;
    CREATE FUNCTION `mycat_seq_currval`(seq_name VARCHAR(50)) RETURNS varchar(64) CHARSET utf8
    DETERMINISTIC
    BEGIN
    DECLARE retval VARCHAR(64);
    SET retval="-999999999,null";
    SELECT concat(CAST(current_value AS CHAR),",",CAST(increment AS CHAR)) INTO retval FROM mycat_sequence WHERE name = seq_name;
    RETURN retval;
    END
    ;;
    DELIMITER;

    DROP FUNCTION IF EXISTS `mycat_seq_setval`;
    DELIMITER;;
    CREATE FUNCTION `mycat_seq_setval`(seq_name VARCHAR(50),value INTEGER) RETURNS varchar(64) CHARSET utf8
    DETERMINISTIC
    BEGIN
    UPDATE mycat_sequence SET current_value = value WHERE name = seq_name;
    RETURN mycat_seq_currval(seq_name);
    END;;
    DELIMITER;

    DROP FUNCTION IF EXISTS `mycat_seq_nextval`;
    DELIMITER ;;
    CREATE FUNCTION `mycat_seq_nextval`(seq_name VARCHAR(50)) RETURNS varchar(64) CHARSET utf8
    DETERMINISTIC
    BEGIN
    UPDATE mycat_sequence SET current_value = current_value + increment WHERE name = seq_name;
    RETURN mycat_seq_currval(seq_name);
    END
    ;;
    DELIMITER;
## 5. 测试步骤
### 5.1 测试步长10
我们先开始测试步长10
    进入mycat 8066 端口, 执行下面语句

    DELETE FROM mycat_sequence;
    INSERT INTO mycat_sequence(name,current_value,increment) VALUES ('PERSON', 0, 10);
    
在mycat测试工具bin目录,执行

    ./test_stand_insert_perf.sh jdbc:mysql://localhost:8066/TESTDB test test 50  "file=person-insert.sql"

最后结果如下
![2016-02-25_20-44-29](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_20-44-29.jpg)

我们看到10万笔记录，失败了300笔，时间耗时很长，tps很差。
### 5.2 测试步长100
接下来我们先开始测试步长100，进入mycat 8066 端口, 执行下面语句

    DELETE FROM mycat_sequence;
    INSERT INTO mycat_sequence(name,current_value,increment) VALUES ('PERSON', 0, 100);

清空person已有的数据

    drop table person;
    create table person 
    (
    Id bigint not null primary key AUTO_INCREMENT,
    name varchar(16) ,
    school int,
    age int,
    addr varchar(32),
    zcode int,
    birth date,
    score int
    ) AUTO_INCREMENT = 1;

![2016-02-25_20-20-31](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_20-20-31.jpg)

在mycat测试工具bin目录,执行

    ./test_stand_insert_perf.sh jdbc:mysql://localhost:8066/TESTDB test test 50  "file=person-insert.sql"

最后结果如下
![2016-02-25_20-50-59](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_20-50-59.jpg)
我们看到10万笔记录，失败了200笔，耗时缩短了，tps增加了。
### 5.3 测试步长10000
接下来我们先开始测试步长10000，进入mycat 8066 端口, 执行下面语句
    
    DELETE FROM mycat_sequence;
    INSERT INTO mycat_sequence(name,current_value,increment) VALUES ('PERSON', 0, 10000);

清空person已有的数据

    drop table person;
    create table person 
    (
    Id bigint not null primary key AUTO_INCREMENT,
    name varchar(16) ,
    school int,
    age int,
    addr varchar(32),
    zcode int,
    birth date,
    score int
    ) AUTO_INCREMENT = 1;

在mycat测试工具bin目录,执行

    ./test_stand_insert_perf.sh jdbc:mysql://localhost:8066/TESTDB test test 50  "file=person-insert.sql"

最后结果如下
![2016-02-25_20-59-09](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-25_20-59-09.jpg)

我们看到10万笔记录，全部成功插入，耗时很短，tps显著增加。
结论，从以上三种步长测试结果看，明显步长越大，成功率越高，效果越佳。
参考：[Mycat权威指南](http://www.mycat.io/document/mycat1.5.2.pdf)