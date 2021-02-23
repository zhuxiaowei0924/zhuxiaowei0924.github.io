---
title: linux就该这么学-17：使用ISCSI服务部署网络存储
date: 2021-02-20 21:13:51
tags:
- 第17章节：使用ISCSI服务部署网络存储
categories:
- linux就该这么学

---

# 1.ISCSI技术介绍

为了进一步提升硬盘存储设备的读写速度和性能，人们一直在努力改进物理硬盘设备的接口协议。当前的硬盘接口类型主要有IDE、SCSI和SATA这3种。

<!--more-->

	IDE是一种成熟稳定、价格便宜的并行传输接口。
	
	SATA是一种传输速度更快、数据校验更完整的串行传输接口。
	
	SCSI是一种用于计算机和硬盘、光驱等设备之间系统级接口的通用标准，具有系统资源占用率低、转速高、传输速度快等优点。

iSCSI：Internet Small Computer System Interface，互联网小型计算机系统接口！

这是一种将SCSI接口与以太网技术相结合的新型存储技术，可以用来在网络中传输SCSI接口的命令和数据。这样，不仅克服了传统SCSI接口设备的物理局限性，实现了跨区域的存储资源共享，还可以在不停机的状态下扩展存储容量。

# 2.创建RAID磁盘阵列

第1步：首先在虚拟机中添加4块新硬盘，用于创建RAID 5磁盘阵列和备份盘：

{% asset_img 1.jpg %}

第2步：启动虚拟机系统，使用mdadm命令创建RAID磁盘阵列。其中，-Cv参数为创建阵列并显示过程，/dev/md0为生成的阵列组名称，-n 3参数为创建RAID 5磁盘阵列所需的硬盘个数，-l 5参数为RAID磁盘阵列的级别，-x 1参数为磁盘阵列的备份盘个数。在命令后面要逐一写上使用的硬盘名称。

	[root@linuxprobe ~]# mdadm -Cv /dev/md0 -n 3 -l 5 -x 1 /dev/sdb /dev/sdc /dev/sdd /dev/sde
	mdadm: layout defaults to left-symmetric
	mdadm: layout defaults to left-symmetric
	mdadm: chunk size defaults to 512K
	mdadm: size set to 20954624K
	mdadm: Defaulting to version 1.2 metadata
	mdadm: array /dev/md0 started.

在上述命令成功执行之后，得到一块名称为/dev/md0的新设备，大家可使用mdadm -D命令来查看设备的详细信息。另外，由于在使用远程设备时极有可能出现设备识别顺序发生变化的情况，因此，如果直接在fstab挂载配置文件中写入/dev/sdb、/dev/sdc等设备名称的话，就有可能在下一次挂载了错误的存储设备。而UUID值是设备的唯一标识符，可以用于精确地区分本地或远程设备。于是我们可以把这个值记录下来，一会儿准备填写到挂载配置文件中。

	[root@linuxprobe ~]# mdadm -D /dev/md0
	/dev/md0:
	        Version : 1.2
	…………
	**UUID : 3370f643:c10efd6a:44e91f2a:20c71f3e**
	         Events : 26
	    Number   Major   Minor   RaidDevice State
	       0       8       16        0      active sync   /dev/sdb
	       1       8       32        1      active sync   /dev/sdc
	       4       8       48        2      active sync   /dev/sdd
	       3       8       64        -      spare   /dev/sde

# 3.配置iSCSI服务端

iSCSI技术在工作形式上分为服务端（target）与客户端（initiator）。iSCSI服务端即用于存放硬盘存储资源的服务器，它作为前面创建的RAID磁盘阵列的存储端，能够为用户提供可用的存储资源。iSCSI客户端则是用户使用的软件，用于访问远程服务端的存储资源。

 iSCSI服务端和客户端的操作系统以及IP地址：


| 主机名称 | 操作系统 | IP地址 |
| --- | --- | --- |
| iSCSI服务端 | RHEL 7 | 192.168.10.10 |
| iSCSI客户端 | RHEL 7 | 192.168.10.20 |

