---
title: Docker构建镜像实例
tags:
  - Docker
id: 418
categories:
  - 云计算及虚拟化
date: 2016-01-01 23:58:07
---
## 1. 目标
本文目标是使用centos7为基础镜像，构建一个包括java 8,tomcat7,mysql(mariadb),mycat的镜像，并验证从本机（非宿主机）能访问 8080 ，能用mysql客户端连接mycat的 8066,9066端口。
## 2. 步骤
### 2.1 下拉镜像
首先从仓库下拉centos7镜像，为了加速下拉，我这里选择国内的daoclould作为代理镜像
![2015-12-27_10-04-15](/uploads/2016/01/2015-12-27_10-04-15.jpg)

### 2.2 制作Dockfile
Dockfile

    from centos:7
    MAINTAINER Alex Wu
    RUN yum -y install wget
    RUN wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

    #install java8
    #ADD jdk-8u51-linux-x64.gz /usr/local
    #RUN ln -s /usr/local/jdk1.8.0_51 /usr/local/java
    #ENV JAVA_HOME /usr/local/java
    #ENV PATH $PATH:$JAVA_HOME/bin
    RUN yum -y install java-1.8.0-openjdk

    #install tomcat
    ADD apache-tomcat-7.0.67.tar.gz /usr/local
    RUN ln -s /usr/local/apache-tomcat-7.0.67 /usr/local/tomcat

    #install mysql
    #ADD mariadb-5.5.36-linux-x86_64.tar.gz /usr/local
    #RUN groupadd  -r  mysql &amp;&amp; useradd -g mysql -r mysql
    #RUN ln -s /usr/local/mariadb-5.5.36-linux-x86_64 /usr/local/mysql
    #RUN chown -R mysql:mysql /usr/local/mysql/data
    #RUN /usr/local/mysql/scripts/mysql_install_db --user=mysql --datadir=/usr/local/mysql/data --basedir=/usr/local/mysql
    RUN yum -y install mariadb-server mariadb
    RUN /usr/bin/mysql_install_db --user=mysql --datadir=/var/lib/mysql

    #install mycat
    ADD Mycat-server-1.4-release-20151019230038-linux.tar.gz /opt
    COPY ./schema.xml /opt/mycat/conf
    ENV MYCAT_USER mycatuser
    ENV MYCAT_PASS mycatpass
    RUN sed -i 's/user name="test"/user name=\"'"$MYCAT_USER"'"/' /opt/mycat/conf/server.xml && sed -i 's/name="password">test/name="password" >'"$MYCAT_PASS"'/' /opt/mycat/conf/server.xml

    EXPOSE 8080 8066 9066
    COPY ./start.sh .
    ENTRYPOINT ./start.sh
    
    
### 2.3 Start脚本
    
    #!/bin/bash
    #/usr/local/mysql/bin/mysqld_safe --datadir='/usr/local/mysql/data' &
    /usr/bin/mysqld_safe --datadir=/var/lib/mysql &
    /opt/mycat/bin/mycat start &
    /usr/local/tomcat/bin/catalina.sh run

### 2.4 build镜像
	
	#docker build -t myimage/first .
build 完毕，我们可以看到image
	
	#docker images
![2016-01-01_23-36-21](/uploads/2016/01/2016-01-01_23-36-21.jpg)

### 2.5 启动容器
	docker create --name mycontainer01 -p 8080:8080 -p 8066:8066 -p 9066:9066 myimage/first
	docker start mycontainer01
	docker exec -it mycontainer01 /bin/bash
![2015-12-27_20-31-33](/uploads/2016/01/2015-12-27_20-31-33.jpg)

我们可以看到容器已经起来。
![2016-01-01_23-51-43](/uploads/2016/01/2016-01-01_23-51-43.jpg)
## 3. 验证
### 3.1 访问8080端口
![2015-12-27_20-43-03](/uploads/2016/01/2015-12-27_20-43-03.jpg)

### 3.2 其他主机mysql访问8066，9066端口
	
	mysql -umycatuser -pmycatpass -h192.168.199.111 –P8066
![2015-12-27_20-44-27](/uploads/2016/01/2015-12-27_20-44-27.jpg)

	mysql -umycatuser -pmycatpass -h192.168.199.111 –P9066
![2015-12-27_20-45-51](/uploads/2016/01/2015-12-27_20-45-51.jpg)