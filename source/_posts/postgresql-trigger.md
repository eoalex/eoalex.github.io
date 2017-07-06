---
title: postgresql trigger调用C语言函数实例
tags:
  - postgresql
  - trigger
id: 484
categories:
  - 数据库
date: 2016-01-09 10:41:41
---
## 1. 简介
postgresql 的一个触发器是一种声明，告诉数据库应该在执行特定的操作的时候执行特定的函数。本文我们将通过一个C语言编写的函数来演示触发器的机制。
## 2. 准备
### 2.1 安装数据库，初始化并启动
	
	#/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data start
### 2.2 建立测试数据库及表

	#/usr/local/pgsql/bin/createdb trigger_testdb
	#/usr/local/pgsql/bin/psql trigger_testdb

    CREATE TABLE ttest (
        x integer
    );
   
![2016-01-09_9-39-44](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-09_9-39-44.jpg)
## 3. 步骤
### 3.1 函数编译
编写C函数并编译，代码如下:

```
#include "postgres.h"
#include "executor/spi.h"       /* this is what you need to work with SPI */
#include "commands/trigger.h"   /* ... triggers ... */
#include "utils/rel.h"          /* ... and relations */
#ifdef PG_MODULE_MAGIC
PG_MODULE_MAGIC;
#endif

extern Datum trigf(PG_FUNCTION_ARGS);

PG_FUNCTION_INFO_V1(trigf);

Datum
trigf(PG_FUNCTION_ARGS)
{
TriggerData *trigdata = (TriggerData *) fcinfo-&gt;context;
TupleDesc   tupdesc;
HeapTuple   rettuple;
char       *when;
bool        checknull = false;
bool        isnull;
int         ret, i;

/* make sure it's called as a trigger at all */
if (!CALLED_AS_TRIGGER(fcinfo))
    elog(ERROR, "trigf: not called by trigger manager");

/* tuple to return to executor */
if (TRIGGER_FIRED_BY_UPDATE(trigdata-&gt;tg_event))
    rettuple = trigdata-&gt;tg_newtuple;
else
    rettuple = trigdata-&gt;tg_trigtuple;

/* check for null values */
if (!TRIGGER_FIRED_BY_DELETE(trigdata-&gt;tg_event)
    &amp;&amp; TRIGGER_FIRED_BEFORE(trigdata-&gt;tg_event))
    checknull = true;

if (TRIGGER_FIRED_BEFORE(trigdata-&gt;tg_event))
    when = "before";
else
    when = "after ";

tupdesc = trigdata-&gt;tg_relation-&gt;rd_att;

/* connect to SPI manager */
if ((ret = SPI_connect()) &lt; 0)
    elog(ERROR, "trigf (fired %s): SPI_connect returned %d", when, ret);

/* get number of rows in table */
ret = SPI_exec("SELECT count(*) FROM ttest", 0);

if (ret vals[0],
                                SPI_tuptable-&gt;tupdesc,
                                1,
                                &amp;isnull));

elog (INFO, "trigf (fired %s): there are %d rows in ttest", when, i);

SPI_finish();

if (checknull)
{
    SPI_getbinval(rettuple, tupdesc, 1, &amp;isnull);
    if (isnull)
        rettuple = NULL;
}

return PointerGetDatum(rettuple);
}
```
用以下命令编译
	
	#gcc -fPIC -I /home/src/pgsql/src/include -shared -o trigger_testc.so trigger_testc.c
	
把编译好的so文件复制到lib目录下
    
    #cp trigger_testc.so /usr/local/pgsql/lib
### 3.2 创建触发器
加载编译好的so文件到数据库
    
    load 'trigger_testc'
创建函数
   
    CREATE FUNCTION trigf() RETURNS trigger
        AS 'trigger_testc'
        LANGUAGE C;
    
创建两个触发器，一个触发在INSERT/UPDATE/DELETE事务之前，一个触发在INSERT/UPDATE/DELETE事务之后。触发后执行trigf函数，这个函数也是我们刚才编译的C语言函数。
    
    CREATE TRIGGER tbefore BEFORE INSERT OR UPDATE OR DELETE ON ttest
        FOR EACH ROW EXECUTE PROCEDURE trigf();
    CREATE TRIGGER tafter AFTER INSERT OR UPDATE OR DELETE ON ttest
        FOR EACH ROW EXECUTE PROCEDURE trigf();

![2016-01-09_10-06-43](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-09_10-06-43.jpg)
## 4. 测试
1) 插入空值
	
	INSERT INTO ttest VALUES (NULL);
![2016-01-09_10-09-38](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-09_10-09-38.jpg)

2)插入第一条记录
	
	INSERT INTO ttest VALUES (1);

3)插入第二条记录
	
	INSERT INTO ttest SELECT x * 2 FROM ttest;
![2016-01-09_10-11-15](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-09_10-11-15.jpg)

4)更新第二条记录
	
	UPDATE ttest SET x = NULL WHERE x = 2;
	UPDATE ttest SET x = 4 WHERE x = 2;
![2016-01-09_10-13-10](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-09_10-13-10.jpg)

5)删除记录
	
	DELETE FROM ttest;
![2016-01-09_10-14-03](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-09_10-14-03.jpg)

[参考官网](http://www.postgresql.org/docs/9.4/interactive/triggers.html)