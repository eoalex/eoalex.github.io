---
title: COGNOS BI 10 集群部署示列
tags:
  - Cognos
id: 173
categories:
  - 商业智能
date: 2015-11-15 14:09:40
---

# 1.Cognos简介

Cognos是在BI核心平台之上，以服务为导向进行架构的一种数据模型，是唯一可以通过单一产品和在单一可靠架构上提供完整业务智能功能的解决方案。它可以提供无缝密合的报表、分析、记分卡、仪表盘等解决方案，通过提供所有的系统和资料资源，以简化公司各员工处理资讯的方法。作为一个全面、灵活的产品，Cognos业务智能解决方案可以容易地整合到现有的多系统和数据源架构中。

# 2.本文目标

2.1 将cognos部署到多台电脑上（这里用三台虚拟机）
2.2 设置两个以上的cognos使用者权限，一个为管理员（全部权限），另一个为报表开发者（只能进行报表开发）
2.3 使用自己的数据生成数据包，并且制作3个以上的报表。

# 3.准备

准备3台机器（或虚拟机）
环境：
Windows 2008 R2 64Bit；
COGNOS 10.2;
DB2 10.1 64Bit;
Apache HTTP SERVER 2.2;
OpenLDAP 2.2.29;
安装配置如下图：
<table>
<tbody>
<tr>
<td width="73">Name</td>
<td width="102">IP</td>
<td width="92">Memory/CPU</td>
<td width="181">Main function</td>
<td width="143">Other</td>
</tr>
<tr>
<td width="73">Cognos11</td>
<td width="102">192.168.199.7</td>
<td width="92">2G/1CPU</td>
<td width="181">Content Manager Server</td>
<td width="143">DB2 Server，OpenLDAP</td>
</tr>
<tr>
<td width="73">Cognos12</td>
<td width="102">192.168.199.8</td>
<td width="92">4G/2CPU</td>
<td width="181">BI Server</td>
<td width="143">DB2 Client，Framework Manager</td>
</tr>
<tr>
<td width="73">Cognos13</td>
<td width="102">192.168.199.9</td>
<td width="92">2G/1CPU</td>
<td width="181">Gateway Server</td>
<td width="143">Apache Server</td>
</tr>
</tbody>
</table>
安装顺序DB2 SERVER-&gt;Content Manager-&gt;BI server -&gt; Gateway server-&gt;Apache Server-&gt;OpenLDAP-&gt;DB2 Client -&gt; Framework Manager.
注：Framework Manager可安装在任一电脑，前提是安装DB2 client，因为这里DB2 Client本来需要安装在BI SERVER.所以我这里偷懒把Framework Manager也安装在BI server上了。
[![2015-09-13_19-57-53](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_19-57-53.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_19-57-53.png)

# 4.安装部署

## 4.1安装DB2

在COGOS11,192.168.199.7电脑上，启动db2 setup程序。
[![2015-09-13_16-03-21](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_16-03-21.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_16-03-21.png)
[![2015-09-13_16-06-51](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_16-06-51.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_16-06-51.png)

## 4.2安装Content Manager

1)在COGOS11,192.168.199.7电脑上启动cognos setup程序,选择Content Manger,选择安装路径C:\C10\CM
[![2015-09-13_16-41-23](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_16-41-23.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_16-41-23.png)
2)复制C:\Program Files\IBM\SQLLIB\java下两个文件db2jcc.jar db2jcc_license_cu.jar 到 content manager安装目录的\webapps\p2pd\web-inf\lib
3)启动配置，修改以下localhost为本机IP 192.168.199.7，端口号不变。
[![2015-09-13_17-01-11](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_17-01-11.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_17-01-11.png)
4)配置content Store
[![2015-09-15_16-37-59](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_16-37-59.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_16-37-59.png)
输入以上信息，右键点击生成DDL文件
[![2015-09-15_16-28-16](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_16-28-16.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_16-28-16.png)
[![2015-09-15_0-24-21](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_0-24-21.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_0-24-21.png)
在DB2命令行运行DB2 -tvf C:/C10/CM/configuration/schemas/content/db2/createDb.sql
[![2015-09-13_17-51-33](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_17-51-33.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_17-51-33.png)
5)保存，启动服务,保持启动状态。
[![2015-09-13_17-55-17](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_17-55-17.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_17-55-17.png)
在WEB浏览器输入http://192.168.199.7:9300/p2pd/servlet 显示成功运行。
[![2015-09-13_18-33-32](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_18-33-32.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_18-33-32.png)

## 4.3安装BI SERVER

1)在COGOS12,192.168.199.8电脑上启动cognos setup程序,选择应用程序组件, 选择安装路径C:\C10\BI
[![2015-09-13_15-18-39](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_15-18-39.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_15-18-39.png)
2)同样复制C:\Program Files\IBM\SQLLIB\java下两个文件
db2jcc.jar db2jcc_license_cu.jar 到 content manager安装目录的\webapps\p2pd\web-inf\lib
3)启动配置,Content Manager URI改为Content Manger 服务器IP 192.168.199.7,分派器URI改为本机IP地址192.168.199.8
[![2015-09-13_18-05-53](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_18-05-53.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_18-05-53.png)
通知存储库设置与content store 一样，注意使用IP地址。
[![2015-09-13_18-12-11](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_18-12-11.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_18-12-11.png)
4)保存启动（注意保持content manager已启动)
[![2015-09-14_13-35-46](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_13-35-46.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_13-35-46.png)
在WEB浏览器输入http://192.168.199.8:9300/p2pd/servlet/dispatch 显示以下信息，表示成功。
[![2015-09-15_18-51-57](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_18-51-57.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_18-51-57.png)

## 4.4安装Gateway

1)在COGNOS13,192.168.199.9电脑上启动cognos setup程序,选择Gateway, 选择安装路径C:\C10\GW
[![2015-09-13_15-45-00](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_15-45-00.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_15-45-00.png)
2)启动配置,把分派器URI 改成IP192.168.199.8,网关URI 改成本机IP：192.168.199.9
[![2015-09-13_18-30-30](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_18-30-30.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-13_18-30-30.png)
3)安装Apache http server,在安装路径的httpd.conf文件最后添加如下代码：

    ScriptAlias /ibmcognos/cgi-bin/ "C:/C10/GW/cgi-bin/" 
    &lt;Directory "C:/C10/GW/cgi-bin/"&gt;
    AllowOverride None
    AcceptPathInfo On
    Options None
    Order allow,deny
    Allow from all

    Alias /ibmcognos/ "C:/C10/GW/webcontent/"
    &lt;Directory "C:/C10/GW/webcontent/"&gt; 
    Options Indexes MultiViews
    AllowOverride None
    AcceptPathInfo On
    Order allow,deny 
    Allow from all 

在WEB浏览器输入http://192.168.199.9/ibmcognos/,出现以下界面表示成功。
[![2015-09-14_14-03-07](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_14-03-07.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_14-03-07.png)

## 4.5安装openLDAP

1)在COGOS11,192.168.199.7电脑上，启动openLDAP 安装程序.安装配置过程略，参见文档（为OpenLDAP 配置LDAP命名空间 v2）
2)配置结束，重启BI SERVER
在WEB浏览器输入http://192.168.199.9/ibmcognos/,右上角出现需要登录。表示成功。
[![2015-09-14_19-20-57](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_19-20-57.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_19-20-57.png)

## 4.6安装DB2 CLIENT

在COGNOS12,192.168.199.8电脑上，安装 db2 client.过程略。

## 4.7安装Framework Manager

1)在COGNOS12,192.168.199.8电脑上，安装 Framework Manager,路径 C:\C10\FM
2)把分派器URI 改成IP192.168.199.8,网关URI 改成IP：192.168.199.9
[![2015-09-14_20-35-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_20-35-36.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_20-35-36.png)
至此，COGNOS三台机器安装部署基本结束。

# 5.权限配置

1)设置openLDAP 的admin为系统管理员,用admin登录，在管理-&gt;安全-&gt;用户,组,角色找到角色系统管理员.
[![2015-09-15_20-13-38](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-13-38.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-13-38.png)
添加admin
[![2015-09-14_19-36-47](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_19-36-47.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_19-36-47.png)
禁止匿名角色
[![diable item](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/diable-item.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/diable-item.png)
2)添加角色报表开发者,加入openLDAP用户cog _user1.
[![report dev](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/report-dev.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/report-dev.png)
在功能区域禁止角色报表开发者其他权限。用cog_user1 登录看到只有报表开发权限。
[![2015-09-14_20-12-13](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_20-12-13.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_20-12-13.png)
3)修改Content Manger安全认证，把匿名登录项改为false.
[![2015-09-14_19-47-12](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_19-47-12.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_19-47-12.png)
访问http://192/168.199.9/ibmcognos/将显示需要验证登录
[![2015-09-15_20-07-22](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-07-22.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-07-22.png)

# 6.导入数据

1)因为手上也没有自己的数据，所以还是安装SAMPLE里的GS_DB数据库至DB2 SERVER，并起名为GOSALES,这个过程略。
2)配置DB2 CLIENT
使用 catalog 将DB2 SERVER上的数据库映射到cognos BI服务器.
db2 catalog tcpip node db2db remote 192.168.199.7 server 50000
db2 catalog db GOSales at node db2db
db2 terminate
[![2015-09-14_22-58-54](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_22-58-54.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-14_22-58-54.png)
3)制作数据包,并发布。
a.启动Framework Manager,新建项目fmPackage3
b.设置数据源.
[![2015-09-15_20-49-01](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-49-01.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-49-01.jpg)
[![2015-09-15_20-47-57](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-47-57.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-47-57.jpg)
[![2015-09-15_20-47-36](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-47-36.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-47-36.jpg)
c.选择元数据
[![2015-09-15_11-33-17](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_11-33-17.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_11-33-17.png)
d.制作数据包，为简单起见，只选择销售目标为唯一度量维度，相应选择产品品牌，产品类型为常规维度，具体过程略。
[![2015-09-15_14-26-41](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-26-41.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-26-41.png)
业务视图
[![2015-09-15_21-11-22](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_21-11-22.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_21-11-22.jpg)
维度视图
[![2015-09-15_21-12-30](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_21-12-30.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_21-12-30.jpg)
5)发布数据包
新建数据包，首先验证，如果没有问题，就可以发布。在公共文件夹下看到了刚发布的包.
[![2015-09-15_20-54-53](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-54-53.jpg)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_20-54-53.jpg)

# 7.制作报表

1)制作Query Studio报表
用cog_user2登录，选择新建Query Studio，我们将制作一个按品牌显示的销售目标报表。按销售目标降序排序。
[![2015-09-15_14-38-46](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-38-46.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-38-46.png)
2)制作Analysis studio报表，如下图:
[![2015-09-15_14-46-21](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-46-21.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-46-21.png)
3)制作Report studio报表，带图表显示。
[![2015-09-15_14-57-27](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-57-27.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-57-27.png)
[![2015-09-15_14-57-05](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-57-05.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2015/11/2015-09-15_14-57-05.png)