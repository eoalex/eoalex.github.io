---
title: 'OpenStack入门之四:故障诊断实例'
tags:
  - openstack
id: 1142
categories:
  - 云计算及虚拟化
date: 2016-06-12 14:27:47
---

## 1. 简介
对于openstack入门者来说，最怕的是出现各种错误而不知如何解决，本文简单介绍openstack日志的路径并通过实际解决一个故障来演示。
## 2. 日志的级别
首先我们来回顾一下日志里各种信息的级别。
INFO:突出强调程序的运行过程
WARNING:程序可能存在潜在的问题
ERROR:程序发生的错误
DEBUG:程序调试信息
TRACE:记录代码执行的方法和属性
## 3. HTTP状态码
由于OpenStack的各个服务之间是通过统一的REST风格的API调用，所以我们有必要回顾一下HTTP状态码。
1)100-199 用于指定客户端应相应的某些动作。

    100 (Continue/继续)
    101 (Switching Protocols/转换协议)
    
2)200-299 用于表示请求成功。 
    
    200 (OK/正常)
    201 (Created/已创建)
    202 (Accepted/接受)
    203 (Non-Authoritative Information/非官方信息)
    204 (No Content/无内容)
    205 (Reset Content/重置内容)
    206 (Partial Content/局部内容)
    
3)300-399 用于已经移动的文件并且常被包含在定位头信息中指定新的地址信息。
    
    300 (Multiple Choices/多重选择)
    301 (Moved Permanently)
    302 (Found/找到)
    303 (See Other/参见其他信息)
    304 (Not Modified/为修正)
    305 (Use Proxy/使用代理)
    305 (SC_USE_PROXY)表示所请求的文档要通过定位头信息中的代理服务器获得。这个状态码是新加入 HTTP 1.1中的。 
    307 (Temporary Redirect/临时重定向)
    
4)400-499 用于指出客户端的错误。 
    
    400 (Bad Request/错误请求)
    401 (Unauthorized/未授权)
    403 (Forbidden/禁止)
    404 (Not Found/未找到)
    405 (Method Not Allowed/方法未允许)
    406 (Not Acceptable/无法访问)
    407 (Proxy Authentication Required/代理服务器认证要求)
    408 (Request Timeout/请求超时)
    409 (Conflict/冲突)
    410 (Gone/已经不存在)
    411 (Length Required/需要数据长度)
    412 (Precondition Failed/先决条件错误)
    413 (Request Entity Too Large/请求实体过大)
    414 (Request URI Too Long/请求URI过长)
    415 (Unsupported Media Type/不支持的媒体格式)
    416 (Requested Range Not Satisfiable/请求范围无法满足)
    417 (Expectation Failed/期望失败)
    
 5)500-599 用于支持服务器错误。 
    
    500 (Internal Server Error/内部服务器错误)
    501 (Not Implemented/未实现)
    502 (Bad Gateway/错误的网关)
    503 (Service Unavailable/服务无法获得)
    504 (Gateway Timeout/网关超时)
    505 (HTTP Version Not Supported/不支持的 HTTP 版本)
    
## 4. 日志路径
 想要查看日志，必须知道各组件日志的路径。
 
|组件	|组件log路径	|所在节点|
|:--|:--|:--|
|keystone	|/var/log/httpd,/var/log/keystone|	controller|
|nova	|/var/log/nova	|controller,compute-node-01|
|qemu	|/var/log/libvirt/qemu	|compute-node-01|
|glance	|/var/log/glance	|controller|
|cinder	|/var/log/cinder	|controller,block-node-01|
|neutron	|/var/log/neutron	|controller,compute-node-01 network-node-01|
|openVSwtich	|/var/log/openvswitch	|compute-node-01 network-node-01|
|swift	|/var/log/swift	|controller,block-node-01,block-node-02|
|horizon	|/var/log/httpd	|controller|
|Mariadb	|/var/log/mariadb	|controller|
|rabbitMQ	|/var/log/rabbitmq	|controller|


