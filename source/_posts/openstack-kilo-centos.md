---
title: 'OpenStack入门之三:Kilo版本centos 7平台搭建'
tags:
  - openstack
id: 1129
categories:
  - 云计算及虚拟化
date: 2016-06-11 23:14:22
---

## 1. 虚拟机环境

|节点名称	|硬件配置	|功能/用途	|IP地址|
|:--|:--|:--|:--|
|controller|	4G内存, 1块硬盘，1块网卡|	控制节点|	192.168.199.80|
|compute-node-01|	2G内存, 1块硬盘，1块网卡|	计算节点|	192.168.199.81|
|network-node-01|	1G内存, 1块硬盘，2块网卡|	网络节点|	192.168.199.82|
|block-node-01|	1G内存, 2块硬盘，1块网卡|	块存储节点|	192.168.199.83|
|object-node-01|	1G内存, 3块硬盘，1块网卡|	对象存储节点1|	192.168.199.84|
|object-node-02|	1G内存, 3块硬盘，1块网卡|	对象存储节点2|	192.168.199.85|

## 2. 组件节点分布

[![2016-06-12_9-43-08](/uploads/2016/06/2016-06-12_9-43-08.jpg)](/uploads/2016/06/2016-06-12_9-43-08.jpg)

## 3. 准备工作

以下操作在每个节点上
### 3.1 更新centos源
	#yum install wget
	#wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
### 3.2 停止防火墙服务
	systemctl stop firewalld.service
	systemctl disable firewalld.service
### 3.3 安装基础组件及源
	#yum install ntp -y
	#yum install yum-plugin-priorities -y
	#yum install http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-6.noarch.rpm -y
	#yum install http://rdo.fedorapeople.org/openstack-kilo/rdo-release-kilo.rpm -y
	#yum install https://repos.fedorapeople.org/repos/openstack/openstack-kilo/rdo-release-kilo-2.noarch.rpm -y
	#yum upgrade -y
	#yum install openstack-selinux -y
### 3.4 更新时区
	#cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
### 3.5 配置时钟同步(controller节点)
#### 3.5.1 备份配置文件
	
	#cp /etc/ntp.conf /etc/ntp.conf.bak
#### 3.5.2 添加参数
	#vi /etc/ntp.conf
	    server controller iburst
	    restrict -4 default kod notrap nomodify
	    restrict -6 default kod notrap nomodify
    
#### 3.5.3 注释以下参数
    #restrict default nomodify notrap nopeer noquery
    #restrict 127.0.0.1 
    #restrict ::1
#### 3.5.4 启动时钟同步服务
    #systemctl enable ntpd.service
    #systemctl start ntpd.service
### 3.6 配置时钟同步(其它节点)
#### 3.6.1 备份配置文件
    #cp /etc/ntp.conf /etc/ntp.conf.bak
#### 3.6.2添加参数
    #vi /etc/ntp.conf
        server controller iburst
    
注释其他server项
### 3.7 本环境所涉及到组件密码。

|用户名	|密码|	所属组件	|用途|
|:--|:--|:--|:--|
|openstack|	123456|	rabbitMQ	|Openstack组件间MQ消息通讯|
|admin	|123456	|Keystone|	Keystone管理员|
|demo|	123456	|Keystone	|访问租户项目demo|
|glance|	123456	|Keystone	|glance用户|
|nova	|123456|	Keystone|	nova用户|
|neutron|	123456	|Keystone|	neutron用户|
|cinder	|123456	|Keystone|	cinder用户|
|swift	|123456|	Keystone|	swift用户|
|root	|123456|	mysql|	mysql系统管理员|
|keystone|	keystone|	mysql|	keystone组件访问mysql数据库|
|glance|	glance	|mysql|	Glance组件访问mysql数据库|
|nova	|nova|	mysql	|Nova组件访问mysql数据库|
|neutron	|neutron|	mysql|	neutron组件访问mysql数据库|
|cinder	|cinder|	mysql	|Cinder组件访问mysql数据库|

## 4. controller节点安装
### 4.1 安装数据库
#### 4.1.1 安装mariadb
    #yum install mariadb mariadb-server MySQL-python -y
#### 4.1.2备份配置文件
    #cp /etc/my.cnf /etc/my.cnf.bak
#### 4.1.3修改配置文件
    #vi /etc/my.cnf  
   	    [mysqld]
	    bind-address = controller
	    default-storage-engine = innodb
	    innodb_file_per_table
	    collation-server = utf8_general_ci
	    init-connect = 'SET NAMES utf8'
	    character-set-server = utf8
#### 4.1.4 启动数据库服务
    #systemctl enable mariadb.service
    #systemctl start mariadb.service
#### 2.1.5 配置数据库服务安全参数,设置root密码
    #mysql_secure_installation 
    Remove anonymous users? [Y/n] n
    Disallow root login remotely? [Y/n] Y
    Remove test database and access to it? [Y/n] n
    Reload privilege tables now? [Y/n] Y
#### 4.1.6 验证mysql是否登录正常
    #mysql -uroot -p

### 4.2 安装rabbitmq
#### 4.2.1安装rabbitmq-server
    yum install rabbitmq-server -y
#### 4.2.2启动服务
    #systemctl enable rabbitmq-server.service
    #systemctl start rabbitmq-server.service
#### 4.2.3添加用户，并允许远程访问
    #rabbitmqctl add_user openstack 123456
    #rabbitmqctl set_permissions openstack ".*" ".*" ".*"
#### 4.2.4验证
默认端口5672

    #telent controller 5672
或者

    #rabbitmqctl status

### 4.3 安装keystone
#### 4.3.1 创建数据库keystone
    #mysql -uroot -p123456
    
    create database keystone;
    GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone';
    GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone';
    
#### 4.3.2 安装keystone组件
    #yum install openstack-keystone httpd mod_wsgi python-openstackclient memcached python-memcached -yh
    
#### 4.3.3 启动memcached
    #systemctl enable memcached.service
    #systemctl start memcached.service
    
#### 4.3.4 配置keystone.conf
##### 4.3.4.1 备份配置文件
    #cp /etc/keystone/keystone.conf /etc/keystone/keystone.conf.bak
##### 4.3.4.2 生成token
    #openssl rand -hex 10
    > 0cadc88a6437bc7da5cd
##### 4.3.4.3 修改配置文件
    #vi /etc/keystone/keystone.conf
    
    [DEFAULT]
    admin_token = 0cadc88a6437bc7da5cd  --用你生成的token替换
    verbose = True
    [database]
    connection = mysql://keystone:keystone@localhost/keystone
    [memcache]
    servers = localhost:11211
    [revoke]
    driver = keystone.contrib.revoke.backends.sql.Revoke
    [token]
    provider = keystone.token.providers.uuid.Provider
    driver = keystone.token.persistence.backends.memcache.Token
    

#### 4.3.5初始化keystone数据库
##### 4.3.5.1 执行创建表脚本
    #su -s /bin/sh -c "keystone-manage db_sync" keystone
