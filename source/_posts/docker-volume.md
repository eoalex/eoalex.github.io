---
title: Docker Volume定义及实例应用
tags:
  - Docker
id: 432
categories:
  - 云计算及虚拟化
date: 2016-01-02 15:46:11
---
## 1. 简介
Volume真正的目的是将容器以及容器产生的数据分离开来。如果你删除容器的时候，Volume的数据不会自动删除，除非删除容器的时候加了-v选项。很多人都对Volume有一个误解，他们认为Volume是为了持久化，但其实容器已经持久化了，容器自创建后会一直存在，除非你删除它们。
Volume可以使用以下两种方式创建,这两种方式有些细小而又重要的差别。
* 在Dockerfile中指定`VOLUME /some/dir`
* 执行`docker run -v /some/dir`命令来指定

无论哪种方式都是做了同样的事情。它们告诉Docker在主机上创建一个目录（默认情况下是在/var/lib/docker下），然后将其挂载到指定的路径。当删除使用该Volume的容器时，Volume本身不会受到影响，它可以一直存在下去。如果在容器中不存在指定的路径，那么该目录将会被自动创建。
## 2. 实例
下面我们将通过实例来加深理解。
这是一个官方mysql镜像的[Dockerfile](https://github.com/docker-library/mysql/blob/master/5.7/Dockerfile)

```
FROM debian:jessie
# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mysql && useradd -r -g mysql mysql
RUN mkdir /docker-entrypoint-initdb.d
# FATAL ERROR: please install the following Perl modules before executing /usr/local/mysql/scripts/mysql_install_db:
# File::Basename
# File::Copy
# Sys::Hostname
# Data::Dumper
RUN apt-get update && apt-get install -y perl pwgen --no-install-recommends && rm -rf /var/lib/apt/lists/*

# gpg: key 5072E1F5: public key "MySQL Release Engineering " imported
RUN apt-key adv --keyserver ha.pool.sks-keyservers.net --recv-keys A4A9406876FCBD3C456770C88C718D3B5072E1F5

ENV MYSQL_MAJOR 5.7
ENV MYSQL_VERSION 5.7.10-1debian8

RUN echo "deb http://repo.mysql.com/apt/debian/ jessie mysql-${MYSQL_MAJOR}" > /etc/apt/sources.list.d/mysql.list

# the "/var/lib/mysql" stuff here is because the mysql-server postinst doesn't have an explicit way to disable the mysql_install_db codepath besides having a database already "configured" (ie, stuff in /var/lib/mysql/mysql)
# also, we set debconf keys to make APT a little quieter
RUN { \
	echo mysql-community-server mysql-community-server/data-dir select ''; \
	echo mysql-community-server mysql-community-server/root-pass password ''; \
	echo mysql-community-server mysql-community-server/re-root-pass password ''; \
	echo mysql-community-server mysql-community-server/remove-test-db select false; \
	} | debconf-set-selections \
	&& apt-get update && apt-get install -y mysql-server="${MYSQL_VERSION}" && rm -rf /var/lib/apt/lists/* \
	&& rm -rf /var/lib/mysql &amp;&amp; mkdir -p /var/lib/mysql

# comment out a few problematic configuration values
# don't reverse lookup hostnames, they are usually another container
RUN sed -Ei 's/^(bind-address|log)/#&/' /etc/mysql/my.cnf \
	&amp;&amp; echo 'skip-host-cache\nskip-name-resolve' | awk '{ print } $1 == "[mysqld]" && c == 0 { c = 1; system("cat") }' /etc/mysql/my.cnf > /tmp/my.cnf \
	&amp;&amp; mv /tmp/my.cnf /etc/mysql/my.cnf

VOLUME /var/lib/mysql

COPY docker-entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]

EXPOSE 3306
CMD ["mysqld"]
```

我们看到Dockerfile里指定了VOLUME /var/lib/mysql。
### 2.1 实验1
我们通过以下命令启动一个容器
	
	docker run --name mysqlcontainer01 -e MYSQL_ROOT_PASSWORD=p123456 -d mysql:latest
![2016-01-02_15-34-19](/uploads/2016/01/2016-01-02_15-34-19.jpg)
通过`docker inspect mysqlcontainer01`可以看到mount了一个目录在主机`/var/lib/docker/volumes`下
![2016-01-02_15-33-19](/uploads/2016/01/2016-01-02_15-33-19.jpg)
我们可以看到这个目录其实就是容器中mysql创建的数据库目录`/var/lib/mysql`
![2016-01-02_15-40-07](/uploads/2016/01/2016-01-02_15-40-07.jpg)
现在我们试图停止并删除容器，再看一下`/var/lib/docker/volumes`目录下是否还存在。
![2016-01-02_15-43-29](/uploads/2016/01/2016-01-02_15-43-29.jpg)
我们看到启动容器时创建的目录还是存在的，而这个容器现在已经删除，通过这个Volume保证了容器中创建的数据还存在。

### 2.2 实验2
我们再通过以下命令创建一个容器

	docker run --name mysqlcontainer02 -v /data/mysqldb:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=p123456 -d mysql:latest

事先，主机已经有一个mysql数据库目录`/data/mysqldb`
![2016-01-02_17-22-52](/uploads/2016/01/2016-01-02_17-22-52.jpg)
我们看到已经有个数据库test目录及一个testfile文件，我们看容器启动后是否会挂载至容器中。
![2016-01-02_17-27-21](/uploads/2016/01/2016-01-02_17-27-21.jpg)
我们看到确实已经有test数据库了。容器中查看`/var/lib/mysql`也可以看到数据库test目录及一个testfile文件
![2016-01-02_17-28-21](/uploads/2016/01/2016-01-02_17-28-21.jpg)

我们再在容器中`/var/lib/mysql` 下新增一个文件testfile2
![2016-01-02_17-40-18](/uploads/2016/01/2016-01-02_17-40-18.jpg)

然后在主机`/data/mysqldb`看到也有了
![2016-01-02_17-41-10](/uploads/2016/01/2016-01-02_17-41-10.jpg)
这就证明通过volume挂载使得主机跟容器数据互通。