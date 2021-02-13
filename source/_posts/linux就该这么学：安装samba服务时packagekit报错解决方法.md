---
title: linux就该这么学：安装samba服务时packagekit报错解决方法
date: 2021-02-12 10:28:09
tags:
- 安装samba服务时packagekit报错解决方法
categories:
- linux就该这么学
- 
---

在linux虚拟机Redhat7.0版本系统终端中输入命令：yum install samba，安装samba服务时，在安装过程中，出现以下报错信息：

	Error: initscripts conflicts with redhat-release-server-7.0-1.el7.x86_64
	 You could try using --skip-broken to work around the problem
	Found 2 pre-existing rpmdb problem(s), 'yum check' output follows:
	PackageKit-0.8.9-11.el7.x86_64 has missing requires of PackageKit-backend
	rhn-check-2.0.2-5.el7.noarch has missing requires of yum-rhn-plugin >= ('0', '1.6.4', '1')

有2个problem，1个error；先解决problem：

	（1）在http://rpm.pbone.net/网站中下载PackageKit-yum-0.8.9-11.el7.x86_64.rpm,然后将rpm文件拖入linux系统中，输入命令：rpm -Uvh PackageKit-yum-0.8.9-11.el7.x86_64.rpm，解决第1个problem。
	（2）在终端中输入命令：yum install yum-rhn-plugin，，解决第2个problem。

再解决initscripts conflicts的error：

	首先，先查一下本机的initscripts的版本，有没有更适合自己系统的，输入以下命令：yum list initscripts，

	显示以下信息：
	
	Loading mirror speeds from cached hostfile
	Installed Packages
	initscripts.x86_64                9.49.17-1.el7                @anaconda/7.0
	Available Packages
	initscripts.x86_64                9.49.53-1.el7_9.1                updates  
	
	说明当前版本是initscripts.x86_64  9.49.17-1.el7   @anaconda/7.0  

	可以用更适合系统的initscripts.x86_64   9.49.53-1.el7_9.1版本来替换。

	再输入命令：yum list centos-release，看看合适的centos-release版本：
	
	Loading mirror speeds from cached hostfile
	Available Packages
	centos-release.x86_64               7-9.2009.1.el7.centos                updates

	使用以上命令可以知道，最合适的版本是centos-release.x86_64  7-9.2009.1.el7.centos

	从http://rpm.pbone.net/网站中分别下载initscripts-9.49.53-1.el7.x86_64.rpm和centos-release-7-9.2009.1.el7.centos.x86_64.rpm，然后输入命令：rpm -e redhat-release-server-7.0-1.el7.x86_64 --nodeps，将有冲突的版本删掉，删除redhat-release以后，不重启机器，马上安装centos-release.x86_64：

	[root@linuxprobe Desktop]# rpm -Uvh centos-release-7-9.2009.1.el7.centos.x86_64.rpm
	Preparing...                          ################################# [100%]
	Updating / installing...
   	1:centos-release-7-9.2009.1.el7.cenwarning: /etc/yum.repos.d/CentOS-Base.repo created as /etc/yum.repos.d/CentOS-Base.repo.rpmnew
	################################# [100%]
	error: unpacking of archive failed on file /usr/share/doc/redhat-release: cpio: rename failed - Is a directory
	error: centos-release-7-9.2009.1.el7.centos.x86_64: install failed

安装时出错！提示“/usr/share/doc/redhat-release 失败：cpio: rename 失败 - 是一个目录”

推测：安装centos-release时，这个redhat-release目录碍事了，所以我们手动把它删除掉（注意备份）。

	[root@linuxprobe doc]# rm -r /usr/share/doc/redhat-release 
	rm：是否删除目录 "/usr/share/doc/redhat-release"？y
	
	如果还是报错，则再在/usr/share目录中删除redhat-release：
	
	[root@linuxprobe share]# rm -r /usr/share/redhat-release 
	rm：是否删除目录 "/usr/share/redhat-release"？y

	然后再终端中输入命令：rpm -Uvh centos-release-7-9.2009.1.el7.centos.x86_64.rpm：

	[root@linuxprobe Desktop]# rpm -Uvh centos-release-7-9.2009.1.el7.centos.x86_64.rpm
	Preparing...                          ################################# [100%]
	Updating / installing...
  	1:centos-release-7-9.2009.1.el7.cenwarning: /etc/yum.repos.d/CentOS-Base.repo created as /etc/yum.repos.d/CentOS-Base.repo.rpmnew
	################################# [100%]

到这一步，基本安装samba服务过程中的problem和error都已经解决，但此时如果输入yum install samba命令安装samba服务时，

可能会引起SELinux的报警，因此需要将SELinux关闭；在/etc/selinux/config配置文件中查看SELinux的默认状态。如果是

enforcing，建议赶紧修改为permissive或disabled；SELinux服务的主配置文件中，定义的是SELinux的默认运行状态，可以将其

理解为系统重启后的状态，因此它不会在更改后立即生效。可以使用getenforce命令获得当前SELinux服务的运行模式：

	[root@linuxprobe ~]# getenforce 
	Enforcing

再用setenforce [0|1]命令修改SELinux当前的运行模式（0为禁用，1为启用）。注意，这种修改只是临时的，在系统重启后就会失效：

	[root@linuxprobe ~]# setenforce 0
	[root@linuxprobe ~]# getenforce
	Permissive

之后在终端中输入：yum install samba，安装samba服务，成功！

具体方法可参考以下链接：

http://www.likecs.com/default/index/show?id=20904
https://blog.csdn.net/zz24_com/article/details/105127746

