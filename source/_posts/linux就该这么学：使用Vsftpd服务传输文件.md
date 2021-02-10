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

第1步：在**服务器**中创建用于进行FTP认证的用户数据库文件，其中奇数行为账户名，偶数行为密码。例如，我们分别创建出zhangsan和lisi两个用户，密码均为redhat：

	[root@linuxprobe ~]# cd /etc/vsftpd/
	[root@linuxprobe vsftpd]# vim vuser.list
	zhangsan
	redhat
	lisi
	redhat

第2步：明文信息既不安全，也不符合让vsftpd服务程序直接加载的格式，因此需要使用db_load命令用哈希（hash）算法将原始的明文信息文件转换成数据库文件，并且降低数据库文件的权限（避免其他人看到数据库文件的内容），然后再把原始的明文信息文件删除。

	[root@linuxprobe vsftpd]# db_load -T -t hash -f vuser.list vuser.db
	[root@linuxprobe vsftpd]# file vuser.db
	vuser.db: Berkeley DB (Hash, version 9, native byte-order)
	[root@linuxprobe vsftpd]# chmod 600 vuser.db
	[root@linuxprobe vsftpd]# rm -f vuser.list

第3步：创建vsftpd服务程序用于存储文件的根目录以及虚拟用户映射的系统本地用户。FTP服务用于存储文件的根目录指的是，当虚拟用户登录后所访问的默认位置。

为了方便管理FTP服务器上的数据，可以把这个系统本地用户的家目录设置为/var目录（该目录用来存放经常发生改变的数据）。并且为了安全起见，我们将这个系统本地用户设置为不允许登录FTP服务器，这不会影响虚拟用户登录，而且还可以避免黑客通过这个系统本地用户进行登录。

	[root@linuxprobe ~]# useradd -d /var/ftproot -s /sbin/nologin virtual
	[root@linuxprobe ~]# ls -ld /var/ftproot/
	drwx------. 3 virtual virtual 74 Jul 14 17:50 /var/ftproot/
	[root@linuxprobe ~]# chmod -Rf 755 /var/ftproot/

第4步：建立用于支持虚拟用户的PAM文件；新建一个用于虚拟用户认证的PAM文件vsftpd.vu，其中PAM文件内的“db=”参数为使用db_load命令生成的账户密码数据库文件的路径，但不用写数据库文件的后缀：

	[root@linuxprobe ~]# vim /etc/pam.d/vsftpd.vu
	auth       required     pam_userdb.so db=/etc/vsftpd/vuser
	account    required     pam_userdb.so db=/etc/vsftpd/vuser

第5步：在vsftpd服务程序的主配置文件中通过pam_service_name参数将PAM认证文件的名称修改为vsftpd.vu：


| 参数 | 作用 |
| --- | --- |
| anonymous_enable=NO | 禁止匿名开放模式 |
| local_enable=YES | 允许本地用户模式 |
| guest_enable=YES | 开启虚拟用户模式 |
| guest_username=virtual | 指定虚拟用户账户 |
| pam_service_name=vsftpd.vu | 指定PAM文件 |
| allow_writeable_chroot=YES | 允许对禁锢的FTP根目录执行写入操作，而且不拒绝用户的登录请求 |

	[root@linuxprobe ~]# vim /etc/vsftpd/vsftpd.conf
	1 anonymous_enable=NO
	2 local_enable=YES
	3 guest_enable=YES
	4 guest_username=virtual
	5 allow_writeable_chroot=YES
	6 write_enable=YES
	7 local_umask=022
	8 dirmessage_enable=YES
	9 xferlog_enable=YES
	10 connect_from_port_20=YES
	11 xferlog_std_format=YES
	12 listen=NO
	13 listen_ipv6=YES
	14 pam_service_name=vsftpd.vu
	15 userlist_enable=YES
	16 tcp_wrappers=YES

第6步：为虚拟用户设置不同的权限。虽然账户zhangsan和lisi都是用于vsftpd服务程序认证的虚拟账户，但是我们依然想对这两人进行区别对待。比如，允许张三上传、创建、修改、查看、删除文件，只允许李四查看文件。这可以通过vsftpd服务程序来实现。只需新建一个目录，在里面分别创建两个以zhangsan和lisi命名的文件，其中在名为zhangsan的文件中写入允许的相关权限（使用匿名用户的参数）
	
	[root@linuxprobe ~]# mkdir /etc/vsftpd/vusers_dir/
	[root@linuxprobe ~]# cd /etc/vsftpd/vusers_dir/
	[root@linuxprobe vusers_dir]# touch lisi
	[root@linuxprobe vusers_dir]# vim zhangsan
	anon_upload_enable=YES
	anon_mkdir_write_enable=YES
	anon_other_write_enable=YES

