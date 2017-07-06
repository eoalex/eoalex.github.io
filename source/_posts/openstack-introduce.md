---
title: 'OpenStack入门之一:OpenStack概述'
tags:
  - openstack
id: 1091
categories:
  - 云计算及虚拟化
date: 2016-05-08 23:51:33
---

## 一、OpenStack定义：
OpenStack既是一个社区，也是一个项目和一个开源软件，它提供了一个部署云的操作平台或工具集。其宗旨在于，帮助组织运行为虚拟计算或存储服务的云，为公有云、私有云，也为大云、小云提供可扩展的、灵活的云计算。OpenStack通过各种互补的服务提供了基础设施即服务（IaaS）的解决方案，每个服务提供API以进行集成。
## 二、Openstack项目组成
### 2.1 6个核心项目
1)Compute Service (“Nova”): 计算资源生命周期管理组件
2)NetWork Service (“Neutron”):提供云计算环境下的虚拟网路功能
3)Block Storage Service("Cinder"):管理计算实例所使用的块级存储
4)Object Storage Service("Swift"):对象存储，用于永久类型的静态数据的长期存储
5)Image Service (“Glance”):提供虚拟机镜像的发现注册获取服务
6)Identity Service(“Keystone”):提供了用户信息管理为其他组件提供认证服务
### 2.2 13个可选服务
1)Horizon:Dashboard OpenStack中各种服务的Web管理门户，用于简化用户对服务的操作，例如：启动实例、分配IP地址、配置访问控制等。
2)Ceilometer:Telemetry 像一个漏斗一样，能把OpenStack内部发生的几乎所有的事件都收集起来，然后为计费和监控以及其它服务提供数据支撑。
3)Heat:Orchestration 提供了一种通过模板定义的协同部署方式，实现云基础设施软件运行环境（计算、存储和网络资源）的自动化部署。
4)Trove:Database 为用户在OpenStack的环境提供可扩展和可靠的关系和非关系数据库引擎服务。
5)Sahara:Elastic Map Reduce 刻意快速且经济高效地处理大数据。
6)Ironic:Bare-Metal Provisioning  提供裸机管理服务。
7)Zaqar:Messaging Service  为web开发者提供多租户的云消息服务。
8)Manila:Shared Filesystems 将不同的文件接口接到统一的API上，跨越不同文件系统形成共享池。
9)Designate:DNS Service 提供DNS服务
10)Barbican:Key Management 提供密钥管理
11)Magnum:Containers 容器服务
12)Murano:Application Catalog 应用目录
13)Congress:Governance 治理
## 三、Openstack的发展历史
Openstack自从2010年发布一个版本Austin以来，以每六个月的速度更新一个版本，每个版本以26个英文字母为首字母从A到Z顺序命名。目前最新版本为Mitaka。
![2016-05-09_0-09-53](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-09_0-09-53.jpg)

## 四、Openstack开发语言
所有代码均采用Python开发
## 五、架构设计
![2016-05-09_0-37-50](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-09_0-37-50.jpg)

特点：
1.无中心：可以通过增加组件部署实例来实现水平扩展
2.分布式：由多个逻辑和物理上均可分离的组件共同实现
3.无状态：所有组件无本地持久化状态数据
4.异步执行：部分执行流通过消息机制实现异步执行
5.插件化可配置：大量使用插件机制、配置参数实现灵活的扩展与变更
6.Restful API