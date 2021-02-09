---
title: linux就该这么学：使用Vsftpd服务传输文件
date: 2021-02-09 09:45:32
tags:
- 第11章节：使用Vsftpd服务传输文件
categories:
- linux就该这么学
---

# 1.文件传输协议

FTP是一种在互联网中进行文件传输的协议，基于客户端/服务器模式，默认使用20、21号端口，其中端口20（数据端口）用于进行数据传输，端口21（命令端口）用于接受客户端发出的相关FTP命令与参数。

<!--more-->

FTP服务器是按照FTP协议在互联网上提供文件存储和访问服务的主机，FTP客户端则是向服务器发送连接请求，以建立数据传输链路的主机。FTP协议有下面两种工作模式。

	主动模式：FTP服务器主动向客户端发起连接请求。

	被动模式：FTP服务器等待客户端发起连接请求（FTP的默认工作模式）。

在配置妥当Yum软件仓库之后，就可以安装vsftpd服务程序了。

	[root@linuxprobe ~]# yum install vsftpd

iptables防火墙管理工具默认禁止了FTP传输协议的端口号，因此在正式配置vsftpd服务程序之前，为了避免这些默认的防火墙策略“捣乱”，还需要清空iptables防火墙的默认策略，并把当前已经被清理的防火墙策略状态保存下来：

	[root@linuxprobe ~]# iptables -F
	[root@linuxprobe ~]# service iptables save

vsftpd服务程序主配置文件中常用的参数以及作用：


| 参数 | 作用 |
| --- | --- |
| listen=[YES|NO] | 是否以独立运行的方式监听服务 |
| listen_address=IP地址 | 设置要监听的IP地址 |
| listen_port=21 | 设置FTP服务的监听端口 |
| download_enable＝[YES|NO] | 是否允许下载文件 |
| userlist_enable=[YES|NO]
  userlist_deny=[YES|NO] | 设置用户列表为“允许”还是“禁止”操作 |
| max_clients=0 | 最大客户端连接数，0为不限制 |
| max_per_ip=0 | 同一IP地址的最大连接数，0为不限制 |
| anonymous_enable=[YES|NO] | 是否允许匿名用户访问 |
| anon_upload_enable=[YES|NO] | 是否允许匿名用户上传文件 |
| anon_umask=022 | 匿名用户上传文件的umask值 |
| anon_root=/var/ftp | 匿名用户的FTP根目录 |
| anon_mkdir_write_enable=[YES|NO] | 是否允许匿名用户创建目录 |
| anon_other_write_enable=[YES|NO] | 是否开放匿名用户的其他写入权限（包括重命名、删除等操作权限） |
| anon_max_rate=0 | 匿名用户的最大传输速率（字节/秒），0为不限制 |
| local_enable=[YES|NO] | 是否允许本地用户登录FTP |
| local_umask=022 | 本地用户上传文件的umask值 |
| local_root=/var/ftp | 本地用户的FTP根目录 |
| chroot_local_user=[YES|NO] | 是否将用户权限禁锢在FTP目录，以确保安全 |
| local_max_rate=0 | 本地用户最大传输速率（字节/秒），0为不限制 |

# 2.Vsftpd服务程序

vsftpd作为更加安全的文件传输的服务程序，允许用户以三种认证模式登录到FTP服务器上。

	匿名开放模式：是一种最不安全的认证模式，任何人都可以无需密码验证而直接登录到FTP服务器。
	
	本地用户模式：是通过Linux系统本地的账户密码信息进行认证的模式，相较于匿名开放模式更安全，而且配置起来也很简单。但是如果被黑客破解了账户的信息，就可以畅通无阻地登录FTP服务器，从而完全控制整台服务器。
	
	虚拟用户模式：是这三种模式中最安全的一种认证模式，它需要为FTP服务单独建立用户数据库文件，虚拟出用来进行口令验证的账户信息，而这些账户信息在服务器系统中实际上是不存在的，仅供FTP服务程序进行认证使用。

	[root@linuxprobe ~]# yum install ftp

## 2.1匿名访问模式

