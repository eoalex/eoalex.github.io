---
title: Redis加速你的WordPress站点
tags:
  - Redis
  - WordPress
id: 1297
categories:
  - 数据库
  - 运维
date: 2016-10-10 20:25:15
---
## 1. 简介
Redis是一个开源的内存存储的数据结构服务器，可用作数据库，高速缓存和消息队列代理。WordPress是一种使用PHP语言开发的博客平台，用户可以在支持PHP和MySQL数据库的服务器上架设属于自己的网站。也可以把 WordPress当作一个内容管理系统（CMS）来使用。WordPress简单又功能强大让大家爱不释手，但也导致了WordPress在架构大型网站和博客时成为了消耗资源“大户”，如何让Wordpress更好更有效率地运行，是我们一直不断追求的目标。那么是否可用Redis来做Wordpress的缓存来加速访问呢？答案是肯定的。步骤也非常简单。

## 2. 安装Redis
首先要安装Redis。访问[Redis官网](http://redis.io/download),下载源代码，并编译。

    wget http://download.redis.io/releases/redis-3.2.4.tar.gz
    tar xzf redis-3.2.4.tar.gz
    cd redis-3.2.4
    make
安装过程中，如果发现没有编译器，可安装gcc，
    
    sudo yum install gcc
    
## 3. 设置Redis
建立redis命令，配置，log等文件存放的文件夹。
    
    sudo mkdir -p /usr/local/redis/{bin,var,etc}
    cd src/
    sudo cp redis-benchmark redis-check-aof redis-check-rdb redis-cli redis-sentinel redis-server /usr/local/redis/bin/
    sudo cp ../redis.conf /usr/local/redis/etc
    sudo ln -s /usr/local/redis/bin/* /usr/bin/
    
编辑配置文件

    sudo vim /usr/local/redis/etc/redis.conf
    
设置以下参数
    
    daemonize yes
    bind 127.0.0.1
    logfile /usr/local/redis/var/redis.log
    dir /usr/local/redis/var
    
## 4. 编辑Redis启动文件
下载初始配置文件,复制到/etc/init.d
    
    sudo wget https://raw.github.com/saxenap/install-redis-amazon-linux-centos/master/redis-server
    sudo mv redis-server /etc/init.d
    sudo chmod 755 /etc/init.d/redis-server
编辑文件`sudo vim /etc/init.d/redis-server`
修改redis的文件路径，这个路径就是我们第2步复制的地方。
    
    redis="/usr/local/redis/bin/redis-server"
## 5. 设置Redis开机启动
接下来我们设置开机启动，并启动Redis。
    
    sudo chkconfig --add redis-server
    sudo chkconfig --level 345 redis-server on
    sudo service redis-server restart
## 6. Redis验证

![redis-test1](/uploads/2016/10/redis-test1.jpg)

## 7. 安装Redis的PHP客户端
为了使WordPress的PHP与Redis通信，必须安装Redis的PHP客户端，主要有三种predis，phpredis和Rediska。这里采用predis。 下载[predis.php](https://uploads.staticjw.com/ji/jim/predis.php)并上传至网站wordpress的根目录。并且使用[index-with-redis.php](index-with-redis.php)替换index.php文件
    
    sudo wget http://uploads.staticjw.com/ji/jim/predis.php
    sudo chown nginx:nginx predis.php
    sudo wget https://gist.githubusercontent.com/JimWestergren/3053250/raw/d9e279e31cbee4a1520f59108a4418ae396b2dde/index-with-redis.php
    sudo chown nginx:nginx index-with-redis.php
    sudo mv index.php index.php.bak
    sudo mv index-with-redis.php index.php
注意根据需要修改以下参数值。
    
    $cf = 0; // set to 1 if you are using cloudflare
    $debug = 1; // set to 1 if you wish to see execution time and cache actions
    $display_powered_by_redis = 0; // set to 1 if you want to display a powered by redis message with execution time, see below

## 8. 测试效果
所有工作完成，你可以刷新你的网页，查看一下效果。如果你想在页面上看到脚本执行时间和缓存加载时间，请设置$debug = 1; 浏览器最下方第一次会显示cache is set:0.23617,接下来访问你会看到速度明显快了，浏览器最下方会显示this is a cache:0.00418,如果你用F5刷新，cache会清除。浏览器最下方会显示cache of page deleted:0.24268， 如果再访问会重新cache。
![2016-10-10_8-43-46](/uploads/2016/10/2016-10-10_8-43-46.jpg)

![2016-10-10_8-40-29](/uploads/2016/10/2016-10-10_8-40-29.jpg)

![2016-10-10_8-42-38](/uploads/2016/10/2016-10-10_8-42-38.jpg)