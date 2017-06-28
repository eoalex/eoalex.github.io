---
title: Kubernetes入门实战(6)：使用ku8eye快速构建Kubernetes集群
tags:
  - ku8eye
  - Kubernetes
id: 1036
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-04-10 13:44:59
---

[Ku8eye](https://github.com/bestcloud/ku8eye)是Kubernetes的Web一站式管理系统,具有如下的目标：
1.图形化一键安装部署多节点的Kubernetes集群。是安装部署谷歌Kubernetes集群的最快以及最佳方式，安装流程会参考当前系统环境，提供默认优化的集群安装参数，实现最佳部署。
2.支持多角色多租户的Portal管理界面。通过一个集中化的Portal界面，运营团队可以很方便的调整集群配置以及管理集群资源，实现跨部门的角色及用户管理、多租户管理，通过自助服务可以很容易完成Kubernetes集群的运维管理工作。
3.制定一个Kubernetes应用的程序发布包标准(ku8package)并提供一个向导工具，使得专门为Kubernetes设计的应用能够很容易从本地环境中发布到公有云和其他环境中，更进一步的，我们还提供了Kubernetes应用可视化的构建工具，实现Kubernetes Service、RC、Pod以及其他资源的可视化构建和管理功能
4.可定制化的监控和告警系统。内建很多系统健康检查工具用来检测和发现异常并触发告警事件，不仅可以监控集群中的所有节点和组件（包括Docker与Kubernetes），还能够很容易的监控业务应用的性能，我们提供了一个强大的Dashboard，可以用来生成各种复杂的监控图表以展示历史信息，并且可以用来自定义相关监控指标的告警阀值。
5.具备的综合的、全面的故障排查能力。平台提供唯一的、集中化的日志管理工具，日志系统从集群中各个节点拉取日志并做聚合分析，拉取的日志包括系统日志和用户程序日志，并且提供全文检索能力以方便故障分析和问题排查，检索的信息包括相关告警信息，而历史视图和相关的度量数据则告诉你，什么时候发生了什么事情，有助于快速了解相关时间内系统的行为特征。
6.实现Dockers与kubernetes项目的持续集成功能。提供一个可视化工具驱动持续集成的整个流程，包括创建新的Docker镜像、Push镜像到私有仓库中、创建一个Kubernetes测试环境进行测试以及最终滚动升级到生产环境中等各个主要环节。本文就从制作ku8eye镜像文件开始，演示如何使用ku8eye。
准备四台虚拟机，环境centos 7,内存2G。
<table>
<tbody>
<tr>
<td width="120">**IP**</td>
<td width="120">**功能**</td>
</tr>	
<tr>
<td width="120">192.168.199.50</td>
<td width="120">Ku8eye</td>
</tr>
<tr>
<td width="120">192.168.199.54</td>
<td width="120">Master</td>
</tr>
<tr>
<td width="120">192.168.199.55</td>
<td width="120">Node1</td>
</tr>	
<tr>
<td width="120">192.168.199.56</td>
<td width="120">Node2</td>
</tr>		
</tbody>
</table>
一、制作Ku8eye镜像文件
1.Ku8eye镜像文件

    from centos:7
    MAINTAINER Alex Wu
    RUN yum -y install wget
    #get update from aliyun
    RUN wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    # set timezone
    ENV TZ Asia/Shanghai
    # 1\. install ansible 
    RUN yum clean all &amp;&amp; \
        yum -y install epel-release &amp;&amp; \
        yum -y install PyYAML python-jinja2 python-httplib2 python-keyczar python-paramiko python-setuptools git python-pip
    RUN mkdir /etc/ansible/ &amp;&amp; echo -e '[local]\nlocalhost' &gt; /etc/ansible/hosts
    RUN pip install ansible
    # 2\. install sshpass, and generate ssh keys
    RUN yum -y install sshpass
    RUN ssh-keygen -q -t rsa -N "" -f ~/.ssh/id_rsa
    # make ansible not do key checking from ~/.ssh/known_hosts file
    ENV ANSIBLE_HOST_KEY_CHECKING false
    # 3\. install MariaDB (mysql)
    RUN yum -y install MariaDB-server MariaDB-client
    # 4\. install supervisor 
    RUN pip install supervisor
    # 5\. add JRE1.8
    COPY jre1.8.0_65 /root/jre1.8.0_65
    ENV JAVA_HOME="/root/jre1.8.0_65" PATH="$PATH:/root/jre1.8.0_65/bin"
    RUN chmod +x /root/jre1.8.0_65/bin/*
    # 6\. install openssh
    RUN yum install -y openssh openssh-server
    RUN mkdir -p /var/run/sshd &amp;&amp; echo "root:root" | chpasswd
    RUN /usr/sbin/sshd-keygen
    RUN sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config &amp;&amp; sed -ri 's/#UsePAM no/UsePAM no/g' /etc/ssh/sshd_config
    # 7\. add ku8eye-ansible binary and config files
    COPY kubernetes_cluster_setup /root/kubernetes_cluster_setup
    # 8\. copy shell scripts, SQL scripts, config files 
    # db init SQL
    COPY db_scripts /root/db_scripts
    # shell scripts
    COPY shell_scripts /root/shell_scripts
    RUN chmod +x /root/shell_scripts/*.sh
    COPY ku8eye-startup.sh /root/ku8eye-startup.sh
    RUN chmod +x /root/ku8eye-startup.sh
    # latest jar
    COPY ku8eye-web.jar /root/ku8eye-web.jar
    # 9\. start mariadb, init db data, and start ku8eye-web app
    # supervisor config file
    COPY supervisord.conf /etc/supervisord.conf
    ENTRYPOINT /usr/bin/supervisord
    `</pre>
    2.制作镜像
    #docker build -t myku8eye .
    [![2016-03-26_14-46-50](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-03-26_14-46-50.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-03-26_14-46-50.jpg)
    <pre>`docker images
    REPOSITORY                              TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
    myku8eye                                latest              af10ef44e73d        3 hours ago         2.25 GB
    `</pre>
    二、启动安装
    1.启动容器
    <pre>`docker run -tid --name ku8eye-web -p 3306:3306 -p 8080:8080 -p 9001:9001 myku8eye`</pre>
    2.进入容器
    <pre>`docker exec -ti ku8eye-web /bin/bash`</pre>
    3.启动安装命令
    ku8eye支持图形化一键安装及命令行一键安装，这里使用命令行一键安装。
    <pre>`/root/ku8eye-startup.sh "192.168.199.54,192.168.199.55,192.168.199.56" "172.0.0.0/16" "123456"

安装过程信息略...
最后显示
Node  192.168.199.54  sumary:SUCESS
Node  192.168.199.55  sumary:SUCESS
Node  192.168.199.56  sumary:SUCESS

在master主机我们查看，我们看到kube nodes已启动。
[![2016-03-26_14-36-58](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-03-26_14-36-58.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-03-26_14-36-58.jpg)
[![2016-03-26_14-37-54](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-03-26_14-37-54.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-03-26_14-37-54.jpg)
我们看到build镜像文件后，我们只要一个命令就可以轻松安装kubernetes集群。
三、web界面
访问http://192.168.199.50:8080端口，看到ku8 manager已启动。
[![2016-04-10_14-00-26](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-04-10_14-00-26.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-04-10_14-00-26.jpg)
使用guest/123456登录，我们可以看到左侧菜单有资源管理，应用管理及集群监控。有兴趣的朋友自己测试一下，这里不再赘述。
[![2016-04-10_14-05-42](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-04-10_14-05-42.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/04/2016-04-10_14-05-42.jpg)

参考：
https://github.com/bestcloud/ku8eye/blob/master/doc/ku8eye-web-dev-env.md