## 5. 一个故障实例-Nova instance boot
在本次搭建openstack平台的过程中，出现了不同错误，这里举一个可能大家都会遇到的一个问题，创建实例出错！

最初通过dashboard遇到的是这样的问题。

![2016-05-07_23-42-54](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-05-07_23-42-54.jpg)

这样的信息我们只知道是服务器内部错误，无法知道具体是哪里的问题。
所以我们通过命令行加入debug选项
	
	[root@controller ~]# nova --debug boot --flavor m1.nano --image cirros-0.3.4-x86_64 --nic net-id=40f3f1c5-4ce4-4d38-8539-23dd553f3b2f --security-group default --key-name demo-key inst1
    
屏幕打印的debug信息如下
    
    DEBUG (session:195) REQ: curl -g -i -X GET http://controller:5000/v3 -H "Accept: application/json" -H "User-Agent: python-keystoneclient"
    INFO (connectionpool:213) Starting new HTTP connection (1): controller
    DEBUG (connectionpool:393) "GET /v3 HTTP/1.1" 200 249
    DEBUG (session:224) RESP: [200] Content-Length: 249 Vary: X-Auth-Token Keep-Alive: timeout=5, max=100 Server: Apache/2.4.6 (CentOS) mod_wsgi/3.4 Python/2.7.5 Connection: Keep-Alive Date: Fri, 10 Jun 2016 13:49:29 GMT Content-Type: application/json x-openstack-request-id: req-9abd7d61-323e-4d76-a778-bcb8f3b381a1 
    RESP BODY: {"version": {"status": "stable", "updated": "2015-03-30T00:00:00Z", "media-types": [{"base": "application/json", "type": "application/vnd.openstack.identity-v3+json"}], "id": "v3.4", "links": [{"href": "http://controller:5000/v3/", "rel": "self"}]}}
    DEBUG (base:171) Making authentication request to http://controller:5000/v3/auth/tokens
    DEBUG (connectionpool:393) "POST /v3/auth/tokens HTTP/1.1" 201 3247
    DEBUG (session:195) REQ: curl -g -i -X GET http://controller:8774/v2/276eb659fd4b44859849c773dcfdf138/images -H "User-Agent: python-novaclient" -H "Accept: application/json" -H "X-Auth-Token: {SHA1}c327ae73d3652d44c33cd59907566d40909e631a"
    INFO (connectionpool:213) Starting new HTTP connection (1): controller
    DEBUG (connectionpool:393) "GET /v2/276eb659fd4b44859849c773dcfdf138/images HTTP/1.1" 200 508
    DEBUG (session:224) RESP: [200] Date: Fri, 10 Jun 2016 13:49:30 GMT Connection: keep-alive Content-Type: application/json Content-Length: 508 X-Compute-Request-Id: req-9b2129f1-f24e-453a-88f3-f05580894e07 
    RESP BODY: {"images": [{"id": "c9509d39-78b3-4509-94b5-8498e8e6acea", "links": [{"href": "http://controller:8774/v2/276eb659fd4b44859849c773dcfdf138/images/c9509d39-78b3-4509-94b5-8498e8e6acea", "rel": "self"}, {"href": "http://controller:8774/276eb659fd4b44859849c773dcfdf138/images/c9509d39-78b3-4509-94b5-8498e8e6acea", "rel": "bookmark"}, {"href": "http://controller:9292/images/c9509d39-78b3-4509-94b5-8498e8e6acea", "type": "application/vnd.openstack.image", "rel": "alternate"}], "name": "cirros-0.3.4-x86_64"}]}
    DEBUG (session:195) REQ: curl -g -i -X GET http://controller:8774/v2/276eb659fd4b44859849c773dcfdf138/images/c9509d39-78b3-4509-94b5-8498e8e6acea -H "User-Agent: python-novaclient" -H "Accept: application/json" -H "X-Auth-Token: {SHA1}c327ae73d3652d44c33cd59907566d40909e631a"
    DEBUG (connectionpool:393) "GET /v2/276eb659fd4b44859849c773dcfdf138/images/c9509d39-78b3-4509-94b5-8498e8e6acea HTTP/1.1" 500 128
    DEBUG (session:224) RESP:
    DEBUG (shell:914) The server has either erred or is incapable of performing the requested operation. (HTTP 500) (Request-ID: req-e359221b-6be3-4ac2-bb55-5310f5bb0bb2)
    Traceback (most recent call last):
      File "/usr/lib/python2.7/site-packages/novaclient/shell.py", line 911, in main
        OpenStackComputeShell().main(argv)
      File "/usr/lib/python2.7/site-packages/novaclient/shell.py", line 838, in main
        args.func(self.cs, args)
      File "/usr/lib/python2.7/site-packages/novaclient/v2/shell.py", line 495, in do_boot
        boot_args, boot_kwargs = _boot(cs, args)
      File "/usr/lib/python2.7/site-packages/novaclient/v2/shell.py", line 142, in _boot
        image = _find_image(cs, args.image)
      File "/usr/lib/python2.7/site-packages/novaclient/v2/shell.py", line 1894, in _find_image
        return utils.find_resource(cs.images, image)
      File "/usr/lib/python2.7/site-packages/novaclient/utils.py", line 216, in find_resource
        return manager.find(**kwargs)
      File "/usr/lib/python2.7/site-packages/novaclient/base.py", line 196, in find
        matches = self.findall(**kwargs)
      File "/usr/lib/python2.7/site-packages/novaclient/base.py", line 258, in findall
        found.append(self.get(obj.id))
      File "/usr/lib/python2.7/site-packages/novaclient/v2/images.py", line 53, in get
        return self._get("/images/%s" % base.getid(image), "image")
      File "/usr/lib/python2.7/site-packages/novaclient/base.py", line 156, in _get
        _resp, body = self.api.client.get(url)
      File "/usr/lib/python2.7/site-packages/keystoneclient/adapter.py", line 170, in get
        return self.request(url, 'GET', **kwargs)
      File "/usr/lib/python2.7/site-packages/novaclient/client.py", line 96, in request
        raise exceptions.from_response(resp, body, url, method)
    ClientException: The server has either erred or is incapable of performing the requested operation. (HTTP 500) (Request-ID: req-e359221b-6be3-4ac2-bb55-5310f5bb0bb2)
    ERROR (ClientException): The server has either erred or is incapable of performing the requested operation. (HTTP 500) (Request-ID: req-e359221b-6be3-4ac2-bb55-5310f5bb0bb2)
    