##### 4.3.5.2 检查表是否均已创建完成
    MariaDB [keystone]> show tables;
    
    +------------------------+
    | Tables_in_keystone     |
    +------------------------+
    | access_token           |
    | assignment             |
    | consumer               |
    | credential             |
    | domain                 |
    | endpoint               |
    | endpoint_group         |
    | federation_protocol    |
    | group                  |
    | id_mapping             |
    | identity_provider      |
    | idp_remote_ids         |
    | mapping                |
    | migrate_version        |
    | policy                 |
    | policy_association     |
    | project                |
    | project_endpoint       |
    | project_endpoint_group |
    | region                 |
    | request_token          |
    | revocation_event       |
    | role                   |
    | sensitive_config       |
    | service                |
    | service_provider       |
    | token                  |
    | trust                  |
    | trust_role             |
    | user                   |
    | user_group_membership  |
    | whitelisted_config     |
    +------------------------+
    32 rows in set (0.00 sec)
    
#### 2.3.6 配置keystone的http服务
##### 2.3.6.1 备份配置文件httpd.conf
    #cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd.conf.bak
##### 2.3.6.2 修改配置文件httpd.conf
    #vi /etc/httpd/conf/httpd.conf
    
    ServerName controller
    
##### 2.3.6.3 创建keystone的虚拟站点
    #vi /etc/httpd/conf.d/wsgi-keystone.conf
    
    Listen 5000
    Listen 35357
        WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
        WSGIProcessGroup keystone-public
        WSGIScriptAlias / /var/www/cgi-bin/keystone/main
        WSGIApplicationGroup %{GLOBAL}
        WSGIPassAuthorization On
        LogLevel info
        ErrorLogFormat "%{cu}t %M"
        ErrorLog /var/log/httpd/keystone-error.log
        CustomLog /var/log/httpd/keystone-access.log combined

        WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
        WSGIProcessGroup keystone-admin
        WSGIScriptAlias / /var/www/cgi-bin/keystone/admin
        WSGIApplicationGroup %{GLOBAL}
        WSGIPassAuthorization On
        LogLevel info
        ErrorLogFormat "%{cu}t %M"
        ErrorLog /var/log/httpd/keystone-error.log
        CustomLog /var/log/httpd/keystone-access.log combined
    
##### 4.3.6.4 创建wsgi 目录
    #mkdir -p /var/www/cgi-bin/keystone
##### 4.3.6.5 下载程序模板
    #curl http://git.openstack.org/cgit/openstack/keystone/plain/httpd/keystone.py?h=stable/kilo \
    | tee /var/www/cgi-bin/keystone/main /var/www/cgi-bin/keystone/admin
##### 4.3.6.6 修改文件夹及文件权限
    #chown -R keystone:keystone /var/www/cgi-bin/keystone
    #chmod 755 /var/www/cgi-bin/keystone/*
##### 4.3.6.7 启动httpServer
    #systemctl enable httpd.service
    #systemctl start httpd.service
##### 4.3.6.8 检查http服务是否正常启动
    #systemctl status httpd.service
#### 4.3.7 注册keystone api服务
##### 4.3.7.1 配置系统环境变量(ADMIN_TOKEN为2.3.4.2步骤通过openssl命令生成的值)
    export OS_TOKEN=ADMIN_TOKEN
    export OS_URL=http://controller:35357/v2.0
##### 4.3.7.2 创建keystone服务信息
    openstack service create --name keystone --description "OpenStack Identity" identity
    
##### 4.3.7.3 创建keystone服务实例
    #openstack endpoint create \
    --publicurl http://controller:5000/v2.0 \
    --internalurl http://controller:5000/v2.0 \
    --adminurl http://controller:35357/v2.0 \
    --region RegionOne \
    identity
    
#### 4.3.8 创建项目、用户、角色等信息
##### 4.3.8.1 创建项目:admin
    #openstack project create --description "Admin Project" admin
    
##### 4.3.8.2 为项目admin创建用户：admin
    #openstack user create --password-prompt admin
##### 4.3.8.3 创建角色:admin
    #openstack role create admin
##### 4.3.8.4 将角色admin授权给用户admin
    #openstack role add --project admin --user admin admin
#### 4.3.9 同样创建项目demo及相关用户信息
    #openstack project create --description "Demo Project" demo
    #openstack user create --password-prompt demo
    #openstack role create user
    #openstack role add --project demo --user demo user
#### 4.3.10 创建项目：service
    #openstack project create --description "Service Project" service
#### 4.3.11 验证keystone服务
##### 4.3.11.1 删除系统配置参数
    #unset OS_TOKEN
    #unset OS_URL
##### 4.3.11.2 备份并修改keystone-dist-paste.ini配置文件
    #cp /usr/share/keystone/keystone-dist-paste.ini /usr/share/keystone/keystone-dist-paste.ini.bak
删除以下配置参数中的admin_token_auth字符串

    #vi /usr/share/keystone/keystone-dist-paste.ini
    
##### 4.3.11.3 为项目admin配置环境变量脚本
    #vi /root/admin-openrc.sh
	    export OS_PROJECT_DOMAIN_ID=default
	    export OS_USER_DOMAIN_ID=default
	    export OS_PROJECT_NAME=admin
	    export OS_TENANT_NAME=admin
	    export OS_USERNAME=admin
	    export OS_PASSWORD=123456
	    export OS_AUTH_URL=http://controller:5000/v3
    
##### 4.3.11.4 为项目demo配置环境变量脚本
    #vi /root/demo-openrc.sh  
	    export OS_PROJECT_DOMAIN_ID=default
	    export OS_USER_DOMAIN_ID=default
	    export OS_PROJECT_NAME=demo
	    export OS_TENANT_NAME=demo
	    export OS_USERNAME=demo
	    export OS_PASSWORD=123456
	    export OS_AUTH_URL=http://controller:5000/v3
##### 4.3.11.5 使用admin用户查看项目列表
    [root@controller ~]#chmod  +x /root/admin-openrc.sh
    [root@controller ~]#source /root/admin-openrc.sh
    [root@controller ~]#openstack project list
    
    +----------------------------------+---------+
    | ID                               | Name    |
    +----------------------------------+---------+
    | 54b423c7655146e691207d5c2e91d464 | demo    |
    | 80c94cfaac6d4649b3cdb282cae6f406 | service |
    | dd3ebd91af2c40e7b2ddc411d8826321 | admin   |
    +----------------------------------+---------+
    
##### 4.3.11.6 使用demo用户获取token
    [root@controller ~]# source demo-openrc.sh
    [root@controller ~]# openstack token issue
    
    +------------+----------------------------------+
    | Field      | Value                            |
    +------------+----------------------------------+
    | expires    | 2016-05-07T10:09:52.940266Z      |
    | id         | 0808f1b47d794029951d9f0dcb4da62d |
    | project_id | 54b423c7655146e691207d5c2e91d464 |
    | user_id    | a8ebdcfb9d174e05b4d135a911aa5aad |
    +------------+----------------------------------+
    
### 4.4 安装镜像服务glance
#### 4.4.1 创建数据库
    #mysql -uroot -p123456
    
    CREATE DATABASE glance;
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance';
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance';
    
