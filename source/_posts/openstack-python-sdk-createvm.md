---
title: 'OpenStack入门之八:使用python SDK代码启动虚拟机实例'
tags:
  - openstack
  - Python
id: 1203
categories:
  - 云计算及虚拟化
date: 2016-06-26 12:46:32
---

本文基于[OpenStack入门之二:devstack开发环境搭建](/2016/05/openstack-devstack/)一文配置的环境，使用python SDK的代码来启动一个虚拟机实例。
## 1. 建立PyDev项目
### 1.1 新建PyDev项目
启动eclipse，新建PyDev项目，命名为nova-createvm
### 1.2 新建包
设置包名com.nova.demo
### 1.3 新建Python文件
新建Python源代码文件NovaApiCreateVM.py，要启动一个新实例，可以使用 Client.servers.create 方法，需要传递一个 image 对象和 flavor 对象，代码如下：

```python
import time
from keystoneclient.auth.identity import v3
from keystoneclient import session
from keystoneclient.v3 import client as keystoneapi
from novaclient import client as novapi
auth_url = 'http://192.168.199.20:35357/v3/'
username = 'demo'
user_domain_name = 'Default'
project_name = 'demo'
project_domain_name = 'Default'
password = 'passw0rd'
auth = v3.Password(auth_url=auth_url,
                   username=username,
                   password=password,
                   project_name=project_name,
                   project_domain_name=project_domain_name,
                   user_domain_name=user_domain_name)
sess = session.Session(auth=auth)
keystone = keystoneapi.Client(session=sess)
nova = novapi.Client(2, session=keystone.session)
image = nova.images.find(name="cirros-0.3.4-x86_64-uec")
flavor = nova.flavors.find(name="m1.tiny")
instance = nova.servers.create(name="test", image=image, flavor=flavor)
# Poll at 5 second intervals, until the status is no longer 'BUILD'
status = instance.status
while status == 'BUILD':
    time.sleep(5)
    # Retrieve the instance again so the status field updates
    instance = nova.servers.get(instance.id)
    status = instance.status
print "status: %s" % status
```    
nova-createvm文件如下，
![2016-06-26_12-29-37](/uploads/2016/06/2016-06-26_12-29-37.jpg)

## 2. 运行验证
### 2.1 运行
右键文件名-->Run As--> Python Run，等待几秒钟，我们看到运行结果的状态为acvtive
![2016-06-26_12-08-50](/uploads/2016/06/2016-06-26_12-08-50.jpg)
### 2.2 验证
进入网站http://192.168.199.20.使用用户名demo/passw0rd登录，点击instance，我们看到test的instance已经运行中。
![2016-06-26_12-06-37](/uploads/2016/06/2016-06-26_12-06-37.jpg)
以上的例子还可以使用 Client.keypairs.create 方法创建ssh key来验证，有兴趣的同学可以试验一下。

```python
creds = get_nova_creds()
nova = nvclient.Client(**creds)
if not nova.keypairs.findall(name="mykey"):
    with open(os.path.expanduser('~/.ssh/id_rsa.pub')) as fpubkey:
        nova.keypairs.create(name="mykey", public_key=fpubkey.read())
```    