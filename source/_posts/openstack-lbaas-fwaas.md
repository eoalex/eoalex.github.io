---
title: 'OpenStack入门之五:负载均衡和防火墙设置'
tags:
  - openstack
id: 1155
categories:
  - 云计算及虚拟化
date: 2016-06-21 08:03:49
---

本文我们在[OpenStack入门之三:Kilo版本centos 7平台搭建](http://blog.yaodataking.com/2016/06/openstack-kilo-centos.html)基础上设置负载均衡和防火墙。

## 1.负载均衡设置

1.1 网络节点安装配置
1.1.1安装
#yum install openstack-neutron-lbaas haproxy
1.1.2配置
#vi /etc/neutron/lbaas_agent.ini

    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    device_driver = neutron.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver
    `</pre>
    #vi /etc/neutron/neutron.conf
    <pre></code>
    service_plugins = router,lbaas
    </code></pre>
    1.1.3启用，重启
    #systemctl enable neutron-lbaas-agent.service
    #systemctl start neutron-lbaas-agent.service
    #systemctl restart neutron-openvswitch-agent.service
    1.2 控制节点安装配置
    1.2.1 安装
    #yum install openstack-neutron-lbaas -y
    1.2.2 配置
    #vi /etc/neutron/neutron.conf
    <pre>`
    service_plugins = router,lbaas
    [service_providers]
    service_provider = LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.driver.HaproxyDriver:default
    `</pre>
    1.2.3 重启服务
    #systemctl restart neutron-server.service
    1.3 Dashboard配置及验证
    1.3.1 添加资源池
    [![2016-06-11_9-11-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_9-11-13.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_9-11-13.jpg)
    1.3.2 添加成员
    [![2016-06-11_9-15-48](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_9-15-48.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_9-15-48.jpg)
    1.3.3 设置vip ip
    [![2016-06-11_9-13-03](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_9-13-03.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_9-13-03.jpg)
    [![2016-06-11_9-14-17](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_9-14-17.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_9-14-17.jpg)
    1.3.4 验证
    SSH连接VIP 192.168.2.5我们看到实际连接的是192.168.2.3
    [![2016-06-11_20-08-47](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-08-47.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-08-47.jpg)
    我们停止实例192.168.2.3.再去连接192.168.2.5,看到连接到了192.168.2.4去了，这证明负载均衡是起作用了。
    [![2016-06-11_20-13-58](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-13-58.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-13-58.jpg)
    [![2016-06-11_20-13-44](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-13-44.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-13-44.jpg)

    ## 2.防火墙设置

    2.1控制节点安装配置
    2.1.1安装
    #yum install openstack-neutron-fwaas -y
    2.1.2配置
    #vi /etc/neutron/neutron.conf
    <pre>`
    service_plugins = router,lbaas,firewall
    [service_providers]
    service_provider = FIREWALL:Iptables:neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver:default
    `</pre>
    2.1.3重启服务
    #systemctl restart neutron-server.service
    2.1.4更新数据库
    #neutron-db-manage --service fwaas upgrade head
    <pre>`
    INFO  [alembic.runtime.migration] Context impl MySQLImpl.
    INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
    INFO  [alembic.runtime.migration] Context impl MySQLImpl.
    INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
    INFO  [alembic.runtime.migration] Running upgrade  -&gt; start_neutron_fwaas, start neutron-fwaas chain
    INFO  [alembic.runtime.migration] Running upgrade start_neutron_fwaas -&gt; 4202e3047e47, add_index_tenant_id
    INFO  [alembic.runtime.migration] Running upgrade 4202e3047e47 -&gt; 540142f314f4, FWaaS router insertion
    INFO  [alembic.runtime.migration] Running upgrade 540142f314f4 -&gt; 796c68dffbb, cisco_csr_fwaas
    INFO  [alembic.runtime.migration] Running upgrade 796c68dffbb -&gt; kilo, kilo
    `</pre>
    2.2 网络节点安装配置
    2.2.1安装
    #yum install openstack-neutron-fwaas -y
    2.2.2配置
    #vi /etc/neutron/fwaas_driver.ini
    <pre>`
    driver = neutron_fwaas.services.firewall.drivers.linux.iptables_fwaas.IptablesFwaasDriver
    enabled = True
    `</pre>
    #vi /etc/neutron/neutron.conf
    <pre>`
    service_plugins = router,lbaas,firewall

2.2.3重启
#systemctl restart neutron-openvswitch-agent.service neutron-l3-agent.service
2.3 Dashboard配置及验证
2.3.1 设置规则
[![2016-06-11_20-32-06](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-32-06.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-32-06.jpg)
2.3.2 设置策略
[![2016-06-11_20-32-44](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-32-44.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-32-44.jpg)
2.3.3 设置防火墙并关联路由
[![2016-06-11_20-33-25](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-33-25.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/06/2016-06-11_20-33-25.jpg)