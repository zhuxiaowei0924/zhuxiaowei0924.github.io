---
title: linux就该这么学：使用ssh服务管理远程主机
date: 2021-02-04 22:00:33
tags:
- 第9章节：使用ssh服务管理远程主机
categories:
- linux就该这么学

---

# 1.配置网卡服务

# 1.1配置网卡参数

{% asset_img 1.png %}

{% asset_img 2.png %}

{% asset_img 3.png %}

{% asset_img 4.png %}

{% asset_img 5.png %}

{% asset_img 6.png %}

{% asset_img 7.png %}

{% asset_img 8.png %}

用Vim编辑器将网卡配置文件中的ONBOOT参数修改成yes，这样在系统重启后网卡就被激活了:

	[root@linuxprobe ~]# vim /etc/sysconfig/network-scripts/ifcfg-eno16777736
	TYPE=Ethernet
	BOOTPROTO=none
	...
	ONBOOT=yes

修改完Linux系统中的服务配置文件后,要手动重启相应的服务:

	[root@linuxprobe ~]# systemctl restart network

## 1.2创建网络会话

NetworkManager：是一种动态管理网络配置的守护进程，能够让网络设备保持连接状态；可以使用nmcli命令来管理Network Manager服务。

可以使用nmcli命令并按照“connection add con-name type ifname”的格式来创建网络会话。

使用con-name参数指定使用的网络会话名称company，然后依次用ifname参数指定网卡名称，用autoconnect no参数设置该网络会话默认不被自动激活，以及用ip4及gw4参数手动指定网络的IP地址：

	[root@linuxprobe ~]# nmcli connection add con-name company ifname eno16777736 autoconnect no type ethernet ip4 192.168.10.10/24 gw4 192.168.10.1

使用con-name参数指定网络会话名称house，使用外部DHCP服务器自动获得IP地址，不需要进行手动指定：

	[root@linuxprobe ~]# nmcli connection add con-name house type ethernet ifname eno16777736

使配置过的网络会话生效：

	[root@linuxprobe ~]# nmcli connection up house 

## 1.3绑定两块网卡

{% asset_img 9.png %}

{% asset_img 10.png %}

使用Vim文本编辑器来配置网卡设备的绑定参数：
	[root@linuxprobe ~]# vim /etc/sysconfig/network-scripts/ifcfg-eno16777736
	TYPE=Ethernet
	BOOTPROTO=none
	ONBOOT=yes
	USERCTL=no
	DEVICE=eno16777736
	MASTER=bond0
	SLAVE=yes
	[root@linuxprobe ~]# vim /etc/sysconfig/network-scripts/ifcfg-eno33554968
	TYPE=Ethernet
	BOOTPROTO=none
	ONBOOT=yes
	USERCTL=no
	DEVICE=eno33554968
	MASTER=bond0
	SLAVE=yes

将绑定后的设备命名为bond0并把IP地址等信息填写进去，这样当用户访问相应服务的时候，实际上就是由这两块网卡设备在共同提供服务：

	[root@linuxprobe ~]# vim /etc/sysconfig/network-scripts/ifcfg-bond0
	TYPE=Ethernet
	BOOTPROTO=none
	ONBOOT=yes
	USERCTL=no
	DEVICE=bond0
	IPADDR=192.168.10.10
	PREFIX=24
	DNS=192.168.10.1
	NM_CONTROLLED=no

让Linux内核支持网卡绑定驱动。常见的网卡绑定驱动有三种模式—mode0、mode1和mode6。下面以绑定两块网卡为例，讲解使用的情景。

	mode0（平衡负载模式）：平时两块网卡均工作，且自动备援，但需要在与服务器本地网卡相连的交换机设备上进行端口聚合来支持绑定技术。

	mode1（自动备援模式）：平时只有一块网卡工作，在它故障后自动替换为另外的网卡。

	mode6（平衡负载模式）：平时两块网卡均工作，且自动备援，无须交换机设备提供辅助支持。

使用Vim文本编辑器创建一个用于网卡绑定的驱动文件，使得绑定后的bond0网卡设备能够支持绑定技术（bonding）；同时定义网卡以mode6模式进行绑定，且出现故障时自动切换的时间为100毫秒：

	[root@linuxprobe ~]# vim /etc/modprobe.d/bond.conf
	alias bond0 bonding
	options bond0 miimon=100 mode=6

重启网络服务后网卡绑定操作即可成功。正常情况下只有bond0网卡设备才会有IP地址等信息：

	[root@linuxprobe ~]# systemctl restart network