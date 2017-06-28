---
title: docker ps 命令详解
tags:
  - Docker
id: 1602
categories:
  - 运维
date: 2017-04-09 14:19:35
---

对docker容器的管理，docker ps命令必不可少。
1.基本命令格式
基本命令格式为，docker ps [OPTIONS]，其中OPTIONS有如下选项

    --all, -a	false	显示所有容器默认只显示正在运行中的
    --filter, -f	 	根据条件过滤
    --format	 	显示所需字段
    --last, -n	-1	显示最后创建的n个容器
    --latest, -l	false	显示最后创建的容器
    --no-trunc	false	输出时不截断字段
    --quiet, -q	false	只显示容器id
    --size, -s	false	显示文件大小`</pre>
    2.过滤条件使用
    docker ps 给了我们查看容器的能力，但是如果容器的数量足够多，显然需要一个过滤条件，目前支持的过滤命令有：
    <pre>`id (容器的id)
    label (label=<key> or label=<key>=<value>)
    name (容器的名字)
    exited (列出已退出的容器. 需要与 --all选项一起用)
    status (状态created|restarting|running|removing|paused|exited|dead)
    ancestor (<image-name>[:<tag>], <image id> or <image@digest>) - 过滤所有含某个镜像或层产生的容器
    before (容器的ID或名字) - 过滤给定容器ID或名字之前创建的容器
    since (容器的ID或名字) - 过滤给定容器ID或名字之后创建的容器
    isolation (default|process|hyperv) (Windows daemon only)
    volume (volume name或mount point) - 过滤有mount volume的容器
    network (network id或name) - 过滤指定network id或名字的容器
    health (starting	healthy	unhealthy	none) - 过滤指定health状态的容器`</pre>
    以下是一些具体使用例子
    <pre>`
    $ docker ps --filter "label=color"
    $ docker ps --filter "label=color=blue"
    下列筛选器匹配所有容器的名称包含kafka字符串。
    $ docker ps --filter "name=kafka"
    容器正常停止的过滤
    $ docker ps -a --filter 'exited=0'
    使用kill命令退出的容器
    $ docker ps -a --filter 'exited=137'
    正在运行中的容器
    $ docker ps --filter status=running
    所有含ubuntu字符串的镜像产生的容器
    $ docker ps --filter ancestor=ubuntu
    所有层d0e008c6cf02的镜像产生的容器
    $ docker ps --filter ancestor=d0e008c6cf02
    过滤容器9c3527ed70ce之前创建的容器
    $ docker ps -f before=9c3527ed70ce
    过滤容器6e63f6ff38b0之后创建的容器
    $ docker ps -f since=6e63f6ff38b0
    过滤含有指定卷的容器
    $ docker ps --filter volume=remote-volume
    过滤含网络net1的容器
    $ docker ps --filter network=net1
    过滤publish或expose指定端口的容器
    $ docker ps --filter publish=80
    $ docker ps --filter expose=8000-8080/tcp
    `</pre>
    3.自定义显示字段
    <pre>`--format选项给了我们可以自定义显示字段的能力
    Placeholder	Description
    .ID	Container ID
    .Image	Image ID
    .Command	Quoted command
    .CreatedAt	Time when the container was created.
    .RunningFor	Elapsed time since the container was started.
    .Ports	Exposed ports.
    .Status	Container status.
    .Size	Container disk size.
    .Names	Container names.
    .Labels	All labels assigned to the container.
    .Label	Value of a specific label for this container. For example '{{.Label "com.docker.swarm.cpu"}}'
    .Mounts	Names of the volumes mounted in this container.
    .Networks	Names of the networks attached to this container.
    `</pre>
    以下具体使用例子
    <pre>`
    显示容器的id和命令
    $ docker ps --format "{{.ID}}: {{.Command}}"
    a87ecb4f327c: /bin/sh -c #(nop) MA
    01946d9d34d8: /bin/sh -c #(nop) MA
    c1d3b0166030: /bin/sh -c yum -y up
    41d50ecd2f57: /bin/sh -c #(nop) MA
    显示容器的id和label并带字段名
    $ docker ps --format "table {{.ID}}\t{{.Labels}}"
    CONTAINER ID        LABELS
    a87ecb4f327c        com.docker.swarm.node=ubuntu,com.docker.swarm.storage=ssd
    01946d9d34d8
    c1d3b0166030        com.docker.swarm.node=debian,com.docker.swarm.cpu=6
    41d50ecd2f57        com.docker.swarm.node=fedora,com.docker.swarm.cpu=3,com.docker.swarm.storage=ssd
    