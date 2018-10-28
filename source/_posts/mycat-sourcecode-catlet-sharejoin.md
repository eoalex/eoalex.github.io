---
title: Mycat源码分析之Catlet Sharejoin分析
tags:
  - catlet
  - Mycat
id: 1079
categories:
  - 数据库
date: 2016-04-30 23:47:48
---

## 1. 简介
Catlet是一个实现了Catlet接口的无状态Java类，负责将编码实现某个SQL的处理过程，并返回响应报文给客户端，目前主要用于人工编码实现跨分片SQL的处理逻辑，至今Mycat已经提供了BatchInsertSequence，MyHellowJoin和ShareJoin三个demo案例，本文我们通过分析ShareJoin来看看Catlet是怎么处理的。
ShareJoin是一个简单的跨分片Join,基于HBT的方式实现。目前支持2个表的join,原理就是解析SQL语句，拆分成单表的SQL语句执行，然后把各个节点的数据汇集。

## 2. Catlet接口定义
首先我们看到Catlet接口定义了processSQL和route。这样我们就可以通过编写自己的processSQL和route来实现特殊的要求。

```Java
    public interface Catlet {

    	/*
    	 * execute sql in EngineCtx and return result to client
    	 */
    	void processSQL(String sql, EngineCtx ctx);

    	void route(SystemConfig sysConfig, SchemaConfig schema,
    			int sqlType, String realSQL, String charset, ServerConnection sc,
    			LayerCachePool cachePool) ;
    	//void setRoute(RouteResultset rrs);
    }
```

## 3. ShareJoin类的processSQL
![2016-04-30_23-06-26](/uploads/2016/04/2016-04-30_23-06-26.jpg)

我们看到JoinParser首先解析SQL语句，ShareDBJoinHandler是执行后获取数据的handler，由sqlengine.EngineCtx的executeNativeSQLSequnceJob去执行SQLjob，并有AllJobFinishedListener侦听是否所有任务完成。

## 4. 相关类图
下图使整个与sharejoin相关的类图，
![class](/uploads/2016/04/class.png)    
## 5. 实际SQL测试
那么实际的sharejoin是不是就是这么实现的呢？我们来实际测试一下。
我们假设两个表company是全局表，customer是跨两个分区的表，我们现在需要JOIN这两个表来抓取数据。SQL语句如下，
    `select a.*,b.id, b.name as tit from customer a,company b on a.company_id=b.id;`
正常情况下，mycat能不能解析这个SQL语句呢？
![2016-05-01_10-01-32](/uploads/2016/04/2016-05-01_10-01-32.jpg)

以上我们看到这样一个普通的JOIN,正常的路由是无法解析这个SQL语句的。好，我们现在来看看sharejoin的威力。

![2016-05-01_10-01-48](/uploads/2016/04/2016-05-01_10-01-48.jpg)

我们看到，通过sharejoin的路由，成功的解析了这个SQL语句并且返回了正确的数据。查看mycat.log, 我们看到执行的步骤是跟上面源码看到的是一致的，catlet分别拆分成单表的SQL语句执行并汇总，最后侦听是否所有任务完成后发送数据ok。

    04/30 14:02:44.726   INFO [$_NIOREACTOR-0-RW] (JoinParser.java:70) -SQL: select a.*,b.id, b.name as tit from customer a,company b on a.company_id=b.id
    04/30 14:02:44.731   INFO [$_NIOREACTOR-0-RW] (ShareJoin.java:157) -Catlet exec:dn1,dn2, sql:select *, company_id from customer
    04/30 14:02:44.742   INFO [$_NIOREACTOR-0-RW] (ShareJoin.java:241) -SQLParallJob:dn1, sql:select id, id , name as tit from company where id in (1,2,3)
    04/30 14:02:44.751   INFO [$_NIOREACTOR-0-RW] (EngineCtx.java:171) -all job finished  for front connection: ServerConnection [id=102, schema=TESTDB, host=172.17.0.1, user=test,txIsolation=3, autocommit=true, schema=TESTDB]
    04/30 14:02:44.751   INFO [$_NIOREACTOR-0-RW] (EngineCtx.java:159) -write  eof ,packgId:13
    04/30 14:02:44.751   INFO [$_NIOREACTOR-0-RW] (ShareJoin.java:166) -发送数据OK

小结，解决跨分片的SQL JOIN的问题，远比我们想象的复杂，而且往往无法实现高效的处理，Mycat提供的这个Catlet接口可以让我们自己实现一些特定的算法。目前的实现sharejoin还是相对来说简单的，我们可以在此基础上去实现更加复杂的JOIN算法，分组算法，排序算法等等。

参考：Mycat权威指南