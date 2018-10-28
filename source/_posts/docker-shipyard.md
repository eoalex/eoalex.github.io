---
title: Docker管理工具Shipyard安装配置
tags:
  - Docker
  - Shipyard
  - swarm
id: 545
categories:
  - 云计算及虚拟化
date: 2016-01-22 23:45:16
---
## 1. 简介
Shipyard是一个基于Web的Docker管理工具，基于[Docker Swarm](https://docs.docker.com/swarm)，支持多主机，可以把多个Docker主机上的容器统一管理；可以查看镜像，下拉镜像；可以管理私有镜像仓库；并提供 RESTful API 等。本文将在两台docker主机上安装配置shipyard并分别在不同的Docker上发布两个MySQL实例，MySQL-dev与MySQL-Online。
## 2. Shipyard生态介绍
shipyard是由shipyard控制器以及周围生态系统构成，都是以容器封装，以下按照启动顺序进行介绍。
1)RethinkDB
首先启动的就是RethinkDB容器，shipyard采用RethinkDB作为数据库来保存账户，引擎，服务键值以及元信息等信息。
2)Discovery
为了使用Swarm的选举机制，我们需要一个外部的密钥值存储容器，shipyard默认采用了[etcd](https://github.com/coreos/etcd)。
3)shipyard_certs
证书管理容器，实现证书验证功能
4)Proxy
默认情况下，Docker引擎只监听Socket，我们可以重新配置引擎使用TLS或者使用一个代理容器，转发请求从TCP到Docker监听的UNIX Socket。
5)Swarm Manager
Swarm管理器
6)Swarm Agent
Swarm代理，运行在每个节点上。
7)Controller
shipyard控制器，Remote API的实现和web的实现。
## 3. 准备
环境准备 centos 7 + docker 1.9.1,准备两台。
主节点：docker41，IP:192.168.199.41；
从节点：docker42，IP:192.168.199.42。
在两台主机上分别做几下两点。
1)首先清除iptables
	
	#iptables -F
![2016-01-22_19-22-52](/uploads/2016/01/2016-01-22_19-22-52.jpg)

2)设置daemon参数，重启docker
	
	#vi /usr/lib/systemd/system/docker.service

    ExecStart=/usr/bin/docker daemon -H unix:///var/run/docker.sock
![2016-01-22_19-22-23](/uploads/2016/01/2016-01-22_19-22-23.jpg)
## 4. 主节点安装
Shipyard主节点安装（IP:192.168.199.41)
### 4.1 下拉镜像
虽然官网提供了一键安装的命令，`curl -sSL https://shipyard-project.com/deploy | bash -s` , 但我们还是体验一下non-TLS手工安装，也就是说不安装shipyard_certs。所以我们先下拉以下几个镜像。
    
    docker pull rethinkdb
    docker pull microbox/etcd 
    docker pull shipyard/docker-proxy
    docker pull swarm:latest
    docker pull shipyard/shipyard
![2016-01-22_19-18-51](/uploads/2016/01/2016-01-22_19-18-51.jpg)

如果在下拉镜像速度上有问题，建议使用国内的镜像代理。
### 4.2 启动容器shipyard-rethinkdb
    docker run \
        -ti \
        -d \
        --restart=always \
        --name shipyard-rethinkdb \
        rethinkdb
### 4.3 启动容器shipyard-discovery
    docker run \
        -ti \
        -d \
        -p 4001:4001 \
        -p 7001:7001 \
        --restart=always \
        --name shipyard-discovery \
        microbox/etcd -name discovery
![2016-01-22_19-25-11](/uploads/2016/01/2016-01-22_19-25-11.jpg)
### 4.4 启动shipyard-proxy
    docker run \
        -ti \
        -d \
        -p 2375:2375 \
        --hostname=$HOSTNAME \
        --restart=always \
        --name shipyard-proxy \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -e PORT=2375 \
        shipyard/docker-proxy:latest
![2016-01-22_19-25-34](/uploads/2016/01/2016-01-22_19-25-34.jpg)

### 4.5 启动shipyard-swarm-manager 
    docker run \
        -ti \
        -d \
        --restart=always \
        --name shipyard-swarm-manager \
        swarm:latest \
        manage --replication --addr 192.168.199.41:3375 --host tcp://0.0.0.0:3375 etcd://192.168.199.41:4001
![2016-01-22_22-46-33](/uploads/2016/01/2016-01-22_22-46-33.jpg)

### 4.6 启动shipyard-swarm-agent
    docker run \
        -ti \
        -d \
        --restart=always \
        --name shipyard-swarm-agent \
        swarm:latest \
        join --addr 192.168.199.41:2375 etcd://192.168.199.41:4001
![2016-01-22_19-26-28](/uploads/2016/01/2016-01-22_19-26-28.jpg)
### 4.7 启动shipyard-controller
    docker run \
        -ti \
        -d \
        --restart=always \
        --name shipyard-controller \
        --link shipyard-rethinkdb:rethinkdb \
        --link shipyard-swarm-manager:swarm \
        -p 8080:8080 \
        shipyard/shipyard:latest \
        server \
        -d tcp://swarm:3375

