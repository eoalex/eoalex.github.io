---
title: Kubernetes入门实战(2)：Kubernetes集群初探
tags:
  - Docker
  - Kubernetes
  - Mysql
id: 802
categories:
  - 云计算及虚拟化
date: 2016-02-27 20:18:55
---

上文我们在一台虚机上演示了Kubernetes基于redis和docker的guestbook留言簿案例，本文我们将通过配置Kubernetes集群的方式继续深入研究。集群组件安装如下配置。
<table width="553">
<tbody>
<tr>
<td width="109">IP</td>
<td width="156">NAME</td>
<td width="288">Component</td>
</tr>
<tr>
<td width="109">192.168.199.51</td>
<td width="156">centos-master</td>
<td width="288">etcd,kube-apiserver,kube-controller-manager,kube-scheduler</td>
</tr>
<tr>
<td width="109">192.168.199.52</td>
<td width="156">centos-minion01</td>
<td width="288">kube-proxy,kubelet,docker </td>
</tr>
<td width="109">192.168.199.53</td>
<td width="156">centos-minion02</td>
<td width="288">kube-proxy,kubelet,docker </td>
</tr>
</tbody>
</table>

主机环境：centos 7，三台虚机。
1.准备工作
以下工作在每台虚机执行。
1.1 停止防火墙
#systemctl disable firewalld
#systemctl stop firewalld

