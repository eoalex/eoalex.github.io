---
title: Docker 快速搭建私有仓库
tags:
  - Docker
  - Registry
id: 596
categories:
  - 云计算及虚拟化
date: 2016-01-31 21:19:37
---
## 1. 简介
[Docker Hub](https://hub.docker.com/explore/)这样的公共仓库虽然给了用户很多便利，但是对大多数用户来说，建立本地私有仓库还是很有必要的。本文以centos 7 为例介绍如何快速建立本地私有仓库。
## 2. 安装
### 2.1 准备环境
准备centos 7 主机一台（IP:192.168.199.39)。
### 2.2 安装Docker Engine
增加docker 官方yum库

    #sudo tee /etc/yum.repos.d/docker.repo &lt;&lt;-'EOF'
	    [dockerrepo]
	    name=Docker Repository
	    baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
	    enabled=1
	    gpgcheck=1
	    gpgkey=https://yum.dockerproject.org/gpg
	    EOF
安装
    
    #sudo yum install docker-engine
启动docker服务

    #sudo service docker start
### 2.3 安装私有仓库
好了，现在我们可以下拉`docker-registry`镜像了，它是官方提供的工具，可以用于构建私有的镜像仓库。
    
    #docker pull registry
建立本地私有仓库地址

    #mkdir -pv /opt/data/registry
启动私有仓库容器

    docker run -d -p 5000:5000 --restart=always --name registry \
      -v /opt/data/registry:/var/lib/registry \
      registry

好了，其实现在已经搭建好私有仓库了，很简单吧。
## 3. 测试
### 3.1 测试本机上传镜像至私有仓库
我们先下拉一个镜像

    #docker pull centos:7
打个标签（本机IP+PORT)

    #docker tag centos:7 127.0.0.1:5000/centos:7
上传

    #docker push 127.0.0.1:5000/centos:7
在本机可以用下面命令访问私有仓库

    #curl 127.0.0.1:5000/v1/search
也可以从浏览器访问

    http://192.168.199.39:5000/v1/search
### 3.2 测试其它主机从私有仓库下载镜像
首先修改docker配置文件，重启docker服务

    #vi /etc/sysconfig/docker
	    OPTIONS='--selinux-enabled --insecure-registry 192.168.199.39:5000'

	#systemctl restart docker
使用下面命令我们就可以下载镜像了。
	
	#docker pull 192.168.199.39:5000/centos:7

参考：
[https://docs.docker.com/registry/deploying/](https://docs.docker.com/registry/deploying/)
[https://docs.docker.com/engine/installation/centos/](https://docs.docker.com/engine/installation/centos/)