第7步：再次修改vsftpd主配置文件，通过添加user_config_dir参数来定义这两个虚拟用户不同权限的配置文件所存放的路径。为了让修改后的参数立即生效，需要重启vsftpd服务程序并将该服务添加到开机启动项中：

	[root@linuxprobe ~]# vim /etc/vsftpd/vsftpd.conf
	anonymous_enable=NO
	local_enable=YES
	guest_enable=YES
	guest_username=virtual
	allow_writeable_chroot=YES
	write_enable=YES
	local_umask=022
	dirmessage_enable=YES
	xferlog_enable=YES
	connect_from_port_20=YES
	xferlog_std_format=YES
	listen=NO
	listen_ipv6=YES
	pam_service_name=vsftpd.vu
	userlist_enable=YES
	tcp_wrappers=YES
	user_config_dir=/etc/vsftpd/vusers_dir
	[root@linuxprobe ~]# systemctl restart vsftpd
	[root@linuxprobe ~]# systemctl enable vsftpd
	 ln -s '/usr/lib/systemd/system/vsftpd.service' '/etc/systemd/system/multi-user.target.wants/vsftpd.service

第8步：设置SELinux域允许策略，然后使用虚拟用户模式登录FTP服务器。

	[root@linuxprobe ~]# getsebool -a | grep ftp
	ftp_home_dir –> off
	ftpd_anon_write –> off
	ftpd_connect_all_unreserved –> off
	ftpd_connect_db –> off
	ftpd_full_access –> off
	……
	[root@linuxprobe ~]# setsebool -P ftpd_full_access=on

第9步：然后在**客户端**上重新安装FTP服务器；然后用虚拟用户模式成功登录FTP服务器，还可以分别使用账户zhangsan和lisi来检验他们的权限。

	[root@linuxprobe ~]# ftp 192.168.10.10
	Connected to 192.168.10.10 (192.168.10.10).
	220 (vsFTPd 3.0.2)
	Name (192.168.10.10:root): lisi
	331 Please specify the password.
	Password:此处输入虚拟用户的密码
	230 Login successful.
	Remote system type is UNIX.
	Using binary mode to transfer files.
	ftp> mkdir files
	550 Permission denied.
	ftp> exit
	221 Goodbye.
	[root@linuxprobe ~]# ftp 192.168.10.10
	Connected to 192.168.10.10 (192.168.10.10).
	220 (vsFTPd 3.0.2)
	Name (192.168.10.10:root): zhangsan
	331 Please specify the password.
	Password:此处输入虚拟用户的密码
	230 Login successful.
	Remote system type is UNIX.
	Using binary mode to transfer files.
	ftp> mkdir files
	257 "/files" created
	ftp> rename files database
	350 Ready for RNTO.
	250 Rename successful.
	ftp> rmdir database
	250 Remove directory operation successful.
	ftp> exit
	221 Goodbye.

# 3.TFTP简单文件传输协议

简单文件传输协议（Trivial File Transfer Protocol，TFTP）是一种基于UDP协议在客户端和服务器之间进行简单文件传输的协议。顾名思义，它提供不复杂、开销不大的文件传输服务（可将其当作FTP协议的简化版本）。

第1步：在服务器上安装tftp-server、tftp和xinetd：

	[root@linuxprobe ~]# yum install tftp-server tftp xinetd

第2步：TFTP服务是使用xinetd服务程序来管理的。xinetd服务可以用来管理多种轻量级的网络服务，而且具有强大的日志功能。简单来说，在安装TFTP软件包后，还需要在xinetd服务程序中将其开启，把默认的禁用（disable）参数修改为no：

	[root@linuxprobe ~]# vim /etc/xinetd.d/tftp
	service tftp
	{
	        socket_type             = dgram
	        protocol                = udp
	        wait                    = yes
	        user                    = root
	        server                  = /usr/sbin/in.tftpd
	        server_args             = -s /var/lib/tftpboot
	        disable                 = no
	        per_source              = 11
	        cps                     = 100 2
	        flags                   = IPv4
	}


第3步：重启xinetd服务并将它添加到系统的开机启动项中，以确保TFTP服务在系统重启后依然处于运行状态：

	[root@linuxprobe ~]# systemctl restart xinetd
	[root@linuxprobe ~]# systemctl enable xinetd

第4步：考虑到有些系统的防火墙默认没有允许UDP协议的69端口，因此需要手动将该端口号加入到防火墙的允许策略中（看电脑实际情况而定）：
	
	[root@linuxprobe ~]# firewall-cmd --permanent --add-port=69/udp
	success
	[root@linuxprobe ~]# firewall-cmd --reload 
	success

第5步：TFTP的根目录为/var/lib/tftpboot，可以在改目录中放置或新建自己想要传输的数据文件，如：

[root@linuxprobe ~]# echo "i love linux" > /var/lib/tftpboot/readme.txt

第6步：在客户端上按照以上步骤重新安装tftp；安装完成后，我们可以使用刚安装好的tftp命令尝试访问服务器端目录中的文件，亲身体验TFTP服务的文件传输过程。


| 命令 | 作用 |
| --- | --- |
| ? | 帮助信息 |
| put | 上传文件 |
| get | 下载文件 |
| verbose | 显示详细的处理信息 |
| status | 显示当前的状态信息 |
| binary | 使用二进制进行传输 |
| ascii | 使用ASCII码进行传输 |
| timeout | 设置重传的超时时间 |
| quit | 退出 |

	[root@linuxprobe ~]# tftp 192.168.10.10
	tftp> get readme.txt
	tftp> quit
	[root@linuxprobe ~]# ls
	anaconda-ks.cfg Documents initial-setup-ks.cfg Pictures readme.txt Videos
	Desktop Downloads Music Public Templates
	[root@linuxprobe ~]# cat readme.txt 
	i love linux















