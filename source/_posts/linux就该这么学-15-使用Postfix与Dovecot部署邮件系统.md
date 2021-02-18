---
title: 'linux就该这么学-15:使用Postfix与Dovecot部署邮件系统'
date: 2021-02-16 15:14:21
tags:
- 第15章节：使用Postfix与Dovecot部署邮件系统
categories:
- linux就该这么学

---

# 1.电子邮件系统

电子邮件系统基于邮件协议来完成电子邮件的传输，常见的邮件协议有下面这些：

<!--more-->

	简单邮件传输协议（Simple Mail Transfer Protocol，SMTP）：用于发送和中转发出的电子邮件，占用服务器的25/TCP端口。
	
	邮局协议版本3（Post Office Protocol 3）：用于将电子邮件存储到本地主机，占用服务器的110/TCP端口。
	
	Internet消息访问协议版本4（Internet Message Access Protocol 4）：用于在本地主机上访问邮件，占用服务器的143/TCP端口。

MUA：Mail User Agent，邮件用户代理；	MDA：Mail Delivery Agent，邮件投递代理；	

MTA：Mail Transfer Agent，邮件传输代理。

{% asset_img 1.png %}

在生产环境中部署企业级的电子邮件系统时，有4个注意事项请留意。

	1.添加反垃圾与反病毒模块：它能够很有效地阻止垃圾邮件或病毒邮件对企业信箱的干扰。
	2.对邮件加密：可有效保护邮件内容不被黑客盗取和篡改。
	3.添加邮件监控审核模块：可有效地监控企业全体员工的邮件中是否有敏感词、是否有透露企业资料等违规行为。
	4.保障稳定性：电子邮件系统的稳定性至关重要，运维人员应做到保证电子邮件系统的稳定运行，并及时做好防范分布式拒绝服务（Distributed Denial of Service，DDoS）攻击的准备。

# 2.部署基础的电子邮件系统

一个最基础的电子邮件系统肯定要能提供发件服务和收件服务，为此需要使用基于SMTP协议的Postfix服务程序提供发件服务功能，并使用基于POP3协议的Dovecot服务程序提供收件服务功能。

要想更好地检验电子邮件系统的配置效果，需要先部署bind服务程序，为电子邮件服务器和客户端提供DNS域名解析服务。

第1步：部署bind服务程序。

	[root@linuxprobe ~]# yum install bind-chroot

第2步：配置服务器主机名称，需要保证服务器主机名称与发信域名保持一致：

	[root@linuxprobe ~]# vim /etc/hostname
	mail.linuxprobe.com
	[root@linuxprobe ~]# hostname
	mail.linuxprobe.com
	[root@linuxprobe ~]# vim /etc/hosts
	192.168.10.10 mail.linuxprobe.com

第3步：清空iptables防火墙默认策略，并保存策略状态，避免因防火墙中默认存在的策略阻止了客户端DNS解析域名及收发邮件：

	[root@localhost ~]# iptables -F
	[root@localhost ~]# service iptables save
	iptables: Saving firewall rules to /etc/sysconfig/iptables:[  OK  ]

