---
title: 使用Keepalived实现将lvs进行高可用
tags:
  - keepalived
  - lvs
id: 312
categories:
  - 运维
date: 2015-12-06 20:31:46
---

LVS是Linux Virtual Server的简写，意即Linux虚拟服务器，是一个虚拟的服务器集群系统。目前有三种IP负载均衡技术（VS/NAT、VS/TUN和VS/DR）；十种调度算法（rrr|wrr|lc|wlc|lblc|lblcr|dh|sh|sed|nq）。
Keepalived在这里主要用作RealServer的健康状态检查以及LoadBalance主机和BackUP主机之间failover的实现。
本文我们将实现下图的实验。
[![keepalive](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/keepalive.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/keepalive.png)
环境centos 6.6
<table>
<tbody>
<tr>
<td width="73">服务器功能</td>
<td width="92">IP</td>
<td width="92">VIP</td>
</tr>
<tr>
<td width="73">Lvs MASTER</td>
<td width="92">192.168.199.33</td>
<td width="92">192.168.199.88</td>
</tr>
<tr>
<td width="73">Lvs BACKUP</td>
<td width="92">192.168.199.34</td>
<td width="92">192.168.199.88</td>
</tr>
<tr>
<td width="73">Web serverA</td>
<td width="92">192.168.199.31</td>
<td width="92">192.168.199.88</td>
</tr>
<tr>
<td width="73">Web serverB</td>
<td width="102">192.168.199.32</td>
<td width="92">192.168.199.88</td>
</tr>
</tbody>
</table>
准备工作：
1)每台服务器关闭SELinux和防火墙
#setenforce 0
#service iptables stop
2)每台服务器同步时间
tzselect
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ntpdate -u ntp.api.bz
1.配置两台WEB服务器
1.1开启httpd服务
#yum install httpd
1.2设置访问主页
#cd /var/www/html
//192.168.199.31
#vi index.html
This is web server 192.168.199.31
//192.168.199.32
This is web server 192.168.199.32

1.3编写WEB服务器启动脚本
#vi realserver.sh
***************************************************************************
#!/bin/bash
VIP=192.168.199.88
/etc/rc.d/init.d/functions
case "$1" in
start)
echo " start LVS of REALServer"
/sbin/ifconfig lo:0 $VIP broadcast $VIP netmask 255.255.255.255 up
/sbin/route add -host $VIP dev lo:0
echo "1" &gt;/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "2" &gt;/proc/sys/net/ipv4/conf/lo/arp_announce
echo "1" &gt;/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" &gt;/proc/sys/net/ipv4/conf/all/arp_announce
sysctl -p &gt;/dev/null 2&gt;&amp;1
;;
stop)
/sbin/ifconfig lo:0 down
echo "close LVS Directorserver"
echo "0" &gt;/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "0" &gt;/proc/sys/net/ipv4/conf/lo/arp_announce
echo "0" &gt;/proc/sys/net/ipv4/conf/all/arp_ignore
echo "0" &gt;/proc/sys/net/ipv4/conf/all/arp_announce
;;
*)
echo "Usage: $0 {start|stop}"
exit 1
esac
******************************************************************************
//设置可执行
#chmod +x realserver.sh
//设置可读写
#chmod 755 /etc/rc.d/init.d/functions