vsftpd服务程序默认开启了匿名开放模式，我们需要做的就是开放匿名用户的上传、下载文件的权限，以及让匿名用户创建、删除、更名文件的权限。


| 参数 | 作用 |
| --- | --- |
| anonymous_enable=YES | 允许匿名访问模式 |
| anon_umask=022 | 匿名用户上传文件的umask值 |
| anon_upload_enable=YES | 允许匿名用户上传文件 |
| anon_mkdir_write_enable=YES | 允许匿名用户创建目录 |
| anon_other_write_enable=YES | 允许匿名用户修改目录名称或删除目录 |

在**服务器**中修改vsftpd.conf的配置文件：

	[root@linuxprobe ~]# vim /etc/vsftpd/vsftpd.conf
	1 anonymous_enable=YES
	2 anon_umask=022
	3 anon_upload_enable=YES
	4 anon_mkdir_write_enable=YES
	5 anon_other_write_enable=YES
	6 local_enable=YES
	7 write_enable=YES
	……

在vsftpd服务程序的主配置文件中正确填写参数，然后保存并退出。

在vsftpd服务程序的匿名开放认证模式下，默认访问的是/var/ftp目录。查看该目录的权限得知，只有root管理员才有写入权限，修改目录的所有者身份改为系统账户ftp：

	[root@linuxprobe ~]# ls -ld /var/ftp/pub
	drwxr-xr-x. 3 root root 16 Jul 13 14:38 /var/ftp/pub
	[root@linuxprobe ~]# chown -Rf ftp /var/ftp/pub
	[root@linuxprobe ~]# ls -ld /var/ftp/pub
	drwxr-xr-x. 3 ftp root 16 Jul 13 14:38 /var/ftp/pub

再使用getsebool命令查看与FTP相关的SELinux域策略：

	[root@linuxprobe ~]# getsebool -a | grep ftp
	ftp_home_dir --> off
	ftpd_anon_write --> off
	ftpd_connect_all_unreserved --> off
	ftpd_connect_db --> off
	ftpd_full_access --> off
	ftpd_use_cifs --> off
	ftpd_use_fusefs --> off
	ftpd_use_nfs --> off
	……

修改该策略规则，并且在设置时使用-P参数让修改过的策略永久生效，确保在服务器重启后依然能够顺利写入文件：

	[root@linuxprobe ~]# setsebool -P ftpd_full_access=on

再重启vsftpd服务程序并加入启动项，让新的配置参数生效。

	[root@linuxprobe ~]# systemctl restart vsftpd
	[root@linuxprobe ~]# systemctl enable vsftpd

现在就可以在**客户端**执行ftp命令连接到远程的FTP服务器了。在vsftpd服务程序的匿名开放认证模式下，其账户统一为anonymous，密码为空。可以切换到该目录下的pub目录中，然后尝试创建一个新的目录文件，以检验是否拥有写入权限：

	[root@linuxprobe ~]# ftp 192.168.10.10
	Connected to 192.168.10.10 (192.168.10.10).
	220 (vsFTPd 3.0.2)
	Name (192.168.10.10:root): anonymous
	331 Please specify the password.
	Password:此处敲击回车即可
	230 Login successful.
	Remote system type is UNIX.
	Using binary mode to transfer files.
	ftp> cd pub
	250 Directory successfully changed.
	ftp> mkdir files
	257 "/pub/files" created
	ftp> rename files database
	350 Ready for RNTO.
	250 Rename successful.
	ftp> rmdir database
	250 Remove directory operation successful.
	ftp> exit
	221 Goodbye.

## 2.2本地用户模式

本地用户模式的权限参数以及作用如下：

| 参数 | 作用 |
| --- | --- |
| anonymous_enable=NO | 禁止匿名访问模式 |
| local_enable=YES | 允许本地用户模式 |
| write_enable=YES | 设置可写权限 |
| local_umask=022 | 本地用户模式创建文件的umask值 |
| userlist_deny=YES | 启用“禁止用户名单”，名单文件为ftpusers和user_list |
| userlist_enable=YES | 开启用户作用名单文件功能 |

