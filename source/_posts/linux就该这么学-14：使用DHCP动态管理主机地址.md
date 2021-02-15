---
title: linux就该这么学-14：使用DHCP动态管理主机地址
date: 2021-02-15 14:08:50
tags:
- 第14章节：使用DHCP动态管理主机地址
categories:
- linux就该这么学

---

# 1.动态主机地址管理协议

DHCP：Dynamic Host Configuration Protocol，动态主机配置协议；该协议用于自动管理局域网内主机的IP地址、子网掩码、网关地址及DNS地址等参数，可以有效地提升IP地址的利用率，提高配置效率，并降低管理与维护成本。

<!--more-->

	作用域：一个完整的IP地址段，DHCP协议根据作用域来管理网络的分布、分配IP地址及其他配置参数。
	
	超级作用域：用于管理处于同一个物理网络中的多个逻辑子网段。超级作用域中包含了可以统一管理的作用域列表。
	
	排除范围：把作用域中的某些IP地址排除，确保这些IP地址不会分配给DHCP客户端。
	
	地址池：在定义了DHCP的作用域并应用了排除范围后，剩余的用来动态分配给DHCP客户端的IP地址范围。
	
	租约：DHCP客户端能够使用动态分配的IP地址的时间。
	
	预约：保证网络中的特定设备总是获取到相同的IP地址。

# 2.部署dhcpd服务程序

	[root@linuxprobe ~]# yum install dhcp

查看dhcpd服务程序的配置文件内容：

	[root@linuxprobe ~]# cat /etc/dhcp/dhcpd.conf

dhcp的服务程序的配置文件中只有3行注释语句，这意味着我们需要自行编写这个文件。如果读者不知道怎么编写，可以看一下配置文件中第2行的参考示例文件，其组成架构如图：

{% asset_img 1.png %}

一个标准的配置文件应该包括全局配置参数、子网网段声明、地址配置选项以及地址配置参数。其中，全局配置参数用于定义dhcpd服务程序的整体运行参数；子网网段声明用于配置整个子网段的地址属性。

dhcpd服务程序配置文件中使用的常见参数以及作用:


| 参数 | 作用 |
| --- | --- |
| ddns-update-style 类型 | 定义DNS服务动态更新的类型，类型包括：none（不支持动态更新）、interim（互动更新模式）与ad-hoc（特殊更新模式） |
| allow/ignore client-updates | 允许/忽略客户端更新DNS记录 |
| default-lease-time 21600 | 默认超时时间 |
| max-lease-time 43200 | 最大超时时间 |
| option domain-name-servers 8.8.8.8 | 定义DNS服务器地址 |
| option domain-name "domain.org" | 定义DNS域名 |
| range | 定义用于分配的IP地址池 |
| option subnet-mask | 定义客户端的子网掩码 |
| option routers | 定义客户端的网关地址 |
| broadcast-address 广播地址 | 定义客户端的广播地址 |
| ntp-server IP地址 | 定义客户端的网络时间服务器（NTP） |
| nis-servers IP地址 | 定义客户端的NIS域服务器的地址 |
| hardware 硬件类型 MAC地址 | 指定网卡接口的类型与MAC地址 |
| server-name 主机名 | 向DHCP客户端通知DHCP服务器的主机名 |
| fixed-address IP地址 | 将某个固定的IP地址分配给指定主机 |
| time-offset 偏移差 | 指定客户端与格林尼治时间的偏移差 |

# 3.自动管理IP地址

DHCP协议的设计初衷是为了更高效地集中管理局域网内的IP地址资源。DHCP服务器会自动把IP地址、子网掩码、网关、DNS地址等网络信息分配给有需要的客户端，而且当客户端的租约时间到期后还可以自动回收所分配的IP地址，以便交给新加入的客户端。

DHCP服务器以及客户端的配置信息：


| 主机类型 | 操作系统 | IP地址 |
| --- | --- | --- |
| DHCP服务器 | RHEL 7 | 192.168.10.1 |
| DHCP客户机 | RHEL 7 | DHCP自动获取地址 |

作用域一般是个完整的IP地址段，而地址池中的IP地址才是真正供客户端使用的，因此地址池应该小于或等于作用域的IP地址范围。另外，由于VMware Workstation虚拟机软件自带DHCP服务，为了避免与自己配置的dhcpd服务程序产生冲突，应该将虚拟机软件（包括服务器和客户端）自带的DHCP功能关闭。

{% asset_img 2.png %}

{% asset_img 3.jpg %}

此外还要注意，DHCP客户端与服务器需要处于同一种网络模式—仅主机模式（Hostonly），否则就会产生物理隔离，从而无法获取IP地址。