#### 4.4.2 创建用户：glance，并授权
    #source /root/admin-openrc.sh
    #openstack user create --password-prompt glance
    #openstack role add --project service --user glance admin
#### 4.4.3 创建服务及服务实例
    #openstack service create --name glance --description "OpenStack Image service" image
    #openstack endpoint create \
    --publicurl http://controller:9292 \
    --internalurl http://controller:9292 \
    --adminurl http://controller:9292 \
    --region RegionOne \
    image
#### 4.4.4 安装glance
    #yum install openstack-glance python-glance python-glanceclient -y
#### 4.4.5 配置glance api组件
##### 4.4.5.1 备份配置文件
    #cp /etc/glance/glance-api.conf /etc/glance/glance-api.conf.bak
##### 4.4.5.2 修改配置文件
    #vi /etc/glance/glance-api.conf
	   
	    [DEFAULT]
	    verbose = True
	    notification_driver = noop
	    [database]
	    connection = mysql://glance:glance@localhost/glance
	    [keystone_authtoken]
	    auth_uri = http://controller:5000
	    auth_url = http://controller:35357
	    auth_plugin = password
	    project_domain_id = default
	    user_domain_id = default
	    project_name = service
	    username = glance
	    password = 123456
	    [paste_deploy]
	    flavor = keystone
	    [glance_store]
	    default_store = file
	    filesystem_store_datadir = /var/lib/glance/images/
    
#### 4.4.6 配置glance registry组件
##### 4.4.6.1 备份配置文件
    #cp /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.bak
##### 4.4.6.2 修改配置文件
    #vi /etc/glance/glance-registry.conf
    
	    [DEFAULT]
	    verbose = True
	    notification_driver = noop
	    [database]
	    connection = mysql://glance:glance@localhost/glance
	    [keystone_authtoken]
	    auth_uri = http://controller:5000
	    auth_url = http://controller:35357
	    auth_plugin = password
	    project_domain_id = default
	    user_domain_id = default
	    project_name = service
	    username = glance
	    password = 123456
	    [paste_deploy]
	    flavor = keystone
    
#### 4.4.7 初始化glance数据库
    #su -s /bin/sh -c "glance-manage db_sync" glance
#### 4.4.8 启动glance服务
    #systemctl enable openstack-glance-api.service openstack-glance-registry.service
    #systemctl start openstack-glance-api.service openstack-glance-registry.service
#### 4.4.9 验证glance服务
    #echo "export OS_IMAGE_API_VERSION=2" | tee -a admin-openrc.sh \
    demo-openrc.sh
    #source /root/admin-openrc.sh
    #glance image-create --name "cirros-0.3.4-x86_64" \
    --file /root/cirros-0.3.4-x86_64-disk.img \
    --disk-format qcow2 --container-format bare --visibility public --progress
使用openstack image list查看镜像是否上传完成。

## 5. Nova组件安装
### 5.1 在controller节点安装
#### 5.1.1 创建数据库
    #mysql -uroot -p123456
    
    CREATE DATABASE nova;
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'nova';
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova';
    
#### 5.1.2 创建用户：nova，并授权用户权限
    #openstack user create --password-prompt nova
    #openstack role add --project service --user nova admin
#### 5.1.3 创建服务及服务实例
    #openstack service create --name nova \
    --description "OpenStack Compute" compute
    #openstack endpoint create --publicurl http://controller:8774/v2/%\(tenant_id\)s \
    --internalurl http://controller:8774/v2/%\(tenant_id\)s \
    --adminurl http://controller:8774/v2/%\(tenant_id\)s \
    --region RegionOne \
    compute
#### 5.1.4 安装nova组件
    #yum install openstack-nova-api openstack-nova-cert openstack-nova-conductor \
    openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler \
    python-novaclient -y
#### 5.1.5 配置nova.conf
##### 5.1.5.1 备份配置文件
    #cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
##### 5.1.5.2 修改配置文件
    #vi /etc/nova/nova.conf
    
    [DEFAULT]
    verbose = True
    rpc_backend = rabbit
    auth_strategy = keystone
    #必须用ip
    my_ip = 192.168.199.80
    vncserver_listen = 192.168.199.80
    vncserver_proxyclient_address = 192.168.199.80
    [oslo_messaging_rabbit]
    rabbit_host = controller
    rabbit_userid = openstack
    rabbit_password = 123456
    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_url = http://controller:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = nova
    password = 123456
    [glance]
    host = controller
    [oslo_concurrency]
    lock_path = /var/lib/nova/tmp
    [database]
    connection = mysql://nova:nova@localhost/nova
    
#### 5.1.6 初始化nova数据库
    #su -s /bin/sh -c "nova-manage db sync" nova
#### 5.1.7 启动nova
    #systemctl enable openstack-nova-api.service openstack-nova-cert.service \
    openstack-nova-consoleauth.service openstack-nova-scheduler.service \
    openstack-nova-conductor.service openstack-nova-novncproxy.service
    #systemctl start openstack-nova-api.service openstack-nova-cert.service \
    openstack-nova-consoleauth.service openstack-nova-scheduler.service \
    openstack-nova-conductor.service openstack-nova-novncproxy.service

### 5.2 compute-node-01安装
#### 5.2.1 安装nova组件
    #yum install openstack-nova-compute sysfsutils -y
#### 5.2.2 配置nova
##### 5.2.2.1备份配置文件
    #cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
##### 5.2.2.2修改配置文件
    #vi /etc/nova/nova.conf
    
    [DEFAULT]
    verbose = True
    rpc_backend = rabbit
    auth_strategy = keystone
    my_ip = 192.168.199.81
    vnc_enabled = True
    vncserver_listen = 0.0.0.0
    vncserver_proxyclient_address = 192.168.199.81
    novncproxy_base_url = http://controller:6080/vnc_auto.html
    [oslo_messaging_rabbit]
    rabbit_host = controller
    rabbit_userid = openstack
    rabbit_password = 123456
    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_url = http://controller:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = nova
    password = 123456
    [glance]
    host = controller
    [oslo_concurrency]
    lock_path = /var/lib/nova/tmp
    [libvirt]
    virt_type = qemu
    
#### 5.2.3 启动nova
    #systemctl enable libvirtd.service openstack-nova-compute.service
    #systemctl start libvirtd.service openstack-nova-compute.service
