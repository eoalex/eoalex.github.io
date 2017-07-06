---
title: Docker 容器网络机制初探(2)-Open vSwitch
tags:
  - Container
  - Docker
  - open vswitch
id: 499
categories:
  - 云计算及虚拟化
date: 2016-01-16 16:30:52
---
## 1. Open vSwitch简介
Open vSwitch的目标，是做一个具有产品级质量的多层虚拟交换机。通过可编程扩展，可以实现大规模网络的自动化（配置、管理、维护）。它支持现有标准管理接口和协议（比如netFlow，sFlow，SPAN，RSPAN，CLI，LACP，802.1ag等，熟悉物理网络维护的管理员可以毫不费力地通过Open vSwitch转向虚拟网络管理）。
相比于Linux bridge，Open vSwitch有以下好处。
* 网络隔离
* QoS配置
* 流量监控，Netflow，sFlow
* 数据包分析，Packet Mirror。
本文我们使用Open vSwitch的GRE通道简单实现下图的网络结构，并用tshark或tcpdump等工具分析网络的流向。
![2016-01-16_15-07-57](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_15-07-57.jpg)

## 2. Open vSwitch安装
### 2.1 环境准备 
centos 7 + docker 1.9.1(准备两台)
### 2.2 安装组件
在docker41主机上
安装Linux网桥配置命令brctl
	
	#yum install bridge-utils
安装编译所需组件
	
	#yum install gcc make python-devel openssl-devel kernel-devel graphviz kernel-debug-devel autoconf automake rpm-build libtool
### 2.3 下载
	#wget http://openvswitch.org/releases/openvswitch-2.4.0.tar.gz
	#mkdir -p /root/rpmbuild/SOURCES
	#cp openvswitch-2.4.0.tar.gz /root/rpmbuild/SOURCES
	#tar -xzf openvswitch-2.4.0.tar.gz
### 2.4 编译
	#cd openvswitch-2.4.0
	#rpmbuild -bb --without check rhel/openvswitch.spec
![2016-01-16_10-09-01](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_10-09-01.jpg)
编译完成，文件存放位置`/root/rpmbuild/RPMS/x86_64`
![2016-01-16_15-35-17](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_15-35-17.jpg)
### 2.5 安装
	#cd /root/rpmbuild/RPMS/x86_64
	#yum install openvswitch-2.4.0-1.x86_64.rpm
![2016-01-16_10-11-32](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_10-11-32.jpg)

	#systemctl start openvswitch.service 
	#systemctl status openvswitch.service 
![2016-01-16_16-11-43](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_16-11-43.jpg)

### 2.6 复制
复制安装文件至另一台主机docker42，并启动服务
	
	#scp openvswitch-2.4.0-1.x86_64.rpm root@192.168.199.42:/root/
![2016-01-16_10-56-53](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_10-56-53.jpg)

## 3. 配置脚本
在主机docker41上,编辑`41-ovs-docker.sh`

```
# Base on http://goldmann.pl/blog/2014/01/21/connecting-docker-containers-on-multiple-hosts/
# Edit this variable: the 'other' host.
REMOTE_IP=192.168.199.42
# Edit this variable: the bridge address on 'this' host.
BRIDGE_ADDRESS=172.17.41.1/24
ROUTE_ADDRESS=172.17.0.0/16
# Name of the bridge and GRE 
DOCKER_BRIDGE=docker0
OVS_BRIDGE=br0
GRE_TUNNEL=gre0
# Deactivate the bridge
ip link set $DOCKER_BRIDGE down
ip link set $OVS_BRIDGE down
# Remove the docker0 bridge
brctl delbr $DOCKER_BRIDGE
# Delete the Open vSwitch bridge
ovs-vsctl del-br $OVS_BRIDGE
# Add the docker bridge
brctl addbr $DOCKER_BRIDGE
# Set up the IP for the docker bridge
ip a add $BRIDGE_ADDRESS dev $DOCKER_BRIDGE
# Add the br0 Open vSwitch bridge
ovs-vsctl add-br $OVS_BRIDGE
# Create the tunnel to the other host and attach it to the ovs bridge
ovs-vsctl add-port $OVS_BRIDGE $GRE_TUNNEL -- set interface $GRE_TUNNEL type=gre options:remote_ip=$REMOTE_IP
# Add the ovs bridge to docker bridge
brctl addif $DOCKER_BRIDGE $OVS_BRIDGE
# Activate the bridge
ip link set $DOCKER_BRIDGE up
ip link set $OVS_BRIDGE up
#add route to docker bridge
ip route add $ROUTE_ADDRESS dev $DOCKER_BRIDGE
# Restart Docker daemon to use the new DOCKER_BRIDGE
systemctl restart docker.service
```	    
    
    #chmode +x 41-ovs-docker.sh
