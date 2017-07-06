---
title: Mac 10.12下搭建Eclipse的maven开发环境
tags:
  - eclipse
  - maven
id: 1390
categories:
  - 运维
date: 2016-11-06 09:58:00
---

## 1. Java安装
从Oralce官网下载
http://download.oracle.com/otn-pub/java/jdk/8u112-b16/jdk-8u112-macosx-x64.dmg
下载完毕直接双击dmg文件安装，安装完毕，运行java -version查看

    Downloads alex$ java -version
    java version "1.8.0_112"
    Java(TM) SE Runtime Environment (build 1.8.0_112-b16)
    Java HotSpot(TM) 64-Bit Server VM (build 25.112-b16, mixed mode)

## 2. Eclipse安装
以下镜像地址下载。

    http://mirrors.neusoft.edu.cn/eclipse/technology/epp/downloads/release/neon/1a/eclipse-java-neon-1a-macosx-cocoa-x86_64.tar.gz

直接解压至应用程序目录。
    
    tar -xzvf  eclipse-java-neon-1a-macosx-cocoa-x86_64.tar.gz /Applications

## 3. Maven安装
下载地址

http://mirror.bit.edu.cn/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz

解压
    
    tar -xzvf  apache-maven-3.3.9-bin.tar /opt
添加以下命令行至/etc/bashrc
    
    export MAVEN_HOME=/opt/apache-maven-3.3.9
    export PATH=${PATH}:${MAVEN_HOME}/bin
执行source命令生效
    
    source /etc/bashrc
验证maven是否安装成功。
    
    $ mvn -v
    Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T00:41:47+08:00)
    Maven home: /opt/apache-maven-3.3.9
    Java version: 1.8.0_112, vendor: Oracle Corporation
    Java home: /Library/Java/JavaVirtualMachines/jdk1.8.0_112.jdk/Contents/Home/jre
    Default locale: zh_CN, platform encoding: UTF-8
    OS name: "mac os x", version: "10.12", arch: "x86_64", family: "mac"

## 4. 配置
打开Eclipse，选择Help->Install New SoftWare
Add new repo
http://m2eclipse.sonatype.org/sites/m2e,
![%e5%b1%8f%e5%b9%95%e5%bf%ab%e7%85%a7-2016-11-05-%e4%b8%8b%e5%8d%889-32-42](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/11/屏幕快照-2016-11-05-下午9.32.42.png)

点击OK，等待安装完成.

## 5. 验证
重启eclipse， Help --> About Eclipse --> Installation Details
在Installed Software标签中检查刚才选择的模块是否在这个列表中
检查eclipse是否已经支持创建Maven项目：
File --> New --> Other ，找到Maven一项，如果展开一切正常，说明m2eclipse已经正确安装了。