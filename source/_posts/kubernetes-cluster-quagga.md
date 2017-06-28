---
title: Kubernetes入门实战(4)：Kubernetes 集群网络之直接路由方案
tags:
  - Kubernetes
  - Quagga
id: 960
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-03-13 14:15:30
---

[上文](http://blog.yaodataking.com/2016/02/kubernetes-cluster-1.html)我们已经初步搭建了Kubernetes两个Node节点的集群，并且成功地在启动mysql pod的Node节点上连接，但是在另一个Node节点上我们还是无法连接到。本文我们就用直接路由加Quagga的方式实现不同Node节点间的pod互联。

环境及准备工作同[上文](http://blog.yaodataking.com/2016/02/kubernetes-cluster-1.html)
1\. 修改docker0地址
我们重新规划docker0的网址，分别使用10.1.10.1及10.1.20.1网段。
在centos-minion01上，修改docker0网段地址。
ifconfig docker0 10.1.10.1/24
还要修改docker配置文件

    # /etc/sysconfig/docker
    # Modify these options if you want to change the way the docker daemon runs
    OPTIONS='--bip=10.1.10.1/24'
    `</pre>
    在centos-minion02上，修改docker0网段地址。
    ifconfig docker0 10.1.20.1/24
    还要修改docker配置文件
    <pre>`# /etc/sysconfig/docker
    # Modify these options if you want to change the way the docker daemon runs
    OPTIONS='--bip=10.1.20.1/24'
    `</pre>
    重启docker服务。

    2\. 启动mysql POD和services
    <pre>`[root@centos-master mysqlpod]# kubectl get pods
    NAME      READY     STATUS    RESTARTS   AGE
    mysql     1/1       Running   0          31m
    [root@centos-master mysqlpod]# kubectl get services
    NAME         CLUSTER_IP       EXTERNAL_IP   PORT(S)    SELECTOR     AGE
    kubernetes   10.254.0.1               443/TCP           15d
    mysql        10.254.145.138           3306/TCP   name=mysql   51m
    `</pre>
    我们看到mysql pod启动在centos-minion02上。
    <pre>`[root@centos-master mysqlpod]# kubectl describe pods mysql|grep centos-minion
    Node:                           centos-minion02/192.168.199.53
    `</pre>
    此时在centos-minion01上连接mysql，不能连接。
    <pre>`[root@centos-minion01 ~]# mysql -uroot -p -h10.254.145.138
    `</pre>
    3\. 添加路由
    在centos-minion01上, 添加到centos-minion02上docker0的静态路由地址。
    route add -net 10.1.20.0 netmask 255.255.255.0 gw 192.168.199.52
    在centos-minion02上, 添加到centos-minion01上docker0的静态路由地址。
    route add -net 10.1.10.0 netmask 255.255.255.0 gw 192.168.199.53
    再次连接mysql(使用service IP)，我们看到连接已经成功。
    <pre>`[root@centos-minion01 ~]# mysql -uroot -p -h10.254.145.138
    Enter password:
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 5
    Server version: 5.7.10 MySQL Community Server (GPL)

    Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql&gt; exit
    Bye
    `</pre>
    使用pod的内部IP，我们看到连接也成功。
    <pre>`[root@centos-minion01 ~]# mysql -uroot -p -h10.1.20.2
    Enter password:
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 6
    Server version: 5.7.10 MySQL Community Server (GPL)

    Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql&gt; exit
    Bye
    [root@centos-minion01 ~]#
    `</pre>

    4\. 使用Quagga动态添加路由
    为了减少手工添加路由，可以使用Quagga实现路由规则的动态添加。为简单起见，我们使用docker镜像。
    #docker pull  index.alauda.cn/georce/router
    在每个Node上启动Quagga容器，Quagga需要以--privileged特权模式运行，并且指定--net=host，表示直接使用物理机的网络。
    #docker run -itd --name=router --privileged --net=host index.alauda.cn/georce/router
    启动成功后，Quagga会相互学习来完成到其他机器的docker0路由规则的添加。
    <pre>`[root@centos-minion02 ~]# route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    0.0.0.0         192.168.199.1   0.0.0.0         UG    0      0        0 eno16777736
    10.1.10.0       192.168.199.52  255.255.255.0   UG    20     0        0 eno16777736
    10.1.20.0       0.0.0.0         255.255.255.0   U     0      0        0 docker0
    169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eno16777736
    192.168.199.0   0.0.0.0         255.255.255.0   U     0      0        0 eno16777736
    `</pre>
     我们看到已经可以在centos-minion02上ping通centos-minion01的docker0
    <pre>`[root@centos-minion02 ~]# ping 10.1.10.1
    PING 10.1.10.1 (10.1.10.1) 56(84) bytes of data.
    64 bytes from 10.1.10.1: icmp_seq=1 ttl=64 time=1.20 ms
    64 bytes from 10.1.10.1: icmp_seq=2 ttl=64 time=0.204 ms
    64 bytes from 10.1.10.1: icmp_seq=3 ttl=64 time=0.236 ms
    64 bytes from 10.1.10.1: icmp_seq=4 ttl=64 time=0.234 ms
    64 bytes from 10.1.10.1: icmp_seq=5 ttl=64 time=0.423 ms
    