#### 5.2.4验证
以下操作均在controller节点上操作

    [root@controller ~]# source /root/admin-openrc.sh
    [root@controller ~]# nova service-list
    
    +----+------------------+-----------------+----------+---------+-------+----------------------------+-----------------+
    | Id | Binary           | Host            | Zone     | Status  | State | Updated_at                 | Disabled Reason |
    +----+------------------+-----------------+----------+---------+-------+----------------------------+-----------------+
    | 1  | nova-consoleauth | controller      | internal | enabled | up    | 2016-05-07T13:55:42.000000 | -               |
    | 2  | nova-conductor   | controller      | internal | enabled | up    | 2016-05-07T13:55:42.000000 | -               |
    | 3  | nova-scheduler   | controller      | internal | enabled | up    | 2016-05-07T13:55:43.000000 | -               |
    | 4  | nova-cert        | controller      | internal | enabled | up    | 2016-05-07T13:55:43.000000 | -               |
    | 5  | nova-compute     | compute-node-01 | nova     | enabled | up    | 2016-05-07T13:55:44.000000 | -               |
    +----+------------------+-----------------+----------+---------+-------+----------------------------+-----------------+
    
    [root@controller ~]# nova image-list
    
    +--------------------------------------+---------------------+--------+--------+
    | ID                                   | Name                | Status | Server |
    +--------------------------------------+---------------------+--------+--------+
    | 66f03a7f-cdb2-4111-84c0-0af53327329e | cirros-0.3.4-x86_64 | ACTIVE |        |
    +--------------------------------------+---------------------+--------+--------+
    

## 6. Neutron组件安装

### 6.1 在controller节点安装
#### 6.1.1 创建数据库
    #mysql -uroot -p123456
    
    create database neutron;
    GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'neutron';
    GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron';
    
#### 6.1.2 创建用户：neutron，并授权用户权限
    #openstack user create --password-prompt neutron
    #openstack role add --project service --user neutron admin
#### 6.1.3 注册服务及服务实例
    #openstack service create --name neutron \
    --description "OpenStack Networking" network
    #openstack endpoint create \
    --publicurl http://controller:9696 \
    --adminurl http://controller:9696 \
    --internalurl http://controller:9696 \
    --region RegionOne \
    network
#### 6.1.4 安装neutron组件
    #yum install openstack-neutron openstack-neutron-ml2 \
    python-neutronclientwhich -y

#### 6.1.5 配置neutron.conf
##### 6.1.5.1 备份
    #cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
##### 6.1.5.2 修改配置文件
    #vi /etc/neutron/neutron.conf 
    
    [DEFAULT]
    verbose = True
    rpc_backend = rabbit
    auth_strategy = keystone
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    notify_nova_on_port_status_changes = True
    notify_nova_on_port_data_changes = True
    nova_url = http://controller:8774/v2

    [oslo_messaging_rabbit]
    rabbit_host = controller
    rabbit_userid = openstack
    rabbit_password = 123456

    [database]
    connection = mysql://neutron:neutron@localhost/neutron

    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_url = http://controller:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron
    password = 123456

    [nova]
    auth_url = http://controller:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    region_name = RegionOne
    project_name = service
    username = nova
    password = 123456
    
#### 6.1.6 配置ml2插件
##### 6.1.6.1 备份
    #cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
##### 6.1.6.2 配置
    #vi /etc/neutron/plugins/ml2/ml2_conf.ini
    
    [ml2]
    type_drivers = flat,vlan,gre,vxlan
    tenant_network_types = gre
    mechanism_drivers = openvswitch

    [ml2_type_gre]
    tunnel_id_ranges = 1:1000

    [securitygroup]
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    
##### 6.1.6.3 建立link文件
    #ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
#### 6.1.7 配置nova.conf
##### 6.1.7.1 备份
    #cp /etc/nova/nova.conf /etc/nova/nova.conf.bak1
##### 6.1.7.2 配置
    #vi /etc/nova/nova.conf
    
    [DEFAULT]
    network_api_class = nova.network.neutronv2.api.API
    security_group_api = neutron
    linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver = nova.virt.firewall.NoopFirewallDriver

    [neutron]
    url = http://controller:9696
    auth_strategy = keystone
    admin_auth_url = http://controller:35357/v2.0
    admin_tenant_name = service
    admin_username = neutron
    admin_password = 123456
   
#### 6.1.8 初始化数据库
    #su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
    --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
#### 6.1.9 重启nova服务
    #systemctl restart openstack-nova-api.service openstack-nova-scheduler.service \
    openstack-nova-conductor.service
#### 6.1.10 启动neutron服务
    #systemctl enable neutron-server.service
    #systemctl start neutron-server.service
#### 6.1.11 验证
    [root@controller ~]#source /root/admin-openrc.sh
    [root@controller ~]# neutron ext-list
    
    +-----------------------+-----------------------------------------------+
    | alias                 | name                                          |
    +-----------------------+-----------------------------------------------+
    | security-group        | security-group                                |
    | l3_agent_scheduler    | L3 Agent Scheduler                            |
    | net-mtu               | Network MTU                                   |
    | ext-gw-mode           | Neutron L3 Configurable external gateway mode |
    | binding               | Port Binding                                  |
    | provider              | Provider Network                              |
    | agent                 | agent                                         |
    | quotas                | Quota management support                      |
    | subnet_allocation     | Subnet Allocation                             |
    | dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
    | l3-ha                 | HA Router extension                           |
    | multi-provider        | Multi Provider Network                        |
    | external-net          | Neutron external network                      |
    | router                | Neutron L3 Router                             |
    | allowed-address-pairs | Allowed Address Pairs                         |
    | extraroute            | Neutron Extra Route                           |
    | extra_dhcp_opt        | Neutron Extra DHCP opts                       |
    | dvr                   | Distributed Virtual Router                    |
    +-----------------------+-----------------------------------------------+
   
### 6.2 在network-node-01节点安装
#### 6.2.1 启用IP转发功能
##### 6.2.1.1 备份配置文件
    #cp /etc/sysctl.conf /etc/sysctl.conf.bak
##### 6.2.1.2 修改配置
    #vi /etc/sysctl.conf
    
    net.ipv4.ip_forward=1
    net.ipv4.conf.all.rp_filter=0
    net.ipv4.conf.default.rp_filter=0
    
##### 6.2.1.3 启用配置
    #sysctl -p
#### 6.2.2 安装neutron组件
    #yum install openstack-neutron openstack-neutron-ml2 \
    openstack-neutron-openvswitch -y
#### 6.2.3 配置neutron.conf
##### 6.2.3.1 备份
    #cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
##### 6.2.3.2 配置
    #vi /etc/neutron/neutron.conf
    
    [DEFAULT]
    verbose = True
    rpc_backend = rabbit
    auth_strategy = keystone
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True

    [oslo_messaging_rabbit]
    rabbit_host = controller
    rabbit_userid = openstack
    rabbit_password = 123456

    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_url = http://controller:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron
    password = 123456
    
#### 6.2.4 配置ML2插件
    备注：配置【OVS】段中的IP为network-node-01节点的桥接网卡的地址，br-ex为虚拟网桥
##### 6.2.4.1 备份
    #cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
##### 6.2.4.2 配置
    #vi /etc/neutron/plugins/ml2/ml2_conf.ini
    
    [ml2]
    type_drivers = flat,vlan,gre,vxlan
    tenant_network_types = gre
    mechanism_drivers = openvswitch

    [ml2_type_flat]
    flat_networks = external

    [ml2_type_gre]
    tunnel_id_ranges = 1:1000

    [securitygroup]
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [ovs]
    local_ip = 192.168.199.82
    bridge_mappings = external:br-ex

    [agent]
    tunnel_types = gre
    
#### 6.2.5 创建链接文件
    #ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