在每台WEB服务器上执行
#./realserver.sh start
1.4测试WEB服务器独立可访问
[![2015-10-29_21-56-01](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_21-56-01.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_21-56-01.jpg)
[![2015-10-29_21-56-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_21-56-13.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_21-56-13.jpg)
此时还不能访问192.168.199.88
[![2015-10-29_21-58-35](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_21-58-35.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_21-58-35.png)
2 安装配置Lvs Master 服务器
#yum install keepalived
[![2015-10-29_22-04-42](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-04-42.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-04-42.png)
#cd /etc/keepalived/
#cp keepalived.conf keepalived.conf.bak
#vim keepalived.conf
***********************************************************************
! Configuration File for keepalived
global_defs {
notification_email {
acassen@firewall.loc
failover@firewall.loc
sysadmin@firewall.loc
}
notification_email_from Alexandre.Cassen@firewall.loc
smtp_server 192.168.200.1
smtp_connect_timeout 30
router_id LVS_DEVEL
}

vrrp_instance VI_1 {
state MASTER
interface eth0
virtual_router_id 51
priority 100
advert_int 1
authentication {
auth_type PASS
auth_pass 1111
}
track_interface {
eth0
}
virtual_ipaddress {
192.168.199.88/24 dev eth0 label eth0:0
} }

virtual_server 192.168.199.88 80 {
delay_loop 6
lb_algo wrr
lb_kind DR
nat_mask 255.255.255.0
persistence_timeout 30
protocol TCP

real_server 192.168.199.31 80 {
weight 3
TCP_CHECK {
connect_timeout 10
nb_get_retry 3
delay_before_retry 3
connect_port 80
}
}
real_server 192.168.199.32 80 {
weight 3
TCP_CHECK {
connect_timeout 10
nb_get_retry 3
delay_before_retry 3
connect_port 80
}
}
}
*******************************************************************
#service keepalived start
[![2015-10-29_22-47-42](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-47-42.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-47-42.png)
3\. 安装配置Lvs BACKUP服务器
#yum install keepalived
#cd /etc/keepalived/
#cp keepalived.conf keepalived.conf.bak
#vim keepalived.conf
*******************************************************************
! Configuration File for keepalived
global_defs {
notification_email {
acassen@firewall.loc
failover@firewall.loc
sysadmin@firewall.loc
}
notification_email_from Alexandre.Cassen@firewall.loc
smtp_server 192.168.200.1
smtp_connect_timeout 30
router_id LVS_DEVEL
}

vrrp_instance VI_1 {
state BACKUP
interface eth0
virtual_router_id 51
priority 99
advert_int 1
authentication {
auth_type PASS
auth_pass 1111
}
track_interface {
eth0
}
virtual_ipaddress {
192.168.199.88/24 dev eth0 label eth0:0
} }

virtual_server 192.168.199.88 80 {
delay_loop 6
lb_algo wrr
lb_kind DR
nat_mask 255.255.255.0
persistence_timeout 30
protocol TCP

real_server 192.168.199.31 80 {
weight 3
TCP_CHECK {
connect_timeout 10
nb_get_retry 3
delay_before_retry 3
connect_port 80
}
}
real_server 192.168.199.32 80 {
weight 3
TCP_CHECK {
connect_timeout 10
nb_get_retry 3
delay_before_retry 3
connect_port 80
}
}
}
*****************************************************************************
#service keepalived start
[![2015-10-29_22-48-14](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-48-14.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-48-14.png)
访问192.168.199.88, 访问正常，一段时间会自动切换web服务器
[![2015-10-29_22-26-12](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-26-12.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-26-12.png)
[![2015-10-29_22-44-50](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-44-50.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-44-50.png)
4.实验1:将WEB服务器31停止，看访问是否还是正常。
#service httpd stop
[![2015-10-29_22-51-37](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-51-37.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-51-37.png)
我们看到还能正常访问，只是切换到WEB 服务器B
[![2015-10-29_22-53-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-53-36.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-53-36.png)
5.实验2:将负载均衡服务器master停止，看访问是否还是正常。
#service keepalived stop
[![2015-10-29_22-56-51](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-56-51.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-56-51.png)
我们可以从master服务器的LOG可以看到,VIP192.168.199.88已释放。
[![2015-10-29_23-06-08](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_23-06-08.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_23-06-08.png)
我们可以从BACKUP服务器的LOG可以看到，BACKUP服务器已接管VIP192.168.199.88
[![2015-10-29_23-03-41](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_23-03-41.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_23-03-41.png)
访问192.168.199.88还是正常
[![2015-10-29_22-58-45](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-58-45.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/12/2015-10-29_22-58-45.png)
至此使用Keepalived实现将lvs进行高可用成功部署。