![2016-01-22_19-26-40](/uploads/2016/01/2016-01-22_19-26-40.jpg)
至此主节点安装完毕，我们可以查看log来检查是否安装成功。
### 4.8 Web界面访问
    访问http://192.168.199.41:8080，输入默认用户名和密码：admin/shipyard
查看启动的容器
![2016-01-22_19-27-55](/uploads/2016/01/2016-01-22_19-27-55.jpg)
查看镜像
![2016-01-22_19-28-11](/uploads/2016/01/2016-01-22_19-28-11.jpg)
查看节点，此时只有docker41这个节点
![2016-01-22_19-29-11](/uploads/2016/01/2016-01-22_19-29-11.jpg)

## 5. 从节点安装
添加从节点（IP:192.168.199.42)
### 5.1 下拉镜像
    docker pull shipyard/docker-proxy
    docker pull swarm:latest
        
### 5.2 启动容器shipyard-proxy
    docker run \
        -ti \
        -d \
        -p 2375:2375 \
        --hostname=$HOSTNAME \
        --restart=always \
        --name shipyard-proxy \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -e PORT=2375 \
        shipyard/docker-proxy:latest
### 5.3 启动容器shipyard-swarm-manager
    docker run \
        -ti \
        -d \
        --restart=always \
        --name shipyard-swarm-manager \
        swarm:latest \
        manage --replication --addr 192.168.199.42:3375 --host tcp://0.0.0.0:3375 etcd://192.168.199.41:4001
### 5.4 启动容器shipyard-swarm-agent
    docker run \
        -ti \
        -d \
        --restart=always \
        --name shipyard-swarm-agent \
        swarm:latest \
        join --addr 192.168.199.42:2375 etcd://192.168.199.41:4001
![2016-01-22_22-49-38](/uploads/2016/01/2016-01-22_22-49-38.jpg)
查看容器shipyard-swarm-manager的log，看到已经注册
![2016-01-22_21-04-51](/uploads/2016/01/2016-01-22_21-04-51.jpg)
### 5.5 Web界面访问。
还是访问http://192.168.199.41:8080，查看容器，已经多了节点docker42上的容器。
![2016-01-22_21-06-18](/uploads/2016/01/2016-01-22_21-06-18.jpg)
查看节点，多了 docker42
![2016-01-22_21-05-59](/uploads/2016/01/2016-01-22_21-05-59.jpg)
## 6. 给节点设置label
为了分清节点的功能，我们分别给节点设置不同label,Shipyard好像取消了直接维护，我们只能在docker daemon设置。在docker41上修改。

    vi /usr/lib/systemd/system/docker.service
	    ExecStart=/usr/bin/docker daemon -H unix:///var/run/docker.sock --label docker=dev
在docker42上修改。

	    ExecStart=/usr/bin/docker daemon -H unix:///var/run/docker.sock --label docker=online
分别重启docker，我们再看node节点label已改变。
![2016-01-22_22-07-02](/uploads/2016/01/2016-01-22_22-07-02.jpg)
## 7. 添加Mysql实例
1)添加Mysql-dev, 按图配置参数，端口，路径，容器名。swarm约束，我们选择标签为dev的那台。
![2016-01-22_22-35-02](/uploads/2016/01/2016-01-22_22-35-02.jpg)
查看容器详细信息。
![2016-01-22_22-38-36](/uploads/2016/01/2016-01-22_22-38-36.jpg)

2)添加Mysql-online，按图配置参数，端口，路径，容器名。swarm约束，我们选择标签为online的那台。
![2016-01-22_22-37-33](/uploads/2016/01/2016-01-22_22-37-33.jpg)
查看容器详细信息。
![2016-01-22_22-39-26](/uploads/2016/01/2016-01-22_22-39-26.jpg)
看到新增的两台容器，分别在不同的docker上。
![2016-01-22_22-40-02](/uploads/2016/01/2016-01-22_22-40-02.jpg)
以上，我们看到Shipyard确实可以很方便的统一管理位于不同主机的容器。
PS.
启动从节点的容器shipyard-swarm-manager时，出现ID duplicated错误的解决办法。

    ERRO[0008] ID duplicated. UGJV:S4FQ:NSHM:GL2I:OBON:UOJF:OZHB:E6FA:DVML:L67M:5ZLT:YM2G shared by 192.168.199.41:2375 and 192.168.199.42:2375

这个错误产生的原因，应该是安装docker后，我们两台主机是复制的，所以key值相同。
解决办法是，删除`/etc/docker/key.json`这个文件，重启docker。

参考：
[https://docs.docker.com/swarm/multi-manager-setup/](https://docs.docker.com/swarm/multi-manager-setup/)
[https://shipyard-project.com/deploy](https://shipyard-project.com/deploy)
[https://shipyard-project.com/docs/deploy/manual/](https://shipyard-project.com/docs/deploy/manual/)