#### 6.2.6 配置ML3插件
##### 6.2.6.1 备份
    #cp /etc/neutron/l3_agent.ini /etc/neutron/l3_agent.ini.bak
##### 6.2.6.2 配置
    #vi /etc/neutron/l3_agent.ini
    
    [DEFAULT]
    verbose = True
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    router_delete_namespaces = True
    
#### 6.2.7 配置DHCP插件
##### 6.2.7.1 备份
    #cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bak
##### 6.2.7.2 配置
    #vi /etc/neutron/dhcp_agent.ini
    
    verbose = True
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    dhcp_delete_namespaces = True
    dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
    
##### 6.2.7.3 创建dnsmasq-neutron.conf并添加以下参数：
    #vi /etc/neutron/dnsmasq-neutron.conf
    
    dhcp-option-force=26,1454
   
#### 6.2.8 配置元数据插件
    备注：metadata_proxy_shared_secret密码可以根据需求自行设置
##### 6.2.8.1 备份
    #cp /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.bak
##### 6.2.8.2 配置
    #vi /etc/neutron/metadata_agent.ini
删除[DEFAULT]段中的默认配置
在[DEFAULT]段中增加以下内容

    verbose = True
    auth_uri = http://controller:5000
    auth_url = http://controller:35357
    auth_region = RegionOne
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron
    password = 123456
    nova_metadata_ip = controller
    metadata_proxy_shared_secret = 123456

#### 6.2.9 配置nova.conf(在controller节点配置)
备注：metadata_proxy_shared_secret密码为4.2.8.2节配置的密码
##### 6.2.9.1 备份
    #cp /etc/nova/nova.conf /etc/nova/nova.conf.bak2
##### 6.2.9.2 在[neutron]节点中增加以下内容
    #vi /etc/nova/nova.conf
 
    service_metadata_proxy = True
    metadata_proxy_shared_secret = 123456
 
##### 6.2.9.3 重启nova服务
    #systemctl restart openstack-nova-api.service openstack-nova-scheduler.service \
    openstack-nova-conductor.service
#### 6.2.10启动服务
    #systemctl enable openvswitch.service
    #systemctl start openvswitch.service
#### 6.2.11 创建虚拟路由器
##### 6.2.11.1 创建虚拟路由器
    #ovs-vsctl add-br br-ex
##### 6.2.11.2 添加网卡到虚拟路由器（enp0s8为网络节点第二块网卡名，设备名称先通过ifconfig命令查看，另外这块网卡不需要分配IP地址）
    #ovs-vsctl add-port br-ex enp0s8
##### 6.2.11.3 禁止GRO
    #ethtool -K enp0s8 gro off
#### 6.2.12 启动服务
    #systemctl enable neutron-openvswitch-agent.service neutron-l3-agent.service \
    neutron-dhcp-agent.service neutron-metadata-agent.service neutron-ovs-cleanup.service
    #systemctl start neutron-openvswitch-agent.service neutron-l3-agent.service \
    neutron-dhcp-agent.service neutron-metadata-agent.service
    
#### 6.2.13 验证(在controller节点)
    [root@controller ~]# source /root/admin-openrc.sh
    [root@controller ~]# neutron agent-list
    
    +--------------------------------------+--------------------+-----------------+-------+----------------+---------------------------+
    | id                                   | agent_type         | host            | alive | admin_state_up | binary                    |
    +--------------------------------------+--------------------+-----------------+-------+----------------+---------------------------+
    | 39affea9-d74b-4eea-857c-10d4e955278a | L3 agent           | network-node-01 | :-)   | True           | neutron-l3-agent          |
    | 56552345-4e2a-488c-a2a0-4cd5c4752a46 | Open vSwitch agent | network-node-01 | :-)   | True           | neutron-openvswitch-agent |
    | 8dc9de3c-723d-4e17-9da9-44671401f349 | Metadata agent     | network-node-01 | :-)   | True           | neutron-metadata-agent    |
    | a34a75b7-c3a1-4809-9b22-151604ea1e28 | DHCP agent         | network-node-01 | :-)   | True           | neutron-dhcp-agent        |
    +--------------------------------------+--------------------+-----------------+-------+----------------+---------------------------+
    
### 6.3 在compute-node-01节点安装
#### 6.3.1 配置IP转发
##### 6.3.1.1 备份
    #cp /etc/sysctl.conf /etc/sysctl.conf.bak
##### 6.3.1.2 配置
    #vi /etc/sysctl.conf
    
    net.ipv4.conf.all.rp_filter=0
    net.ipv4.conf.default.rp_filter=0
    net.bridge.bridge-nf-call-iptables=1
    net.bridge.bridge-nf-call-ip6tables=1
    
##### 6.3.1.3 生效配置
    #sysctl -p
    #modprobe bridge
#### 6.3.2安装
    #yum install openstack-neutron openstack-neutron-ml2 \
    openstack-neutron-openvswitch -y
#### 6.3.3 配置neutron
##### 6.3.3.1 备份
    #cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
##### 6.3.3.2 配置（需要删除[keystone_authtoken]中已存在的配置）
    #vi /etc/neutron/neutron.conf
    
    [DEFAULT]
    verbose = True
    rpc_backend = rabbit
    auth_strategy = keystone
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True

    [oslo_messaging_rabbit]
    rabbit_host = controller
    rabbit_userid = openstack
    rabbit_password = 123456

    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_url = http://controller:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron
    password = 123456
    
#### 6.3.4 配置ml2插件
    备注：配置【OVS】段中的IP为compute-node-01节点的桥接网卡的地址，br-ex为虚拟网桥
##### 6.3.4.1 备份
    #cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
##### 6.3.4.2 配置
    #vi /etc/neutron/plugins/ml2/ml2_conf.ini
    
    [ml2]
    type_drivers = flat,vlan,gre,vxlan
    tenant_network_types = gre
    mechanism_drivers = openvswitch

    [ml2_type_gre]
    tunnel_id_ranges = 1:1000

    [securitygroup]
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [ovs]
    local_ip = 192.168.199.81

    [agent]
    tunnel_types = gre
    
#### 6.3.5 创建link文件
    #ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
#### 6.3.6 配置nova.conf
##### 6.3.6.1 备份
    #cp /etc/nova/nova.conf /etc/nova/nova.conf.bak1
##### 6.3.6.2 配置
    vi /etc/nova/nova.conf 
    
    [DEFAULT]
    network_api_class = nova.network.neutronv2.api.API
    security_group_api = neutron
    linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver = nova.virt.firewall.NoopFirewallDriver

    [neutron]
    url = http://controller:9696
    auth_strategy = keystone
    admin_auth_url = http://controller:35357/v2.0
    admin_tenant_name = service
    admin_username = neutron
    admin_password = 123456
    
#### 6.3.7 启动
    #systemctl enable openvswitch.service
    #systemctl start openvswitch.service
    #systemctl enable neutron-openvswitch-agent.service
    #systemctl start neutron-openvswitch-agent.service

