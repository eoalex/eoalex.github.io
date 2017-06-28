---
title: Kubernetes入门实战(3)：运维之扩容与升级
tags:
  - Kubernetes
id: 871
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-03-05 09:07:04
---

本文我们将对Kubernetes的常用运维操作**扩容与升级**做简单说明。

## 1.Node的扩容

Node的扩容简单言之就是增加新的Node节点。在节点上安装Kubelet，Kube-proxy及Docker, 并修改参数使其指向Master地址。基于Kuberlet的自动注册机制，新的Node将会自动加入现有的Kubernetes集群中。

## 2.Pod的动态扩容和缩放

在实际运维过程中，我们常常需要对某个服务动态扩容以满足突增的流量，或者动态减少服务实例节约服务器资源。
下面我们将动态增加redis-slave的pod副本由2个增加为3个。
#kubectl scale rc redis-slave --replicas=3

    [root@localhost ~]# kubectl get pods
    NAME                 READY     STATUS    RESTARTS   AGE
    frontend-ob32o       1/1       Running   1          30d
    frontend-zcyms       1/1       Running   1          30d
    frontend-zllt8       1/1       Running   1          30d
    redis-master-gjkqh   1/1       Running   1          30d
    redis-slave-geq4m    1/1       Running   1          30d
    redis-slave-o692g    1/1       Running   1          30d
    [root@localhost ~]# kubectl scale rc redis-slave --replicas=3
    scaled
    [root@localhost ~]# kubectl get pods
    NAME                 READY     STATUS    RESTARTS   AGE
    frontend-ob32o       1/1       Running   1          30d
    frontend-zcyms       1/1       Running   1          30d
    frontend-zllt8       1/1       Running   1          30d
    redis-master-gjkqh   1/1       Running   1          30d
    redis-slave-geq4m    1/1       Running   1          30d
    redis-slave-o692g    1/1       Running   1          30d
    redis-slave-qozrp    0/1       Running   0          6s
    `</pre>

    同样的我们也可以减少pod的副本，以下我们将redis-slave由3个副本减为1个。

    <pre>`[root@localhost ~]# kubectl get pods
    NAME                 READY     STATUS    RESTARTS   AGE
    frontend-ob32o       1/1       Running   1          30d
    frontend-zcyms       1/1       Running   1          30d
    frontend-zllt8       1/1       Running   1          30d
    redis-master-gjkqh   1/1       Running   1          30d
    redis-slave-geq4m    1/1       Running   1          30d
    redis-slave-o692g    1/1       Running   1          30d
    redis-slave-qozrp    1/1       Running   0          2m
    [root@localhost ~]# kubectl scale rc redis-slave --replicas=1
    scaled
    [root@localhost ~]# kubectl get pods
    NAME                 READY     STATUS    RESTARTS   AGE
    frontend-ob32o       1/1       Running   1          30d
    frontend-zcyms       1/1       Running   1          30d
    frontend-zllt8       1/1       Running   1          30d
    redis-master-gjkqh   1/1       Running   1          30d
    redis-slave-qozrp    1/1       Running   0          2m
    `</pre>

    ## 3.应用的滚动升级

    在实际运维过程中，如何不停止服务而进行升级将变得越来越常见，Kubernetes提供了Rolling-update的功能来解决上述场景。
    我们假设PHP的image有一个新的v2版本，我们需要将现有PHP服务滚动升级为v2。

    ### 3.1制作新镜像

    简单起见，我们通过docker commit来制作一个新镜像，首先用原镜像启动一个新容器，你可以在容器里修改，然后退出。

    <pre>`docker run -it docker.io/kubeguide/guestbook-php-frontend /bin/bash
    root@1b605f52b5f1:/var/www/html# exit
    exit
    `</pre>

    好，现在我们用docker commit来保存刚才我们编辑过的容器,我们把它命名为guestbook-php-frontend:v2

    <pre>`[root@localhost ~]# docker commit 1b605f52b5f1 guestbook-php-frontend:v2
    103579c2c2774b62017de760475e7a1ae3addacc9f1c88ea59ec1f3280ae93fb
    [root@localhost ~]# docker images
    REPOSITORY                                   TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
    guestbook-php-frontend                       v2                  103579c2c277        5 seconds ago       509.6 MB
    docker.io/kubeguide/guestbook-php-frontend   latest              38658844a359        5 months ago        509.6 MB
    docker.io/kubeguide/redis-master             latest              423e126c2ad4        6 months ago        419.1 MB
    docker.io/kubeguide/guestbook-redis-slave    latest              5429ea4e7990        6 months ago        109.5 MB
    gcr.io/google_containers/pause               0.8.0               2c40b0526b63        11 months ago       241.7 kB
    `</pre>

    我们看到新增加了一个镜像guestbook-php-frontend，tag为v2，接下来我们将通过两种方法来演示滚动升级。

    ### 3.2通过配置文件

    创建 frontend-controller-v2.yaml

    <pre>`apiVersion: v1
    kind: ReplicationController
    metadata:
      name: frontend-v2
      labels:
        name: frontend
    spec:
      replicas: 3
      selector:
        name: frontend-v2
      template:
        metadata:
          labels:
            name: frontend-v2
        spec:
          containers:
          - name: php-redis
            image: guestbook-php-frontend:v2
            env:
            - name: GET_HOSTS_FROM
              value: env
            ports:
            - containerPort: 80
    `</pre>

    Kubectl的执行过程如下：

    <pre>`[root@localhost ~]# kubectl rolling-update frontend -f frontend-controller-v2.yaml
    Creating frontend-v2
    At beginning of loop: frontend replicas: 2, frontend-v2 replicas: 1
    Updating frontend replicas: 2, frontend-v2 replicas: 1
    At end of loop: frontend replicas: 2, frontend-v2 replicas: 1
    At beginning of loop: frontend replicas: 1, frontend-v2 replicas: 2
    Updating frontend replicas: 1, frontend-v2 replicas: 2
    At end of loop: frontend replicas: 1, frontend-v2 replicas: 2
    At beginning of loop: frontend replicas: 0, frontend-v2 replicas: 3
    Updating frontend replicas: 0, frontend-v2 replicas: 3
    At end of loop: frontend replicas: 0, frontend-v2 replicas: 3
    Update succeeded. Deleting frontend
    frontend-v2

    `</pre>

    查看pods创建过程, 我们看到新的POD副本从1开始增加，旧的POD副本从3逐步减少，最终旧的POD副本被删除。这样就完成了应用的升级。

    <pre>`[root@localhost ~]# kubectl get rc
    CONTROLLER     CONTAINER(S)   IMAGE(S)                                     SELECTOR            REPLICAS
    frontend       php-redis      docker.io/kubeguide/guestbook-php-frontend   name=frontend       3
    frontend-v2    php-redis      guestbook-php-frontend:v2                    name=frontend-v2    1
    [root@localhost ~]# kubectl get rc
    CONTROLLER     CONTAINER(S)   IMAGE(S)                                     SELECTOR            REPLICAS
    frontend       php-redis      docker.io/kubeguide/guestbook-php-frontend   name=frontend       2
    frontend-v2    php-redis      guestbook-php-frontend:v2                    name=frontend-v2    2
    [root@localhost ~]# kubectl get rc
    CONTROLLER     CONTAINER(S)   IMAGE(S)                                     SELECTOR            REPLICAS
    frontend       php-redis      docker.io/kubeguide/guestbook-php-frontend   name=frontend       1
    frontend-v2    php-redis      guestbook-php-frontend:v2                    name=frontend-v2    3
    [root@localhost ~]# kubectl get rc
    CONTROLLER     CONTAINER(S)   IMAGE(S)                                    SELECTOR            REPLICAS
    frontend-v2    php-redis      guestbook-php-frontend:v2                   name=frontend-v2    3
    `</pre>

    ### 3.3通过新版镜像

    另一种方法是不使用配置文件，直接用kubectl rolling-update 加上--image参数指定新版镜像名称来滚动升级
    kubectl rolling-update frontend --image=guestbook-php-frontend:v2

    <pre>`[root@localhost ~]# kubectl rolling-update frontend --image=guestbook-php-frontend:v2
    Creating frontend-cdb7f26e49a90eae43e257284310b1cf
    At beginning of loop: frontend replicas: 2, frontend-cdb7f26e49a90eae43e257284310b1cf replicas: 1
    Updating frontend replicas: 2, frontend-cdb7f26e49a90eae43e257284310b1cf replicas: 1
    At end of loop: frontend replicas: 2, frontend-cdb7f26e49a90eae43e257284310b1cf replicas: 1
    At beginning of loop: frontend replicas: 1, frontend-cdb7f26e49a90eae43e257284310b1cf replicas: 2
    Updating frontend replicas: 1, frontend-cdb7f26e49a90eae43e257284310b1cf replicas: 2
    At end of loop: frontend replicas: 1, frontend-cdb7f26e49a90eae43e257284310b1cf replicas: 2
    At beginning of loop: frontend replicas: 0, frontend-cdb7f26e49a90eae43e257284310b1cf replicas: 3
    Updating frontend replicas: 0, frontend-cdb7f26e49a90eae43e257284310b1cf replicas: 3
    At end of loop: frontend replicas: 0, frontend-cdb7f26e49a90eae43e257284310b1cf replicas: 3
    Update succeeded. Deleting old controller: frontend
    Renaming frontend-cdb7f26e49a90eae43e257284310b1cf to frontend
    frontend
    `</pre>

    更新完成，查看RC，我们看到与配置文件不同，Kubectl给rc增加了一个key为"deployment"的label，当然这个名字可以通过--deployment-label-key参数修改。

    <pre>`[root@localhost ~]# kubectl get rc
    CONTROLLER     CONTAINER(S)   IMAGE(S)                                    SELECTOR                                                    REPLICAS
    frontend       php-redis      guestbook-php-frontend:v2                   deployment=cdb7f26e49a90eae43e257284310b1cf,name=frontend   3

如果在更新过程中发现配置错误，可以通过执行kubectl rolling-update --rollback完成回滚
kubectl rolling-update frontend --image=guestbook-php-frontend:v2 --rollback