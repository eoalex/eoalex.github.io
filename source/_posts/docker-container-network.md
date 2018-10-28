---
title: Docker 容器网络机制初探(1)-Linux路由
tags:
  - Container
  - Docker
  - 容器
id: 464
categories:
  - 云计算及虚拟化
date: 2016-01-08 22:22:00
---
## 1. 简介
我们知道Docker利用容器来运行应用的，容器是从镜像创建的运行实例。它可以被启动、开始、停止、删除。每个容器都是相互隔离的、保证安全的平台。但是容器之间如何通信呢？本文我们使用Linux的路由简单实现下图的网络结构，并用`tshark`或`tcpdump`等工具分析网络的流向。
![2016-01-08_20-31-33](/uploads/2016/01/2016-01-08_20-31-33.jpg)
## 2. 准备
环境准备 centos 7 + docker 1.9.1(准备两台)
### 2.1 静态IP设置
	
	#vi /etc/sysconfig/network-scripts/ifcfg-eno16777736
		BOOTPROTO=static
		IPADDR=192.168.199.42
		NETMASK=255.255.255.0
		GATEWAY=192.168.199.1
		DNS1=114.114.114.114
		DNS2=114.114.115.115
		NM_CONTROLLED=no
### 2.2 主机名设置
	#hostnamectl set-hostname docker42
	#hostname
![2016-01-08_11-07-05](/uploads/2016/01/2016-01-08_11-07-05.jpg)
	
	#cat /etc/hostname
![2016-01-08_11-15-29](/uploads/2016/01/2016-01-08_11-15-29.jpg)

同理设置另外一台主机。
## 3. 步骤
### 3.1 Docker0 IP设置
在docker41主机上，查看docker的服务文件位置。
	
	#systemctl status docker
![2016-01-08_11-26-26](/uploads/2016/01/2016-01-08_11-26-26.jpg)

修改`docker.service`文件，加入`--bip=172.41.1.1/16`
	
	#vi /usr/lib/systemd/system/docker.service
	    [Unit]
	    Description=Docker Application Container Engine
	    Documentation=https://docs.docker.com
	    After=network.target docker.socket
	    Requires=docker.socket
	
	    [Service]
	    Type=notify
	    ExecStart=/usr/bin/docker daemon --bip=172.41.1.1/16 -H fd:// 
	    MountFlags=slave
	    LimitNOFILE=1048576
	    LimitNPROC=1048576
	    LimitCORE=infinity
	
	    [Install]
	    WantedBy=multi-user.target
   
    #systemctl daemon-reload
    #systemctl restart docker
    
同样修改docker42主机的`docker.service`文件
### 3.2 主机之间的docker0互通
在主机docker41上ping 主机docker42的IP 172.42.1.1，发现不通，原因是没有加入路由。
    
    #route add -net 172.42.0.0/16 gw 192.168.199.42
![2016-01-08_11-36-02](/uploads/2016/01/2016-01-08_11-36-02.jpg)

 发现可以ping通了。同样的在docker42主机加入下面路由
    
    #route add -net 172.41.0.0/16 gw 192.168.199.41
 如果要永久保存，编辑`/etc/sysconfig/network-scripts`下route开头文件，如果没有，可以新建。
    
    #vi /etc/sysconfig/network-scripts/route-eno16777736
    	172.42.0.0/16 via 192.168.199.42

### 3.3 主机之间容器互通
在主机docker41上启动一容器
	
	#docker run --rm=true -it java:latest /bin/bash
ip addr 获得容器ip地址
![2016-01-08_12-25-29](/uploads/2016/01/2016-01-08_12-25-29.jpg)

在主机docker42上ping 容器ip地址，发现目标主机禁止
![2016-01-08_12-29-59](/uploads/2016/01/2016-01-08_12-29-59.jpg)

在主机docker41清空iptable
	
	#iptables -F
	#iptables -t nat -F
![2016-01-08_12-30-52](/uploads/2016/01/2016-01-08_12-30-52.jpg)

这样再ping就通了。
![2016-01-08_12-31-16](/uploads/2016/01/2016-01-08_12-31-16.jpg)

### 3.4 tshark分析ping网络流向
tshark是wireshark的命令行工具，所以在centos可以直接安装
	
	#yun install -y wireshark
我们在docker42主机发送一个ping命令到docker41主机的容器172.41.0.1，然后我们在主机docker41和docker42分别执行下列两条语句监控icmp协议。
	
	#tshark -i eno16777736 -f icmp
	#tshark -i docker0 -f icmp
![2016-01-08_19-57-50](/uploads/2016/01/2016-01-08_19-57-50.jpg)

![2016-01-08_19-58-12](/uploads/2016/01/2016-01-08_19-58-12.jpg)

![2016-01-08_19-58-59](/uploads/2016/01/2016-01-08_19-58-59.jpg)

![2016-01-08_19-58-33](/uploads/2016/01/2016-01-08_19-58-33.jpg)

通过上四图我们可知，ping命令发出路径是主机docker42的docker0到en0，然后通过路由找到docker41的en0和docker0，结果返回的方向正好相反，从主机docker41的docker0到en0，然后返回到docker42的en0和docker0。
### 3.5 tshark分析mysql的数据流向
在主机docker41启动一个mysql容器
	
	#docker run --name mysqlcontainer41 -e MYSQL_ROOT_PASSWORD=p123456 -d mysql:latest
	#docker exec -it mysqlcontainer41 /bin/bash
ip addr查看IP地址
在主机docker42启动一个mysql容器
	
	#docker run --name mysqlcontainer42 -e MYSQL_ROOT_PASSWORD=p123456 -d mysql:latest
	#docker exec -it mysqlcontainer42 /bin/bash
ip addr查看IP地址
从主机docker42的容器去连接主机docker41的mysql容器
	
	#docker exec -it mysqlcontainer42 /bin/bash
	#mysql -uroot -p -h172.41.0.1
我们在主机docker41和docker42分别执行下列两条语句，分别监控docker0及en0的3306端口。
	
	#tshark -i eno16777736 -f 'tcp dst port 3306'
	#tshark -i docker0 -f 'tcp dst port 3306'
我们执行一条select语句，`select * from mysql.user;`看看报文是怎么走向的。
![2016-01-08_21-30-17](/uploads/2016/01/2016-01-08_21-30-17.jpg)

从上图我们可以知道，查询命令发出路径是主机docker42的docker0到en0，然后通过路由找到docker41的en0和docker0，结果返回的方向正好相反，从主机docker41的docker0到en0，然后返回到docker42的en0和docker0。
同样的我们可以通过下面命令打印mysql查询语句。
	
	#tshark -i eno16777736 -f tcp -R 'mysql.query' -T fields -e mysql.query
![2016-01-08_13-58-52](/uploads/2016/01/2016-01-08_13-58-52.jpg)