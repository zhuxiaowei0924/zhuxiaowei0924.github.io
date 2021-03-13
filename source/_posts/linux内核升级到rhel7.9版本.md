---
title: linux内核升级到rhel7.9版本
date: 2021-03-08 22:06:37
tags:
- linux内核升级
categories:
- linux系统
---
当前的系统版本为rhel7.0，内核版本为“Linux 3.10.0-123.el7.x86_64”

<!--more-->

	[root@rhel7_0 ~]# hostnamectl
		……
		Operating System: Red Hat Enterprise Linux Server 7.0 (Maipo)
		Kernel: Linux 3.10.0-123.el7.x86_64
		……
	[root@rhel0 ~]# uname -rs
		Linux 3.10.0-123.el7.x86_64

1、从网上下载RHEL7.9版本的系统镜像文件，然后在 RHEL7.0 服务器上进行挂载，并配置在本地YUM源：

	[root@rhel7_0 ~]# mount /dev/cdrom /mnt/
	[root@rhel7_0 ~]# cat /etc/yum.repos.d/rhel_local.repo 
		[mnt]
		name=mnt
		baseurl=file:///mnt
		enable=1
		gpgcheck=0

2、在RHEL7.0服务器上先执行yum clean all，再使用yum list kernel命令查看当前服务器上已安装的内核以及可更新的高版本内核。

	[root@rhel7_0 ~]# yum list kernel
	Loaded plugins: product-id, search-disabled-repos, subscription-manager
	This system is not registered to Red Hat Subscription Management. You can use subscription-manager to register.
	Installed Packages
	kernel.x86_64                                                    3.10.0-123.el7                                                    @anaconda/7.2
	Available Packages
	kernel.x86_64                                                    3.10.0-1160.el7                                                    mnt          

3、安装新内核.

	[root@rhel7_0 ~]# yum install -y kernel   
	[root@rhel7_0 ~]# uname -rs
　　	Linux 3.10.0-123.el7.x86_64

安装完成之后，再查看内核，发现没有变化，还是跟之前的一样。

查看/boot/grub2/grub.cfg 配置文件，可以看到新内核3.10.0-1160.el7.x86_64已经是排在最前边了，RHEL7 可以不用调整内核启动顺序，默认会从新内核启动。

这是进行系统重启reboot，系统会从新内核3.10.0-1160.el7.x86_64启动，但在登录界面显示一片黑屏或其他报错，因此在系统升级完后，不要着急重启，需要在终端输入：**yum update**进行更新，之后再重启，一切正常！

正常情况下，是不需要修改内核启动顺序的，如果发现内核启动顺序不对，可以按下面的方式来修改：

	[root@rhel7_0 ~]# grub2-editenv list  --查看系统默认内核版本
	saved_entry=Red Hat Enterprise Linux Server (3.10.0-1160.el7.x86_64) 7.9 (Maipo)
	[root@rhel7_0 ~]# grep "menuentry " /boot/grub2/grub.cfg  --查看配置文件里的内核顺序；这个菜单条目对应上图中的三条选项。
	menuentry 'Red Hat Enterprise Linux Server (3.10.0-1160.el7.x86_64) 7.9 (Maipo)' --class red --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-123.el7.x86_64-advanced-e2de1cc9-e059-4306-a1a5-2389e3f83c70' {
	menuentry 'Red Hat Enterprise Linux Server (3.10.0-123.el7.x86_64) 7.0 (Maipo)' --class red --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-123.el7.x86_64-advanced-e2de1cc9-e059-4306-a1a5-2389e3f83c70' {
	menuentry 'Red Hat Enterprise Linux Server (0-rescue-1fbd678500124daea255a3b7a98e320c) 7.0 (Maipo)' --class red --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-0-rescue-1fbd678500124daea255a3b7a98e320c-advanced-e2de1cc9-e059-4306-a1a5-2389e3f83c70' {
	[root@rhel7_0 ~]# grub2-set-default 'Red Hat Enterprise Linux Server (3.10.0-1160.el7.x86_64) 7.9 (Maipo)'   --配置默认内核
	[root@rhel7_0 ~]# grub2-mkconfig -o /boot/grub2/grub.cfg    --将修改的内容写入到grub.cfg配置文件。
	Generating grub configuration file ...
	Found linux image: /boot/vmlinuz-3.10.0-1160.el7.x86_64
	Found initrd image: /boot/initramfs-3.10.0-1160.el7.x86_64.img
	Found linux image: /boot/vmlinuz-3.10.0-123.el7.x86_64
	Found initrd image: /boot/initramfs-3.10.0-123.el7.x86_64.img
	Found linux image: /boot/vmlinuz-0-rescue-1fbd678500124daea255a3b7a98e320c
	Found initrd image: /boot/initramfs-0-rescue-1fbd678500124daea255a3b7a98e320c.img
	done

系统内核更新成功！




