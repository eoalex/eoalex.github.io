---
title: Kubernetes入门实战(1)：基于redis和docker的guestbook留言簿案例
tags:
  - Docker
  - Kubernetes
  - Redis
id: 604
categories:
  - 云计算及虚拟化
date: 2016-02-04 22:14:38
---

## 1. Kubernetes介绍

### 1.1 简介

Kubernetes是什么？
首先，它是一个全新的基于容器技术的分布式架构领先方案。
其次，它是一个开放的开发平台。
最后，它是一个完备的分布式系统支撑平台。
Kubernetes是Google团队发起的开源项目，它的目标是管理跨多个主机的容器，提供基本的部署，维护以及运用伸缩，主要实现语言为Go语言。
Kubernetes特点是：
•易学：轻量级，简单，容易理解
•便携：支持公有云，私有云，混合云，以及多种云平台
•可拓展：模块化，可插拔，支持钩子，可任意组合
•自修复：自动重调度，自动重启，自动复制。
Kubernets目前在https://github.com/kubernetes/kubernetes进行维护。

### 1.2 基本概念

* **Node(节点)：**在Kubernetes中，节点是实际工作的点，较早版本称为Minion。节点可以是虚拟机或者物理机器，依赖于一个集群环境。每个节点都有一些必要的服务以运行Pod容器组，并且它们都可以通过主节点来管理。在Node上运行的服务进程包括docker daemon，Kubelet和 Kube-Proxy。
* **Pod(容器组)：**是Kubernetes的基本操作单元，把相关的一个或多个容器构成一个Pod，通常Pod里的容器运行相同的应用。Pod包含的容器运行在同一个节点上，看作一个统一管理单元，共享相同的volumes和network namespace/IP和Port空间。
* **Pod的生命周期：**Pod的生命周期是通过Replication Controller来管理的。在整个过程中，Pod处于4种状态之一：Pending, Running, Succeeded, Failed。
* **Replication Controller(RC)：**用于定义Pod副本的数量。确保任何时候Kubernetes集群中有指定数量的Pod副本在运行， 如果少于指定数量的Pod副本，Replication Controller会启动新的Pod，反之会杀死多余的以保证数量不变。
* **Service(服务)：**一个Service可以看作一组提供相同服务的Pod的对外访问接口。
* **Volume(存储卷)：**Volume是Pod中能够被多个容器访问的共享目录。
* **Label(标签)：**用于区分Pod、Service、Replication Controller的key/value键值对，Pod、Service、 Replication Controller可以有多个label，但是每个label的key只能对应一个value。Labels是Service和Replication Controller运行的基础，为了将访问Service的请求转发给后端提供服务的多个容器，正是通过标识容器的labels来选择正确的容器。同样，Replication Controller也使用labels来管理通过pod 模板创建的一组容器，这样Replication Controller可以更加容易，方便地管理多个容器，无论有多少容器。
* **Proxy(代理)：**是为了解决外部网络能够访问跨机器集群中容器提供的应用服务而设计的。Proxy提供TCP/UDP sockets的proxy，每创建一种Service，Proxy主要从etcd获取Services和Endpoints的配置信息，或者也可以从file获取，然后根据配置信息在Minion上启动一个Proxy的进程并监听相应的服务端口，当外部请求发生时，Proxy会根据Load Balancer将请求分发到后端正确的容器处理。
* **Namespace(命名空间)：**通过将系统内部的对象“分配”到不同的Namespace中，形成逻辑上的不同分组，便于在共享使用整个集群的资源同时还能分别管理。
* **Annotation(注解)：**与Label类似，但Label定义的是对象的元数据，而Annotation则是用户任意定义的“附加”信息。

### 1.3 组件

#### 1.3.1 Master运行三个组件：

* **apiserver：**作为kubernetes系统的入口，封装了核心对象的增删改查操作，以RESTFul接口方式提供给外部客户和内部组件调用。它维护的REST对象将持久化到etcd（一个分布式强一致性的key/value存储）。
* **scheduler：**负责集群的资源调度，为新建的Pod分配机器。这部分工作分出来变成一个组件，意味着可以很方便地替换成其他的调度器。
* **controller-manager：**负责执行各种控制器，目前有两类：
(1)endpoint-controller：定期关联service和Pod(关联信息由endpoint对象维护)，保证service到Pod的映射总是最新的。
(2)replication-controller：定期关联replicationController和Pod，保证replicationController定义的复制数量与实际运行Pod的数量总是一致的。

#### 1.3.2 Worker运行两个组件：

* **kubelet：**负责管控docker容器，如启动/停止、监控运行状态等。它会定期从etcd获取分配到本机的Pod，并根据Pod信息启动或停止相应的容器。同时，它也会接收apiserver的HTTP请求，汇报Pod的运行状态。
* **proxy：**负责为Pod提供代理。它会定期从etcd获取所有的service，并根据service信息创建代理。当某个客户Pod要访问其他Pod时，访问请求会经过本机proxy做转发。

## 2. Kubernetes安装

### 2.1 关闭自带的防火墙服务

	#systemctl disable firewalld
	#systemctl stop firewalld

### 2.2 安装

安装etcd和Kubernetes软件，会自动安装docker
	
	#yum install -y etcd kubernetes

### 2.3 修改docker配置文件

	#vi /etc/sysconfig/docker
    OPTIONS='--selinux-enabled=false --insecure-registry gcr.io'