在vsftpd服务程序的主配置文件中正确填写参数，然后保存并退出：

	[root@linuxprobe ~]# vim /etc/vsftpd/vsftpd.conf
	anonymous_enable=NO
	local_enable=YES
	write_enable=YES
	local_umask=022
	dirmessage_enable=YES
	xferlog_enable=YES
	connect_from_port_20=YES
	xferlog_std_format=YES
	listen=NO
	listen_ipv6=YES
	pam_service_name=vsftpd
	userlist_enable=YES
	tcp_wrappers=YES

重启vsftpd服务程序并加入启动项，让新的配置参数生效：

	[root@linuxprobe ~]# systemctl restart vsftpd
	[root@linuxprobe ~]# systemctl enable vsftpd
	 ln -s '/usr/lib/systemd/system/vsftpd.service' '/etc/systemd/system/multi-user.target.wants/vsftpd.service

vsftpd服务程序为了保证服务器的安全性而默认禁止了root管理员和大多数系统用户的登录行为，这样可以有效地避免黑客通过FTP服务对root管理员密码进行暴力破解。如果您确认在生产环境中使用root管理员不会对系统安全产生影响，只需按照上面的提示删除掉root用户名即可。

	[root@linuxprobe ~]# cat /etc/vsftpd/user_list 
	1 # vsftpd userlist
	2 # If userlist_deny=NO, only allow users in this file
	3 # If userlist_deny=YES (default), never allow users in this file, and
	4 # do not even prompt for a password.
	5 # Note that the default vsftpd pam config also checks /etc/vsftpd/ftpusers
	6 # for users that are denied.
	7 root
	8 bin
	9 daemon
	10 adm
	11 lp
	12 sync
	13 shutdown
	14 halt
	15 mail
	16 news
	17 uucp
	18 operator
	19 games
	20 nobody
	[root@linuxprobe ~]# cat /etc/vsftpd/ftpusers 
	# Users that are not allowed to login via ftp
	1 root
	2 bin
	3 daemon
	4 adm
	5 lp
	6 sync
	7 shutdown
	8 halt
	9 mail
	10 news
	11 uucp
	12 operator
	13 games
	14 nobody

也可以选择ftpusers和user_list文件中没有的一个普通用户尝试登录FTP服务器：

	[root@linuxprobe ~]# ftp 192.168.10.10 
	Connected to 192.168.10.10 (192.168.10.10).
	220 (vsFTPd 3.0.2)
	Name (192.168.10.10:root): linuxprobe
	331 Please specify the password.
	Password:此处输入该用户的密码
	230 Login successful.
	Remote system type is UNIX.
	Using binary mode to transfer files.
	ftp> mkdir files
	550 Create directory operation failed.

在采用本地用户模式登录FTP服务器后，默认访问的是该用户的家目录，也就是说，访问的是/home/linuxprobe目录。而且该目录的默认所有者、所属组都是该用户自己，因此不存在写入权限不足的情况。但是当前的操作仍然被拒绝，是因为我们刚才将虚拟机系统还原到最初的状态了。为此，需要再次开启SELinux域中对FTP服务的允许策略：

	[root@linuxprobe ~]# getsebool -a | grep ftp
	ftp_home_dir --> off
	ftpd_anon_write --> off
	ftpd_connect_all_unreserved --> off
	ftpd_connect_db --> off
	ftpd_full_access --> off
	……
	[root@linuxprobe ~]# setsebool -P ftpd_full_access=on

配置妥当后再使用本地用户尝试登录下FTP服务器，分别执行文件的创建、重命名及删除等命令。操作均成功！

	[root@linuxprobe ~]# ftp 192.168.10.10
	Connected to 192.168.10.10 (192.168.10.10).
	220 (vsFTPd 3.0.2)
	Name (192.168.10.10:root): linuxprobe
	331 Please specify the password.
	Password:此处输入该用户的密码
	230 Login successful.
	Remote system type is UNIX.
	Using binary mode to transfer files.
	ftp> mkdir files
	257 "/home/linuxprobe/files" created
	ftp> rename files database
	350 Ready for RNTO.
	250 Rename successful.
	ftp> rmdir database
	250 Remove directory operation successful.
	ftp> exit
	221 Goodbye.

## 2.3虚拟用户模式