在确认DHCP服务器的IP地址等网络信息配置妥当后就可以配置dhcpd服务程序了。请注意，在配置dhcpd服务程序时，配置文件中的每行参数后面都需要以分号（;）结尾，这是规定。

	[root@linuxprobe ~]# vim /etc/dhcp/dhcpd.conf
	ddns-update-style none;
	ignore client-updates;
	subnet 192.168.10.0 netmask 255.255.255.0 {
	range 192.168.10.50 192.168.10.150;
	option subnet-mask 255.255.255.0;
	option routers 192.168.10.1;
	option domain-name "linuxprobe.com";
	option domain-name-servers 192.168.10.1;
	default-lease-time 21600;
	max-lease-time 43200;
	}

dhcpd服务程序配置文件中使用的参数以及作用：


| 参数 | 作用 |
| --- | --- |
| ddns-update-style none; | 设置DNS服务不自动进行动态更新 |
| ignore client-updates; | 忽略客户端更新DNS记录 |
| subnet 192.168.10.0 netmask 255.255.255.0 { | 作用域为192.168.10.0/24网段 |
| range 192.168.10.50 192.168.10.150; | IP地址池为192.168.10.50-150（约100个IP地址） |
| option subnet-mask 255.255.255.0; | 定义客户端默认的子网掩码 |
| option routers 192.168.10.1; | 定义客户端的网关地址 |
| option domain-name "linuxprobe.com"; | 定义默认的搜索域 |
| option domain-name-servers 192.168.10.1; | 定义客户端的DNS地址 |
| default-lease-time 21600; | 定义默认租约时间（单位：秒） |
| max-lease-time 43200; | 定义最大预约时间（单位：秒） |
| } | 结束符 |

把配置过的dhcpd服务加入到开机启动项中，以确保当服务器下次开机后dhcpd服务依然能自动启动，并顺利地为客户端分配IP地址等信息：

	[root@linuxprobe ~]# systemctl start dhcpd
	[root@linuxprobe ~]# systemctl enable dhcpd
	 ln -s '/usr/lib/systemd/system/dhcpd.service' '/etc/systemd/system/multi-user.target.wants/dhcpd.service'

把dhcpd服务程序配置妥当之后就可以开启客户端来检验IP分配效果了。重启客户端的网卡服务后即可看到自动分配到的IP地址：

{% asset_img 4.jpg %}

# 4.分配固定IP地址

在DHCP协议中有个术语是“预约”，它用来确保局域网中特定的设备总是获取到固定的IP地址。换句话说，就是dhcpd服务程序会把某个IP地址私藏下来，只将其用于相匹配的特定设备。

要想把某个IP地址与某台主机进行绑定，就需要用到这台主机的MAC地址。

{% asset_img 5.jpg %}

也可以启动dhcpd服务程序，为主机分配一个IP地址，这样就会在DHCP服务器本地的日志文件中保存这次的IP地址分配记录，然后通过查看日志文件，获悉主机的MAC地址：

	[root@linuxprobe ~]# tail -f /var/log/messages 
	Mar 30 05:33:17 localhost dhcpd: Copyright 2004-2013 Internet Systems Consortium.
	Mar 30 05:33:17 localhost dhcpd: All rights reserved.
	Mar 30 05:33:17 localhost dhcpd: For info, please visit https://www.isc.org/software/dhcp/
	Mar 30 05:33:17 localhost dhcpd: Not searching LDAP since ldap-server, ldap-port and ldap-base-dn were not specified in the config file
	Mar 30 05:33:17 localhost dhcpd: Wrote 0 leases to leases file.
	Mar 30 05:33:17 localhost dhcpd: Listening on LPF/eno16777728/00:0c:29:c4:a4:09/192.168.10.0/24
	Mar 30 05:33:17 localhost dhcpd: Sending on LPF/eno16777728/00:0c:29:c4:a4:09/192.168.10.0/24
	Mar 30 05:33:17 localhost dhcpd: Sending on Socket/fallback/fallback-net
	Mar 30 05:33:26 localhost dhcpd: DHCPDISCOVER from **00:0c:29:27:c6:12** via eno16777728

在dhcpd服务程序的配置文件中，按照如下格式将IP地址与MAC地址进行绑定（MAC地址的间隔符为冒号（:））：

{% asset_img 6.png %}

确认参数填写正确后就可以保存退出配置文件，然后就可以重启dhcpd服务程序了：

	[root@linuxprobe ~]# systemctl restart dhcpd

需要说明的是，如果刚刚为这台主机分配了IP地址，则它的IP地址租约时间还没有到期，因此不会立即换成新绑定的IP地址。要想立即查看绑定效果，则需要重启一下客户端的网络服务：

{% asset_img 7.png %}