---
title: Docker proxy关停测试
tags:
  - Docker
  - userland-proxy
id: 449
categories:
  - 云计算及虚拟化
date: 2016-01-02 23:56:40
---
## 1. 简介
正常情况下，当启动一个端口映射的容器时，`docker –proxy`进程就会起来，实现宿主机上0.0.0.0地址上对容器的访问代理，在`docker-proxy`加入Docker之后相当长的一段时间内。Docker爱好者普遍感受到，很多场景下，`docker-proxy`并非必需，甚至会带来一些其他的弊端。
影响较大的场景主要有两种。
第一，单个容器需要和宿主机有多个端口的映射。此场景下，若容器需要映射1000个端口甚至更多，那么宿主机上就会创建1000个甚至更多的 `docker-proxy`进程。据不完全测试，每一个`docker-proxy`占用的内存是4-10MB不等。如此一来，直接消耗至少4-10GB内存， 以及至少1000个进程，无论是从系统内存，还是从系统CPU资源来分析，这都会是很大的负担。
第二，众多容器同时存在于宿主机的情况，单个容器映射端口极少。这种场景下，关于宿主机资源的消耗并没有如第一种场景下那样暴力，而且一种较为慢性的方式侵噬资源。
如今，`Docker Daemon`引入`--userland-proxy`这个flag，将以上场景的控制权完全交给了用户，由用户决定是否开启，也为用户的场景的proxy代理提供了灵活性。
![2016-01-02_21-31-27](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-02_21-31-27.jpg)
那么我们怎么通过`--userland-proxy`这个flag来设置呢？不同版本位置稍有不同。
## 2. 演示
本文通过centos 7来演示。
首先我们要知道docker的配置文件安装在哪里。
	
	#systemctl status docker
![2016-01-02_23-19-28](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-02_23-19-28.jpg)
>ubuntu 可以用 `service docker status`查看

![2016-01-03_0-01-14](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-03_0-01-14.jpg)

可以看到centos 7中docker的服务文件在`/usr/lib/systemd/system`目录下。
停止正在运行的容器，停止docker服务，编辑这个服务文件
	
	#vi /usr/lib/systemd/system/docker.service
在`ExecStart=/usr/bin/docker daemon -H fd://` 后加上`--userland-proxy=false`
![2016-01-02_23-25-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-02_23-25-13.jpg)
编辑完成，重新启动docker服务
![2016-01-02_22-50-43](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-02_22-50-43.jpg)
先用`ps –ef|grep docker` 查看docker进程，只有`docker daemon`，那么`docker-proxy`会不会还会起来呢？我们启动一个mysql容器，再查看docker进程，发现`--userland-proxy=false` 这个参数起作用了，`docker-proxy`没有起来。
![2016-01-02_23-05-37](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-02_23-05-37.jpg)
那么访问数据会不会影响呢？我们用其他主机连上mysql数据库，发现一切正常。有兴趣的同学还可以测试是否访问速度是否加快了。
![2016-01-02_23-36-27](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/01/2016-01-02_23-36-27.jpg)