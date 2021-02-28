---
title: 'linux就该这么学-19:使用PXE+Kickstart无人值守安装服务'
date: 2021-02-25 22:54:02
tags:
- 第19章节：使用PXE+Kickstart无人值守安装服务
categories:
- linux就该这么学

---

# 1.无人值守系统

无人值守安装系统的工作流程：

<!--more-->

{% asset_img 1.png %}

PXE（Preboot eXecute Environment，预启动执行环境）是由Intel公司开发的技术，可以让计算机通过网络来启动操作系统（前提是计算机上安装的网卡支持PXE技术），主要用于在无人值守安装系统中引导客户端主机安装Linux操作系统。

Kickstart是一种无人值守的安装方式，其工作原理是预先把原本需要运维人员手工填写的参数保存成一个ks.cfg文件，当安装过程中需要填写参数时则自动匹配Kickstart生成的文件。所以只要Kickstart文件包含了安装过程中需要人工填写的所有参数，那么从理论上来讲完全不需要运维人员的干预，就可以自动完成安装工作。

由于当前的客户端主机并没有完整的操作系统，也就不能完成FTP协议的验证了，所以需要使用TFTP协议帮助客户端获取引导及驱动文件。vsftpd服务程序用于将完整的系统安装镜像通过网络传输给客户端。当然，只要能将系统安装镜像成功传输给客户端即可，因此也可以使用httpd来替代vsftpd服务程序。

# 2.部署相关服务程序

## 2.1配置DHCP服务程序

DHCP服务程序用于为客户端主机分配可用的IP地址，而且这是服务器与客户端主机进行文件传输的基础，因此我们先行配置DHCP服务程序。

无人值守系统与客户端的设置：


| 主机名称 | 操作系统 | IP地址 |
| --- | --- | --- |
| 无人值守系统 | RHEL 7 | 192.168.10.10 |
| 客户端 | 未安装操作系统 | - |

{% asset_img 2.png %}

{% asset_img 3.jpg %}

	[root@linuxprobe ~]# yum install dhcp
	Loaded plugins: langpacks, product-id, subscription-manager
	This system is not registered to Red Hat Subscription Management. You can use subscription-manager to register.
	rhel | 4.1 kB 00:00 
	……
	Installed:
	 dhcp.x86_64 12:4.2.5-27.el7 
	Complete!

在这里使用的配置文件与之前的配置文件有两个主要区别：允许了BOOTP引导程序协议，旨在让局域网内暂时没有操作系统的主机也能获取静态IP地址；在配置文件的最下面加载了引导驱动文件pxelinux.0（这个文件会在下面的步骤中创建），其目的是让客户端主机获取到IP地址后主动获取引导驱动文件，自行进入下一步的安装过程。

	[root@linuxprobe ~]# vim /etc/dhcp/dhcpd.conf
	allow booting;
	allow bootp;
	ddns-update-style interim;
	ignore client-updates;
	subnet 192.168.10.0 netmask 255.255.255.0 {
	        option subnet-mask      255.255.255.0;
	        option domain-name-servers  192.168.10.10;
	        range dynamic-bootp 192.168.10.100 192.168.10.200;
	        default-lease-time      21600;
	        max-lease-time          43200;
	        next-server             192.168.10.10;
	        filename                "pxelinux.0";
	}

在确认DHCP服务程序的参数都填写正确后，重新启动该服务程序，并将其添加到开机启动项中。这样在设备下一次重启之后，在无须人工干预的情况下，自动为客户端主机安装系统。

	[root@linuxprobe ~]# systemctl restart dhcpd
	[root@linuxprobe ~]# systemctl enable dhcpd
	ln -s '/usr/lib/systemd/system/dhcpd.service' '/etc/systemd/system/multi-user.target.wants/dhcpd.service'

## 2.2 配置TFTP服务程序

vsftpd是一款功能丰富的文件传输服务程序，允许用户以匿名开放模式、本地用户模式、虚拟用户模式来进行访问认证。但是，当前的客户端主机还没有安装操作系统，该如何进行登录认证呢？而TFTP作为一种基于UDP协议的简单文件传输协议，不需要进行用户认证即可获取到所需的文件资源。因此接下来配置TFTP服务程序，为客户端主机提供引导及驱动文件。当客户端主机有了基本的驱动程序之后，再通过vsftpd服务程序将完整的光盘镜像文件传输过去。

	[root@linuxprobe ~]# yum install tftp-server

TFTP是一种非常精简的文件传输服务程序，它的运行和关闭是由xinetd网络守护进程服务来管理的。xinetd服务程序会同时监听系统的多个端口，然后根据用户请求的端口号调取相应的服务程序来响应用户的请求。需要开启TFTP服务程序，只需在xinetd服务程序的配置文件中把disable参数改成no就可以了。保存配置文件并退出，然后重启xinetd服务程序，并将其加入到开机启动项中。

	[root@linuxprobe ~]# yum install xinetd
	[root@linuxprobe ~.d]# vim /etc/xinetd.d/tftp
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
	[root@linuxprobe xinetd.d]# systemctl restart xinetd
	[root@linuxprobe xinetd.d]# systemctl enable xinetd