同理在docker42上,编辑`42-ovs-docker.sh`

```    
#Base on http://goldmann.pl/blog/2014/01/21/connecting-docker-containers-on-multiple-hosts/
# Edit this variable: the 'other' host.
REMOTE_IP=192.168.199.41
# Edit this variable: the bridge address on 'this' host.
BRIDGE_ADDRESS=172.17.42.1/24
ROUTE_ADDRESS=172.17.0.0/16
# Name of the bridge and GRE
DOCKER_BRIDGE=docker0
OVS_BRIDGE=br0
GRE_TUNNEL=gre0
# Deactivate the bridge
ip link set $DOCKER_BRIDGE down
ip link set $OVS_BRIDGE down
# Remove the docker0 bridge
brctl delbr $DOCKER_BRIDGE
# Delete the Open vSwitch bridge
ovs-vsctl del-br $OVS_BRIDGE
# Add the docker bridge
brctl addbr $DOCKER_BRIDGE
# Set up the IP for the docker bridge
ip a add $BRIDGE_ADDRESS dev $DOCKER_BRIDGE
# Add the br0 Open vSwitch bridge
ovs-vsctl add-br $OVS_BRIDGE
# Create the tunnel to the other host and attach it to the ovs bridge
ovs-vsctl add-port $OVS_BRIDGE $GRE_TUNNEL -- set interface $GRE_TUNNEL type=gre options:remote_ip=$REMOTE_IP
# Add the ovs bridge to docker bridge
brctl addif $DOCKER_BRIDGE $OVS_BRIDGE
# Activate the bridge
ip link set $DOCKER_BRIDGE up
ip link set $OVS_BRIDGE up
#add route to docker bridge
ip route add $ROUTE_ADDRESS dev $DOCKER_BRIDGE
# Restart Docker daemon to use the new DOCKER_BRIDGE
systemctl restart docker.service
```	
	
	chmode +x 42-ovs-docker.sh
## 4. 测试
在docker41主机执行
	
	#./41-ovs-docker.sh
![2016-01-16_15-48-12](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_15-48-12.jpg)

在docker42主机执行
	
	#./42-ovs-docker.sh
![2016-01-16_15-48-33](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_15-48-33.jpg)

### 4.1 docker0互ping
在docker41主机ping主机docker42的docker0
	
	#ping 172.17.42.1
![2016-01-16_15-50-04](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_15-50-04.jpg)
在docker42主机ping主机docker41的docker0
	
	#ping 172.17.41.1
![2016-01-16_15-50-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_15-50-36.jpg)

### 4.2 不同主机容器互ping
我们测试一下在主机docker41的一个容器内ping主机docker42的一个容器。
在主机docker41启动一个容器，获取ip地址172.17.41.2
	
	#docker run -it --rm=true java:latest /bin/bash
在主机docker42启动一个容器，获取IP地址172.17.42.2
	
	#docker run -it --rm=true java:latest /bin/bash
在容器172.17.41.2内ping172.17.42.2
![2016-01-16_15-55-44](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_15-55-44.jpg)

tshark分析
	
	#tshark -i br0 -R ip proto gre
![2016-01-16_16-07-32](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_16-07-32.jpg)

	#tshark -i eno16777736 ip proto gre
![2016-01-16_16-05-43](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_16-05-43.jpg)

在容器172.17.42.2内ping172.17.41.2
![2016-01-16_15-56-29](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_15-56-29.jpg)

tshark分析
	
	#tshark -i br0 -R ip proto gre
![2016-01-16_16-07-32](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_16-07-32.jpg)
	
	#tshark -i eno16777736 ip proto gre
![2016-01-16_16-04-52](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-16_16-04-52.jpg)