第1步：配置好Yum软件仓库后安装iSCSI服务端程序以及配置命令工具。通过在yum命令的后面添加-y参数，在安装过程中就不需要再进行手动确认了：

	[root@linuxprobe ~]# yum -y install targetd targetcli
	Loaded plugins: langpacks, product-id, subscription-manager
	………………省略部分输出信息………………
	 python-setproctitle.x86_64 0:1.1.6-5.el7 
	 python-urwid.x86_64 0:1.1.1-3.el7 
	Complete!

安装完成后启动iSCSI的服务端程序targetd，然后把这个服务程序加入到开机启动项中，以便下次在服务器重启后依然能够为用户提供iSCSI共享存储资源服务：

	[root@linuxprobe ~]# systemctl start targetd
	[root@linuxprobe ~]# systemctl enable targetd
	 ln -s '/usr/lib/systemd/system/targetd.service' '/etc/systemd/system/multi-user.target.wants/targetd.service'

第2步：配置iSCSI服务端共享资源。在执行targetcli命令后就能看到交互式的配置界面了。在该界面中可以使用很多Linux命令，比如利用ls查看目录参数的结构，使用cd切换到不同的目录中。/backstores/block是iSCSI服务端配置共享设备的位置。我们需要把刚刚创建的RAID 5磁盘阵列md0文件加入到配置共享设备的“资源池”中，并将该文件重新命名为disk0，这样用户就不会知道是由服务器中的哪块硬盘来提供共享存储资源，而只会看到一个名为disk0的存储设备。

	[root@linuxprobe ~]# targetcli
	Warning: Could not load preferences file /root/.targetcli/prefs.bin.
	targetcli shell version 2.1.fb34
	Copyright 2011-2013 by Datera, Inc and others.
	For help on commands, type 'help'.
	/> ls
	o- / ..................................................................... [...]
	o- backstores .......................................................... [...]
	| o- block .............................................. [Storage Objects: 0]
	| o- fileio ............................................. [Storage Objects: 0]
	| o- pscsi .............................................. [Storage Objects: 0]
	| o- ramdisk ............................................ [Storage Objects: 0]
	o- iscsi ........................................................ [Targets: 0]
	o- loopback ..................................................... [Targets: 0
	/> cd /backstores/block
	/backstores/block> create disk0 /dev/md0
	Created block storage object disk0 using /dev/md0.
	/backstores/block> cd /
	/> ls
	o- / ..................................................................... [...]
	  o- backstores .......................................................... [...]
	  | o- block .............................................. [Storage Objects: 1]
	  | | o- disk0 ..................... [/dev/md0 (40.0GiB) write-thru deactivated]
	  | o- fileio ............................................. [Storage Objects: 0]
	  | o- pscsi .............................................. [Storage Objects: 0]
	  | o- ramdisk ............................................ [Storage Objects: 0]
	  o- iscsi ........................................................ [Targets: 0]
	  o- loopback ..................................................... [Targets: 0]

第3步：创建iSCSI target名称及配置共享资源。iSCSI target名称是由系统自动生成的，这是一串用于描述共享资源的唯一字符串。稍后用户在扫描iSCSI服务端时即可看到这个字符串，因此我们不需要记住它。系统在生成这个target名称后，还会在/iscsi参数目录中创建一个与其字符串同名的新“目录”用来存放共享资源。我们需要把前面加入到iSCSI共享资源池中的硬盘设备添加到这个新目录中，这样用户在登录iSCSI服务端后，即可默认使用这硬盘设备提供的共享存储资源了。
	
	/> cd iscsi
	/iscsi> 
	/iscsi> create
	Created target iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80.
	Created TPG 1.
	/iscsi> cd iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80/
	/iscsi/iqn.20....d497c356ad80> ls
	o- iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80 ...... [TPGs: 1]
	  o- tpg1 ............................................... [no-gen-acls, no-auth]
	    o- acls .......................................................... [ACLs: 0]
	    o- luns .......................................................... [LUNs: 0]
	    o- portals .................................................... [Portals: 0]
	/iscsi/iqn.20....d497c356ad80> cd tpg1/luns
	/iscsi/iqn.20...d80/tpg1/luns> create /backstores/block/disk0 
	Created LUN 0.
	
第4步：设置访问控制列表（ACL）。iSCSI协议是通过客户端名称进行验证的，也就是说，用户在访问存储共享资源时不需要输入密码，只要iSCSI客户端的名称与服务端中设置的访问控制列表中某一名称条目一致即可，因此需要在iSCSI服务端的配置文件中写入一串能够验证用户信息的名称。acls参数目录用于存放能够访问iSCSI服务端共享存储资源的客户端名称。刘遄老师推荐在刚刚系统生成的iSCSI target后面追加上类似于:client的参数，这样既能保证客户端的名称具有唯一性，又非常便于管理和阅读：

	/iscsi/iqn.20...d80/tpg1/luns> cd ..
	/iscsi/iqn.20...c356ad80/tpg1> cd acls 
	/iscsi/iqn.20...d80/tpg1/acls> create iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80:client
	Created Node ACL for iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80:client
	Created mapped LUN 0

第5步：设置iSCSI服务端的监听IP地址和端口号。即在portals参数目录中写上服务器的IP地址。接下来将由系统自动开启服务器192.168.10.10的3260端口将向外提供iSCSI共享存储资源服务：

	/iscsi/iqn.20...d80/tpg1/acls> cd ..
	/iscsi/iqn.20...c356ad80/tpg1> cd portals 
	/iscsi/iqn.20.../tpg1/portals> create 192.168.10.10
	Using default IP port 3260
	Created network portal 192.168.10.10:3260.

第6步：配置妥当后检查配置信息，重启iSCSI服务端程序并配置防火墙策略。在参数文件配置妥当后，可以浏览刚刚配置的信息，确保与下面的信息基本一致。在确认信息无误后输入exit命令来退出配置。注意，千万不要习惯性地按Ctrl + C组合键结束进程，这样不会保存配置文件，我们的工作也就白费了。最后重启iSCSI服务端程序，再设置firewalld防火墙策略，使其放行3260/tcp端口号的流量。

	/iscsi/iqn.20.../tpg1/portals> ls /
	o- / ........................... [...]
	  o- backstores................. [...]
	  | o- block ................... [Storage Objects: 1]
	  | | o- disk0 ................. [/dev/md0 (40.0GiB) write-thru activated]
	  | o- fileio .................. [Storage Objects: 0]
	  | o- pscsi ................... [Storage Objects: 0]
	  | o- ramdisk ................. [Storage Objects: 0]
	  o- iscsi ..................... [Targets: 1]
	  | o- iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80 .... [TPGs: 1]
	  |   o- tpg1 .................. [no-gen-acls, no-auth]
	  |     o- acls ........................................................ [ACLs: 1]
	  |     | o- iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80:client [Mapped LUNs: 1]
	  |     |   o- mapped_lun0 ............................................. [lun0 block/disk0 (rw)]  
	    o- luns .................... [LUNs: 1]
	  |     | o- lun0 .............. [block/disk0 (/dev/md0)]
	  |     o- portals ............. [Portals: 1]
	  |       o- 192.168.10.10:3260  [OK]
	  o- loopback .................. [Targets: 0]
	/> exit
	Global pref auto_save_on_exit=true
	Last 10 configs saved in /etc/target/backup.
	Configuration saved to /etc/target/saveconfig.json
	[root@linuxprobe ~]# systemctl restart targetd
	[root@linuxprobe ~]# firewall-cmd --permanent --add-port=3260/tcp 
	success 
	[root@linuxprobe ~]# firewall-cmd --reload 
	success

但在配置这部分时，遇到一个问题：在targetcli界面输入“exit”时，并没有显示配置已保存，反而提示有报错：'Implict and Explict' is not in list；网上搜索查询得知是由于系统内核版本（kernel-3.10.0-123.el7.x86_64）太低导致，需要升级kernel内核：在升级内核时，也比较麻烦，一台虚拟机系统在终端中输入命令：yum update kernel，即成功升级版本（kernel 3.10.0-1160.15.2.el7.x86_64）；而另一台虚拟机则使用此种方式不成功，查询资料，采用另一种方式：

1.直接运行yum update kernel，显示报错error:kernel conflicts kmod等等……；

2.输入：yum clean all，可能有些效果，但无法彻底解决；

3.上面报错：kernel conflicts kmod，可能是由于系统中安装有多个kmod版本导致；

4.输入：rpm -qa kmod，查询系统中安装的kmod数量，然后使用 rpm -e kmod-version，卸载低版本，之后重新输入；yum update kernel，升级内核！

5.升级完成后，输入：awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg ，查看下内核的默认启动顺序；

6.默认启动的顺序是从0开始，新内核是从头插入，所以需要选择0，输入命令：grub2-set-default 0

7.重启系统，开机后输入：uname -r，查看系统版本号！

# 4.配置linux客户端

第1步：在RHEL 7系统中，已经默认安装了iSCSI客户端服务程序initiator。如果您的系统没有安装的话，可以使用Yum软件仓库手动安装。

	[root@linuxprobe ~]# yum install iscsi-initiator-utils 
	Loaded plugins: langpacks, product-id, subscription-manager 
	Package iscsi-initiator-utils-6.2.0.873-21.el7.x86_64 already installed and latest version 
	Nothing to do

第2步：编辑iSCSI客户端中的initiator名称文件，把服务端的访问控制列表名称填写进来，然后重启客户端iscsid服务程序并将其加入到开机启动项中：

	[root@linuxprobe ~]# vim /etc/iscsi/initiatorname.iscsi
	InitiatorName=iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80:client
	[root@linuxprobe ~]# systemctl restart iscsid
	[root@linuxprobe ~]# systemctl enable iscsid
	 ln -s '/usr/lib/systemd/system/iscsid.service' '/etc/systemd/system/multi-user.target.wants/iscsid.service'

第3步：iSCSI客户端访问并使用共享存储资源的步骤很简单，只需要记住刘遄老师的一个小口诀“先发现，再登录，最后挂载并使用”。iscsiadm是用于管理、查询、插入、更新或删除iSCSI数据库配置文件的命令行工具，用户需要先使用这个工具扫描发现远程iSCSI服务端，然后查看找到的服务端上有哪些可用的共享存储资源。其中，-m discovery参数的目的是扫描并发现可用的存储资源，-t st参数为执行扫描操作的类型，-p 192.168.10.10参数为iSCSI服务端的IP地址：

	[root@linuxprobe ~]# iscsiadm -m discovery -t st -p 192.168.10.10
	192.168.10.10:3260,1 iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80

第4步：在使用iscsiadm命令发现了远程服务器上可用的存储资源后，接下来准备登录iSCSI服务端。其中，-m node参数为将客户端所在主机作为一台节点服务器，-T  iqn.2003-01. org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80参数为要使用的存储资源（大家可以直接复制前面命令中扫描发现的结果，以免录入错误），-p 192.168.10.10参数依然为对方iSCSI服务端的IP地址。最后使用--login或-l参数进行登录验证。

	[root@linuxprobe ~]# iscsiadm -m node -T iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80 -p 192.168.10.10 --login
	Logging in to [iface: default, target: iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80, portal: 192.168.10.10,3260] (multiple)
	Login to [iface: default, target: iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80, portal: 192.168.10.10,3260] successful.

第5步：在iSCSI客户端成功登录之后，会在客户端主机上多出一块名为/dev/sdb的设备文件。

	[root@linuxprobe ~]# file /dev/sdb 
	/dev/sdb: block special

第6步：格式化、挂载硬盘；

	[root@linuxprobe ~]# mkfs.xfs /dev/sdb
	[root@linuxprobe ~]# mkdir /iscsi
	[root@linuxprobe ~]# mount /dev/sdb /iscsi
	[root@linuxprobe ~]# df -h

需要提醒大家的是，由于udev服务是按照系统识别硬盘设备的顺序来命名硬盘设备的，当客户端主机同时使用多个远程存储资源时，如果下一次识别远程设备的顺序发生了变化，则客户端挂载目录中的文件也将随之混乱。为了防止发生这样的问题，我们应该在/etc/fstab配置文件中使用设备的UUID唯一标识符进行挂载，这样，不论远程设备资源的识别顺序再怎么变化，系统也能正确找到设备所对应的目录。

blkid命令用于查看设备的名称、文件系统及UUID。可以使用管道符（详见第3章）进行过滤，只显示与/dev/sdb设备相关的信息：

	[root@linuxprobe ~]# blkid | grep /dev/sdb
	/dev/sdb: UUID="eb9cbf2f-fce8-413a-b770-8b0f243e8ad6" TYPE="xfs" 

由于/dev/sdb是一块网络存储设备，而iSCSI协议是基于TCP/IP网络传输数据的，因此必须在/etc/fstab配置文件中添加上_netdev参数，表示当系统联网后再进行挂载操作，以免系统开机时间过长或开机失败：

	[root@linuxprobe ~]# vim /etc/fstab
	/dev/mapper/rhel-swap swap swap defaults 0 0
	/dev/cdrom /media/cdrom iso9660 defaults 0 0 
	**UUID=eb9cbf2f-fce8-413a-b770-8b0f243e8ad6 /iscsi xfs defaults,_netdev 0 0**

如果我们不再需要使用iSCSI共享设备资源了，可以用iscsiadm命令的-u参数将其设备卸载：

	[root@linuxprobe ~]# iscsiadm -m node -T iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80 -u
	
	Logging out of session [sid: 7, target : iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80, portal: 192.168.10.10,3260]
	
	Logout of [sid: 7, target: iqn.2003-01.org.linux-iscsi.linuxprobe.x8664:sn.d497c356ad80,portal:192.168.10.10,3260] successful.

# 5.配置windows客户端

使用Windows系统的客户端也可以正常访问iSCSI服务器上的共享存储资源，而且操作原理及步骤与Linux系统的客户端基本相同。在进行下面的实验之前，请先关闭Linux系统客户端，以免这两台客户端主机同时使用iSCSI共享存储资源而产生潜在问题。

iSCSI服务器和客户端的操作系统以及IP地址：


| 主机名称 | 操作系统 | IP地址 |
| --- | --- | --- |
| iSCSI服务端 | RHEL 7 | 192.168.10.10 |
| windows客户端 | windows 7 | 192.168.10.30 |

第1步：运行iSCSI发起程序。在Windows 7/10操作系统中已经默认安装了iSCSI客户端程序，我们只需在控制面板中找到“系统和安全”标签，然后单击“管理工具”，进入到“管理工具”页面后即可看到“iSCSI发起程序”图标。双击该图标。在第一次运行iSCSI发起程序时，系统会提示“Microsoft iSCSI服务端未运行”，单击“是”按钮即可自动启动并运行iSCSI发起程序：

{% asset_img 2.png %}

{% asset_img 3.png %}

第2步：扫描发现iSCSI服务端上可用的存储资源。不论是Windows系统还是Linux系统，要想使用iSCSI共享存储资源都必须先进行扫描发现操作。运行iSCSI发起程序后在“目标”选项卡的“目标”文本框中写入iSCSI服务端的IP地址，然后单击“快速连接”按钮:

{% asset_img 4.png %}

在弹出的“快速连接”提示框中可看到共享的硬盘存储资源，单击“完成”按钮即可:

{% asset_img 5.png %}

回到“目标”选项卡页面，可以看到共享存储资源的名称已经出现.

第3步：准备连接iSCSI服务端的共享存储资源。由于在iSCSI服务端程序上设置了ACL，使得只有客户端名称与ACL策略中的名称保持一致时才能使用远程存储资源，因此需要在“配置”选项卡中单击“更改”按钮，把iSCSI发起程序的名称修改为服务端:

{% asset_img 6.png %}

{% asset_img 7.png %}

在确认客户端发起程序的名称修改正确后即可返回到“目标”选项卡页面中，然后单击“连接”按钮进行连接请求，成功连接到远程共享存储资源的页面:

{% asset_img 8.png %}

第4步：访问iSCSI远程共享存储资源。右键单击桌面上的“计算机”图标，打开计算机管理程序:

{% asset_img 9.png %}

开始对磁盘进行初始化操作:

{% asset_img 10.png %}

{% asset_img 11.png %}

{% asset_img 12.png %}

{% asset_img 13.png %}

{% asset_img 14.png %}

{% asset_img 15.png %}

{% asset_img 16.png %}

{% asset_img 17.png %}