TFTP服务程序默认使用的是UDP协议，占用的端口号为69，所以在生产环境中还需要在firewalld防火墙管理工具中写入使其永久生效的允许策略，以便让客户端主机顺利获取到引导文件。

	[root@linuxprobe ~]# firewall-cmd --permanent --add-port=69/udp
	success
	[root@linuxprobe ~]# firewall-cmd --reload 
	success

## 2.3 配置SYSLinux服务程序

SYSLinux是一个用于提供引导加载的服务程序。与其说SYSLinux是一个服务程序，不如说更需要里面的引导文件，在安装好SYSLinux服务程序软件包后，/usr/share/syslinux目录中会出现很多引导文件。

	[root@linuxprobe ~]# yum install syslinux

我们首先需要把SYSLinux提供的引导文件复制到TFTP服务程序的默认目录中，也就是前文提到的文件pxelinux.0，这样客户端主机就能够顺利地获取到引导文件了。另外在RHEL 7系统光盘镜像中也有一些我们需要调取的引导文件。确认光盘镜像已经被挂载到/media/cdrom目录后，使用复制命令将光盘镜像中自带的一些引导文件也复制到TFTP服务程序的默认目录中。

	[root@linuxprobe ~]# cd /var/lib/tftpboot
	[root@linuxprobe tftpboot]# cp /usr/share/syslinux/pxelinux.0 .
	[root@linuxprobe tftpboot]# cp /media/cdrom/images/pxeboot/{vmlinuz,initrd.img} .
	[root@linuxprobe tftpboot]# cp /media/cdrom/isolinux/{vesamenu.c32,boot.msg} .

然后在TFTP服务程序的目录中新建pxelinux.cfg目录，虽然该目录的名字带有后缀，但依然也是目录，而非文件！将系统光盘中的开机选项菜单复制到该目录中，并命名为default。

	[root@linuxprobe tftpboot]# mkdir pxelinux.cfg
	[root@linuxprobe tftpboot]# cp /media/cdrom/isolinux/isolinux.cfg pxelinux.cfg/default

默认的开机菜单中有两个选项，要么是安装系统，要么是对安装介质进行检验。既然我们已经确定采用无人值守的方式安装系统，还需要为每台主机手动选择相应的选项，未免与我们的主旨（无人值守安装）相悖。现在我们编辑这个default文件，把第1行的default参数修改为linux，这样系统在开机时就会默认执行那个名称为linux的选项了。对应的linux选项大约在64行，我们将默认的光盘镜像安装方式修改成FTP文件传输方式，并指定好光盘镜像的获取网址以及Kickstart应答文件的获取路径：

	[root@linuxprobe tftpboot]# vim pxelinux.cfg/default
	 1 default linux
	 2 timeout 600
	 3
	 4 display boot.msg
	 5
	 61 label linux
	 62 menu label ^Install Red Hat Enterprise Linux 7.0
	 63 kernel vmlinuz
	 64 append initrd=initrd.img inst.stage2=ftp://192.168.10.10 ks=ftp://192.168.10.10/pub/ks.cfg quiet
	 65
	………………省略部分输出信息………………

## 2.4 配置VSftpd服务程序

在我们这套无人值守安装系统的服务中，光盘镜像是通过FTP协议传输的，因此势必要用到vsftpd服务程序。当然，也可以使用httpd服务程序来提供Web网站访问的方式，只要能确保将光盘镜像顺利传输给客户端主机即可。如果打算使用Web网站服务来提供光盘镜像，一定记得将上面配置文件中的光盘镜像获取网址和Kickstart应答文件获取网址修改一下。

	[root@linuxprobe ~]# yum install vsftpd

在配置文件修改正确之后，一定将相应的服务程序添加到开机启动项中，这样无论是在生产环境中还是在红帽认证考试中，都可以在设备重启之后依然能提供相应的服务。

	[root@linuxprobe ~]# systemctl restart vsftpd
	[root@linuxprobe ~]# systemctl enable vsftpd
	ln -s '/usr/lib/systemd/system/vsftpd.service' '/etc/systemd/system/multi-user.target.wants/vsftpd.service'

在确认系统光盘镜像已经正常挂载到/media/cdrom目录后，把目录中的光盘镜像文件全部复制到vsftpd服务程序的工作目录中。

	[root@linuxprobe ~]# cp -r /media/cdrom/* /var/ftp

这个过程大约需要3～5分钟。在此期间，我们也别闲着，在firewalld防火墙管理工具中写入使FTP协议永久生效的允许策略，然后在SELinux中放行FTP传输：

	[root@linuxprobe ~]# firewall-cmd --permanent --add-service=ftp
	success
	[root@linuxprobe ~]# firewall-cmd --reload 
	success
	[root@linuxprobe ~]# setsebool -P ftpd_connect_all_unreserved=on

## 2.5 创建KickStart应答文件

