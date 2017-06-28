---
title: 'OpenStack入门之二:devstack开发环境搭建'
tags:
  - devstack
  - openstack
id: 1104
categories:
  - 云计算及虚拟化
date: 2016-05-15 14:26:59
---

devstack是用来给开发人员快速部署Openstack的开发环境。
操作系统环境：Ubuntu 14.04
操作系统ISO安装可以从各大镜像网站下载，这里选择阿里镜像。
http://mirrors.aliyun.com/ubuntu-releases/ 
操作系统安装过程略。我们给操作系统分配2块网卡，一为桥接模式，网卡地址192.168.199.20，一为NAT模式。
以下是安装devstack过程。
1.更新源
在软件包管理中心“软件源”中选择“中国的服务器”下mirros.aliyun.com即可自动使用
或者直接编辑文件
sudo vi /etc/apt/source.list
sudo apt-get update
2.安装git
sudo apt-get install git
3.下载devstack源代码
sudo git clone -b stable/kilo https://git.openstack.org/openstack-dev/devstack
4.创建用户
sudo devstack/tools/create-stack-user.sh
sudo su - stack
5.拷贝devstack目录到/opt/stack下
sudo cp -r /home/alex/devstack .
6.修改devstack目录权限
sudo chown -R stack:stack devstack
7.执行安装脚本
cd devstack
./stack.sh
漫长的安装过程，如果安装过程有问题，可以使用git单独下载源代码
7.1 使用git单独下载源代码

    git clone -b stable/kilo git://git.openstack.org/openstack/horizon.git /opt/stack/horizon
    git clone -b stable/kilo git://git.openstack.org/openstack/keystone.git /opt/stack/keystone
    git clone -b stable/kilo git://git.openstack.org/openstack/nova.git /opt/stack/nova
    git clone -b stable/kilo git://git.openstack.org/openstack/neutron.git /opt/stack/neutron
    git clone -b stable/kilo git://git.openstack.org/openstack/glance.git /opt/stack/glance
    git clone -b stable/kilo git://git.openstack.org/openstack/cinder.git /opt/stack/cinder
    `</pre>
    安装完成，显示结果如下。如果安装过程错误，使用./unstack.sh卸载，再使用./stack.sh重新安装。
    [![2016-05-14_0-16-45](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-14_0-16-45.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-14_0-16-45.jpg)
    8.修改配置文件
    编辑local.conf，主要更新IP地址。
    <pre>`
    [[local|localrc]]
    HOST_IP=192.168.199.20
    SERVICE_HOST=$HOST_IP
    MYSQL_HOST=$HOST_IP
    RABBIT_HOST=$HOST_IP
    GLANCE_HOSTPORT=$HOST_IP:9292
    ADMIN_PASSWORD=passw0rd
    DATABASE_PASSWORD=$ADMIN_PASSWORD
    RABBIT_PASSWORD=$ADMIN_PASSWORD
    SERVICE_PASSWORD=$ADMIN_PASSWORD
    SERVICE_TOKEN=$ADMIN_PASSWORD

    # Do not use Nova-Network
    disable_service n-net

    ENABLED_SERVICES+=,q-svc,q-dhcp,q-meta,q-agt,q-l3

    ## Neutron options
    Q_USE_SECGROUP=True
    FLOATING_RANGE="192.168.199.0/24"
    FIXED_RANGE="10.0.0.0/24"
    Q_FLOATING_ALLOCATION_POOL=start=192.168.199.21,end=192.168.199.29
    PUBLIC_NETWORK_GATEWAY="192.168.199.1"
    Q_L3_ENABLED=True
    PUBLIC_INTERFACE=eth1
    Q_USE_PROVIDERNET_FOR_PUBLIC=True
    OVS_PHYSICAL_BRIDGE=br-ex
    PUBLIC_BRIDGE=br-ex
    OVS_BRIDGE_MAPPINGS=public:br-ex
    `</pre>
    9.设置屏幕监控
    screen -list
    screen -r 4573
    script /dev/null
    [![2016-05-14_0-24-47](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-14_0-24-47.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-14_0-24-47.jpg)
    10.安装jdk
    10.1.安装
    cd /opt/stack
    sudo mkdir -p /usr/lib/java
    sudo tar -zxvf jdk-8u51-linux-x64.gz -C /usr/lib/java
    注，jdk文件可以Oracle网站下载，这里不再赘述。
    10.2.配置环境变量
    vi .bashrc
    <pre>`
    export JAVA_HOME=/usr/lib/java/jdk1.8.0_51
    export JRE_HOME=${JAVA_HOME}/jre 
    export CLASSPATH=${JAVA_HOME}/lib:${JRE_HOME}/jre:${CLASSPATH} 
    export PATH=${JAVA_HOME}/bin:${PATH}
    `</pre>
    source .bashrc
    10.安装eclipse
    10.1从以下网址下载eclipse luna版本
    http://www.eclipse.org/downloads/download.php?file=/technology/epp/downloads/release/luna/SR2/eclipse-java-luna-SR2-linux-gtk-x86_64.tar.gz
    10.2 编辑配置文件，修改内存参数
    vi eclipse.ini
    <pre>`
    XX maxPermSize=1024m
    -Xms40m
    -Xmx1024m
    `</pre>
    10.3启动xhost，使所有用户都能访问Xserver.
    Xhost +
    10.4启动eclipse
    [![2016-05-15_13-49-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-15_13-49-13.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-15_13-49-13.jpg) 
    设置workspace为/opt/stack
    11.安装pydev
    以下地址下载pydev插件
    https://marketplace.eclipse.org/content/pydev-python-ide-eclipse
    选择信任证书
    [![2016-05-15_13-52-26](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-15_13-52-26.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-15_13-52-26.jpg) 
    配置
    eclipse-&gt;windows-&gt;preference-&gt;PyDev-&gt;interpreters-&gt;python interpreter-&gt;Quick Auto config
    [![2016-05-14_1-33-27](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-14_1-33-27.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-14_1-33-27.jpg) 
    至此开发环境搭建完成。
    12.验证开发环境是否搭建成功
    我们试着创建horizon项目，首先备份原有配置文件
    12.1备份原有配置文件
    cd /opt/stack/horizon/openstack-dashboard/local/
    备份原有的配置文件
    mv local_settings.py local_settings.py.bak
    mv local_settings.pyc local_settings.pyc.bak
    cp local_settings.py.example local_settings.py
    12.2设置并启动debug
    manager.py debug configration -&gt;arguments
    runserver 0.0.0.0:9000
    [![2016-05-15_14-04-34](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-15_14-04-34.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-15_14-04-34.jpg) 
    12.3登录9000端口
    http://192.168.199.20:9000
    使用admin 账号登录
    [![2016-05-14_1-55-17](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-14_1-55-17.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/05/2016-05-14_1-55-17.jpg) 
    我们看到成功登录，至此可以验证开发环境搭建成功。
    13.devstack终止或重启服务
    在devstack目录下，运行 ./rejoin-stack.sh，进入控制台。如果出现以下错误，使用script /dev/null重定向。
    stack@devstack:~/devstack$ ./rejoin-stack.sh
    Attaching to already started screen session..
    Cannot open your terminal '/dev/pts/39' - please check.
    stack@devstack:~/devstack$ script /dev/null
    Script started, file is /dev/null
    stack@devstack:~/devstack$ ./rejoin-stack.sh
    下面是显示信息
    <pre>`
    2016-05-15 10:17:41.278 DEBUG nova.openstack.common.periodic_task [req-39a9042c-a2f3-4e52-8a5d-5bb1ca80daf8 None None] Running periodic task ComputeManager._check_instance_build_time from (pid=4175) run_periodic_tasks /opt/stack/nova/nova/openstack/common/periodic_task.py:219
    2016-05-15  10:17:41.279 DEBUG nova.openstack.common.loopingcall [req-39a9042c-a2f3-4e52-8a5d-5bb1ca80daf8 None None] Dynamic looping call &lt;bound method Service.periodic_tasks of &gt; sleeping for 1.01 seconds from (pid=4175) _inner /opt/stack/nova/nova/openstack/common/loopingcall.py:132
    2016-05-15  10:17:42.289 DEBUG nova.openstack.common.periodic_task [req-39a9042c-a2f3-4e52-8a5d-5bb1ca80daf8 None None] Running periodic task ComputeManager._poll_rebooting_instances from (pid=4175) run_periodic_tasks /opt/stack/nova/nova/openstack/common/periodic_task.py:219
    2016-05-15  10:17:42.291 DEBUG nova.openstack.common.periodic_task [req-39a9042c-a2f3-4e52-8a5d-5bb1ca80daf8 None None] Running periodic task ComputeManager._poll_unconfirmed_resizes from (pid=4175) run_periodic_tasks /opt/stack/nova/nova/openstack/common/periodic_task.py:219
    2016-05-15  10:17:42.292 DEBUG nova.openstack.common.loopingcall [req-39a9042c-a2f3-4e52-8a5d-5bb1ca80daf8 None None] Dynamic looping call &lt;bound method Service.periodic_tasks of &gt; sleeping for 56.00 seconds from (pid=4175) _inner /opt/stack/nova/nova/openstack/common/loopingcall.py:132

    n-sch  15$(L) n-novnc   16$(L) n-cpu*  17$(L) c-api  18-$(L) c-sch  19$(L) c-vol

屏幕最后一行的“n-cpu*”表示的是nova-compute服务，前面的16表示这个服务的编号，*表示是当前服务。
切换不同服务的方法为按ctrl+a+'(即ctrl+a+单引号)，这时屏幕左下角会显示“Switch to window:”表示要前往的服务控制台，你可以输入17，表示看c-api (cinder-api)服务的情况。
停止服务的方法是在在相应控制台下使用：ctrl+c，再启动这个服务是按下“↑”（即向上键），然后在按enter键。退出控制的方法是使用ctrl+d。