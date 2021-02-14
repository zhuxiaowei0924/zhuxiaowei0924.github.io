---
title: linux就该这么学-13：使用Bind提供域名解析服务
date: 2021-02-13 22:06:57
tags:
- 第13章节：使用Bind提供域名解析服务
categories:
- linux就该这么学

---

# 1.DNS域名解析服务

DNS：Domain Name System，域名系统。

<!--more-->

主服务器：在特定区域内具有唯一性，负责维护该区域内的域名与IP地址之间的对应关系。

从服务器：从主服务器中获得域名与IP地址的对应关系并进行维护，以防主服务器宕机等情况。

缓存服务器：通过向其他域名解析服务器查询获得域名与IP地址的对应关系，并将经常查询的域名信息保存到服务器本地，以此来提高重复查询时的效率。

在执行用户发起的域名查询请求时，具有递归查询和迭代查询两种方式。所谓递归查询，是指DNS服务器在收到用户发起的请求时，必须向用户返回一个准确的查询结果。如果DNS服务器本地没有存储与之对应的信息，则该服务器需要询问其他服务器，并将返回的查询结果提交给用户。而迭代查询则是指，DNS服务器在收到用户发起的请求时，并不直接回复查询结果，而是告诉另一台DNS服务器的地址，用户再向这台DNS服务器提交请求，这样依次反复，直到返回查询结果。

# 2.安装Bind服务程序

BIND：Berkeley Internet Name Domain，伯克利因特网名称域。

chroot：俗称牢笼机制。

	[root@linuxprobe ~]# yum install bind-chroot

在bind服务程序中有下面这三个比较关键的文件：

	主配置文件（/etc/named.conf）：只有58行，而且在去除注释信息和空行之后，实际有效的参数仅有30行左右，这些参数用来定义bind服务程序的运行。
	
	区域配置文件（/etc/named.rfc1912.zones）：用来保存域名和IP地址对应关系的所在位置。类似于图书的目录，对应着每个域和相应IP地址所在的具体位置，当需要查看或修改时，可根据这个位置找到相关文件。
	
	数据配置文件目录（/var/named）：该目录用来保存域名和IP地址真实对应关系的数据配置文件。

