---
title: PostgreSQL压力测试工具pgbench入门
tags:
  - pgbench
  - postgresql
id: 642
categories:
  - 数据库
date: 2016-02-11 15:35:55
---

pgbench 是一个简单的测试PostgreSQL性能的程序。它可以运行在多个并发数据库会话中，并不停地执行同一顺序SQL命令，然后计算平均事务速度，也就是tps。pgbench 默认使用[TPC-B](http://www.tpc.org/tpcb/)方法来测试五个包含 SELECT、UPDATE、和 INSERT命令的脚本。当然，我们也可以使用自己的脚本来测试。本文将使用pgbench 默认提供的脚本来测试。
1.环境及安装
环境centos 6.5 ,内存 4G，postgesql 版本 9.3.4
进入postgres中的pgbench源代码目录
#cd /home/src/pgsql/contrib/pgbench
#make all
#make install
[![2016-02-11_14-11-50](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-11_14-11-50.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-11_14-11-50.jpg)
2.初始化数据
启动数据库
#cd /usr/local/pgsql
#pg_ctl -D data start
#createdb pgbench
初始化测试数据
#pgbench -i pgbench
[![2016-02-11_14-18-41](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-11_14-18-41.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-11_14-18-41.jpg)
3\. 测试
3.1 模拟1个客户端
#pgbench -M simple -c 1 -T 100 -r pgbench

    starting vacuum...end.
    transaction type: TPC-B (sort of)
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 100 s
    number of transactions actually processed: 53477
    tps = 534.764738 (including connections establishing)
    tps = 534.785402 (excluding connections establishing)
    statement latencies in milliseconds:
            0.004993        \set nbranches 1 * :scale
            0.000899        \set ntellers 10 * :scale
            0.000661        \set naccounts 100000 * :scale
            0.000827        \setrandom aid 1 :naccounts
            0.000522        \setrandom bid 1 :nbranches
            0.000544        \setrandom tid 1 :ntellers
            0.000577        \setrandom delta -5000 5000
            0.076682        BEGIN;
            0.250163        UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
            0.155333        SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
            0.171949        UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
            0.152673        UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
            0.130653        INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
            0.909966        END;
    `</pre>
    3.2 模拟50个客户端
    #pgbench -M simple -c 50 -T 100 -r pgbench
    <pre>`
    starting vacuum...end.
    transaction type: TPC-B (sort of)
    scaling factor: 1
    query mode: simple
    number of clients: 50
    number of threads: 1
    duration: 100 s
    number of transactions actually processed: 36036
    tps = 359.955238 (including connections establishing)
    tps = 360.320538 (excluding connections establishing)
    statement latencies in milliseconds:
            0.003963        \set nbranches 1 * :scale
            0.001221        \set ntellers 10 * :scale
            0.000811        \set naccounts 100000 * :scale
            0.001040        \setrandom aid 1 :naccounts
            0.000667        \setrandom bid 1 :nbranches
            0.000662        \setrandom tid 1 :ntellers
            0.000673        \setrandom delta -5000 5000
            0.585555        BEGIN;
            1.312489        UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
            1.089996        SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
            109.413961      UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
            23.461208       UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
            0.577524        INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
            2.212504        END;
    `</pre>
    3.3 模拟100个客户端（数据库默认设置最大连接数)
    #pgbench -M simple -c 100 -T 100 -r pgbench
    <pre>`
    starting vacuum...end.
    transaction type: TPC-B (sort of)
    scaling factor: 1
    query mode: simple
    number of clients: 100
    number of threads: 1
    duration: 100 s
    number of transactions actually processed: 33268
    tps = 331.936164 (including connections establishing)
    tps = 333.080359 (excluding connections establishing)
    statement latencies in milliseconds:
            0.003908        \set nbranches 1 * :scale
            0.001205        \set ntellers 10 * :scale
            0.000798        \set naccounts 100000 * :scale
            0.001023        \setrandom aid 1 :naccounts
            0.000649        \setrandom bid 1 :nbranches
            0.000658        \setrandom tid 1 :ntellers
            0.000684        \setrandom delta -5000 5000
            0.909069        BEGIN;
            1.845696        UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
            1.291101        SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
            266.673504      UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
            26.131598       UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
            0.605307        INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
            2.406303        END;
    `</pre>
    3.4 模拟100个客户端,使用扩张调用接口
    #pgbench -M extended -c 100 -T 100 -r pgbench
    <pre>`
    starting vacuum...end.
    transaction type: TPC-B (sort of)
    scaling factor: 1
    query mode: extended
    number of clients: 100
    number of threads: 1
    duration: 100 s
    number of transactions actually processed: 29687
    tps = 295.368409 (including connections establishing)
    tps = 295.968232 (excluding connections establishing)
    statement latencies in milliseconds:
            0.004401        \set nbranches 1 * :scale
            0.001297        \set ntellers 10 * :scale
            0.000929        \set naccounts 100000 * :scale
            0.001026        \setrandom aid 1 :naccounts
            0.000715        \setrandom bid 1 :nbranches
            0.000711        \setrandom tid 1 :ntellers
            0.000681        \setrandom delta -5000 5000
            0.955081        BEGIN;
            2.016270        UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
            1.409169        SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
            300.175762      UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
            29.409881       UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
            0.749236        INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
            2.665060        END;
    `</pre>
    3.5 模拟100个客户端,使用绑定变量调用接口
    #pgbench -M prepared -c 100 -T 100 -r pgbench
    <pre>`
    starting vacuum...end.
    transaction type: TPC-B (sort of)
    scaling factor: 1
    query mode: prepared
    number of clients: 100
    number of threads: 1
    duration: 100 s
    number of transactions actually processed: 35649
    tps = 354.816112 (including connections establishing)
    tps = 355.568016 (excluding connections establishing)
    statement latencies in milliseconds:
            0.003984        \set nbranches 1 * :scale
            0.001321        \set ntellers 10 * :scale
            0.000875        \set naccounts 100000 * :scale
            0.001053        \setrandom aid 1 :naccounts
            0.000693        \setrandom bid 1 :nbranches
            0.000697        \setrandom tid 1 :ntellers
            0.000728        \setrandom delta -5000 5000
            0.849722        BEGIN;
            1.347952        UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
            0.922049        SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
            250.321603      UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
            24.354052       UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
            0.465177        INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
            2.328246        END;

我们主要关心的是最后的输出报告中的TPS值，里面有两个,一个是包含连接开销(including),另一个是不包含连接开销的(excluding),这个值是反映的每秒处理的事务数，这里也列出了每个事务所消耗的平均时间。一般认为能将硬件用到极致，速度越快越好。 
参考：
[http://www.postgresql.org/docs/9.3/static/pgbench.html](http://www.postgresql.org/docs/9.3/static/pgbench.html)