#### 6.3.8 验证(在controller节点)
    [root@controller ~]# neutron agent-list
    
    +--------------------------------------+--------------------+-----------------+-------+----------------+---------------------------+
    | id                                   | agent_type         | host            | alive | admin_state_up | binary                    |
    +--------------------------------------+--------------------+-----------------+-------+----------------+---------------------------+
    | 07647c62-f1af-4108-9f70-a5c01b870a9c | Open vSwitch agent | compute-node-01 | :-)   | True           | neutron-openvswitch-agent |
    | 39affea9-d74b-4eea-857c-10d4e955278a | L3 agent           | network-node-01 | :-)   | True           | neutron-l3-agent          |
    | 56552345-4e2a-488c-a2a0-4cd5c4752a46 | Open vSwitch agent | network-node-01 | :-)   | True           | neutron-openvswitch-agent |
    | 8dc9de3c-723d-4e17-9da9-44671401f349 | Metadata agent     | network-node-01 | :-)   | True           | neutron-metadata-agent    |
    | a34a75b7-c3a1-4809-9b22-151604ea1e28 | DHCP agent         | network-node-01 | :-)   | True           | neutron-dhcp-agent        |
    +--------------------------------------+--------------------+-----------------+-------+----------------+---------------------------+
    
### 6.4 创建网络
#### 6.4.1 创建扩展网络
    #neutron net-create ext-net --router:external \
    --provider:physical_network external --provider:network_type flat
#### 6.4.2 创建扩展网络子网
    #neutron subnet-create ext-net 192.168.56.0/24 --name ext-subnet \
    --allocation-pool start=192.168.56.101,end=192.168.56.200 \
    --disable-dhcp --gateway 192.168.56.1
#### 6.4.3 创建租户网络
    #source /root/demo-openrc.sh
    #neutron net-create demo-net
    #neutron subnet-create demo-net 192.168.2.0/24 \
    --name demo-subnet  --gateway 192.168.2.1
#### 6.4.4 创建路由器
    #neutron router-create demo-router
    #neutron router-interface-add demo-router demo-subnet
    #neutron router-gateway-set demo-router ext-net
#### 6.4.5 设置安全组规则
    #nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
    #nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
#### 6.4.6.验证
    #source /root/admin-openrc.sh
    #neutron router-list
### 6.5 controller节点安装Dashboard
#### 6.5.1 安装
    #yum install openstack-dashboard -y
#### 6.5.2 配置
##### 6.5.2.1 备份
    cp /etc/openstack-dashboard/local_settings /etc/openstack-dashboard/local_settings.bak
##### 6.5.2.2 配置
    vi /etc/openstack-dashboard/local_settings 
    
    import sys
    reload(sys)
    sys.setdefaultencoding('utf8')

    OPENSTACK_HOST = "controller"
    ALLOWED_HOSTS = ['*', ]
    CACHES = {
        'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '127.0.0.1:11211',
        }
    }
    OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
    TIME_ZONE = "Asia/Shanghai"
    
#### 6.5.3 系统权限配置
    #setsebool -P httpd_can_network_connect on
    #chown -R apache:apache /usr/share/openstack-dashboard/static
    #systemctl restart httpd.service memcached.service
#### 6.5.4 验证
    http://192.168.199.80/dashboard
    demo/123456
![2016-05-07_23-19-10](/uploads/2016/06/2016-05-07_23-19-10.jpg)

## 7. Cinder组件安装
### 7.1 Controller节点安装
#### 7.1.1 创建数据库
    #mysql -uroot -p123456
    
    CREATE DATABASE cinder;
    GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'cinder';
    GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'cinder';
    
#### 7.1.2 创建账号及服务
    #source /root/admin-openrc.sh
    #openstack user create --password-prompt cinder
    #openstack role add --project service --user cinder admin
    #openstack service create --name cinder \
    --description "OpenStack Block Storage" volume
    #openstack service create --name cinderv2 \
    --description "OpenStack Block Storage" volumev2
    #openstack endpoint create \
    --publicurl http://controller:8776/v2/%\(tenant_id\)s \
    --internalurl http://controller:8776/v2/%\(tenant_id\)s \
    --adminurl http://controller:8776/v2/%\(tenant_id\)s \
    --region RegionOne \
    volume
    #openstack endpoint create \
    --publicurl http://controller:8776/v2/%\(tenant_id\)s \
    --internalurl http://controller:8776/v2/%\(tenant_id\)s \
    --adminurl http://controller:8776/v2/%\(tenant_id\)s \
    --region RegionOne \
    volumev2
    
#### 7.1.3 安装Cinder组件
    #yum install openstack-cinder python-cinderclient python-oslo-db -y
#### 7.1.4 配置
备注：my_ip为本机IP
    
    #cp /usr/share/cinder/cinder-dist.conf /etc/cinder/cinder.conf
    #chown -R cinder:cinder /etc/cinder/cinder.conf
    #vi /etc/cinder/cinder.conf
	    [DEFAULT]
	    verbose = True
	    rpc_backend = rabbit
	    auth_strategy = keystone
	    my_ip = 192.168.199.80

	    [database]
	    connection = mysql://cinder:cinder@localhost/cinder
	
	    [oslo_messaging_rabbit]
	    rabbit_host = controller
	    rabbit_userid = openstack
	    rabbit_password = 123456
	
	    [keystone_authtoken]
	    auth_uri = http://controller:5000
	    auth_url = http://controller:35357
	    auth_plugin = password
	    project_domain_id = default
	    user_domain_id = default
	    project_name = service
	    username = cinder
	    password = 123456
	
	    [oslo_concurrency]
	    lock_path = /var/lock/cinder

#### 7.1.5 数据初始化
    #su -s /bin/sh -c "cinder-manage db sync" cinder
    
#### 7.1.6 启动服务
    #systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
    #systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
    
### 7.2 在block-node-01节点上安装
#### 7.2.1 安装支持包
##### 7.2.1.1 安装
    #yum install qemu lvm2 -y
##### 7.2.1.2 启动
    #systemctl enable lvm2-lvmetad.service
    #systemctl start lvm2-lvmetad.service
#### 7.2.2 创建逻辑卷
    #fdisk /dev/sdb
    
    [root@block-node-01 ~]# fdisk /dev/sdb
    Welcome to fdisk (util-linux 2.23.2).

    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.

    Device does not contain a recognized partition table
    Building a new DOS disklabel with disk identifier 0xb091be35.

    Command (m for help): n
    Partition type:
       p   primary (0 primary, 0 extended, 4 free)
       e   extended
    Select (default p): p
    Partition number (1-4, default 1):
    First sector (2048-4194303, default 2048):
    Using default value 2048
    Last sector, +sectors or +size{K,M,G} (2048-4194303, default 4194303):
    Using default value 4194303
    Partition 1 of type Linux and of size 2 GiB is set

    Command (m for help): t
    Selected partition 1
    Hex code (type L to list all codes): 8e
    Changed type of partition 'Linux' to 'Linux LVM'

    Command (m for help): p

    Disk /dev/sdb: 2147 MB, 2147483648 bytes, 4194304 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk label type: dos
    Disk identifier: 0xb091be35

       Device Boot      Start         End      Blocks   Id  System
    /dev/sdb1            2048     4194303     2096128   8e  Linux LVM

    Command (m for help):w
    Calling ioctl() to re-read partition table.
    Syncing disks.
    