看上去是nova的问题，所以我们再去查看nova日志。下面是nova-api的日志。
    
    2016-06-10 21:49:30.191 7533 ERROR nova.api.openstack [req-e359221b-6be3-4ac2-bb55-5310f5bb0bb2 03384abb3c864c6bb55b09d33371beb4 276eb659fd4b44859849c773dcfdf138 - - -] Caught error: id
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack Traceback (most recent call last):
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/nova/api/openstack/__init__.py", line 125, in __call__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     return req.get_response(self.application)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/webob/request.py", line 1317, in send
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     application, catch_exc_info=False)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/webob/request.py", line 1281, in call_application
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     app_iter = application(self.environ, start_response)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/webob/dec.py", line 144, in __call__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     return resp(environ, start_response)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/keystonemiddleware/auth_token/__init__.py", line 634, in __call__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     return self._call_app(env, start_response)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/keystonemiddleware/auth_token/__init__.py", line 554, in _call_app
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     return self._app(env, _fake_start_response)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/webob/dec.py", line 144, in __call__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     return resp(environ, start_response)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/webob/dec.py", line 144, in __call__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     return resp(environ, start_response)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/routes/middleware.py", line 131, in __call__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     response = self.app(environ, start_response)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/webob/dec.py", line 144, in __call__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     return resp(environ, start_response)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/webob/dec.py", line 130, in __call__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     resp = self.call_func(req, *args, **self.kwargs)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/webob/dec.py", line 195, in call_func
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     return self.func(req, *args, **kwargs)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/nova/api/openstack/wsgi.py", line 756, in __call__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     content_type, body, accept)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/nova/api/openstack/wsgi.py", line 821, in _process_stack
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     action_result = self.dispatch(meth, request, action_args)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/nova/api/openstack/wsgi.py", line 911, in dispatch
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     return method(req=request, **action_args)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/nova/api/openstack/compute/images.py", line 83, in show
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     image = self._image_api.get(context, id)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/nova/image/api.py", line 93, in get
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     show_deleted=show_deleted)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/nova/image/glance.py", line 309, in show
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     include_locations=include_locations)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/nova/image/glance.py", line 483, in _translate_from_glance
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     include_locations=include_locations)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/nova/image/glance.py", line 545, in _extract_attributes
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     queued = getattr(image, 'status') == 'queued'
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/glanceclient/openstack/common/apiclient/base.py", line 491, in __getattr__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     self.get()
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/glanceclient/openstack/common/apiclient/base.py", line 509, in get
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     new = self.manager.get(self.id)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack   File "/usr/lib/python2.7/site-packages/glanceclient/openstack/common/apiclient/base.py", line 494, in __getattr__
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack     raise AttributeError(k)
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack AttributeError: id
    2016-06-10 21:49:30.191 7533 TRACE nova.api.openstack 
    2016-06-10 21:49:30.192 7533 INFO nova.api.openstack [req-e359221b-6be3-4ac2-bb55-5310f5bb0bb2 03384abb3c864c6bb55b09d33371beb4 276eb659fd4b44859849c773dcfdf138 - - -] http://controller:8774/v2/276eb659fd4b44859849c773dcfdf138/images/c9509d39-78b3-4509-94b5-8498e8e6acea returned with HTTP 500
    2016-06-10 21:49:30.193 7533 INFO nova.osapi_compute.wsgi.server [req-e359221b-6be3-4ac2-bb55-5310f5bb0bb2 03384abb3c864c6bb55b09d33371beb4 276eb659fd4b44859849c773dcfdf138 - - -] 192.168.199.80 "GET /v2/276eb659fd4b44859849c773dcfdf138/images/c9509d39-78b3-4509-94b5-8498e8e6acea HTTP/1.1" status: 500 len: 359 time: 0.0436440

从最后的trace信息来看是AttributeError，是glanceclient的问题。
接下来是各种google，我们也可以在openstack官网查询，发现这么一个地址，https://bugs.launchpad.net/glance/+bug/1476770，原来是glanceclient的一个bug。
你也可以再查看其他相关网址
	
	https://bugs.launchpad.net/python-glanceclient/+bug/1479296	https://bugs.launchpad.net/python-glanceclient/+bug/1323660
按照网站上面给出的建议，似乎要升级glanceclient至0.17.3，好吧，从下面的网址git clone源码， 再安装替换。https://github.com/openstack/python-glanceclient/tree/stable/kilo 
重启启动服务
	
	systemctl restart openstack-glance-api.service openstack-glance-registry.service
最后我们看到实例最终创建成功。
![2016-06-12_14-15-48](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-12_14-15-48.jpg)

当然有的可能不一定是这个原因引起的，如果升级了还是没有解决这个问题，我们还要再尝试其他办法。
像https://review.openstack.org/#/c/244899/这个帖子里说要替换http.py和images.py两个文件。如果还是不行，可能要考虑重新安装组件了。总之，时间允许的情况下，多尝试几次总是没错了，通过几次折腾，也会进一步了解openstack组件间的关系了。
