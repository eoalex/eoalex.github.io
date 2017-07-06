---
title: Kubernetes入门实战(5)：Kubernetes集群网络之flannel网络方案
tags:
  - flannel
  - Kubernetes
id: 965
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-03-13 19:24:58
---
## 1. 简介
[flannel](https://github.com/coreos/flannel) 是 CoreOS 团队针对 Kubernetes 设计的一个覆盖网络 (overlay network) 工具，其目的在于帮助每一个使用 Kuberentes 的 CoreOS 主机拥有一个完整的子网。Kubernetes 会为每一个 POD 分配一个独立的 IP 地址，这样便于同一个 POD 中的 Containers 彼此连接，而之前的 CoreOS 并不具备这种能力。为了解决这一问题，flannel 通过在集群中创建一个覆盖网格网络 (overlay mesh network) 为主机设定一个子网。具体flannel介绍及原理参见[官网](https://github.com/coreos/flannel)。下面我们实战配置及测试。
>注：本文安装配置是在我的上篇博文[Kubernetes集群初探](/2016/02/27/kubernetes-cluster-1/)的基础上。

## 2. etcd设置
首先我们要对etcd做一些更改
### 2.1 设置flanel网络段
	#etcdctl set /coreos.com/network/config '{ "Network": "10.2.0.0/16" }'
### 2.2 修改配置文件
在配置文件里/etc/etcd/etcd.conf把ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"中的locahost改为0.0.0.0

## 3. flannel安装配置
每台Node节点都要配置.
### 3.1 下载
	#wget https://github.com/coreos/flannel/releases/download/v0.5.5/flannel-0.5.5-linux-amd64.tar.gz
### 3.2 解压
	#tar -xzvf flannel-0.5.5-linux-amd64.tar.gz
### 3.3 安装
直接复制解压出来的两个文件到可执行目录
	
	#cp flannel-0.5.5/flanneld /usr/bin
	#cp flannel-0.5.5/mk-docker-opts.sh /usr/bin
### 3.4 配置
编辑`/etc/sysconfig/flanneld`

    # Flanneld configuration options
    # etcd url location
    FLANNEL_ETCD="http://centos-master:2379"

    # etcs config key
    FLANNEL_ETCD_KEY="/coreos.com/network"

    # Any additonal options
    #FLANNEL_OPTIONS=
    
编辑服务文件`/usr/lib/systemd/system/flanneld.service`
    
    [Unit]
    Description=Flanneld overlay address etcd agent
    After=network.target
    Before=docker.service

    [Service]
    Type=notify
    EnvironmentFile=-/etc/sysconfig/flanneld
    EnvironmentFile=-/etc/sysconfig/docker-network
    ExecStart=/usr/bin/flanneld \
                -etcd-endpoints=${FLANNEL_ETCD} \
                $FLANNEL_OPTIONS

    [Install]
    RequiredBy=docker.service
    WantedBy=multi-user.target
### 3.5 暂停docker服务
    #systemctl stop docker
### 3.6 执行以下脚本
    #systemctl start flanneld
    #mk-docker-opts.sh -i
    #source /run/flannel/subnet.env
    #ifconfig docker0 ${FLANNEL_SUBNET}
### 3.7 重启docker服务
    #systemctl restart docker
检查网络配置，我们看到多了flannel0，
在centos-minion01上
    
    [root@centos-minion01 ~]# ip a
    1: lo:  mtu 65536 qdisc noqueue state UNKNOWN
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: eno16777736:  mtu 1500 qdisc pfifo_fast state UP qlen 1000
        link/ether 00:0c:29:66:c0:bf brd ff:ff:ff:ff:ff:ff
        inet 192.168.199.52/24 brd 192.168.199.255 scope global eno16777736
           valid_lft forever preferred_lft forever
        inet6 fe80::20c:29ff:fe66:c0bf/64 scope link
           valid_lft forever preferred_lft forever
    3: docker0:  mtu 1500 qdisc noqueue state DOWN
        link/ether 02:42:c8:3e:ab:b3 brd ff:ff:ff:ff:ff:ff
        inet 10.2.35.1/24 brd 10.2.35.255 scope global docker0
           valid_lft forever preferred_lft forever
    4: flannel0:  mtu 1472 qdisc pfifo_fast state UNKNOWN qlen 500
        link/none
        inet 10.2.35.0/16 scope global flannel0
           valid_lft forever preferred_lft forever
在centos-minion02上
    
    [root@centos-minion02 ~]# ip a
    1: lo:  mtu 65536 qdisc noqueue state UNKNOWN
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: eno16777736:  mtu 1500 qdisc pfifo_fast state UP qlen 1000
        link/ether 00:0c:29:6a:0c:48 brd ff:ff:ff:ff:ff:ff
        inet 192.168.199.53/24 brd 192.168.199.255 scope global eno16777736
           valid_lft forever preferred_lft forever
        inet6 fe80::20c:29ff:fe6a:c48/64 scope link
           valid_lft forever preferred_lft forever
    3: docker0:  mtu 1500 qdisc noqueue state DOWN
        link/ether 02:42:2e:ab:fa:29 brd ff:ff:ff:ff:ff:ff
        inet 10.2.52.1/24 brd 10.2.52.255 scope global docker0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:2eff:feab:fa29/64 scope link
           valid_lft forever preferred_lft forever
    6: flannel0:  mtu 1472 qdisc pfifo_fast state UNKNOWN qlen 500
        link/none
        inet 10.2.52.0/16 scope global flannel0
           valid_lft forever preferred_lft forever
    
查看etcd上的路由表。
    
    [root@centos-master ~]# etcdctl ls /coreos.com/network/subnets
    /coreos.com/network/subnets/10.2.35.0-24
    /coreos.com/network/subnets/10.2.52.0-24
    [root@centos-master ~]# etcdctl get /coreos.com/network/subnets/10.2.35.0-24
    {"PublicIP":"192.168.199.52"}
    [root@centos-master ~]# etcdctl get /coreos.com/network/subnets/10.2.52.0-24
    {"PublicIP":"192.168.199.53"}
    

## 4. 测试验证
### 4.1 启动两个pod
我们在centos-master上制作两个pod文件。第二个文件把01改为02
    
    [root@centos-master mysqlpod]# cat mysqlpod01.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: mysql01
      labels:
        name: mysql01
    spec:
      containers:
      - name: mysql01
        image: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: p123456
        ports:
        - containerPort: 3306
    
启动这两个pod。

    kubectl create -f mysqlpod01.yaml
    kubectl create -f mysqlpod02.yaml
    [root@centos-master ~]# kubectl get pods -o wide
    NAME      READY     STATUS    RESTARTS   AGE       NODE
    mysql01   1/1       Running   0          10m       centos-minion02
    mysql02   1/1       Running   0          7m        centos-minion01
    
我们看到分别启动在两台Node上。下面我们测试他们的容器能不能互联。
在centos-minion01上进入mysql02容器，获得IP地址为10.2.35.2，同时我们在centos-minion02上也获得mysql01容器的IP地址为10.2.52.2。
    
    [root@centos-minion01 ~]# docker exec -it 309728d3a3f4 /bin/bash
    root@mysql02:/# ip a
    1: lo:  mtu 65536 qdisc noqueue state UNKNOWN group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    5: eth0@if6:  mtu 1500 qdisc noqueue state UP group default
        link/ether 02:42:0a:02:23:02 brd ff:ff:ff:ff:ff:ff
        inet 10.2.35.2/24 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:aff:fe02:2302/64 scope link tentative dadfailed
           valid_lft forever preferred_lft forever
    root@mysql02:/# mysql -uroot -p -h10.2.52.2
    Enter password:
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 2
    Server version: 5.7.10 MySQL Community Server (GPL)

    Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql>
    
我们看到连接没有问题。同时在centos-minion02的上，我们也试着进入mysql01容器，连接在centos-minion01节点上的mysql02容器。
    
    [root@centos-minion02 ~]# docker exec -it 98fd9272ad7a /bin/bash
    root@mysql01:/# ip a
    1: lo:  mtu 65536 qdisc noqueue state UNKNOWN group default
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    7: eth0@if8:  mtu 1500 qdisc noqueue state UP group default
        link/ether 02:42:0a:02:34:02 brd ff:ff:ff:ff:ff:ff
        inet 10.2.52.2/24 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::42:aff:fe02:3402/64 scope link tentative dadfailed
           valid_lft forever preferred_lft forever
    root@mysql01:/# mysql -uroot -p -h10.2.35.2
    Enter password:
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 2
    Server version: 5.7.10 MySQL Community Server (GPL)

    Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql>

同样也可以顺利连接。

注：Flannel不需要在Master节点上部署，因为master节点不参与负载。Flannel不仅控制了Docker引擎子网的分配也控制了容器的IP分配。
参考：Kubernetes权威指南