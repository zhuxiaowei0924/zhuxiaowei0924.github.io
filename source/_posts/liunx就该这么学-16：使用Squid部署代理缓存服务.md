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

## 3.2ACL访问控制

Squid服务程序的ACL是由多个策略规则组成的，它可以根据指定的策略规则来允许或限制访问请求，而且策略规则的匹配顺序与防火墙策略规则一样都是由上至下；在一旦形成匹配之后，则立即执行相应操作并结束匹配过程。为了避免ACL将所有流量全部禁止或全部放行，起不到预期的访问控制效果，运维人员通常会在ACL的最下面写上deny all或者allow all语句，以避免安全隐患。

实验1：只允许IP地址为192.168.10.20的客户端使用服务器上的Squid服务程序提供的代理服务，禁止其余所有的主机代理请求。

下面的配置文件依然是Squid服务程序的配置文件，但是需要留心配置参数的填写位置。如果写的太靠前，则有些Squid服务程序自身的语句都没有加载完，也会导致策略无效。当然也不用太靠后，大约在26~32行的位置就可以，而且采用分行填写的方式也便于日后的修改。

	[root@linuxprobe ~]# vim /etc/squid/squid.conf
	 23 acl Safe_ports port 591 # filemaker
	 24 acl Safe_ports port 777 # multiling http
	 25 acl CONNECT method CONNECT
	 26 acl client src 192.168.10.20
	 27 #
	 28 # Recommended minimum Access Permission configuration:
	 29 #
	 30 # Deny requests to certain unsafe ports
	 31 http_access allow client
	 32 http_access deny all
	 33 http_access deny !Safe_ports
	 34
	[root@linuxprobe ~]# systemctl restart squid

{% asset_img 18.jpg %}

实验2：禁止所有客户端访问网址中包含linux关键词的网站。

	[root@linuxprobe ~]# vim /etc/squid/squid.conf
	 24 acl Safe_ports port 777 # multiling http
	 25 acl CONNECT method CONNECT
	 26 acl deny_keyword url_regex -i linux
	 27 #
	 28 # Recommended minimum Access Permission configuration:
	 29 #
	 30 # Deny requests to certain unsafe ports
	 31 http_access deny deny_keyword
	 33 http_access deny !Safe_ports
	 34
	[root@linuxprobe ~]# systemctl restart squid

{% asset_img 19.jpg %}

实验3：禁止所有客户端访问某个特定的网站。

	[root@linuxprobe ~]# vim /etc/squid/squid.conf
	 24 acl Safe_ports port 777 # multiling http
	 25 acl CONNECT method CONNECT
	 26 acl deny_url url_regex http://www.linuxcool.com
	 27 #
	 28 # Recommended minimum Access Permission configuration:
	 29 #
	 30 # Deny requests to certain unsafe ports
	 31 http_access deny deny_url
	 33 http_access deny !Safe_ports
	 34
	[root@linuxprobe ~]# systemctl restart squid

{% asset_img 20.png %}

实验4：禁止员工在企业网内部下载带有某些后缀的文件。

[root@linuxprobe ~]# vim /etc/squid/squid.conf
	 24 acl Safe_ports port 777 # multiling http
	 25 acl CONNECT method CONNECT
	 26 acl badfile urlpath_regex -i \.mp3$ \.rar$
	 27 #
	 28 # Recommended minimum Access Permission configuration:
	 29 #
	 30 # Deny requests to certain unsafe ports
	 31 http_access deny badfile
	 33 http_access deny !Safe_ports
	 34
	[root@linuxprobe ~]# systemctl restart squid

## 3.3透明正向代理

透明”二字指的是让用户在没有感知的情况下使用代理服务，这样的好处是一方面不需要用户手动配置代理服务器的信息，进而降低了代理服务的使用门槛；另一方面也可以更隐秘地监督员工的上网行为。

在透明代理模式中，用户无须在浏览器或其他软件中配置代理服务器地址、端口号等信息，而是由DHCP服务器将网络配置信息分配给客户端主机。这样只要用户打开浏览器便会自动使用代理服务了。如果大家此时并没有配置DHCP服务器，可以手动配置客户端主机的网卡参数。

{% asset_img 21.png %}

有些时候会因为Windows系统的缓存原因导致依然能看到网页内容，这时可以换个网站尝试一下访问效果。

既然要让用户在无需过多配置系统的情况下就能使用代理服务，作为运维人员就必须提前将网络配置信息与数据转发功能配置好。前面已经配置好的网络参数，要使用SNAT技术完成数据的转发，让客户端主机将数据交给Squid代理服务器，再由后者转发到外网中。简单来说，就是让Squid服务器作为一个中间人，实现内网客户端主机与外部网络之间的数据传输。

