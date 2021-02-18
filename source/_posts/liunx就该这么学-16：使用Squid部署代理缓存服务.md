---
title: liunx就该这么学-16：使用Squid部署代理缓存服务
date: 2021-02-17 13:28:38
tags:
- 第16章节：使用Squid部署代理缓存服务
categories:
- linux就该这么学

---

# 1.代理缓存服务

<!--more-->

Squid是Linux系统中最为流行的一款高性能代理服务软件，通常用作Web网站的前置缓存服务，能够代替用户向网站服务器请求页面数据并进行缓存。简单来说，Squid服务程序会按照收到的用户请求向网站源服务器请求页面、图片等所需的数据，并将服务器返回的数据存储在运行Squid服务程序的服务器上。当有用户再请求相同的数据时，则可以直接将存储服务器本地的数据交付给用户，这样不仅减少了用户的等待时间，还缓解了网站服务器的负载压力。

在使用Squid服务程序为用户提供缓存代理服务时，具有正向代理模式和反向代理模式之分。

所谓正向代理模式，是指让用户通过Squid服务程序获取网站页面等资源，以及基于访问控制列表（ACL）功能对用户访问网站行为进行限制，在具体的服务方式上又分为标准代理模式与透明代理模式。标准正向代理模式是把网站数据缓存到服务器本地，提高数据资源被再次访问时的效率，但是用户在上网时必须在浏览器等软件中填写代理服务器的IP地址与端口号信息，否则默认不使用代理服务。而透明正向代理模式的作用与标准正向代理模式基本相同，区别是用户不需要手动指定代理服务器的IP地址与端口号，所以这种代理服务对于用户来讲是相对透明的。

Squid服务程序提供正向代理服务：

{% asset_img 1.png %}

反向代理模式是指让多台节点主机反向缓存网站数据，从而加快用户访问速度。因为一般来讲，网站中会普遍加载大量的文字、图片等静态资源，而且它们相对来说都是比较稳定的数据信息，当用户发起网站页面中这些静态资源的访问请求时，我们可以使用Squid服务程序提供的反向代理模式来进行响应。而且，如果反向代理服务器中恰巧已经有了用户要访问的静态资源，则直接将缓存的这些静态资源发送给用户，这不仅可以加快用户的网站访问速度，还在一定程度上降低了网站服务器的负载压力。

Squid服务程序提供的反向代理模式:

{% asset_img 2.png %}

# 2.配置Squid服务程序

Squid服务程序的配置步骤虽然十分简单，但依然需要为大家交代一下实验所需的设备以及相应的设置。首先需要准备两台虚拟机，一台用作Squid服务器，另外一台用作Squid客户端；为了能够相互通信，需要将这两台虚拟机**都设置为仅主机模式（Hostonly）**，然后关闭其中一台虚拟机的电源，在添加一块新的网卡为**桥接模式**后开启电源：

{% asset_img 3.png %}

需要注意的是，这块新添加的网卡设备必须选择为桥接模式，否则这两台虚拟机都无法访问外网。

设置好桥接模式后，虚拟机还不能连接外网，还需要到虚拟机的编辑-虚拟网络编辑器，对VMnet0的桥接网卡进行指定：

{% asset_img 4.png %}

{% asset_img 5.png %}

之后回到linux系统，在终端中输入命令：nm-connection-editor，新建网卡：

	[root@linuxprobe ~]#nm-connection-editor

{% asset_img 6.png %}

{% asset_img 7.png %}

{% asset_img 8.png %}

{% asset_img 9.png %}

设置完成后，保存退出，之后重新启动网卡；也可以ping下网站，看是否能够成功：

	[root@linuxprobe ~]#systemctl restart network
	[root@linuxprobe ~]#ping -c 4 www.baidu.com

当前Squid服务器和客户端的操作系统和IP地址信息：


| 主机名称 | 操作系统 | IP地址 |
| --- | --- | --- |
| Squid服务器 | RHEL 7 | 外网卡：桥接DHCP模式 |
|  |  | 内网卡：192.168.10.10 |
| Squid客户端 | Windows 7/RHEL 7 | 192.168.10.20 |

