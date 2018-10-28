---
title: 'OpenStack入门之七:基于NFS共享存储的在线迁移'
tags:
  - openstack
id: 1186
categories:
  - 云计算及虚拟化
date: 2016-06-21 09:14:22
---

## 1. 简介
上文我们做了[虚拟机的在线迁移实验](/2016/06/openstack-scheduler-live-migration/)，由于在两台主机之间迁移，性能和速度可想而知了。本文我们将实验基于NFS共享存储进行在线迁移，以达到秒速的迁移体验。

## 2. 在NFS服务器上设置挂载点
### 2.1 安装NFS
	#yum install -y nfs-utils
### 2.2 创建共享目录
	#mkdir /data
### 2.3 修改/etc/exports文件
添加以下一行

    /data compute-node-01(rw,sync,fsid=0,no_root_squash)  compute-node-02(rw,sync,fsid=0,no_root_squash)

### 2.4 启动nfs服务

	systemctl enable rpcbind.service
	systemctl enable nfs-server.service
	systemctl start rpcbind.service
	systemctl start nfs-server.service

## 3. 在所有计算节点上完成以下操作
### 3.1 安装nfs
	#yum install -y nfs-utils
	#systemctl enable rpcbind.service
	#systemctl start rpcbind.service
### 3.2 挂载
	#mount -t nfs 192.168.199.87:/data /var/lib/nova/instances
192.168.199.87是nfs服务器的IP.
### 3.3 重新设置文件夹所有者及权限
	#chown -R nova:nova /var/lib/nova/instances

## 4. 迁移
### 4.1 创建虚拟机
为了更清楚的演示，我们将之前的虚拟机删除，重新创建一个新的虚拟机。
![2016-06-20_0-45-06](/uploads/2016/06/2016-06-20_0-45-06.jpg)

按照[上文](/2016/06/openstack-scheduler-live-migration/)的定义，我们看到新的虚拟机安装在compute-node-02上。
![2016-06-20_0-46-25](/uploads/2016/06/2016-06-20_0-46-25.jpg)

### 4.2 开始热迁移
![2016-06-20_0-47-05](/uploads/2016/06/2016-06-20_0-47-05.jpg)

![2016-06-20_0-47-24](/uploads/2016/06/2016-06-20_0-47-24.jpg)

### 4.3 迁移完成（很快）
![2016-06-20_0-47-58](/uploads/2016/06/2016-06-20_0-47-58.jpg)
看目录文件，compute-node-01和compute-node-02迁移前后没有变化。

![2016-06-20_0-48-54](/uploads/2016/06/2016-06-20_0-48-54.jpg)

![2016-06-20_0-49-06](/uploads/2016/06/2016-06-20_0-49-06.jpg)

其实他们都共享了nfs server 的data目录，这也是迁移速度很快的原因，也是使用NFS的好处

![2016-06-20_0-54-23](/uploads/2016/06/2016-06-20_0-54-23.jpg)