要想让内网中的客户端主机能够访问外网，客户端主机首先要能获取到DNS地址解析服务的数据，这样才能在互联网中找到对应网站的IP地址。下面通过iptables命令实现DNS地址解析服务53端口的数据转发功能，并且允许Squid服务器转发IPv4数据包。sysctl -p命令的作用是让转发参数立即生效：

	[root@linuxprobe ~]# iptables -F
	[root@linuxprobe ~]# iptables -t nat -A POSTROUTING -p udp --dport 53 -o eno33554968 -j MASQUERADE
	[root@linuxprobe ~]# echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
	[root@linuxprobe ~]# sysctl -p 
	net.ipv4.ip_forward = 1

现在回到客户端主机，再次ping某个外网地址。此时可以发现，虽然不能连通网站，但是此时已经能够获取到外网DNS服务的域名解析数据。这个步骤非常重要，为接下来的SNAT技术打下了扎实的基础。

	C:\Users\linuxprobe>ping www.linuxprobe.com
	正在 Ping www.linuxprobe.com [116.31.127.233] 具有 32 字节的数据:
	请求超时。
	请求超时。
	请求超时。
	请求超时。
	116.31.127.233 的 Ping 统计信息:
	    数据包: 已发送 = 4，已接收 = 0，丢失 = 4 (100% 丢失)，

Squid服务程序透明代理模式的配置过程就十分简单了，只需要在主配置文件中服务器端口号后面追加上transparent单词（意思为“透明的”），然后把第62行的井号（#）注释符删除，设置缓存的保存路径就可以了。保存主配置文件并退出后再使用squid -k parse命令检查主配置文件是否有错误，以及使用squid -z命令对Squid服务程序的透明代理技术进行初始化。

	[root@linuxprobe ~]# vim /etc/squid/squid.conf
	………………省略部分输出信息………………
	58 # Squid normally listens to port 3128
	59 http_port 3128 transparent
	60
	61 # Uncomment and adjust the following to add a disk cache directory.
	62 cache_dir ufs /var/spool/squid 100 16 256
	63 
	………………省略部分输出信息………………
	[root@linuxprobe ~]# squid -k parse
	2017/04/13 06:40:44| Startup: Initializing Authentication Schemes ...
	2017/04/13 06:40:44| Startup: Initialized Authentication Scheme 'basic'
	………………省略部分输出信息………………
	[root@linuxprobe ~]# squid -z
	2017/04/13 06:41:26 kid1| Creating missing swap directories
	2017/04/13 06:41:26 kid1| /var/spool/squid exists
	………………省略部分输出信息………………
	[root@linuxprobe ~]# systemctl restart squid

在配置妥当并重启Squid服务程序且系统没有提示报错信息后，接下来就可以完成SNAT数据转发功能了。它的原理其实很简单，就是使用iptables防火墙管理命令把所有客户端主机对网站80端口的请求转发至Squid服务器本地的3128端口上。SNAT数据转发功能的具体配置参数如下：
	
	[root@linuxprobe ~]# iptables -t nat -A PREROUTING  -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 3128
	[root@linuxprobe ~]# iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o eno33554968 -j SNAT --to 您的桥接网卡IP地址
	[root@linuxprobe ~]# service iptables save
	iptables: Saving firewall rules to /etc/sysconfig/iptables:[ OK ]

这时客户端主机再刷新一下浏览器，就又能访问网络了。

# 4.反向代理

反向代理是Squid服务程序的一种重要模式，其原理是把一部分原本向网站源服务器发起的用户请求交给Squid服务器缓存节点来处理。

当前许多网站都默认禁止了反向代理功能。开启了CDN（内容分发网络）服务的网站也可以避免这种窃取行为。

使用Squid服务程序来配置反向代理服务非常简单。首先找到一个网站源服务器的IP地址，然后编辑Squid服务程序的主配置文件，把端口号3128修改为网站源服务器的地址和端口号，此时正向解析服务会被暂停（它不能与反向代理服务同时使用）。然后按照下面的参数形式写入需要反向代理的网站源服务器的IP地址信息，保存退出后重启Squid服务程序。

	[root@linuxprobe ~]# vim /etc/squid/squid.conf
	………………省略部分输出信息………………
	57 
	58 # Squid normally listens to port 3128
	59 http_port 您的桥接网卡IP地址:80 vhost
	60 cache_peer 网站源服务器IP地址 parent 80 0 originserver
	61 
	………………省略部分输出信息………………
	[root@linuxprobe ~]# systemctl restart squid













































