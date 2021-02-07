---
title: linux虚拟机外网网络访问设置指南
date: 2021-02-07 14:23:03
tags:
- linux虚拟机外网网络访问设置指南
categories:
- linux就该这么学

---

要达到的效果就是在Linux虚拟机中可以上网：ping http://www.baidu.com

VMware中安装完 redhat7 后， 默认初始配置如下：

<!--more-->

# 1.VMware 网络配置 

（编辑 --> 虚拟网络编辑器）查看

默认配置为： (这些配置都不需要发生变更， 看看就好)

子网IP：192.168.154.0

子网掩码 : 255.255.255.0

可设置的IP网段为： 192.168.154.128 ~ 254

{% asset_img 1.jpg %}

点开 NAT设置可以看到 GATEWAY : 192.168.154.2:

{% asset_img 2.jpg %}

下面介绍 redhat 本身配置m修改配置参数如下：

注意各个参数与第一步 VMware 中的默认配置的联系 :

BOOTPROTO ： dhcp --> static （静态IP）

ONBOOT : no --> YES (开机自动连接网络)

IPADDR : 192.168.154.128 ~ 254 之间即可

NETMASK : 子网， 与 VMware 默认一致

GATEWAY : 网关， 与VMware 默认一致

DNS1 : 首选 DNS, 不配置会出现外网不通， 只能与内网互通的问题.

这里DNS要说一下的是，其他资料上有说要跟本机上网的DNS一致，还有说跟网关一致，或者改成
8.8.8.8，都试过，ping www.baidu.com 都可以，不同的机器可能有不同，希望可以共同探讨。

NM_CONTROLLED : “NM_CONTROLLED=no”表示该接口将通过该配置文件进行设置，而不是通过网络管理器进行管理。“ONBOOT=yes”告诉我们，系统将在启动时开启该接口。

编辑/etc/sysconfig/network-scipts/网卡名称：

{% asset_img 3.jpg %}

{% asset_img 4.jpg %}

{% asset_img 5.jpg %}

重启网络服务命令： systemctl restart network；

{% asset_img 6.jpg %}
















