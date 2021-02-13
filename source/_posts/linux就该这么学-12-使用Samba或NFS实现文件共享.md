---
title: 'linux就该这么学-12:使用Samba或NFS实现文件共享'
date: 2021-02-11 13:50:48
tags:
- 第12章节：使用Samba或NFS实现文件共享
categories:
- linux就该这么学

---

# 1.SAMBA文件共享服务

Samba服务程序现在已经成为在Linux系统与Windows系统之间共享文件的最佳选择。

首先需要先通过Yum软件仓库来安装Samba服务程序：

<!--more--t> 

	[root@linuxprobe ~ ]# yum install samba

 Samba服务程序中的参数以及作用:

	[global]									#全局参数。

		workgroup = MYGROUP						#工作组名称

		server string = Samba Server Version %v	#服务器介绍信息，参数%v为显示SMB版本号

		log file = /var/log/samba/log.%m		#定义日志文件的存放位置与名称，参数%m为来访的主机名

		max log size = 50						#定义日志文件的最大容量为50KB

		security = user							#安全验证的方式，总共有4种

		#share：来访主机无需验证口令；比较方便，但安全性很差

		#user：需验证来访主机提供的口令后才可以访问；提升了安全性

		#server：使用独立的远程主机验证来访主机提供的口令（集中管理账户）

		#domain：使用域控制器进行身份验证

		passdb backend = tdbsam					#定义用户后台的类型，共有3种

		#smbpasswd：使用smbpasswd命令为系统用户设置Samba服务程序的密码

		#tdbsam：创建数据库文件并使用pdbedit命令建立Samba服务程序的用户

		#ldapsam：基于LDAP服务进行账户验证

		load printers = yes						#设置在Samba服务启动时是否共享打印机设备

		cups options = raw						#打印机的选项

	[homes]										#共享参数

		comment = Home Directories				#描述信息

		browseable = no							#指定共享信息是否在“网上邻居”中可见

		writable = yes							#定义是否可以执行写入操作，与“read only”相反

	[printers]									#打印机共享参数

		comment = All Printers	

		path = /var/spool/samba					#共享文件的实际路径(重要)。

		browseable = no	
		guest ok = no							#是否所有人可见，等同于"public"参数。

		writable = no	

		printable = yes	

	[root@linuxprobe ~]# cat /etc/samba/smb.conf

## 1.1配置共享资源

用于设置Samba服务程序的参数以及作用：


| 参数 | 作用 |
| --- | --- |
| [database] | 共享名称为database |
| comment = Do not arbitrarily modify the database file | 警告用户不要随意修改数据库 |
| path = /home/database | 共享目录为/home/database |
| public = no | 关闭“所有人可见” |
| writable = yes | 允许写入操作 |

第1步：创建用于访问共享资源的账户信息。在RHEL 7系统中，Samba服务程序默认使用的是用户口令认证模式（user）。这种认证模式可以确保仅让有密码且受信任的用户访问共享资源，而且验证过程也十分简单。不过，只有建立账户信息数据库之后，才能使用用户口令认证模式。另外，Samba服务程序的数据库要求账户必须在当前系统中已经存在，否则日后创建文件时将导致文件的权限属性混乱不堪，由此引发错误。

pdbedit命令用于管理SMB服务程序的账户信息数据库，格式为“pdbedit [选项] 账户”。在第一次把账户信息写入到数据库时需要使用-a参数，以后在执行修改密码、删除账户等操作时就不再需要该参数了。


| 参数 | 作用 |
| --- | --- |
| -a 用户名 | 建立Samba用户 |
| -x 用户名 | 删除Samba用户 |
| -L | 列出用户列表 |
| -Lv | 列出用户详细信息的列表 |

	[root@linuxprobe ~]# id linuxprobe
	uid=1000(linuxprobe) gid=1000(linuxprobe) groups=1000(linuxprobe)
	[root@linuxprobe ~]# pdbedit -a -u linuxprobe
	new password:此处输入该账户在Samba服务数据库中的密码
	retype new password:再次输入密码进行确认
	Unix username: linuxprobe
	NT username: 
	Account Flags: [U ]
	……
	Logon hours : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF

第2步：创建用于共享资源的文件目录。

	[root@linuxprobe ~]# mkdir /home/database
	[root@linuxprobe ~]# chown -Rf linuxprobe:linuxprobe /home/database
	[root@linuxprobe ~]# semanage fcontext -a -t samba_share_t /home/database
	[root@linuxprobe ~]# restorecon -Rv /home/database
	restorecon reset /home/database context unconfined_u:object_r:home_root_t:s0->unconfined_u:object_r:samba_share_t:s0

第3步：设置SELinux服务与策略，使其允许通过Samba服务程序访问普通用户家目录。

	[root@linuxprobe ~]# getsebool -a | grep samba
	samba_create_home_dirs --> off
	samba_domain_controller --> off
	samba_enable_home_dirs --> off
	……
	[root@linuxprobe ~]# setsebool -P samba_enable_home_dirs on

第4步：在Samba服务程序的主配置文件中，写入共享信息。在原始的配置文件中，[homes]参数为来访用户的家目录共享信息，[printers]参数为共享的打印机设备可以手动删除，这没有任何问题。

	[root@linuxprobe ~]# vim /etc/samba/smb.conf 
	[global]
	 workgroup = MYGROUP
	 server string = Samba Server Version %v
	 log file = /var/log/samba/log.%m
	 max log size = 50
	 security = user
	 passdb backend = tdbsam
	 load printers = yes
	 cups options = raw
	[database]
	 comment = Do not arbitrarily modify the database file
	 path = /home/database
	 public = no
	 writable = yes

第5步：Samba服务程序的配置工作基本完毕。接下来重启smb服务（Samba服务程序在Linux系统中的名字为smb）并清空iptables防火墙。

	[root@linuxprobe ~]# systemctl restart smb
	[root@linuxprobe ~]# systemctl enable smb
	 ln -s '/usr/lib/systemd/system/smb.service' '/etc/systemd/system/multi-user.target.wants/smb.service'
	[root@linuxprobe ~]# iptables -F
	[root@linuxprobe ~]# service iptables save
	iptables: Saving firewall rules to /etc/sysconfig/iptables:[ OK ]

## 1.2windows挂载共享

Samba服务器和Windows客户端使用的操作系统以及IP地址（举例）：


| 主机名称 | 操作系统 | IP地址 |
| --- | --- |
| Samba共享服务器 | 	RHEL 7 | 192.168.10.10 |
| Linux客户端 | RHEL 7 | 192.168.10.20 |
| Windows客户端 | Windows 7 | 192.168.10.30 |

要在Windows系统中访问共享资源，只需在Windows的“运行”命令框中输入两个反斜杠，然后再加服务器的IP地址即可：

{% asset_img 1.png %}

在RHEL 7系统中，Samba服务程序使用的果然是独立的账户信息数据库。所以，即便在Linux系统中有一个linuxprobe账户，Samba服务程序使用的账户信息数据库中也有一个同名的linuxprobe账户，大家也一定要弄清楚它们各自所对应的密码。

{% asset_img 2.jpg %}

正确输入linuxprobe账户名以及使用pdbedit命令设置的密码后，就可以登录到共享界面中了!

由于Windows系统的缓存原因，有可能您在第二次登录时提供了正确的账户和密码，依然会报错，这时只需要重新启动一下Windows客户端就没问题了!

## 1.3linux挂载共享

 Samba共享服务器和Linux客户端各自使用的操作系统以及IP地址：


| 主机名称 | 操作系统 | IP地址 |
| --- | --- |
| Samba共享服务器 | 	RHEL 7 | 192.168.10.10 |
| Linux客户端 | RHEL 7 | 192.168.10.20 |
| Windows客户端 | Windows 7 | 192.168.10.30 |

	[root@linuxprobe ~]# yum install cifs-utils
	Loaded plugins: langpacks, product-id, subscription-manager
	rhel | 4.1 kB 00:00 
	Resolving Dependencies
	--> Running transaction check
	---> Package cifs-utils.x86_64 0:6.2-6.el7 will be installed
	--> Finished Dependency Resolution
	Dependencies Resolved
	……
	Installed:
	 cifs-utils.x86_64 0:6.2-6.el7 
	Complete!