这样一来，我们就有了一台既能访问内网，又能访问外网的Squid服务器了；但当前的Squid服务器只能和外网或虚拟机间进行通讯，无法和宿主机进行连接；如果Squid服务器需要和宿主机间通过主机模式（host only）进行通讯的话，还需要在宿主机的VMnet1网络适配器进行以下设置：

{% asset_img 10.png %}

{% asset_img 11.png %}

至此，Squid服务器既能访问外网和内网，又能和宿主机进行通讯。

但有一个问题，Squid服务器目前有两个网卡，一个网卡连接外网，一个网卡连接内网；但/etc/sysconfig/network-scripts目录内只有内网网卡的配置文件，并没有生成外网网卡的配置文件；自己手动新建配置文件的话，重启网卡后反而无法连接外网，会报错，不太清楚原因所在，难道使用DHCP的外网网卡不需要配置文件？

当配置好Yum软件仓库并挂载好设备镜像后，就可以安装Squid服务程序了:

	[root@linuxprobe ~]# yum install squid

Squid服务程序的配置文件也是存放在/etc/squid/squid.conf;常用的Squid服务程序配置参数以及作用:


| 参数 | 作用 |
| --- | --- |
| http_port 3128 | 监听的端口号 |
| cache_mem 64M | 内存缓冲区的大小 |
| cache_dir ufs /var/spool/squid 2000 16 256 | 硬盘缓冲区的大小 |
| cache_effective_user squid | 设置缓存的有效用户 |
| cache_effective_group squid | 设置缓存的有效用户组 |
| dns_nameservers | IP地址	一般不设置，而是用服务器默认的DNS地址 |
| cache_access_log /var/log/squid/access.log | 访问日志文件的保存路径 |
| cache_log /var/log/squid/cache.log | 缓存日志文件的保存路径 |
| visible_hostname linuxprobe.com | 设置Squid服务器的名称 |

# 3.正向代理

## 3.1标准正向代理

	[root@linuxprobe ~]# systemctl restart squid
	[root@linuxprobe ~]# systemctl enable squid
	ln -s '/usr/lib/systemd/system/squid.service' '/etc/systemd/system/multi-user.target.wants/squid.service'

接下来在Squid客户端进行操作，首先是windows7系统：

{% asset_img 12.png %}

{% asset_img 13.png %}

{% asset_img 14.png %}

其次是RHEL 7系统,打开火狐浏览器:

{% asset_img 15.png %}

{% asset_img 16.png %}

{% asset_img 17.png %}

需要注意的是，虽然在浏览器中已经设置好代理服务器，但目前在客户端系统上还不能连接外网；此时需要回到Squid服务器中，输入以下命令，清空防火墙：

	[root@linuxprobe ~]# iptables -F
	[root@linuxprobe ~]# service iptables save

至此，Squid客户端系统可以连接外网！

Squid服务程序默认使用3128、3401与4827等端口号，因此可以把默认使用的端口号修改为其他值，以便起到一定的保护作用：

	[root@linuxprobe ~]# vim /etc/squid/squid.conf
	………………省略部分输出信息………………
	http_port 10000
	………………省略部分输出信息………………
	[root@linuxprobe ~]# systemctl restart squid 
	[root@linuxprobe ~]# systemctl enable squid 
	 ln -s '/usr/lib/systemd/system/squid.service' '/etc/systemd/system/multi-user.target.wants/squid.service'

尽管现在重启Squid服务程序后系统没有报错，但是用户还不能使用代理服务。SElinux安全子系统认为Squid服务程序使用3128端口号是理所当然的，因此在默认策略规则中也是允许的，但是现在Squid服务程序却尝试使用新的10000端口号，而该端口原本并不属于Squid服务程序应该使用的系统资源，因此还需要手动把新的端口号添加到Squid服务程序在SElinux域的允许列表中。

	[root@linuxprobe ~]# semanage port -l | grep squid_port_t
	squid_port_t                   tcp      3128, 3401, 4827
	squid_port_t                   udp      3401, 4827
	[root@linuxprobe ~]# semanage port -a -t squid_port_t -p tcp 10000
	[root@linuxprobe ~]# semanage port -l | grep squid_port_t
	squid_port_t                   tcp      10000, 3128, 3401, 4827
	squid_port_t                   udp      3401, 4827




