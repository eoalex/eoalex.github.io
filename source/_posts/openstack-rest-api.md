---
title: 'OpenStack入门之九:使用REST API启动虚拟机实例'
tags:
  - openstack
  - Python
id: 1216
categories:
  - 云计算及虚拟化
date: 2016-07-03 23:01:35
---
## 1. 简介
上文[OpenStack入门之八:使用python SDK代码启动虚拟机实例](/2016/06/openstack-python-sdk-createvm/)我们使用python SDK的代码来启动一个虚拟机实例。本文将演示如何使用REST API来启动一个虚拟机实例。
如果你不知道什么是REST请参考这篇博文，[理解RESTful架构](http://www.ruanyifeng.com/blog/2011/09/restful.html)
从上文[OpenStack入门之八:使用python SDK代码启动虚拟机实例](/2016/06/openstack-python-sdk-createvm/)我们知道要想创建一个虚拟机，至少知道三个参数，虚拟机名称，镜像名称以及类型模板(flavor)。所以我们首先要获取image ID及flavor ID,但是要得到这些资源，首先要获取TOKEN数据以及NOVA的URL地址。

## 2. 引用库文件
那么通过什么工具来与openstack API 通讯呢，这里我们用到了PYHON的标准库urllib2,另外我们使用JSON格式来传递数据，所以还要引用JSON库。

    import json
    import urllib2
    
## 3. 获取token数据
我们通过POST用户名密码的方式获取token数据,这里简单起见直接定义。
    
    auth = {"auth":{"identity":{"methods":["password"],"password":{"user":{"domain":{"id":"default"},"name":"demo","password":"passw0rd"}}},"scope":{"project":{"domain":{"id":"default"},"name":"demo"}}}}
    
URL地址简单起见，也直接定义，这两个参数后续可以定义在配置文件抓取
    
    auth_url = 'http://192.168.199.20:35357/v3/'
    
现在我们可以获取token数据了。

```python
def token_post(auth,auth_url):
	headers = {"Content-type":"application/json","Accept": "application/json;charset=UTF-8"}
	token_url = auth_url+"auth/tokens"
	req = urllib2.Request(token_url, auth, headers)
	response = urllib2.urlopen(req)
	token = response.info().getheader("X-Subject-Token")
	response.encoding='utf-8'
	response_json = json.loads(response.read())
	response_json = byteify(response_json)
	response_json['authtoken'] = token
	return response_json
```
   
## 4. 获取NOVA URL地址
通过token返回的数据，我们可以直接筛选出NOVA的URL地址
 
```python
def get_nova_endpoint(response):
	services = response['token']['catalog']
	for i in range(0, len(services)):
	if services[i]['name'] == 'nova':
	    nova_endpoint = services[i]['endpoints'][0]['url']
	return nova_endpoint
```

## 5. 获取image ID
通过image名字获取iamge ID

```python
    def get_image_id(novaurl,tokenid,imagename):
        servers_url = novaurl+'/'+'images'
        req = urllib2.Request(servers_url)
        req.add_header('X-Auth-Token', tokenid)
        response = urllib2.urlopen(req)
        services = json.loads(response.read())['images']
        for i in range(0, len(services)):
            if services[i]['name'] == imagename:
                image_id = services[i]['id']
        return image_id
```

## 6. 获取flavor ID
通过flavor名字获取flavor ID
   
```python
    def get_flavor_id(novaurl,tokenid,flavorname):
        servers_url = novaurl+'/'+'flavors'
        req = urllib2.Request(servers_url)
        req.add_header('X-Auth-Token', tokenid)
        response = urllib2.urlopen(req)
        services = json.loads(response.read())['flavors']
        for i in range(0, len(services)):
            if services[i]['name'] == flavorname:
                flavor_id = services[i]['id']
        return flavor_id
```

## 7. 创建虚拟机
现在我们可以创建虚拟机了。
    
```python    
    def create_vmserver(novaurl,tokenid,vmname,imange_id,flaver_id):
        headers = {"Content-type":"application/json","Accept": "application/json;charset=UTF-8","X-Auth-Token":tokenid}
        #print headers
        servers_url = novaurl+"/servers"
        #print servers_url
    	body={"server": {"name": vmname, "imageRef": imange_id, "flavorRef": flaver_id}}
    	body=json.dumps(body)
    	#print body
        req = urllib2.Request(servers_url,body,headers) 
        response = urllib2.urlopen(req)
        response_json = json.loads(response.read())
        return response_json    
```

## 8. 查看结果
运行源代码
![2016-07-03_15-44-08](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/07/2016-07-03_15-44-08.jpg)

我们查看instance，看到test1已经创建。

![2016-07-03_15-17-01](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/07/2016-07-03_15-17-01.jpg)

本文参考
    `http://developer.openstack.org/api-ref/compute/?expanded=create-server-detail`

附整个源代码。
  
```python    
import json
import urllib2
def byteify(input):
    if isinstance(input, dict):
        return {byteify(key): byteify(value) for key, value in input.iteritems()}
    elif isinstance(input, list):
        return [byteify(element) for element in input]
    elif isinstance(input, unicode):
        return input.encode('utf-8')
    else:
        return input

def token_post(auth,auth_url):
    headers = {"Content-type":"application/json","Accept": "application/json;charset=UTF-8"}
    token_url = auth_url+"auth/tokens"
    req = urllib2.Request(token_url, auth, headers)
    response = urllib2.urlopen(req)
    token = response.info().getheader("X-Subject-Token")
    response.encoding='utf-8'
    response_json = json.loads(response.read())
    response_json = byteify(response_json)
    response_json['authtoken'] = token
    return response_json

def get_nova_endpoint(response):
    services = response['token']['catalog']
    for i in range(0, len(services)):
        if services[i]['name'] == 'nova':
            nova_endpoint = services[i]['endpoints'][0]['url']
    return nova_endpoint

def get_image_id(novaurl,tokenid,imagename):
    servers_url = novaurl+'/'+'images'
    req = urllib2.Request(servers_url)
    req.add_header('X-Auth-Token', tokenid)
    response = urllib2.urlopen(req)
    services = json.loads(response.read())['images']
    for i in range(0, len(services)):
        if services[i]['name'] == imagename:
            image_id = services[i]['id']
    return image_id

def get_flavor_id(novaurl,tokenid,flavorname):
    servers_url = novaurl+'/'+'flavors'
    req = urllib2.Request(servers_url)
    req.add_header('X-Auth-Token', tokenid)
    response = urllib2.urlopen(req)
    services = json.loads(response.read())['flavors']
    for i in range(0, len(services)):
        if services[i]['name'] == flavorname:
            flavor_id = services[i]['id']
    return flavor_id

def create_vmserver(novaurl,tokenid,vmname,imange_id,flaver_id):
    headers = {"Content-type":"application/json","Accept": "application/json;charset=UTF-8","X-Auth-Token":tokenid}
    #print headers
    servers_url = novaurl+"/servers"
    #print servers_url
	body={"server": {"name": vmname, "imageRef": imange_id, "flavorRef": flaver_id}}
	body=json.dumps(body)
	#print body
    req = urllib2.Request(servers_url,body,headers) 
    response = urllib2.urlopen(req)
    response_json = json.loads(response.read())
    return response_json    

if __name__ == '__main__':
    auth = {"auth":{"identity":{"methods":["password"],"password":{"user":{"domain":{"id":"default"},"name":"demo","password":"passw0rd"}}},"scope":{"project":{"domain":{"id":"default"},"name":"demo"}}}}
    auth = json.dumps(auth)
	auth_url = 'http://192.168.199.20:35357/v3/'
    response = token_post(auth,auth_url)
	#print response
	tokenid = response['authtoken']
	novaurl = get_nova_endpoint(response)
    #print novaurl
    imange_id=get_image_id(novaurl,tokenid,'cirros-0.3.4-x86_64-uec')
    flaver_id=get_flavor_id(novaurl,tokenid,'m1.tiny')
    print create_vmserver(novaurl,tokenid,'test1',imange_id,flaver_id)
```    