### 2.4 修改apiserver配置文件
 删除以下代码中的ServiceAccount
    
    #vi /etc/kubernetes/apiserver
		KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
   
### 2.5 启动所有服务

    #systemctl start etcd
    #systemctl start docker
    #systemctl start kube-apiserver
    #systemctl start kube-controller-manager
    #systemctl start kube-scheduler
    #systemctl start kubelet
    #systemctl start kube-proxy

## 3. Guestbook部署

* **redis-master:** 用于前端Web应用进行“写”留言的操作，其中已经保存了一条内容为“Hello World！”的留言。
* **guestbook-redis-slave:** 用于前端Web应用进行“读”留言的操作，并与redis-master的数据保持同步。
* **guestbook-php-frontend:** PHP Web服务，在网页上展示留言内容，也提供一个文本输入框供访客添加留言。

### 3.1 下载镜像

    #docker pull kubeguide/redis-master
    #docker pull kubeguide/guestbook-redis-slave
    #docker pull kubeguide/guestbook-php-frontend
![2016-02-01_22-01-11](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-01_22-01-11.jpg)

### 3.2 设置工作目录

    #mkdir kube-guestbook
    #cd kube-guestbook

### 3.3 创建redis-master Pod和服务

    #vi redis-master-controller.yaml
		apiVersion: v1
    	kind: ReplicationController
    	metadata:
      	name: redis-master
      	labels:
        name: redis-master
    	spec:
      		replicas: 1
      		selector:
        		name: redis-master
      template:
        metadata:
          labels:
            name: redis-master
        spec:
          containers:
          - name: master
            image: docker.io/kubeguide/redis-master
            ports:
            - containerPort: 6379

 创建redis-master Pod
 
    #kubectl create -f redis-master-controller.yaml
    #kubectl get pods
 一开始pod在pending状态
 ![2016-02-03_19-38-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-03_19-38-36.jpg)
 第一次启动容器时间比较久，如果没什么问题，状态会更新为Running。
 ![2016-02-03_21-18-38](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-03_21-18-38.jpg)
 
    #vi redis-master-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: redis-master
      labels:
        name: redis-master
    spec:
      ports:
        # the port that this service should serve on
      - port: 6379
        targetPort: 6379
      selector:
        name: redis-master`</pre>

创建redis-master服务

    #kubectl create -f redis-master-service.yaml
    #kubectl get services
![2016-02-03_21-21-15](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-03_21-21-15.jpg)

### 3.4 创建redis-slave Pod和服务

    #vi redis-slave-controller.yaml
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: redis-slave
      labels:
        name: redis-slave
    spec:
      replicas: 2
      selector:
        name: redis-slave
      template:
        metadata:
          labels:
            name: redis-slave
        spec:
          containers:
          - name: slave
            image: docker.io/kubeguide/guestbook-redis-slave
            env:
            - name: GET_HOSTS_FROM
              value: env
            ports:
            - containerPort: 6379

创建redis-slave Pod
    
    #kubectl create -f redis-slave-controller.yaml
    #kubectl get rc
![2016-02-03_21-23-31](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-03_21-23-31.jpg)

    #kubectl get pods
![2016-02-03_21-23-50](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-03_21-23-50.jpg)

    #vi redis-slave-service.yaml
	  	 apiVersion: v1
	    kind: Service
	    metadata:
	      name: redis-slave
	      labels:
	        name: redis-slave
	    spec:
	      ports:
	      - port: 6379
	      selector:
	        name: redis-slave

创建redis-slave服务

    #kubectl create -f redis-slave-service.yaml
    #kubectl get services
![2016-02-03_21-25-17](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-03_21-25-17.jpg)

### 3.5 创建frontend Pod和服务

    #vi frontend-controller.yaml
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: frontend
      labels:
        name: frontend
    spec:
      replicas: 3
      selector:
        name: frontend
      template:
        metadata:
          labels:
            name: frontend
        spec:
          containers:
          - name: php-redis
            image: docker.io/kubeguide/guestbook-php-frontend
            env:
            - name: GET_HOSTS_FROM
              value: env
            ports:
            - containerPort: 80

创建frontend Pod

    #kubectl create -f frontend-controller.yaml
    #kubectl get rc
![2016-02-03_21-27-09](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-03_21-27-09.jpg)

    #kubectl get pods
![2016-02-03_21-27-18](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-03_21-27-18.jpg)
    
    #vi frontend-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: frontend
      labels:
        name: frontend
    spec:
      type: NodePort
      ports:
        - port: 80
          nodePort: 30001    
      selector:
        name: frontend

创建frontend服务
	
	#kubectl create -f frontend-service.yaml
	#kubectl get services
![2016-02-04_11-41-15](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-04_11-41-15.jpg)

### 3.6 通过浏览器访问网页

访问主机30001端口，我们看到网页已经默认有一条Hello World！
![2016-02-04_9-37-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-04_9-37-36.jpg)

我们再输入新的留言Hello Kubernetes！然后submit，看到下面多出来一条新的留言。
![2016-02-04_9-38-38](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/02/2016-02-04_9-38-38.jpg)

### 3.7 停止Pod和服务

	#kubectl stop rc -l "name in (redis-master, redis-slave, frontend)"
	#kubectl delete service -l "name in (redis-master, redis-slave, frontend)"

参考：
[Guestbook Example](http://kubernetes.io/v1.1/examples/guestbook/README.html)
[Kubernetes权威指南](http://item.jd.com/11847020.html)