#### 7.2.3 更新内核中的分区表
    #partprobe
#### 7.2.4 创建物理卷及卷组
    #pvcreate /dev/sdb1
    #vgcreate cinder-volumes /dev/sdb1
#### 7.2.5 修改lvm服务配置
##### 7.2.5.1 备份
    #cp /etc/lvm/lvm.conf /etc/lvm/lvm.conf.bak
##### 7.2.5.2 修改
    #vi /etc/lvm/lvm.conf
     
    devices {
    filter = [ "a/sda/", "a/sdb/", "r/.*/"]
    
##### 7.2.5.3 重启lvm服务
    #systemctl restart lvm2-lvmetad.service
#### 7.2.6 安装cinder组件
    #yum install openstack-cinder targetcli python-oslo-db python-oslo-log MySQL-python -y
#### 7.2.7 配置cinder
##### 7.2.7.1 备份
    #cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
##### 7.2.7.2 配置cinder.conf
    #vi /etc/cinder/cinder.conf
    [DEFAULT]
    rpc_backend = rabbit
    auth_strategy = keystone
    my_ip = 192.168.199.83
    enabled_backends = lvm
    glance_host = controller
    verbose = True

    [oslo_messaging_rabbit]
    rabbit_host = controller
    rabbit_userid = openstack
    rabbit_password = 123456

    [database]
    connection = mysql://cinder:cinder@controller/cinder

    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_url = http://controller:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = cinder
    password = 123456

    [lvm]
    volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
    volume_group = cinder-volumes
    iscsi_protocol = iscsi
    iscsi_helper = lioadm

    [oslo_concurrency]
    lock_path = /var/lock/cinder
    
##### 7.2.7.3 创建文件夹
    #mkdir -p /var/lock/cinder
    #chown cinder:cinder /var/lock/cinder
    
#### 7.2.8 启动cinder服务
    #systemctl enable openstack-cinder-volume.service target.service
    #systemctl start openstack-cinder-volume.service target.service
#### 7.2.9 验证(在controller节点)
    #echo "export OS_VOLUME_API_VERSION=2" | tee -a admin-openrc.sh demo-openrc.sh
    
    [root@controller ~]# source /root/admin-openrc.sh
    [root@controller ~]# cinder service-list
    +------------------+-------------------+------+---------+-------+----------------------------+-----------------+
    |      Binary      |        Host       | Zone |  Status | State |         Updated_at         | Disabled Reason |
    +------------------+-------------------+------+---------+-------+----------------------------+-----------------+
    | cinder-scheduler |     controller    | nova | enabled |   up  | 2016-05-22T04:41:04.000000 |        -        |
    |  cinder-volume   | block-node-01@lvm | nova | enabled |   up  | 2016-05-22T04:41:01.000000 |        -        |
    +------------------+-------------------+------+---------+-------+----------------------------+-----------------+
    [root@controller ~]# source demo-openrc.sh
    [root@controller ~]# cinder create --name demo-volume1 1
    +---------------------------------------+--------------------------------------+
    |                Property               |                Value                 |
    +---------------------------------------+--------------------------------------+
    |              attachments              |                  []                  |
    |           availability_zone           |                 nova                 |
    |                bootable               |                false                 |
    |          consistencygroup_id          |                 None                 |
    |               created_at              |      2016-05-22T04:42:35.000000      |
    |              description              |                 None                 |
    |               encrypted               |                False                 |
    |                   id                  | 75ed439d-9d42-4ae9-8615-c4f1b8ff9ae0 |
    |                metadata               |                  {}                  |
    |              multiattach              |                False                 |
    |                  name                 |             demo-volume1             |
    |      os-vol-tenant-attr:tenant_id     |   54b423c7655146e691207d5c2e91d464   |
    |   os-volume-replication:driver_data   |                 None                 |
    | os-volume-replication:extended_status |                 None                 |
    |           replication_status          |               disabled               |
    |                  size                 |                  1                   |
    |              snapshot_id              |                 None                 |
    |              source_volid             |                 None                 |
    |                 status                |               creating               |
    |                user_id                |   a8ebdcfb9d174e05b4d135a911aa5aad   |
    |              volume_type              |                 None                 |
    +---------------------------------------+--------------------------------------+
    [root@controller ~]# cinder list
    +--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
    |                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Attached to |
    +--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
    | 75ed439d-9d42-4ae9-8615-c4f1b8ff9ae0 | available | demo-volume1 |  1   |      -      |  false   |             |
    +--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
    

## 8. Swift组件安装
### 8.1 在controller节点安装
#### 8.1.1 创建账号及服务
    #source /root/admin-openrc.sh
    #openstack user create --password-prompt swift
    #openstack role add --project service --user swift admin
    #openstack service create --name swift \
    --description "OpenStack Object Storage" object-store
    #openstack endpoint create \
    --publicurl 'http://controller:8080/v1/AUTH_%(tenant_id)s' \
    --internalurl 'http://controller:8080/v1/AUTH_%(tenant_id)s' \
    --adminurl http://controller:8080 \
    --region RegionOne \
    object-store
#### 8.1.2 安装swift组件
    #yum install openstack-swift-proxy python-swiftclient python-keystoneclient \
    python-keystonemiddleware memcached -y
#### 8.1.3 配置proxy-server
##### 8.1.3.1 下载配置文件模板
    #curl -o /etc/swift/proxy-server.conf \
    https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/kilo
    
    [root@controller ~]# curl -o /etc/swift/proxy-server.conf \
    > https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/kilo
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100 28586  100 28586    0     0  13043      0  0:00:02  0:00:02 --:--:-- 13047

    
##### 8.1.3.2 修改配置文件
    #vi /etc/swift/proxy-server.conf
    
    [DEFAULT]
    bind_port = 8080
    user = swift
    swift_dir = /etc/swift

    [pipeline:main]
    pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo proxy-logging proxy-server

    [app:proxy-server]
    use = egg:swift#proxy
    account_autocreate = true
    [filter:keystoneauth]
    use = egg:swift#keystoneauth
    operator_roles = admin,user
    [filter:healthcheck]
    use = egg:swift#healthcheck
    [filter:authtoken]
    paste.filter_factory = keystonemiddleware.auth_token:filter_factory
    auth_uri = http://controller:5000
    auth_url = http://controller:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = swift
    password = 123456
    delay_auth_decision = true

    [filter:cache]
    memcache_servers = 127.0.0.1:11211
    
### 8.2 在object-node-01节点安装
#### 8.2.1 系统配置
##### 8.2.1.1 安装系统组件
    #yum install xfsprogs rsync -y
##### 8.2.1.2 格式化磁盘
    #fdisk /dev/sdb
    #fdisk /dev/sdc
    #mkfs.xfs /dev/sdb1
    #mkfs.xfs /dev/sdc1
##### 8.2.1.3 创建文件夹
    #mkdir -p /srv/node/sdb1
    #mkdir -p /srv/node/sdc1
##### 8.2.1.4 修改fstab
    #vi /etc/fstab
    /dev/sdb1 /srv/node/sdb1 xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
    /dev/sdc1 /srv/node/sdc1 xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
    
##### 8.2.1.5 挂载磁盘到文件夹
    #mount /srv/node/sdb1
    #mount /srv/node/sdc1
##### 8.2.1.6 修改文件同步服务
    #cp /etc/rsyncd.conf /etc/rsyncd.conf.bak
    #vi /etc/rsyncd.conf
    
    uid = swift
    gid = swift
    log file = /var/log/rsyncd.log
    pid file = /var/run/rsyncd.pid
    address = 192.168.199.84
    [account]
    max connections = 2
    path = /srv/node/
    read only = false
    lock file = /var/lock/account.lock
    [container]
    max connections = 2
    path = /srv/node/
    read only = false
    lock file = /var/lock/container.lock
    [object]
    max connections = 2
    path = /srv/node/
    read only = false
    lock file = /var/lock/object.lock
    
##### 8.2.1.7 启动文件同步服务
    #systemctl enable rsyncd.service
    #systemctl start rsyncd.service
#### 8.2.2 安装
    #yum install openstack-swift-account openstack-swift-container \
    openstack-swift-object -y
#### 8.2.3 配置swift
##### 8.2.3.1 下载配置文件模板
    #curl -o /etc/swift/account-server.conf \
    https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/kilo
    #curl -o /etc/swift/container-server.conf \
    https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/kilo
    #curl -o /etc/swift/object-server.conf \
    https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/kilo
    #curl -o /etc/swift/container-reconciler.conf \
    https://git.openstack.org/cgit/openstack/swift/plain/etc/container-reconciler.conf-sample?h=stable/kilo
    #curl -o /etc/swift/object-expirer.conf \
    https://git.openstack.org/cgit/openstack/swift/plain/etc/object-expirer.conf-sample?h=stable/kilo
    #curl -o /etc/swift/swift.conf \
    https://git.openstack.org/cgit/openstack/swift/plain/etc/swift.conf-sample?h=stable/kilo
##### 8.2.3.2 修改/etc/swift/account-server.conf
    
    [DEFAULT]
    bind_ip = 192.168.199.84
    bind_port = 6002
    user = swift
    swift_dir = /etc/swift
    devices = /srv/node

    [pipeline:main]
    pipeline = healthcheck recon account-server
    [filter:healthcheck]
    use = egg:swift#healthcheck
    [filter:recon]
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift
    
##### 8.2.3.3 修改/etc/swift/container-server.conf
    
    [DEFAULT]
    bind_ip = 192.168.199.84
    bind_port = 6001
    user = swift
    swift_dir = /etc/swift
    devices = /srv/node

    [pipeline:main]
    pipeline = healthcheck recon container-server
    [filter:healthcheck]
    use = egg:swift#healthcheck
    [filter:recon]
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift
    
##### 8.2.3.4 修改/etc/swift/object-server.conf
    
    [DEFAULT]
    bind_ip = 192.168.199.84
    bind_port = 6000
    user = swift
    swift_dir = /etc/swift
    devices = /srv/node

    [pipeline:main]
    pipeline = healthcheck recon object-server
    [filter:healthcheck]
    use = egg:swift#healthcheck
    [filter:recon]
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift
    recon_lock_path = /var/lock
    
##### 8.2.3.5 修改/etc/swift/swift.conf
备注：HASH_PATH_SUFFIX 和 HASH_PATH_PREFIX 分别用openssl rand -hex 10命令生成。

    [swift-hash]
    swift_hash_path_suffix = HASH_PATH_SUFFIX
    swift_hash_path_prefix = HASH_PATH_PREFIX

    [storage-policy:0]
    name = Policy-0
    default = yes

##### 8.2.3.6 文件夹授权
	#chown -R swift:swift /srv/node
	#mkdir -p /var/cache/swift
	#chown -R swift:swift /var/cache/swift
	#chown -R swift:swift /etc/swift

### 8.3 在object-node-02节点安装
参照6.2章节安装。配置文件可以自己从object-node-01节点拷贝过来，只需要修改配置文件中的IP地址即可。
### 8.4 controller节点创建环（Ring）
备注：IP地址分别为object-node-01和object-node-02所对应的IP
#### 8.4.1 创建账号环（Account Ring）
	swift-ring-builder account.builder create 10 3 1
	swift-ring-builder account.builder add r1z1-192.168.199.84:6002/sdb1 100
	swift-ring-builder account.builder add r1z1-192.168.199.84:6002/sdc1 100
	swift-ring-builder account.builder add r1z2-192.168.199.85:6002/sdb1 100
	swift-ring-builder account.builder add r1z2-192.168.199.85:6002/sdc1 100
	swift-ring-builder account.builder
	swift-ring-builder account.builder rebalance
#### 8.4.2 创建容器环（Container Ring)
	swift-ring-builder container.builder create 10 3 1
	swift-ring-builder container.builder create 10 3 1
	swift-ring-builder container.builder add r1z1-192.168.199.84:6001/sdb1 100
	swift-ring-builder container.builder add r1z1-192.168.199.84:6001/sdc1 100
	swift-ring-builder container.builder add r1z2-192.168.199.85:6001/sdb1 100
	swift-ring-builder container.builder add r1z2-192.168.199.85:6001/sdc1 100
	swift-ring-builder container.builder
	swift-ring-builder container.builder rebalance
#### 8.4.3 创建对象环（Object Ring)
	swift-ring-builder object.builder create 10 3 1
	swift-ring-builder object.builder create 10 3 1
	swift-ring-builder object.builder add r1z1-192.168.199.84:6000/sdb1 100
	swift-ring-builder object.builder add r1z1-192.168.199.84:6000/sdc1 100
	swift-ring-builder object.builder add r1z2-192.168.199.85:6000/sdb1 100
	swift-ring-builder object.builder add r1z2-192.168.199.85:6000/sdc1 100
	swift-ring-builder object.builder
	swift-ring-builder object.builder rebalance
#### 8.4.4 启动服务(controller节点执行)
	#systemctl enable openstack-swift-proxy.service
	#systemctl start openstack-swift-proxy.service
	#systemctl restart memcached.service
### 8.5 启动服务（object-node-01, object-node-02节点执行）
	#systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service \
	openstack-swift-account-reaper.service openstack-swift-account-replicator.service
	#systemctl start openstack-swift-account.service openstack-swift-account-auditor.service \
	openstack-swift-account-reaper.service openstack-swift-account-replicator.service
	#systemctl enable openstack-swift-container.service openstack-swift-container-auditor.service \	
	openstack-swift-container-replicator.service openstack-swift-container-updater.service
	#systemctl start openstack-swift-container.service openstack-swift-container-auditor.service \
	openstack-swift-container-replicator.service openstack-swift-container-updater.service
	#systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service \
	openstack-swift-object-replicator.service openstack-swift-object-updater.service
	#systemctl start openstack-swift-object.service openstack-swift-object-auditor.service \
	openstack-swift-object-replicator.service openstack-swift-object-updater.service