---
title: linux就该这么学：iptables与firewalld防火墙
date: 2021-02-03 21:22:55
tags:
- 第8章节：iptables与firewalld防火墙
categories:
- linux就该这么学

---

# 1.Iptables

## （1）策略与规则链

iptables服务把用于处理或过滤流量的策略条目称之为规则，多条规则可以组成一个规则链，而规则链则依据数据包处理位置的不同进行分类，具体如下：

<!--more-->

	在进行路由选择前处理数据包（PREROUTING）；

	处理流入的数据包（INPUT）；

	处理流出的数据包（OUTPUT）；

	处理转发的数据包（FORWARD）；

	在进行路由选择后处理数据包（POSTROUTING）。

iptables服务的术语分别是ACCEPT（允许流量通过）、REJECT（拒绝流量通过）、LOG（记录日志信息）、DROP（拒绝流量通过）。REJECT和DROP的不同点：就DROP来说，它是直接将流量丢弃而且不响应；REJECT则会在拒绝流量后再回复一条“您的信息已经收到，但是被扔掉了”信息，从而让流量发送方清晰地看到数据被拒绝的响应信息。

## （2）基本命令参数

防火墙策略规则的匹配顺序是从上至下的，因此要把较为严格、优先级较高的策略规则放到前面，以免发生错误。


| 参数 | 作用|
| --- | --- |
| -P | 设置默认策略 |
| -F | 清空规则链 |
| -L | 查看规则链 |
| -A | 在规则链的末尾加入新规则 |
| -I num | 在规则链的头部加入新规则 |
| -D num | 删除某一条规则 |
| -s | 匹配来源地址IP/MASK，加叹号“!”表示除这个IP外 |
| -d | 匹配目标地址 |
| -p | 匹配协议，如TCP、UDP、ICMP |
| --dport num |	匹配目标端口号 |
| --sport num	| 匹配来源端口号 |

规则链的默认策略拒绝动作只能是DROP，而不能是REJECT。

防火墙策略规则是按照从上到下的顺序匹配的，因此一定要把允许动作放到拒绝动作前面，否则所有的流量就将被拒绝掉，从而导致任何主机都无法访问我们的服务。

向INPUT规则链中添加拒绝192.168.10.5主机访问本机80端口（Web服务）的策略规则：

	[root@linuxprobe ~]# iptables -I INPUT -p tcp -s 192.168.10.5 --dport 80 -j REJECT

向INPUT规则链中添加拒绝所有主机访问本机1000～1024端口的策略规则：

	[root@linuxprobe ~]# iptables -A INPUT -p tcp --dport 1000:1024 -j REJECT
	[root@linuxprobe ~]# iptables -A INPUT -p udp --dport 1000:1024 -j REJECT

使用iptables命令配置的防火墙规则默认会在系统下一次重启时失效，如果想让配置的防火墙策略永久生效，还要执行保存命令：

	[root@linuxprobe ~]# service iptables save
	iptables: Saving firewall rules to /etc/sysconfig/iptables: [ OK ]

# 2.Firewalld


| 区域 | 默认规则策略 |
| --- | ---|
| trusted |	允许所有的数据包 |
| home | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh、mdns、ipp-client、amba-client与dhcpv6-client服务相关，则允许流量| 
| internal | 等同于home区域 | 
| work | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh、ipp-client与dhcpv6-client服务相关，则允许流量 | 
| public | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh、dhcpv6-client服务相关，则允许流量 | 
| external | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh服务相关，则允许流量 | 
| dmz | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh服务相关，则允许流量 | 
| block | 拒绝流入的流量，除非与流出的流量相关 | 
| drop | 拒绝流入的流量，除非与流出的流量相关 | 

## 2.1图形管理工具

{% asset_img 1.png %}

我们先将当前区域中请求http服务的流量设置为允许，但仅限当前生效：

{% asset_img 2.png %}

试添加一条防火墙策略规则，使其放行访问8080～8088端口（TCP协议）的流量，并将其设置为永久生效，配置完毕之后，还需要在Options菜单中单击Reload Firewalld命令，让配置的防火墙策略立即生效：

{% asset_img 3.png %}

{% asset_img 4.png %}

使用iptables命令实现SNAT技术(Source Network Address Translation，源网络地址转换),可以使得多个内网中的用户通过同一个外网IP接入Internet:

{% asset_img 5.png %}

将本机888端口的流量转发到22端口，且要求当前和长期均有效:

{% asset_img 6.png %}

{% asset_img 7.png %}

配置富规则，让192.168.10.20主机访问到本机的1234端口号:

{% asset_img 8.png %}

把网卡与防火墙策略区域进行绑定:

{% asset_img 9.png %}

## 2.3服务的访问控制列表

TCP Wrappers是RHEL 7系统中默认启用的一款流量监控程序，它能够根据来访主机的地址与本机的目标服务程序作出允许或拒绝的操作。换句话说，Linux系统中其实有两个层面的防火墙，第一种是前面讲到的基于TCP/IP协议的流量过滤工具，而TCP Wrappers服务则是能允许或禁止Linux系统提供服务的防火墙，从而在更高层面保护了Linux系统的安全运行。

在配置TCP Wrappers服务时需要遵循两个原则：

	（1）编写拒绝策略规则时，填写的是服务名称，而非协议名称；
	（2）建议先编写拒绝策略规则（/etc/hosts.deny），再编写允许策略规则（/etc/hosts.allow），以便直观地看到相应的效果。