1.2 修改iptables
把icmp-host-prohibited两条注释掉

    # sample configuration for iptables service
    # you can edit this manually or use system-config-firewall
    # please do not ask us to add additional ports/services to this default configuration
    *filter
    :INPUT ACCEPT [0:0]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
    -A INPUT -p icmp -j ACCEPT
    -A INPUT -i lo -j ACCEPT
    -A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
    #-A INPUT -j REJECT --reject-with icmp-host-prohibited
    #-A FORWARD -j REJECT --reject-with icmp-host-prohibited
    COMMIT
    `</pre>
    重启iptables
    #systemctl restart iptables

    1.3 使用阿里镜像
    #wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

    1.4 更新主机列表
    #echo "192.168.199.51 centos-master
    192.168.199.52 centos-minion01
    192.168.199.53 centos-minion02" &gt;&gt; /etc/hosts

    2.安装配置kubernetes master
    2.1 在centos-master上安装
    #yum install kubernetes-master
    #yum install etcd

    2.2配置 Kubernetes services
    #vi /etc/kubernetes/config
    <pre>`###
    # kubernetes system config
    #
    # The following values are used to configure various aspects of all
    # kubernetes services, including
    #
    #   kube-apiserver.service
    #   kube-controller-manager.service
    #   kube-scheduler.service
    #   kubelet.service
    #   kube-proxy.service
    # logging to stderr means we get it in the systemd journal
    KUBE_LOGTOSTDERR="--logtostderr=true"

    # journal message level, 0 is debug
    KUBE_LOG_LEVEL="--v=0"

    # Should this cluster be allowed to run privileged docker containers
    KUBE_ALLOW_PRIV="--allow-privileged=false"

    # How the controller-manager, scheduler, and proxy find the apiserver
    KUBE_MASTER="--master=http://centos-master:8080"
    `</pre>

    2.3配置Kubernetes API server
    #vi /etc/kubernetes/apiserver
    <pre>`###
    # kubernetes system config
    #
    # The following values are used to configure the kube-apiserver
    #
    # The address on the local server to listen to.
    KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"

    # The port on the local server to listen on.
    KUBE_API_PORT="--insecure-port=8080"

    # Port minions listen on
    #KUBELET_PORT="--kubelet-port=10250"

    # Comma separated list of nodes in the etcd cluster
    KUBE_ETCD_SERVERS="--etcd-servers=http://127.0.0.1:2379"

    # Address range to use for services
    KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

    # default admission control policies
    KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"

    # Add your own!
    KUBE_API_ARGS=""
    `</pre>

    2.4 启动服务
    <pre>`for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
        systemctl restart $SERVICES
        systemctl enable $SERVICES
        systemctl status $SERVICES 
    done
    `</pre>
    2.5 停止服务
    <pre>`for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
        systemctl stop $SERVICES
    done
    `</pre>

    3.安装配置kubernetes node
    3.1 在centos-minion01及centos-minion02上安装
    #yum install kubernetes-node
    #vi /etc/kubernetes/config
    <pre>`###
    # kubernetes system config
    #
    # The following values are used to configure various aspects of all
    # kubernetes services, including
    #
    #   kube-apiserver.service
    #   kube-controller-manager.service
    #   kube-scheduler.service
    #   kubelet.service
    #   kube-proxy.service
    # logging to stderr means we get it in the systemd journal
    KUBE_LOGTOSTDERR="--logtostderr=true"

    # journal message level, 0 is debug
    KUBE_LOG_LEVEL="--v=0"

    # Should this cluster be allowed to run privileged docker containers
    KUBE_ALLOW_PRIV="--allow-privileged=false"

    # How the controller-manager, scheduler, and proxy find the apiserver
    KUBE_MASTER="--master=http://centos-master:8080"
    `</pre>
    3.2 配置 kubelet文件
    vi /etc/kubernetes/kubelet
    centos-minion01
    <pre>`###
    # kubernetes kubelet (minion) config

    # The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
    KUBELET_ADDRESS="--address=0.0.0.0"

    # The port for the info server to serve on
    KUBELET_PORT="--port=10250"

    # You may leave this blank to use the actual hostname
    KUBELET_HOSTNAME="--hostname-override=centos-minion01"

    # location of the api-server
    KUBELET_API_SERVER="--api-servers=http://centos-master:8080"

    # pod infrastructure container
    KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"

    # Add your own!
    KUBELET_ARGS=""
    `</pre>
    centos-minion02
    <pre>`
    ###
    # kubernetes kubelet (minion) config

    # The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
    KUBELET_ADDRESS="--address=0.0.0.0"

    # The port for the info server to serve on
    KUBELET_PORT="--port=10250"

    # You may leave this blank to use the actual hostname
    KUBELET_HOSTNAME="--hostname-override=centos-minion02"

    # location of the api-server
    KUBELET_API_SERVER="--api-servers=http://centos-master:8080"

    # pod infrastructure container
    KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"

    # Add your own!
    KUBELET_ARGS=""
    `</pre>
    3.3 配置config文件
    <pre>`###
    # kubernetes system config
    #
    # The following values are used to configure various aspects of all
    # kubernetes services, including
    #
    #   kube-apiserver.service
    #   kube-controller-manager.service
    #   kube-scheduler.service
    #   kubelet.service
    #   kube-proxy.service
    # logging to stderr means we get it in the systemd journal
    KUBE_LOGTOSTDERR="--logtostderr=true"

    # journal message level, 0 is debug
    KUBE_LOG_LEVEL="--v=0"

    # Should this cluster be allowed to run privileged docker containers
    KUBE_ALLOW_PRIV="--allow-privileged=false"

    # How the controller-manager, scheduler, and proxy find the apiserver
    KUBE_MASTER="--master=http://centos-master:8080"
    `</pre>

    3.4 启动服务
    <pre>`for SERVICES in kube-proxy kubelet docker; do 
        systemctl restart $SERVICES
        systemctl enable $SERVICES
        systemctl status $SERVICES 
    done
    `</pre>
    在centos-minion01上启动
    <pre>`[root@centos-minion01 ~]# for SERVICES in kube-proxy kubelet docker; do
    &gt;     systemctl restart $SERVICES
    &gt;     systemctl enable $SERVICES
    &gt;     systemctl status $SERVICES
    &gt; done
    ● kube-proxy.service - Kubernetes Kube-Proxy Server
       Loaded: loaded (/usr/lib/systemd/system/kube-proxy.service; enabled; vendor preset: disabled)
       Active: active (running) since Sat 2016-02-27 07:35:45 CST; 227ms ago
         Docs: https://github.com/GoogleCloudPlatform/kubernetes
     Main PID: 3682 (kube-proxy)
       CGroup: /system.slice/kube-proxy.service
               └─3682 /usr/bin/kube-proxy --logtostderr=true --v=0 --master=http://centos-master:8080

    Feb 27 07:35:45 centos-minion01 systemd[1]: Started Kubernetes Kube-Proxy Server.
    Feb 27 07:35:45 centos-minion01 systemd[1]: Starting Kubernetes Kube-Proxy Server...
    Feb 27 07:35:45 centos-minion01 kube-proxy[3682]: E0227 07:35:45.735033    3682 proxier.go:193] Error re...ory
    Feb 27 07:35:45 centos-minion01 kube-proxy[3682]: Try `iptables -h' or 'iptables --help' for more information.
    Feb 27 07:35:45 centos-minion01 kube-proxy[3682]: E0227 07:35:45.738008    3682 proxier.go:197] Error re...ory
    Feb 27 07:35:45 centos-minion01 kube-proxy[3682]: Try `iptables -h' or 'iptables --help' for more information.
    Hint: Some lines were ellipsized, use -l to show in full.
    ● kubelet.service - Kubernetes Kubelet Server
       Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
       Active: active (running) since Sat 2016-02-27 07:35:46 CST; 201ms ago
         Docs: https://github.com/GoogleCloudPlatform/kubernetes
     Main PID: 3816 (kubelet)
       CGroup: /system.slice/kubelet.service
               └─3816 /usr/bin/kubelet --logtostderr=true --v=0 --api-servers=http://centos-master:8080 --addre...

    Feb 27 07:35:46 centos-minion01 systemd[1]: Started Kubernetes Kubelet Server.
    Feb 27 07:35:46 centos-minion01 systemd[1]: Starting Kubernetes Kubelet Server...
    ● docker.service - Docker Application Container Engine
       Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
       Active: active (running) since Sat 2016-02-27 07:35:46 CST; 188ms ago
         Docs: http://docs.docker.com
     Main PID: 3875 (docker)
       CGroup: /system.slice/docker.service
               └─3875 /usr/bin/docker daemon --selinux-enabled

    Feb 27 07:35:46 centos-minion01 docker[3875]: time="2016-02-27T07:35:46.806484947+08:00" level=info msg...dge"
    Feb 27 07:35:46 centos-minion01 docker[3875]: time="2016-02-27T07:35:46.806514467+08:00" level=info msg...dge"
    Feb 27 07:35:46 centos-minion01 docker[3875]: time="2016-02-27T07:35:46.815103655+08:00" level=warning ...s 1"
    Feb 27 07:35:46 centos-minion01 docker[3875]: time="2016-02-27T07:35:46.820513002+08:00" level=info msg...lse"
    Feb 27 07:35:46 centos-minion01 docker[3875]: time="2016-02-27T07:35:46.877030818+08:00" level=info msg...rt."
    Feb 27 07:35:46 centos-minion01 docker[3875]: ....
    Feb 27 07:35:46 centos-minion01 docker[3875]: time="2016-02-27T07:35:46.880893846+08:00" level=info msg...ne."
    Feb 27 07:35:46 centos-minion01 docker[3875]: time="2016-02-27T07:35:46.880919835+08:00" level=info msg...ion"
    Feb 27 07:35:46 centos-minion01 docker[3875]: time="2016-02-27T07:35:46.880937118+08:00" level=info msg...ntos
    Feb 27 07:35:46 centos-minion01 systemd[1]: Started Docker Application Container Engine.
    Hint: Some lines were ellipsized, use -l to show in full.
    [root@centos-minion01 ~]#
    `</pre>
    在centos-minion02上启动
    <pre>`[root@centos-minion02 kubernetes]# for SERVICES in kube-proxy kubelet docker; do
    &gt;     systemctl restart $SERVICES
    &gt;     systemctl enable $SERVICES
    &gt;     systemctl status $SERVICES
    &gt; done
    ● kube-proxy.service - Kubernetes Kube-Proxy Server
       Loaded: loaded (/usr/lib/systemd/system/kube-proxy.service; enabled; vendor preset: disabled)
       Active: active (running) since Sat 2016-02-27 07:32:22 CST; 221ms ago
         Docs: https://github.com/GoogleCloudPlatform/kubernetes
     Main PID: 3138 (kube-proxy)
       CGroup: /system.slice/kube-proxy.service
               └─3138 /usr/bin/kube-proxy --logtostderr=true --v=0 --master=http://centos-master:8080

    Feb 27 07:32:22 centos-minion02 systemd[1]: Started Kubernetes Kube-Proxy Server.
    Feb 27 07:32:22 centos-minion02 systemd[1]: Starting Kubernetes Kube-Proxy Server...
    Feb 27 07:32:22 centos-minion02 kube-proxy[3138]: E0227 07:32:22.774533    3138 server.go:324] Not tryi...und
    Feb 27 07:32:22 centos-minion02 kube-proxy[3138]: E0227 07:32:22.857247    3138 proxier.go:193] Error r...ory
    Feb 27 07:32:22 centos-minion02 kube-proxy[3138]: Try `iptables -h' or 'iptables --help' for more infor...on.
    Feb 27 07:32:22 centos-minion02 kube-proxy[3138]: E0227 07:32:22.859129    3138 proxier.go:197] Error r...ory
    Feb 27 07:32:22 centos-minion02 kube-proxy[3138]: Try `iptables -h' or 'iptables --help' for more infor...on.
    Hint: Some lines were ellipsized, use -l to show in full.
    ● kubelet.service - Kubernetes Kubelet Server
       Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
       Active: active (running) since Sat 2016-02-27 07:32:23 CST; 201ms ago
         Docs: https://github.com/GoogleCloudPlatform/kubernetes
     Main PID: 3279 (kubelet)
       CGroup: /system.slice/kubelet.service
               └─3279 /usr/bin/kubelet --logtostderr=true --v=0 --api-servers=http://centos-master:8080 --addr...

    Feb 27 07:32:23 centos-minion02 systemd[1]: Started Kubernetes Kubelet Server.
    Feb 27 07:32:23 centos-minion02 systemd[1]: Starting Kubernetes Kubelet Server...
    ● docker.service - Docker Application Container Engine
       Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
       Active: active (running) since Sat 2016-02-27 07:32:23 CST; 186ms ago
         Docs: http://docs.docker.com
     Main PID: 3338 (docker)
       CGroup: /system.slice/docker.service
               └─3338 /usr/bin/docker daemon --selinux-enabled

    Feb 27 07:32:23 centos-minion02 docker[3338]: time="2016-02-27T07:32:23.772348922+08:00" level=info ms...r\""
    Feb 27 07:32:23 centos-minion02 docker[3338]: time="2016-02-27T07:32:23.779372746+08:00" level=info ms...dge"
    Feb 27 07:32:23 centos-minion02 docker[3338]: time="2016-02-27T07:32:23.779397422+08:00" level=info ms...dge"
    Feb 27 07:32:23 centos-minion02 docker[3338]: time="2016-02-27T07:32:23.787899136+08:00" level=warning...s 1"
    Feb 27 07:32:23 centos-minion02 docker[3338]: time="2016-02-27T07:32:23.791497828+08:00" level=info ms...lse"
    Feb 27 07:32:23 centos-minion02 docker[3338]: time="2016-02-27T07:32:23.843065746+08:00" level=info ms...rt."
    Feb 27 07:32:23 centos-minion02 docker[3338]: time="2016-02-27T07:32:23.843241402+08:00" level=info ms...ne."
    Feb 27 07:32:23 centos-minion02 docker[3338]: time="2016-02-27T07:32:23.843258440+08:00" level=info ms...ion"
    Feb 27 07:32:23 centos-minion02 docker[3338]: time="2016-02-27T07:32:23.843271897+08:00" level=info ms...ntos
    Feb 27 07:32:23 centos-minion02 systemd[1]: Started Docker Application Container Engine.
    Hint: Some lines were ellipsized, use -l to show in full.
    [root@centos-minion02 kubernetes]#
    `</pre>

    3.5 停止服务
    <pre>`for SERVICES in kube-proxy kubelet docker; do 
        systemctl stop $SERVICES 
    done
    `</pre>

    4\. 检查及确认状态
    #kubectl get nodes
    #kubectl cluster-info
    我们看到2个节点都正常启动。
    <pre>`[root@centos-master ~]# kubectl get nodes
    NAME              LABELS                                   STATUS    AGE
    centos-minion01   kubernetes.io/hostname=centos-minion01   Ready     1m
    centos-minion02   kubernetes.io/hostname=centos-minion02   Ready     51s
    [root@centos-master ~]# kubectl cluster-info
    Kubernetes master is running at http://localhost:8080
    [root@centos-master ~]#
    `</pre>

    5.创建MYSQL POD
    5.1 建立工作目录并查看API版本
    在kubernetes master节点
    #mkdir mysqlpod
    #cd mysqlpod
    #kubectl api-versions
    我们看到API版本为1,所以设置文件时用v1就可以了。
    <pre>`
    [root@centos-master mysqlpod]# kubectl api-versions
    Available Server Api Versions: v1
    `</pre>
    5.2 编写mysql的pod文件
    #vi mysqlpod.yaml
    <pre>`apiVersion: v1
    kind: Pod
    metadata:
      name: mysql
      labels:
        name: mysql
    spec:
      containers:
      - name: mysql
        image: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: 123456
        ports:
        - containerPort: 3306
    `</pre>
    5.3 启动POD
    #kubectl create -f mysqlpod.yaml
    #kubectl get pods
    #kubectl describe pods mysql
    <pre>`Events:
      FirstSeen     LastSeen        Count   From                            SubobjectPath                           Reason            Message
      ─────────     ────────        ─────   ────                            ─────────────                           ──────            ───────
      49s           49s             1       {scheduler }                                                            Scheduled Successfully assigned mysql to centos-minion02
      49s           49s             1       {kubelet centos-minion02}       implicitly required container POD       Pulled            Container image "registry.access.redhat.com/rhel7/pod-infrastructure:latest" already present on machine
      49s           49s             1       {kubelet centos-minion02}       implicitly required container POD       Created           Created with docker id 31f65f03a960
      48s           48s             1       {kubelet centos-minion02}       implicitly required container POD       Started           Started with docker id 31f65f03a960
      48s           48s             1       {kubelet centos-minion02}       spec.containers{mysql}                  Pulled            Container image "mysql" already present on machine
      48s           48s             1       {kubelet centos-minion02}       spec.containers{mysql}                  Created           Created with docker id aa39d65008dc
      47s           47s             1       {kubelet centos-minion02}       spec.containers{mysql}                  Started           Started with docker id aa39d65008dc
    `</pre>
    我们看到这个POD启动在centos-minion02虚机上，首先它启动了一个叫pod-infrastructure的容器，然后去找本机是否有mysql镜像，没有就去下载，已有的话就直接创建一个mysql容器。
    在centos-minion02上看启动的容器。
    #docker ps
    <pre>`[root@centos-minion02 ~]# docker ps
    CONTAINER ID        IMAGE                                                        COMMAND                  CREATED             STATUS              PORTS               NAMES
    aa39d65008dc        mysql                                                        "/entrypoint.sh mysql"   About an hour ago   Up About an hour                        k8s_mysql.1431d49_mysql_default_c935d35d-dce2-11e5-9ab1-000c29beeacc_95c7a9b7
    31f65f03a960        registry.access.redhat.com/rhel7/pod-infrastructure:latest   "/pod"                   About an hour ago   Up About an hour                        k8s_POD.36d00adb_mysql_default_c935d35d-dce2-11e5-9ab1-000c29beeacc_1cf5c985
    `</pre>
    我们看到启动两个容器，一个是mysql，一个是pod-infrastructure。
    5.4 编写mysql的服务文件
    #vi mysqlservice.yaml
    <pre>`apiVersion: v1
    kind: Service
    metadata:
      labels:
        name: mysql
      name: mysql
    spec:
      ports:
        - port: 3306
      selector:
        name: mysql
    `</pre>
    5.4 启动服务
    #kubectl create -f mysqlservice.yaml
    <pre>`[root@centos-master mysqlpod]# kubectl get services
    NAME         CLUSTER_IP     EXTERNAL_IP   PORT(S)    SELECTOR     AGE
    kubernetes   10.254.0.1             443/TCP           18h
    mysql        10.254.62.21           3306/TCP   name=mysql   9s
    [root@centos-master mysqlpod]#
    `</pre>
    5.5 mysql登录
    在centos-minion02上连接mysql的POD，我们看到连接正常。
    <pre>`[root@centos-minion02 ~]# mysql -uroot -p -h10.254.62.21
    Enter password:
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 3
    Server version: 5.7.10 MySQL Community Server (GPL)

    Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    mysql&gt; show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    4 rows in set (0.01 sec)

    mysql&gt;

参考：Kubernetes权威指南