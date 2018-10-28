---
title: PostgreSQL 集群性能审计工具pgCluu
tags:
  - pgCluu
  - postgresql
id: 853
categories:
  - 数据库
date: 2016-03-04 11:11:11
---
## 1. 简介
[pgCluu](http://pgcluu.darold.net/) 是一个对 PostgreSQL 集群性能进行完整审计的工具，该工具分为两部分：
* collector 用于从 PostgreSQL 集群中获取统计数据，使用 psql 和 sar 工具。
* grapher 用于生成 HTML 报表和图表。

    语法：
        PostgreSQL数据库和系统指标收集器
             pgcluu_collectd [options] output_dir
       报表生成器
             pgcluu [options] -o report_dir input_dir
 

## 2. 安装
好，下面我们介绍如何安装和使用它。
我们可以从pgCluu的[github](https://github.com/darold/pgcluu/releases)上下载。目前gpCluu版本更新到了2.4，下载后解压

    tar -xzvf pgcluu-2.4.tar.gz
检查并编译，可执行文件默认会安装至 `/usr/local/bin/`目录下。

    [root@anzhy pgcluu-2.4]# perl Makefile.PL
    Checking if your kit is complete...
    Looks good
    Writing Makefile for pgCluu
    [root@anzhy pgcluu-2.4]# make
    cp pgcluu_collectd blib/script/pgcluu_collectd
    /usr/bin/perl -MExtUtils::MY -e 'MY-&gt;fixin(shift)' -- blib/script/pgcluu_collectd
    cp pgcluu blib/script/pgcluu
    /usr/bin/perl -MExtUtils::MY -e 'MY-&gt;fixin(shift)' -- blib/script/pgcluu
    Manifying blib/man1/pgcluu.1
    [root@anzhy pgcluu-2.4]# sudo make install
    Installing /usr/local/share/man/man1/pgcluu.1
    Installing /usr/local/bin/pgcluu_collectd
    Installing /usr/local/bin/pgcluu
    Appending installation info to /usr/lib64/perl5/perllocal.pod

## 3. 配置
### 3.1 启动postgres
如果还没有启动postgres，我们先启动它，我们使用postgres用户
    
    su - postgres
    pg_ctl -D /usr/local/pgsql/data start
### 3.2 执行收集器
我们先建立临时目录,然后执行收集器

    mkdir /tmp/stat_db1/
    pgcluu_collectd -D -i 60 /tmp/stat_db1/
过几分钟我们停止收集

    pgcluu_collectd -k
以下就是收集过程

```
[postgres@anzhy ~]$ mkdir /tmp/stat_db1/
[postgres@anzhy ~]$ pgcluu_collectd -D -i 60 /tmp/stat_db1/
[postgres@anzhy ~]$ LOG: Detach from terminal with pid: 2704
[postgres@anzhy ~]$ pgcluu_collectd -k
OK: pgcluu_collectd exited with value 0
[postgres@anzhy ~]$ ll /tmp/stat_db1/
total 280
-rw-rw-r--. 1 postgres postgres  12554 Mar  4 09:19 pg_class_size.csv
-rw-rw-r--. 1 postgres postgres    331 Mar  4 09:19 pg_database_size.csv
-rw-rw-r--. 1 postgres postgres      0 Mar  4 09:19 pg_db_role_setting.csv
-rw-rw-r--. 1 postgres postgres   4476 Mar  4 09:19 pg_hba.conf
-rw-rw-r--. 1 postgres postgres   1636 Mar  4 09:19 pg_ident.conf
-rw-rw-r--. 1 postgres postgres 128800 Mar  4 09:19 pg_settings.csv
-rw-rw-r--. 1 postgres postgres     90 Mar  4 09:19 pg_stat_bgwriter.csv
-rw-rw-r--. 1 postgres postgres      0 Mar  4 09:19 pg_stat_connections.csv
-rw-rw-r--. 1 postgres postgres    342 Mar  4 09:19 pg_stat_database_conflicts.csv
-rw-rw-r--. 1 postgres postgres    781 Mar  4 09:19 pg_stat_database.csv
-rw-rw-r--. 1 postgres postgres    295 Mar  4 09:19 pg_statio_user_indexes.csv
-rw-rw-r--. 1 postgres postgres      0 Mar  4 09:19 pg_statio_user_sequences.csv
-rw-rw-r--. 1 postgres postgres    470 Mar  4 09:19 pg_statio_user_tables.csv
-rw-rw-r--. 1 postgres postgres   1415 Mar  4 09:19 pg_stat_locks.csv
-rw-rw-r--. 1 postgres postgres      0 Mar  4 09:19 pg_stat_missing_fkindexes.csv
-rw-rw-r--. 1 postgres postgres      0 Mar  4 09:19 pg_stat_redundant_indexes.csv
-rw-rw-r--. 1 postgres postgres      0 Mar  4 09:19 pg_stat_replication.csv
-rw-rw-r--. 1 postgres postgres      0 Mar  4 09:19 pg_stat_unused_indexes.csv
-rw-rw-r--. 1 postgres postgres      0 Mar  4 09:19 pg_stat_user_functions.csv
-rw-rw-r--. 1 postgres postgres    328 Mar  4 09:19 pg_stat_user_indexes.csv
-rw-rw-r--. 1 postgres postgres    966 Mar  4 09:19 pg_stat_user_tables.csv
-rw-rw-r--. 1 postgres postgres      0 Mar  4 09:19 pg_stat_xact_user_functions.csv
-rw-rw-r--. 1 postgres postgres    383 Mar  4 09:19 pg_stat_xact_user_tables.csv
-rw-rw-r--. 1 postgres postgres    140 Mar  4 09:19 pg_tablespace_size.csv
-rw-rw-r--. 1 postgres postgres     58 Mar  4 09:19 pg_xlog_stat.csv
-rw-rw-r--. 1 postgres postgres  20543 Mar  4 09:19 postgresql.conf
-rw-rw-r--. 1 postgres postgres  27402 Mar  4 09:19 sar_stats.dat
-rw-rw-r--. 1 postgres postgres  21605 Mar  4 09:19 sysinfo.txt
```
收集的数据保存在csv及txt文件中。
### 3.3 执行报表生成器
接下来我们通过报表生成器生成我们可以看的html网页

    mkdir /tmp/report_db1/
    pgcluu -o /tmp/report_db1/ /tmp/stat_db1/

结果
```
[postgres@anzhy ~]$ mkdir /tmp/report_db1/
[postgres@anzhy ~]$ pgcluu -o /tmp/report_db1/ /tmp/stat_db1/
[postgres@anzhy ~]$ ll /tmp/report_db1
total 1760
-rw-rw-r--. 1 postgres postgres 102595 Mar  4 09:21 bootstrap.min.css
-rw-rw-r--. 1 postgres postgres  27834 Mar  4 09:21 bootstrap.min.js
-rw-rw-r--. 1 postgres postgres 128934 Mar  4 09:21 cluster.html
-rw-rw-r--. 1 postgres postgres  41925 Mar  4 09:21 database-all.html
-rw-rw-r--. 1 postgres postgres  71078 Mar  4 09:21 database-pgbench.html
-rw-rw-r--. 1 postgres postgres  71186 Mar  4 09:21 database-plpython_testdb.html
-rw-rw-r--. 1 postgres postgres  74045 Mar  4 09:21 database-postgres.html
-rw-rw-r--. 1 postgres postgres  70967 Mar  4 09:21 database-test.html
-rw-rw-r--. 1 postgres postgres  71153 Mar  4 09:21 database-trigger_testdb.html
-rw-rw-r--. 1 postgres postgres 125286 Mar  4 09:21 font-awesome.min.css
-rw-rw-r--. 1 postgres postgres  80174 Mar  4 09:21 index.html
-rw-rw-r--. 1 postgres postgres 104684 Mar  4 09:21 jquery.min.js
-rw-rw-r--. 1 postgres postgres  43879 Mar  4 09:21 network-device0.html
-rw-rw-r--. 1 postgres postgres  43873 Mar  4 09:21 network-device1.html
-rw-rw-r--. 1 postgres postgres  43120 Mar  4 09:21 pgbench-index-size.html
-rw-rw-r--. 1 postgres postgres  41934 Mar  4 09:21 pgbench-table-size.html
-rw-rw-r--. 1 postgres postgres   3782 Mar  4 09:21 pgcluu.css
-rw-rw-r--. 1 postgres postgres 104288 Mar  4 09:21 pgcluu.js
-rw-rw-r--. 1 postgres postgres  42871 Mar  4 09:21 plpython_testdb-index-size.html
-rw-rw-r--. 1 postgres postgres  41609 Mar  4 09:21 plpython_testdb-table-size.html
-rw-rw-r--. 1 postgres postgres  42850 Mar  4 09:21 postgres-index-size.html
-rw-rw-r--. 1 postgres postgres  41588 Mar  4 09:21 postgres-table-size.html
-rw-rw-r--. 1 postgres postgres  17899 Mar  4 09:21 sorttable.js
-rw-rw-r--. 1 postgres postgres  53244 Mar  4 09:21 system-device0.html
-rw-rw-r--. 1 postgres postgres  75758 Mar  4 09:21 system.html
-rw-rw-r--. 1 postgres postgres  42838 Mar  4 09:21 test-index-size.html
-rw-rw-r--. 1 postgres postgres  41654 Mar  4 09:21 test-table-size.html
-rw-rw-r--. 1 postgres postgres  42868 Mar  4 09:21 trigger_testdb-index-size.html
-rw-rw-r--. 1 postgres postgres  41673 Mar  4 09:21 trigger_testdb-table-size.html
```
## 4. 结果展示
### 4.1 index 网页
![2016-03-04_9-39-39](/uploads/2016/03/2016-03-04_9-39-39.jpg)
### 4.2 系统信息
![2016-03-04_9-41-46](/uploads/2016/03/2016-03-04_9-41-46.jpg)

### 4.3 数据库信息
![2016-03-04_9-43-44](/uploads/2016/03/2016-03-04_9-43-44.jpg)

更多结果参见pgCluu官网[Example](http://pgcluu.darold.net/example/index.html)

本文参考[pgCluu官网](http://pgcluu.darold.net)