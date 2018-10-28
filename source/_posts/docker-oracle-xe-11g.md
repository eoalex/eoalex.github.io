---
title: Docker快速搭建oracle xe 11g开发环境
tags:
  - Docker
  - Oracle
id: 1350
categories:
  - 数据库
date: 2016-10-29 17:01:44
---
## 1. 简介
Oracle 11g XE 是 Oracle 数据库的免费版本，支持标准版的大部分功能，11g XE 提供 Windows 和 Linux 版本。
做为免费的 Oracle 数据库版本，XE 的限制是：

>最大数据库大小为 11 GB  
可使用的最大内存是 1G
一台机器上只能安装一个 XE 实例
XE 只能使用单 CPU，无法在多CPU上进行分布处理

## 2. 镜像下载
一般情况下你需要去[oracle网站](http://www.oracle.com/technetwork/database/database-technologies/express-edition/downloads/index.html)下载所需版本，然后安装配置。现在你不需要这般麻烦，只需使用docker下载镜像即可。
你可以一键下载，基于ubuntu16.04的版本
    
    docker pull wnameless/oracle-xe-11g:16.04
或者基于ubuntu14.04
    
    docker pull wnameless/oracle-xe-11g:14.04.4
## 3. 容器使用
### 3.1 容器创建
    
    docker create --name oraclexe11g01 -p 49160:22 -p 49161:1521 wnameless/oracle-xe-11g:16.04

其中容器名字，你可以自己命名。如果需要启动多个容器，可以02,03...编下去。宿主机映射的端口号根据你的需要可以自己定义。
    创建时使用ORACLE_ALLOW_REMOTE参数可以打开远程连接功能。
    
    docker create --name oraclexe11g01 -p 49160:22 -p 49161:1521 -e ORACLE_ALLOW_REMOTE=true wnameless/oracle-xe-11g:16.04
### 3.2 容器启动

    docker start oraclexe11g01
### 3.3 容器关闭
    
    docker stop oraclexe11g01
### 3.4 容器登录

oracle连接参数
    
    hostname: localhost
    port: 49161
    sid: xe
    username: system
    password: oracle
    Password for SYS &amp; SYSTEM
    oracle
使用SSH登录容器
    
    ssh root@localhost -p 49160
    password: admin

登陆后，你可以sqlplus登录oracle
![2016-10-29_17-21-21](/uploads/2016/10/2016-10-29_17-21-21.jpg)
你可以随时去[github](https://github.com/wnameless/docker-oracle-xe-11g)检查有无更新。