第4步：为电子邮件系统提供域名解析。

	[root@linuxprobe ~]# cat /etc/named.conf
	 10 options {
	 11 listen-on port 53 { any; };
	 12 listen-on-v6 port 53 { ::1; };
	 13 directory "/var/named";
	 14 dump-file "/var/named/data/cache_dump.db";
	 15 statistics-file "/var/named/data/named_stats.txt";
	 16 memstatistics-file "/var/named/data/named_mem_stats.txt";
	 17 allow-query { any; };
	 18 
	 ………………省略部分输出信息………………

	[root@linuxprobe ~]# cat /etc/named.rfc1912.zones
	zone "linuxprobe.com" IN {
	type master;
	file "linuxprobe.com.zone";
	allow-update {none;};
	};
	
	[root@linuxprobe ~]# cat /var/named/linuxprobe.com.zone

{% asset_img 2.png %}

	[root@linuxprobe ~]# systemctl restart named
	[root@linuxprobe ~]# systemctl enable named
	ln -s '/usr/lib/systemd/system/named.service' 
	'/etc/systemd/system/multi-user.target.wants/named.service'

测试时重启named服务时，显示报错：Job for named.service failed because the control process exited with error code. See “systemctl status named.service” and “journalctl -xe” for details.用named-checkconf -z /etc/named.conf（纠错命令）查看发现是由于配置文件内容写错导致！

这样电子邮件系统所对应的服务器主机名即为mail.linuxprobe.com，而邮件域为@linuxprobe.com。把服务器的DNS地址修改成本地IP地址：

{% asset_img 3.jpg %}

修改完IP地址后，需要重启网卡，之后可用ping mail.linuxprobe.com或nslookup命令测试下配置是否成功：

{% asset_img 4.png %}

用nslookup命令测试时，出现以下报错：server can't find mail.linuxprobe.com:NXDOMAIN，并提示当前的server 192.168.10.2 不能使用；此时可编辑/etc/resolv.conf文件，将报错的nameserver 192.168.10.2删除，保留nameserver 192.168.10.10即可！之后再重启网卡，测试一下！

## 2.1配置Postfix服务程序

第1步：安装Postfix服务程序；在安装完Postfix服务程序后，需要禁用iptables防火墙，否则外部用户无法访问电子邮件系统。

	[root@linuxprobe ~]# yum install postfix
	Loaded plugins: langpacks, product-id, subscription-manager
	rhel7 | 4.1 kB 00:00
	(1/2): rhel7/group_gz | 134 kB 00:00
	(2/2): rhel7/primary_db | 3.4 MB 00:00
	Package 2:postfix-2.10.1-6.el7.x86_64 already installed and latest version
	Nothing to do
	[root@linuxprobe ~]# systemctl disable iptables

第2步：配置Postfix服务程序，Postfix服务程序主配置文件（/etc/ postfix/main.cf）。

Postfix服务程序主配置文件中的重要参数：


| 参数 | 作用 |
| ---  | --- |
| myhostname  | 邮局系统的主机名 |
| mydomain  | 邮局系统的域名 |
| myorigin  | 从本机发出邮件的域名名称 |
| inet_interfaces  | 监听的网卡接口 |
| mydestination  | 可接收邮件的主机名或域名 |
| mynetworks  | 设置可转发哪些主机的邮件 |
| relay_domains  | 设置可转发哪些网域的邮件 |

在Postfix服务程序的主配置文件中，总计需要修改5处。首先是在第76行定义一个名为myhostname的变量，用来保存服务器的主机名称。

	[root@linuxprobe ~]# vim /etc/postfix/main.cf
	………………省略部分输出信息………………
	68 # INTERNET HOST AND DOMAIN NAMES
	69 # 
	70 # The myhostname parameter specifies the internet hostname of this
	71 # mail system. The default is to use the fully-qualified domain name
	72 # from gethostname(). $myhostname is used as a default value for many
	73 # other configuration parameters.
	74 #
	75 #myhostname = host.domain.tld
	76 myhostname = mail.linuxprobe.com
	………………省略部分输出信息………………

然后在第83行定义一个名为mydomain的变量，用来保存邮件域的名称。大家也要记住这个变量名称，下面将调用它：

	78 # The mydomain parameter specifies the local internet domain name.
	79 # The default is to use $myhostname minus the first component.
	80 # $mydomain is used as a default value for many other configuration
	81 # parameters.
	82 #
	83 mydomain = linuxprobe.com

在第99行调用前面的mydomain变量，用来定义发出邮件的域。调用变量的好处是避免重复写入信息，以及便于日后统一修改：

	94 # For the sake of consistency between sender and recipient addresses,
	95 # myorigin also specifies the default domain name that is appended
	96 # to recipient addresses that have no @domain part.
	97 #
	98 #myorigin = $myhostname
	99 myorigin = $mydomain

第4处修改是在第116行定义网卡监听地址。可以指定要使用服务器的哪些IP地址对外提供电子邮件服务；也可以干脆写成all，代表所有IP地址都能提供电子邮件服务：

	111 # Note: you need to stop/start Postfix when this parameter changes.
	112 #
	113 #inet_interfaces = all
	114 #inet_interfaces = $myhostname
	115 #inet_interfaces = $myhostname, localhost
	116 inet_interfaces = all

最后一处修改是在第164行定义可接收邮件的主机名或域名列表。这里可以直接调用前面定义好的myhostname和mydomain变量（如果不想调用变量，也可以直接调用变量中的值）：

	162 # See also below, section "REJECTING MAIL FOR UNKNOWN LOCAL USERS".
	163 #
	164 mydestination = $myhostname , $mydomain
	165 #mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
	166 #mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain,

第3步：创建电子邮件系统的登录账户。Postfix与vsftpd服务程序一样，都可以调用本地系统的账户和密码，因此在本地系统创建常规账户即可。最后重启配置妥当的postfix服务程序，并将其添加到开机启动项中。大功告成！

	[root@linuxprobe ~]# useradd boss
	[root@linuxprobe ~]# echo "linuxprobe" | passwd --stdin boss
	Changing password for user boss. passwd: all authentication tokens updated successfully.
	[root@linuxprobe ~]# systemctl restart postfix
	[root@linuxprobe ~]# systemctl enable postfix
	ln -s '/usr/lib/systemd/system/postfix.service' '/etc/systemd/system/multi-user.target.wants/postfix.service'

## 2.2配置Dovecot服务程序

第1步：安装Dovecot服务程序软件包。大家可自行配置Yum软件仓库、挂载光盘镜像到指定目录，然后输入要安装的dovecot软件包名称即可：

	[root@linuxprobe ~]# yum install dovecot

第2步：配置部署Dovecot服务程序。在Dovecot服务程序的主配置文件中进行如下修改。首先是第24行，把Dovecot服务程序支持的电子邮件协议修改为imap、pop3和lmtp。然后在这一行下面添加一行参数，允许用户使用明文进行密码验证。之所以这样操作，是因为Dovecot服务程序为了保证电子邮件系统的安全而默认强制用户使用加密方式进行登录，而由于当前还没有加密系统，因此需要添加该参数来允许用户的明文登录。

	[root@linuxprobe ~]# vim /etc/dovecot/dovecot.conf
	………………省略部分输出信息………………
	23 # Protocols we want to be serving.
	24 protocols = imap pop3 lmtp
	25 disable_plaintext_auth = no
	………………省略部分输出信息………………

在主配置文件中的第48行，设置允许登录的网段地址，也就是说我们可以在这里限制只有来自于某个网段的用户才能使用电子邮件系统。如果想允许所有人都能使用，则不用修改本参数：

	44 # Space separated list of trusted network ranges. Connections from these
	45 # IPs are allowed to override their IP addresses and ports (for logging and
	46 # for authentication checks). disable_plaintext_auth is also ignored for
	47 # these networks. Typically you'd specify your IMAP proxy servers here.
	48 login_trusted_networks = 192.168.10.0/24

第3步：配置邮件格式与存储路径。在Dovecot服务程序单独的子配置文件中，定义一个路径，用于指定要将收到的邮件存放到服务器本地的哪个位置。这个路径默认已经定义好了，我们只需要将该配置文件中第24行前面的井号（#）删除即可。

	[root@linuxprobe ~]# vim /etc/dovecot/conf.d/10-mail.conf
	21 # See doc/wiki/Variables.txt for full list. Some examples:
	22 #
	23 # mail_location = maildir:~/Maildir
	24 mail_location = mbox:~/mail:INBOX=/var/mail/%u
	25 # mail_location = mbox:/var/mail/%d/%1n/%n:INDEX=/var/indexes/%d/%1n/%n
	………………省略部分输出信息………………

然后切换到配置Postfix服务程序时创建的boss账户，并在家目录中建立用于保存邮件的目录。记得要重启Dovecot服务并将其添加到开机启动项中。
	
	[root@linuxprobe ~]# su - linuxprobe
	Last login: Sat Aug 15 16:15:58 CST 2017 on pts/1
	[linuxprobe@mail ~]$ mkdir -p mail/.imap/INBOX
	[linuxprobe@mail ~]$ exit
	[root@linuxprobe ~]# systemctl restart dovecot 
	[root@linuxprobe ~]# systemctl enable dovecot 
	ln -s '/usr/lib/systemd/system/dovecot.service' '/etc/systemd/system/multi-user.target.wants/dovecot.service'

## 2.3使用电子邮件系统

 服务器与客户端的操作系统与IP地址：


| 主机名称 | 操作系统 | IP地址 |
| ---  | --- | --- |
| 电子邮件系统及DNS服务器  | RHEL 7  | 192.168.10.10 |
| 客户端主机  | Windows 10  | 	192.168.10.30 |

{% asset_img 5.png %}

第1步：在win10系统中安装foxmail软件；安装后在软件账号管理-账号处新建账号，输入E-mail地址（如linuxprobe@linuxprobe.com,linuxprobe是linux用户名）和用户密码：

{% asset_img 6.png %}

第2步：设置接收服务器类型为POP3，输入邮件帐号和密码，设置POP服务器和SMTP服务器地址，两个为同一地址，如192.168.10.10，SSL端口处不勾选，之后创建成功！

{% asset_img 7.png %}

第3步：账号建立后，即可从win 10系统向linux发送邮件：

{% asset_img 8.png %}

第4步：发送邮件后，可在linux系统内部输入mail命令，并输入对应邮件数值编号查看邮件：

{% asset_img 9.png %}

第5步：从linux系统向win10电脑发送邮件，只需输入mail 邮件地址，然后输入subject和正文；最后输入点（.)结束：

{% asset_img 10.png %}

# 3.设置用户别名邮箱

用户别名功能是一项简单实用的邮件账户伪装技术，可以用来设置多个虚拟信箱的账户以接受发送的邮件，从而保证自身的邮件地址不被泄露，还可以用来接收自己的多个信箱中的邮件。

	[root@linuxprobe ~]# cat /etc/aliases
	#
	# Aliases in this file will NOT be expanded in the header from
	# Mail, but WILL be visible over networks or from /bin/mail.
	#
	# >>>>>>>>>> The program "newaliases" must be run after
	# >> NOTE >> this file is updated for any changes to
	# >>>>>>>>>> show through to sendmail.
	#
	# Basic system aliases -- these MUST be present.
	mailer-daemon: postmaster
	postmaster: root
	# General redirections for pseudo accounts.
	bin: root
	daemon: root
	adm: root
	lp: root
	sync: root
	shutdown: root
	halt: root
	mail: root
	news: root
	……

保存并退出aliases邮件别名服务的配置文件后，需要再执行一下newaliases命令，其目的是让新的用户别名配置文件立即生效。