在Linux系统中，bind服务程序的名称为named。首先需要在/etc目录中找到该服务程序的主配置文件，然后把第11行和第17行的地址均修改为any，分别表示服务器上的所有IP地址均可提供DNS域名解析服务，以及允许所有人对本服务器发送DNS查询请求。这两个地方一定要修改准确。

	 [root@linuxprobe ~]# vim /etc/named.conf
	 1 //
	 2 // named.conf
	 3 //
	 4 // Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
	 5 // server as a caching only nameserver (as a localhost DNS resolver only).
	 6 //
	 7 // See /usr/share/doc/bind*/sample/ for example named configuration files.
	 8 //
	 9 
	 10 options {
	 11 listen-on port 53 { any; };
	 12 listen-on-v6 port 53 { ::1; };
	 13 directory "/var/named";
	 14 dump-file "/var/named/data/cache_dump.db";
	 15 statistics-file "/var/named/data/named_stats.txt";
	 16 memstatistics-file "/var/named/data/named_mem_stats.txt";
	 17 allow-query { any; };

## 2.1正向解析实验

正向解析是指根据域名（主机名）查找到对应的IP地址。

第1步：编辑区域配置文件。该文件中默认已经有了一些无关紧要的解析参数，旨在让用户有一个参考。我们可以将下面的参数添加到区域配置文件的最下面，当然，也可以将该文件中的原有信息全部清空，而只保留自己的域名解析信息：

	[root@linuxprobe ~]# vim /etc/named.rfc1912.zones
	zone "linuxprobe.com" IN {
	type master;
	file "linuxprobe.com.zone";
	allow-update {none;};
	};

第2步：编辑数据配置文件。我们可以从/var/named目录中复制一份正向解析的模板文件（named.localhost），然后把域名和IP地址的对应数据填写数据配置文件中并保存。在复制时记得加上-a参数，这可以保留原始文件的所有者、所属组、权限属性等信息，以便让bind服务程序顺利读取文件内容：

	[root@linuxprobe ~]# cd /var/named/
	[root@linuxprobe named]# ls -al named.localhost
	-rw-r-----. 1 root named 152 Jun 21 2007 named.localhost
	[root@linuxprobe named]# cp -a named.localhost linuxprobe.com.zone

{%asset_img 1.png%}

在保存并退出后文件后记得重启named服务程序，让新的解析数据生效:

	[root@linuxprobe named]# vim linuxprobe.com.zone
	[root@linuxprobe named]# systemctl restart named

第3步：检验解析结果;为了检验解析结果，一定要先把Linux系统网卡中的DNS地址参数修改成本机IP地址，这样就可以使用由本机提供的DNS查询服务了。

	[root@linuxprobe ~]# vim /etc/sysconfig/network-scripts/ifcfg-eno16777728（网卡名称）
	TYPE=Ethernet
	BOOTPROTO=none
	DEFROUTE=yes
	IPV4_FAILURE_FATAL=no
	IPV6INIT=yes
	IPV6_AUTOCONF=yes
	IPV6_DEFROUTE=yes
	IPV6_FAILURE_FATAL=no
	NAME=eno16777728
	UUID=f6b8c6f1-3067-47f6-984e-3d0a1f4d7c23
	ONBOOT=yes
	IPADDR0=192.168.10.10
	GATEWAY0=192.168.10.2
	PREFIX0=24
	HWADDR=00:0C:29:2B:CC:85
	DNS=192.168.10.10
	……

nslookup命令用于检测能否从DNS服务器中查询到域名与IP地址的解析记录，进而更准确地检验DNS服务器是否已经能够为用户提供服务。

	[root@linuxprobe ~]# systemctl restart network
	[root@linuxprobe ~]# nslookup
	> www.linuxprobe.com
	Server: 127.0.0.1
	Address: 127.0.0.1#53
	Name: www.linuxprobe.com
	Address: 192.168.10.10
	> bbs.linuxprobe.com
	Server: 127.0.0.1
	Address: 127.0.0.1#53
	Name: bbs.linuxprobe.com
	Address: 192.168.10.20

## 2.2反向解析实验

反向解析：将用户提交的IP地址解析为对应的域名信息。

第1步：编辑区域配置文件。除了不要写错格式之外，还需要记住此处定义的数据配置文件名称，因为一会儿还需要在/var/named目录中建立与其对应的同名文件。反向解析是把IP地址解析成域名格式，因此在定义zone（区域）时应该要把IP地址反写，比如原来是192.168.10.0，反写后应该就是10.168.192，而且只需写出IP地址的网络位即可。把下列参数添加至正向解析参数的后面。

	[root@linuxprobe ~]# vim /etc/named.rfc1912.zones
	zone "linuxprobe.com" IN {
	type master;
	file "linuxprobe.com.zone";
	allow-update {none;};
	};
	zone "10.168.192.in-addr.arpa" IN {
	type master;
	file "192.168.10.arpa";
	};

第2步：编辑数据配置文件。首先从/var/named目录中复制一份反向解析的模板文件（named.loopback），然后把下面的参数填写到文件中。其中，IP地址仅需要写主机位！

{%asset_img 2.png%}

	[root@linuxprobe named]# cp -a named.loopback 192.168.10.arpa
	[root@linuxprobe named]# vim 192.168.10.arpa

{%asset_img 3.png%}

	[root@linuxprobe named]# systemctl restart named

第3步：检验解析结果。在前面的正向解析实验中，已经把系统网卡中的DNS地址参数修改成了本机IP地址，因此可以直接使用nslookup命令来检验解析结果，仅需输入IP地址即可查询到对应的域名信息。

	[root@linuxprobe ~]# nslookup
	> 192.168.10.10
	Server: 127.0.0.1
	Address: 127.0.0.1#53
	10.10.168.192.in-addr.arpa name = ns.linuxprobe.com.
	10.10.168.192.in-addr.arpa name = www.linuxprobe.com.
	10.10.168.192.in-addr.arpa name = mail.linuxprobe.com.
	> 192.168.10.20
	Server: 127.0.0.1
	Address: 127.0.0.1#53
	20.10.168.192.in-addr.arpa name = bbs.linuxprobe.com.

# 3.部署从服务器

在本实验中，主服务器与从服务器分别使用的操作系统和IP地址如下：


| 主机名称 | 操作系统 | IP地址 |
| --- | --- | --- |
| 主服务器 | RHEL 7 | 192.168.10.10 |
| 从服务器 | RHEL 7 | 192.168.10.20 |

第1步：在**主服务器**的区域配置文件中允许该从服务器的更新请求，即修改allow-update {允许更新区域信息的主机地址;};参数，然后重启主服务器的DNS服务程序。
	
	[root@linuxprobe ~]# vim /etc/named.rfc1912.zones
	zone "linuxprobe.com" IN {
	type master;
	file "linuxprobe.com.zone";
	allow-update { 192.168.10.20; };
	};
	zone "10.168.192.in-addr.arpa" IN {
	type master;
	file "192.168.10.arpa";
	allow-update { 192.168.10.20; };
	};
	[root@linuxprobe ~]# systemctl restart named

第2步：在从服务器终端中输入命令：yum install bind-chroot，安装bind服务程序；再根据上面步骤编辑配置文件vim /etc/named.conf；

第3步：安装完之后，在**从服务器**中填写主服务器的IP地址与要抓取的区域信息，然后重启服务。注意此时的服务类型应该是slave（从），而不再是master（主）。masters参数后面应该为主服务器的IP地址，而且file参数后面定义的是同步数据配置文件后要保存到的位置，稍后可以在该目录内看到同步的文件。

	[root@linuxprobe ~]# vim /etc/named.rfc1912.zones
	zone "linuxprobe.com" IN {
	type slave;
	masters { 192.168.10.10; };
	file "slaves/linuxprobe.com.zone";
	};
	zone "10.168.192.in-addr.arpa" IN {
	type slave;
	masters { 192.168.10.10; };
	file "slaves/192.168.10.arpa";
	};
	[root@linuxprobe ~]# systemctl restart named

第4步：检验解析结果。当**从服务器**的DNS服务程序在重启后，一般就已经自动从主服务器上同步了数据配置文件，而且该文件默认会放置在区域配置文件中所定义的目录位置（/var/named/slaves）中。

第5步：随后在终端中输入命令：nmtui，修改从服务器的网络参数，把DNS地址参数修改成192.168.10.20，这样即可使用**从服务器**自身提供的DNS域名解析服务。最后就可以使用nslookup命令顺利看到解析结果了。

	[root@linuxprobe ~]# cd /var/named/slaves
	[root@linuxprobe slaves]# ls 
	192.168.10.arpa linuxprobe.com.zone
	[root@linuxprobe slaves]# nslookup
	> www.linuxprobe.com
	Server: 192.168.10.20
	Address: 192.168.10.20#53
	Name: www.linuxprobe.com
	Address: 192.168.10.10
	> 192.168.10.10
	Server: 192.168.10.20
	Address: 192.168.10.20#53
	10.10.168.192.in-addr.arpa name = www.linuxprobe.com.

# 4.安全的加密传输


| 主机名称 | 操作系统 | IP地址 |
| --- | --- | --- |
| 主服务器 | RHEL 7 | 192.168.10.10 |
| 从服务器 | RHEL 7 | 192.168.10.20 |

在从服务器上配妥bind服务程序并重启后，即可看到从主服务器中获取到的数据配置文件。

	[root@linuxprobe ~]# ls -al /var/named/slaves/
	total 12
	drwxrwx---. 2 named named 54 Jun 7 16:02 .
	drwxr-x---. 6 root named 4096 Jun 7 15:58 ..
	-rw-r--r--. 1 named named 432 Jun 7 16:02 192.168.10.arpa
	-rw-r--r--. 1 named named 439 Jun 7 16:02 linuxprobe.com.zone
	[root@linuxprobe ~]# rm -rf /var/named/slaves/*

第1步：在**主服务器**中生成密钥。dnssec-keygen命令用于生成安全的DNS服务密钥，其格式为“dnssec-keygen [参数]”，常用的参数以及作用如下：


| 参数 | 作用 |
| --- | --- |
| -a | 指定加密算法，包括RSAMD5（RSA）、RSASHA1、DSA、NSEC3RSASHA1、NSEC3DSA等 |
| -b | 密钥长度（HMAC-MD5的密钥长度在1~512位之间） |
| -n | 密钥的类型（HOST表示与主机相关） |

使用下述命令生成一个主机名称为master-slave的128位HMAC-MD5算法的密钥文件。在执行该命令后默认会在当前目录中生成公钥和私钥文件，我们需要把私钥文件中Key参数后面的值记录下来；

	[root@linuxprobe ~]# dnssec-keygen -a HMAC-MD5 -b 128 -n HOST master-slave
	Kmaster-slave.+157+46845
	[root@linuxprobe ~]# ls -al Kmaster-slave.+157+46845.*
	-rw-------. 1 root root 56 Jun 7 16:06 Kmaster-slave.+157+46845.key
	-rw-------. 1 root root 165 Jun 7 16:06 Kmaster-slave.+157+46845.private
	[root@linuxprobe ~]# cat Kmaster-slave.+157+46845.private
	Private-key-format: v1.3
	Algorithm: 157 (HMAC_MD5)
	Key: 1XEEL3tG5DNLOw+1WHfE3Q==
	Bits: AAA=
	Created: 20170607080621
	Publish: 20170607080621
	Activate: 20170607080621

第2步：在**主服务器**中创建密钥验证文件。进入bind服务程序用于保存配置文件的目录，把刚刚生成的密钥名称、加密算法和私钥加密字符串按照下面格式写入到tansfer.key传输配置文件中。为了安全起见，我们需要将文件的所属组修改成named，并将文件权限设置得要小一点，然后把该文件做一个硬链接到/etc目录中。

	[root@linuxprobe ~]# cd /var/named/chroot/etc/
	[root@linuxprobe etc]# vim transfer.key
	key "master-slave" {
	algorithm hmac-md5;
	secret "1XEEL3tG5DNLOw+1WHfE3Q==";
	};
	[root@linuxprobe etc]# chown root:named transfer.key
	[root@linuxprobe etc]# chmod 640 transfer.key
	[root@linuxprobe etc]# ln transfer.key /etc/transfer.key

第3步：开启并加载Bind服务的密钥验证功能。首先需要在主服务器的主配置文件中加载密钥验证文件，然后进行设置，使得只允许带有master-slave密钥认证的DNS服务器同步数据配置文件：

	[root@linuxprobe ~]# vim /etc/named.conf
	  9 include "/etc/transfer.key";
	 10 options {
	 11 listen-on port 53 { any; };
	 12 listen-on-v6 port 53 { ::1; };
	 13 directory "/var/named";
	 14 dump-file "/var/named/data/cache_dump.db";
	 15 statistics-file "/var/named/data/named_stats.txt";
	 16 memstatistics-file "/var/named/data/named_mem_stats.txt";
	 17 allow-query { any; };
	 18 allow-transfer { key master-slave; };
	………………省略部分输出信息………………
	[root@linuxprobe ~]# systemctl restart named

至此，DNS主服务器的TSIG密钥加密传输功能就已经配置完成。此时清空DNS从服务器同步目录中所有的数据配置文件，然后再次重启bind服务程序；

	[root@linuxprobe ~]# rm -rf /var/named/slaves/*
	[root@linuxprobe ~]# ls  /var/named/slaves/
	[root@linuxprobe ~]# systemctl restart named

第4步：配置**从服务器**，使其支持密钥验证。配置DNS从服务器和主服务器的方法大致相同，都需要在bind服务程序的配置文件目录中创建密钥认证文件，并设置相应的权限，然后把该文件做一个硬链接到/etc目录中。

	[root@linuxprobe ~]# cd /var/named/chroot/etc
	[root@linuxprobe etc]# vim transfer.key
	key "master-slave" {
	algorithm hmac-md5;
	secret "1XEEL3tG5DNLOw+1WHfE3Q==";
	};
	[root@linuxprobe etc]# chown root:named transfer.key
	[root@linuxprobe etc]# chmod 640 transfer.key
	[root@linuxprobe etc]# ln transfer.key /etc/transfer.key

第5步：开启并加载从服务器的密钥验证功能。这一步的操作步骤也同样是在主配置文件中加载密钥认证文件，然后按照指定格式写上主服务器的IP地址和密钥名称。注意，密钥名称等参数位置不要太靠前，大约在第43行比较合适，否则bind服务程序会因为没有加载完预设参数而报错：

	[root@linuxprobe etc]# vim /etc/named.conf
	  9 include "/etc/transfer.key";
	 10 options {
	 11 listen-on port 53 { 127.0.0.1; };
	 12 listen-on-v6 port 53 { ::1; };
	 13 directory "/var/named";
	 14 dump-file "/var/named/data/cache_dump.db";
	 15 statistics-file "/var/named/data/named_stats.txt";
	 16 memstatistics-file "/var/named/data/named_mem_stats.txt";
	 17 allow-query { localhost; };
		…………
	 41 session-keyfile "/run/named/session.key";
	 42 };
	 43 server 192.168.10.10
	 44 {
	 45 keys { master-slave; };
	 46 }; 
	 47 logging {

第6步：DNS从服务器同步域名区域数据。现在，两台服务器的bind服务程序都已经配置妥当，并匹配到了相同的密钥认证文件。接下来在从服务器上重启bind服务程序，可以发现又能顺利地同步到数据配置文件了。

	[root@linuxprobe ~]# systemctl restart named
	[root@linuxprobe ~]# ls /var/named/slaves/
	 192.168.10.arpa  linuxprobe.com.zone

# 5.分离解析技术


| 主机名称 | 操作系统 | IP地址 |
| --- | --- | --- |
| DNS服务器 | RHEL 7 | 北京网络：122.71.115.10 |
|          |        | 美国网络：106.185.25.10 |
| 北京用户 | windows 7 | 122.71.115.1 |
| 海外用户 | windows 7 | 106.185.25.1 |

第1步：安装bind域名解析服务，并修改bind服务程序的主配置文件，把第11行的监听端口与第17行的允许查询主机修改为any。由于配置的DNS分离解析功能与DNS根服务器配置参数有冲突，所以需要把第51~54行的根域信息删除。

	[root@linuxprobe ~]# vim /etc/named.conf
	………………省略部分输出信息………………
	 44 logging {
	 45 channel default_debug {
	 46 file "data/named.run";
	 47 severity dynamic;
	 48 };
	 49 };
	 50 
	 51 zone "." IN {
	 52 type hint;
	 53 file "named.ca";
	 54 };
	 55 
	 56 include "/etc/named.rfc1912.zones";
	 57 include "/etc/named.root.key";
	 58
	………………省略部分输出信息………………

第2步：编辑区域配置文件。把区域配置文件中原有的数据清空，然后按照以下格式写入参数。首先使用acl参数分别定义两个变量名称（china与american），当下面需要匹配IP地址时只需写入变量名称即可，这样不仅容易阅读识别，而且也利于修改维护。这里的难点是理解view参数的作用。它的作用是通过判断用户的IP地址是中国的还是美国的，然后去分别加载不同的数据配置文件（linuxprobe.com.china或linuxprobe.com.american）。这样，当把相应的IP地址分别写入到数据配置文件后，即可实现DNS的分离解析功能。这样一来，当中国的用户访问linuxprobe.com域名时，便会按照linuxprobe.com.china数据配置文件内的IP地址找到对应的服务器。

	[root@linuxprobe ~]# vim /etc/named.rfc1912.zones
	1 acl "china" { 122.71.115.0/24; };
	2 acl "american" { 106.185.25.0/24;};
	3 view "china"{
	4 match-clients { "china"; };
	5 zone "linuxprobe.com" {
	6 type master;
	7 file "linuxprobe.com.china";
	8 };
	9 };
	10 view "american" {
	11 match-clients { "american"; };
	12 zone "linuxprobe.com" {
	13 type master;
	14 file "linuxprobe.com.american";
	15 };
	16 };

第3步：建立数据配置文件。分别通过模板文件创建出两份不同名称的区域数据文件，其名称应与上面区域配置文件中的参数相对应。

	[root@linuxprobe ~]# cd /var/named
	[root@linuxprobe named]# cp -a named.localhost linuxprobe.com.china
	[root@linuxprobe named]# cp -a named.localhost linuxprobe.com.american
	[root@linuxprobe named]# vim linuxprobe.com.china

{% asset_img 4.png%}

	[root@linuxprobe named]# vim linuxprobe.com.american

{% asset_img 5.png%}

第4步：重新启动named服务程序，验证结果。

	[root@linuxprobe named]# systemctl restart named

第5步：将客户端主机（Windows系统或Linux系统均可）的IP地址分别设置为122.71.115.1与106.185.25.1，将DNS地址分别设置为服务器主机的两个IP地址。这样，当尝试使用nslookup命令解析域名时就能清晰地看到解析结果，分别如下图所示。

{% asset_img 6.png%}

{% asset_img 7.png%}