在Linux客户端，按照Samba服务的用户名、密码、共享域的顺序将相关信息写入到一个认证文件中。为了保证不被其他人随意看到，最后把这个认证文件的权限修改为仅root管理员才能够读写：

	[root@linuxprobe ~]# vim auth.smb
	username=linuxprobe
	password=redhat
	domain=MYGROUP
	[root@linuxprobe ~]# chmod -Rf 600 auth.smb

现在，在Linux客户端上创建一个用于挂载Samba服务共享资源的目录，并把挂载信息写入到/etc/fstab文件中，以确保共享挂载信息在服务器重启后依然生效：

	[root@linuxprobe ~]# mkdir /database
	[root@linuxprobe ~]# vim /etc/fstab
	/dev/mapper/rhel-swap swap swap defaults 0 0
	/dev/cdrom /media/cdrom iso9660 defaults 0 0 
	//192.168.10.10/database /database cifs credentials=/root/auth.smb 0 0
	[root@linuxprobe ~]# mount -a

Linux客户端成功地挂载了Samba服务的共享资源。进入到挂载目录/database后就可以看到Windows系统访问Samba服务程序时留下来的文件了（即文件Memo.txt）。当然，我们也可以对该文件进行读写操作并保存。

	[root@linuxprobe ~]# cat /database/Memo.txt

# 2.NFS网络文件系统

NFS服务：用于两台linux主机进行共享文件；

	[root@linuxprobe ~]# yum install nfs-utils


| 主机名称 | 操作系统 | IP地址 |
| --- | --- |
| Linux服务端 | 	RHEL 7 | 192.168.10.10 |
| Linux客户端 | RHEL 7 | 192.168.10.20 |

	[root@linuxprobe ~]# iptables -F
	[root@linuxprobe ~]# service iptables save
	iptables: Saving firewall rules to /etc/sysconfig/iptables:[ OK ]

第2步：在NFS服务器上建立用于NFS文件共享的目录，并设置足够的权限确保其他人也有写入权限。

	[root@linuxprobe ~]# mkdir /nfsfile
	[root@linuxprobe ~]# chmod -Rf 777 /nfsfile
	[root@linuxprobe ~]# echo "welcome to linuxprobe.com" > /nfsfile/readme

第3步：NFS服务程序的配置文件为/etc/exports，默认情况下里面没有任何内容。我们可以按照“共享目录的路径 允许访问的NFS客户端（共享权限参数）”的格式，定义要共享的目录与相应的权限。

用于配置NFS服务程序配置文件的参数：


| 参数 | 作用 |
| --- | --- |
| ro | 只读 |
| rw | 读写 |
| root_squash | 当NFS客户端以root管理员访问时，映射为NFS服务器的匿名用户 |
| no_root_squash | 当NFS客户端以root管理员访问时，映射为NFS服务器的root管理员 |
| all_squash | 无论NFS客户端使用什么账户访问，均映射为NFS服务器的匿名用户 |
| sync | 同时将数据写入到内存与硬盘中，保证不丢失数据 |
| async | 优先将数据保存到内存，然后再写入硬盘；这样效率更高，但可能会丢失数据 |

请注意，NFS客户端地址与权限之间没有空格。

	[root@linuxprobe ~]# vim /etc/exports
	/nfsfile 192.168.10.*(rw,sync,root_squash)

第4步：启动和启用NFS服务程序。由于在使用NFS服务进行文件共享之前，需要使用RPC（Remote Procedure Call，远程过程调用）服务将NFS服务器的IP地址和端口号等信息发送给客户端。因此，在启动NFS服务之前，还需要顺带重启并启用rpcbind服务程序，并将这两个服务一并加入开机启动项中。

	[root@linuxprobe ~]# systemctl restart rpcbind
	[root@linuxprobe ~]# systemctl enable rpcbind
	[root@linuxprobe ~]# systemctl start nfs-server
	[root@linuxprobe ~]# systemctl enable nfs-server