Kickstart应答文件中包含了系统安装过程中需要使用的选项和参数信息，系统可以自动调取这个应答文件的内容，从而彻底实现了无人值守安装系统。那么，既然这个文件如此重要，该去哪里找呢？其实在root管理员的家目录中有一个名为anaconda-ks.cfg的文件，它就是应答文件。下面将这个文件复制到vsftpd服务程序的工作目录中（在开机选项菜单的配置文件中已经定义了该文件的获取路径，也就是vsftpd服务程序数据目录中的pub子目录中）。使用chmod命令设置该文件的权限，确保所有人都有可读的权限，以保证客户端主机可以顺利获取到应答文件及里面的内容：

	[root@linuxprobe ~]# cp ~/anaconda-ks.cfg /var/ftp/pub/ks.cfg
	[root@linuxprobe ~]# chmod +r /var/ftp/pub/ks.cfg

Kickstart应答文件并没有想象中的那么复杂，它总共只有46行左右的参数和注释内容，大家完全可以通过参数的名称及介绍来快速了解每个参数的作用。首先把第6行的光盘镜像安装方式修改成FTP协议，仔细填写好FTP服务器的IP地址，并用本地浏览器尝试打开下检查有没有报错。然后把第21行的时区修改成上海(Asia/Shanghai)，最后再把29行的磁盘选项设置为清空所有磁盘内容并初始化磁盘：

	[root@linuxprobe ~]# vim /var/ftp/pub/ks.cfg 
	 1 #version=RHEL7
	 2 # System authorization information
	 3 auth --enableshadow --passalgo=sha512
	 4 
	 5 # Use CDROM installation media
	 6 url --url=ftp://192.168.10.10
	 7 # Run the Setup Agent on first boot
	 20 # System timezone
	 21 timezone Asia/Shanghai --isUtc
	 22 user --name=linuxprobe --password=$6$a9v3InSTNbweIR7D$JegfYWbCdoOokj9sodEccdO.zL F4oSH2AZ2ss2R05B6Lz2A0v2K.RjwsBALL2FeKQVgf640oa/tok6J.7GUtO/ --iscrypted --gecos ="linuxprobe"
	 23 # X Window System configuration information
	 24 xconfig --startxonboot
	 25 # System bootloader configuration
	 26 bootloader --location=mbr --boot-drive=sda
	 27 autopart --type=lvm
	 28 # Partition clearing information
	 29 clearpart --all --initlabel
	 30 
	 31 %packages

如果觉得系统默认自带的应答文件参数较少，不能满足生产环境的需求，则可以通过Yum软件仓库来安装system-config-kickstart软件包。这是一款图形化的Kickstart应答文件生成工具，可以根据自己的需求生成自定义的应答文件，然后将生成的文件放到/var/ftp/pub目录中并将名字修改为ks.cfg即可。

# 3. 自动部署客户机

在按照上文讲解的方法成功部署各个相关的服务程序后，就可以使用PXE + Kickstart无人值守安装系统了。在采用下面的步骤建立虚拟主机时，一定要把客户端的网卡模式设定成与服务端一致的“仅主机模式”，否则两台设备无法进行通信，也就更别提自动安装系统了。其余硬件配置选项并没有强制性要求，大家可参考这里的配置选项来设定。

第1步：打开“新建虚拟机向导”程序，选择“典型（推荐） ”配置类型，然后单击“下一步”按钮，如图19-5所示。

{% asset_img 4.png %}

第2步：将虚拟机操作系统的安装来源设置为“稍后安装操作系统”。这样做的目的是让虚拟机真正从网络中获取系统安装镜像，同时也可避免VMware Workstation虚拟机软件按照内设的方法自行安装系统。单击“下一步”按钮，如图19-6所示。

{% asset_img 5.png %}

第3步：将“客户机操作系统”设置为“Red Hat Enterprise Linux 7 64位”，然后单击“下一步”按钮:

{% asset_img 6.png %}

第4步：对虚拟机进行命名并设置安装位置。大家可自行定义虚拟机的名称，而安装位置则尽量选择磁盘空间较大的分区。然后单击“下一步”按钮:

{% asset_img 7.png %}

第5步：指定磁盘容量。这里将“最大磁盘大小”设置为20GB，指的是虚拟机系统能够使用的最大上限，而不是会被立即占满，因此设置得稍微大一些也没有关系。然后单击“下一步”按钮。

{% asset_img 8.png %}

第6步：结束“新建虚拟机向导程序”后，先不要着急打开虚拟机系统。大家还需要单击图19-10中的“自定义硬件”按钮，在弹出的如下图所示的界面中，把“网络适配器”设备同样也设置为“仅主机模式”（这个步骤非常重要），然后单击“确定”按钮。

{% asset_img 9.png %}

{% asset_img 10.png %}

现在，我们就同时准备好了PXE + Kickstart无人值守安装系统与虚拟主机。在生产环境中，大家只需要将配置妥当的服务器上架，接通服务器和客户端主机之间的网线，然后启动客户端主机即可。接下来就会按照下图那样，开始传输光盘镜像文件并进行自动安装了—期间完全无须人工干预，直到安装完毕时才需要运维人员进行简单的初始化工作。

{% asset_img 11.png %}

{% asset_img 12.png %}





