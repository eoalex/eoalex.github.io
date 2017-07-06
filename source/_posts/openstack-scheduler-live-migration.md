---
title: 'OpenStack入门之六:调度服务与在线迁移实验'
tags:
  - openstack
id: 1171
categories:
  - 云计算及虚拟化
date: 2016-06-21 08:40:01
---

## 1. 简介
本文我们在[OpenStack入门之三:Kilo版本centos 7平台搭建](/2016/06/openstack-kilo-centos/)基础上设置调度服务与在线迁移实验。
为了完成调度，我们需要设置一个新的计算节点compute-node-02。（具体设置过程略)
![2016-06-19_17-30-25](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_17-30-25.jpg)

## 2. 调度服务
### 2.1 创建主机集
	#source /root/admin-openrc.sh
	#nova aggregate-create mynova01 nova

    [root@controller ~]# nova aggregate-create mynova01 nova
    +----+----------+-------------------+-------+--------------------------+
    | Id | Name     | Availability Zone | Hosts | Metadata                 |
    +----+----------+-------------------+-------+--------------------------+
    | 1  | mynova01 | nova              |       | 'availability_zone=nova' |
    +----+----------+-------------------+-------+--------------------------+
    [root@controller ~]# 
    
### 2.2 加入主机
    #nova aggregate-add-host 1 compute-node-02
    
    [root@controller ~]# nova aggregate-add-host 1 compute-node-02
    Host compute-node-02 has been successfully added for aggregate 1 
    +----+----------+-------------------+-------------------+--------------------------+
    | Id | Name     | Availability Zone | Hosts             | Metadata                 |
    +----+----------+-------------------+-------------------+--------------------------+
    | 1  | mynova01 | nova              | 'compute-node-02' | 'availability_zone=nova' |
    +----+----------+-------------------+-------------------+--------------------------+
    [root@controller ~]# 
    
### 2.3 设置元数据
    #nova aggregate-set-metadata 1 test02=true
    
    [root@controller ~]# nova aggregate-set-metadata 1 test02=true
    Metadata has been successfully updated for aggregate 1.
    +----+----------+-------------------+-------------------+-----------------------------------------+
    | Id | Name     | Availability Zone | Hosts             | Metadata                                |
    +----+----------+-------------------+-------------------+-----------------------------------------+
    | 1  | mynova01 | nova              | 'compute-node-02' | 'availability_zone=nova', 'test02=true' |
    +----+----------+-------------------+-------------------+-----------------------------------------+
    [root@controller ~]# 
    
    #nova flavor-key m1.nano set test02=true
### 2.4 修改控制节点上的nova.conf
  		  `scheduler_default_filters=AggregateInstanceExtraSpecsFilter,AvailabilityZoneFilter,RamFilter,ComputeFilter`
    
### 2.5 重启
    #systemctl restart openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-conductor.service
    
### 2.6.创建主机
#### 2.6.1 生成sshkey
    #ssh-keygen -t rsa -f cloud.key
将生成的公钥复制在新增的key pair页面。
![2016-06-19_21-52-42](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_21-52-42.jpg)

#### 2.6.2 设置启动后更改密码
![2016-06-19_21-52-05](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_21-52-05.jpg)

### 2.7 验证
我们看到主机已启动并运行在compute-node-02上
![2016-06-19_21-55-09](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_21-55-09.jpg)
使用更改后的密码登录虚拟机
![2016-06-19_21-56-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_21-56-36.jpg)

## 3. 虚拟机在线迁移
### 3.1 计算节点更新组件
    #yum install libvirt qemu-* -y
### 3.2修改计算节点相关配置
修改/etc/sysconfig/libvirtd 文件。
    
    LIBVIRTD_ARGS="--listen"
    
在/etc/libvirt/libvirtd.conf 文件中做如下配置。
    
    listen_tls=0
    listen_tcp=1
    auth_tcp="none"

### 3.3 重启计算主机
Reboot
### 3.4 在线移动
#### 2.4.1 设置
![2016-06-19_23-13-29](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_23-13-29.jpg)

#### 2.4.2 移动中
![2016-06-19_23-22-15](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_23-22-15.jpg)

#### 2.4.3 移动完成
等待一段时间，我们看到host已变为compute-node-01了。

![2016-06-19_23-35-25](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_23-35-25.jpg)

![2016-06-19_23-42-47](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_23-42-47.jpg)

![2016-06-19_23-43-09](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-19_23-43-09.jpg)

从上面的截图我们看到存放虚拟机信息的目录文件从compute-node-02实际移至compute-node-01了。由于在两台主机之间迁移，性能和速度可想而知了，下节我们将实验基于NFS共享存储进行在线迁移，以达到秒速的迁移体验。