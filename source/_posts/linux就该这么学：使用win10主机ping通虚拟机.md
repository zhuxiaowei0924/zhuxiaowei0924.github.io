---
title: linux就该这么学：使用win10主机ping通虚拟机
date: 2021-02-09 20:33:00
tags:
- 使用win10主机ping通虚拟机
categories:
- linux就该这么学
---

在win10系统中安装了vm12版本，Linux我选用的是redhat 7版本；成功安装redhat 7后，将linux虚拟机的IP地址固定：

<!--more-->

{% asset_img 1.png %}

linux虚拟机的外部连接方式为NAT模式，在linux虚拟机中能够ping通自己的IP，但无法ping通win10系统的IP，win10系统也无法ping通linux虚拟机IP。

{% asset_img 2.png %}

解决方法如下：

1.首先确认了本机的网络有无VMware Network Adapter VMnet8这两个网络：

{% asset_img 3.png %}

有以上两个网络的话，跳过此步骤；没有的话，则将VM虚拟机卸载，之后清理完之后再重新安装VM虚拟机软件。

2.安装好虚拟机后，打开终端，输入：vim /etc/sysconfig/network-scripts/ifcfg-eno16777728（网卡名称），编辑下网卡文件，将BOOTPROTO的值修改为static，并添加参数：IPADDR，GATEWAR，NETMASK，NM_CONTROLLED。IPADDR，GATEWAR这两个设置注意网段要与VMware Network Adapter VMnet8 的一致：

{% asset_img 4.png %}

3.设置好之后，保存退出；之后重启系统，用linux虚拟机ping下win10系统IP地址，测试是否能够ping通，如果可以，那就ok了。

linux虚拟机ping win10系统：

{% asset_img 5.png %}

win10系统ping linux虚拟机：

{% asset_img 6.png %}



















