---
title: BigInSights安装KERBEROS
tags:
  - BigInSights
  - KERBEROS
id: 120
categories:
  - 大数据
date: 2015-11-10 22:04:09
---

本文安装KERBEROS主要基于IBM BigInSights 环境，虚拟机环境可从以下网址下载。

http://www.ibm.com/analytics/us/en/technology/hadoop/

1.安装krb5

#yum -y install krb5-server krb5-libs krb5-auth-dialog krb5-workstation

[![2015-10-27_21-35-23](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_21-35-23-300x152.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_21-35-23.png)

2.配置

#vi /var/lib/ambari-server/resources/scripts/krb5.conf

[![2015-10-27_21-43-38](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_21-43-38-293x300.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_21-43-38.png)

3.创建数据库

#kdb5_util create –s

[![2015-10-27_21-45-14](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_21-45-14-300x66.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_21-45-14.png)

4.启动

#/etc/rc.d/init.d/krb5kdc start

#/etc/rc.d/init.d/kadmin start

[![2015-10-27_21-46-11](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_21-46-11-300x44.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_21-46-11.png)

5.创建admin用户的principal

#kadmin.local -q "addprinc admin/admin "

[![2015-10-27_22-47-23](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-47-23-300x54.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-47-23.png)

6.确认步骤5成功

#cat /var/kerberos/krb5kdc/kadm5.acl

7.重启

#/etc/rc.d/init.d/krb5kdc restart

#/etc/rc.d/init.d/kadmin restart

[![2015-10-27_22-00-37](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-00-37-300x60.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-00-37.png)

8.ambari管理界面配置

[![2015-10-27_22-00-05](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-00-05-300x56.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-00-05.png)

[![2015-10-27_22-11-42](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-11-42-300x161.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-11-42.png)

[![2015-10-27_22-13-43](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-13-43-300x197.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-13-43.png)

[![2015-10-27_22-50-10](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-50-10-300x148.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-50-10.png)

[![2015-10-27_22-50-43](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-50-43-300x169.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-50-43.png)

[![2015-10-27_22-51-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-51-13-300x206.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-51-13.png)

[![2015-10-27_22-54-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-54-13-300x114.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-54-13.png)

[![2015-10-27_22-57-43](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-57-43-300x130.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-10-27_22-57-43.png)