NFS客户端的配置步骤也十分简单。先使用showmount命令（以及必要的参数，见表12-8）查询NFS服务器的远程共享信息，其输出格式为“共享的目录名称 允许使用客户端地址”。

showmount命令中可用的参数以及作用：


| 参数 | 作用 |
| --- | --- |
| -e | 显示NFS服务器的共享列表 |
| -a | 显示本机挂载的文件资源的情况NFS资源的情况 |
| -v | 显示版本号 |

	[root@linuxprobe ~]# showmount -e 192.168.10.10
	Export list for 192.168.10.10:
	/nfsfile 192.168.10.*

然后在NFS客户端创建一个挂载目录。使用mount命令并结合-t参数，指定要挂载的文件系统的类型，并在命令后面写上服务器的IP地址、服务器上的共享目录以及要挂载到本地系统（即客户端）的目录。

	[root@linuxprobe ~]# mkdir /nfsfile
	[root@linuxprobe ~]# mount -t nfs 192.168.10.10:/nfsfile /nfsfile

如果希望NFS文件共享服务能一直有效，则需要将其写入到fstab文件中：

	[root@linuxprobe ~]# cat /nfsfile/readme
		welcome to linuxprobe.com
	[root@linuxprobe ~]# vim /etc/fstab
		……
		/dev/cdrom /media/cdrom iso9660 defaults 0 0 
		192.168.10.10:/nfsfile /nfsfile nfs defaults 0 0

# 3.AutoFs自动挂载服务

	[root@linuxprobe ~]# yum install autofs

在autofs服务程序的主配置文件中需要按照“挂载目录 **子配置文件**”的格式进行填写。挂载目录是设备挂载位置的上一级目录。例如，光盘设备一般挂载到/media/cdrom目录中，那么挂载目录写成/media即可。对应的**子配置文件**则是对这个挂载目录内的挂载设备信息作进一步的说明。**子配置文件**需要用户自行定义，文件名字没有严格要求，但后缀建议以.misc结束。

	[root@linuxprobe ~]# vim /etc/auto.master
	#
	# Sample auto.master file
	# This is an automounter map and it has the following format
	# key [ -mount-options-separated-by-comma ] location
	# For details of the format look at autofs(5).
	#
	/media /etc/iso.misc
	/misc /etc/auto.misc
	#
	# NOTE: mounts done from a hosts map will be mounted with the
	# "nosuid" and "nodev" options unless the "suid" and "dev"
	# options are explicitly given.
	……
		+auto.master

在子配置文件中，应按照“挂载目录 挂载文件类型及权限 :设备名称”的格式进行填写，挂载目录写为iso，而-fstype为文件系统格式参数，iso9660为光盘设备格式，ro、nosuid及nodev为光盘设备具体的权限参数，/dev/cdrom则是定义要挂载的设备名称:

	[root@linuxprobe ~]# vim /etc/iso.misc
	iso   -fstype=iso9660,ro,nosuid,nodev :/dev/cdrom
	[root@linuxprobe ~]# systemctl start autofs 
	[root@linuxprobe ~]# systemctl enable autofs 

	[root@linuxprobe ~]# df -h
	Filesystem Size Used Avail Use% Mounted on
	/dev/mapper/rhel-root 18G 3.0G 15G 17% /
	tmpfs 914M 0 914M 0% /sys/fs/cgroup
	/dev/sda1 497M 119M 379M 24% /boot
	[root@linuxprobe ~]# cd /media
	[root@linuxprobe media]# ls
	[root@linuxprobe media]# cd iso
	[root@linuxprobe iso]# ls -l
	total 812
	dr-xr-xr-x. 4 root root 2048 May 7 2017 addons
	dr-xr-xr-x. 3 root root 2048 May 7 2017 EFI
	[root@linuxprobe ~]# df -h
	Filesystem Size Used Avail Use% Mounted on
	/dev/cdrom 3.5G 3.5G 0 100% /media/iso















