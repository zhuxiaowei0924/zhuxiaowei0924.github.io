---
title: linux就该这么学：使用ssh服务管理远程主机
date: 2021-02-04 22:00:33
tags:
- 第9章节：使用ssh服务管理远程主机
categories:
- linux就该这么学

---

# 1.配置网卡服务

介绍了SSH协议与sshd服务程序的理论知识、Linux系统的远程管理方法以及在系统中配置服务程序的方法，并采用实验的形式演示了使用基于密码验证的sshd服务程序进行远程登录，以及使用screen服务程序远程管理Linux系统的不间断会话等技术。

<!--more-->

# 1.1配置网卡参数

{% asset_img 1.png %}

{% asset_img 2.png %}

{% asset_img 3.png %}

{% asset_img 4.png %}

{% asset_img 5.png %}

{% asset_img 6.png %}

{% asset_img 7.png %}

{% asset_img 8.png %}

用Vim编辑器将网卡配置文件中的ONBOOT参数修改成yes，这样在系统重启后网卡就被激活了:

	[root@linuxprobe ~]# vim /etc/sysconfig/network-scripts/ifcfg-eno16777736
	TYPE=Ethernet
	BOOTPROTO=none
	...
	ONBOOT=yes

修改完Linux系统中的服务配置文件后,要手动重启相应的服务:

	[root@linuxprobe ~]# systemctl restart network

## 1.2创建网络会话

NetworkManager：是一种动态管理网络配置的守护进程，能够让网络设备保持连接状态；可以使用nmcli命令来管理Network Manager服务。

可以使用nmcli命令并按照“connection add con-name type ifname”的格式来创建网络会话。

使用con-name参数指定使用的网络会话名称company，然后依次用ifname参数指定网卡名称，用autoconnect no参数设置该网络会话默认不被自动激活，以及用ip4及gw4参数手动指定网络的IP地址：

	[root@linuxprobe ~]# nmcli connection add con-name company ifname eno16777736 autoconnect no type ethernet ip4 192.168.10.10/24 gw4 192.168.10.1

使用con-name参数指定网络会话名称house，使用外部DHCP服务器自动获得IP地址，不需要进行手动指定：

	[root@linuxprobe ~]# nmcli connection add con-name house type ethernet ifname eno16777736

使配置过的网络会话生效：

	[root@linuxprobe ~]# nmcli connection up house 

## 1.3绑定两块网卡

{% asset_img 9.png %}

{% asset_img 10.png %}

使用Vim文本编辑器来配置网卡设备的绑定参数：
	
	[root@linuxprobe ~]# vim /etc/sysconfig/network-scripts/ifcfg-eno16777736
	TYPE=Ethernet
	BOOTPROTO=none
	ONBOOT=yes
	USERCTL=no
	DEVICE=eno16777736
	MASTER=bond0
	SLAVE=yes
	[root@linuxprobe ~]# vim /etc/sysconfig/network-scripts/ifcfg-eno33554968
	TYPE=Ethernet
	BOOTPROTO=none
	ONBOOT=yes
	USERCTL=no
	DEVICE=eno33554968
	MASTER=bond0
	SLAVE=yes

将绑定后的设备命名为bond0并把IP地址等信息填写进去，这样当用户访问相应服务的时候，实际上就是由这两块网卡设备在共同提供服务：

	[root@linuxprobe ~]# vim /etc/sysconfig/network-scripts/ifcfg-bond0
	TYPE=Ethernet
	BOOTPROTO=none
	ONBOOT=yes
	USERCTL=no
	DEVICE=bond0
	IPADDR=192.168.10.10
	PREFIX=24
	DNS=192.168.10.1
	NM_CONTROLLED=no

让Linux内核支持网卡绑定驱动。常见的网卡绑定驱动有三种模式—mode0、mode1和mode6。下面以绑定两块网卡为例，讲解使用的情景。

	mode0（平衡负载模式）：平时两块网卡均工作，且自动备援，但需要在与服务器本地网卡相连的交换机设备上进行端口聚合来支持绑定技术。

	mode1（自动备援模式）：平时只有一块网卡工作，在它故障后自动替换为另外的网卡。

	mode6（平衡负载模式）：平时两块网卡均工作，且自动备援，无须交换机设备提供辅助支持。

使用Vim文本编辑器创建一个用于网卡绑定的驱动文件，使得绑定后的bond0网卡设备能够支持绑定技术（bonding）；同时定义网卡以mode6模式进行绑定，且出现故障时自动切换的时间为100毫秒：

	[root@linuxprobe ~]# vim /etc/modprobe.d/bond.conf
	alias bond0 bonding
	options bond0 miimon=100 mode=6

重启网络服务后网卡绑定操作即可成功。正常情况下只有bond0网卡设备才会有IP地址等信息：

	[root@linuxprobe ~]# systemctl restart network

# 2.远程控制服务

## 2.1配置sshd服务

想要使用SSH协议来远程管理Linux系统，则需要部署配置sshd服务程序。sshd是基于SSH协议开发的一款远程管理服务程序，不仅使用起来方便快捷，而且能够提供两种安全验证的方法：

	基于口令的验证—用账户和密码来验证登录；

	基于密钥的验证—需要在本地生成密钥对，然后把密钥对中的公钥上传至服务器，并与服务器中的公钥进行比较；该方式相较来说更安全。

sshd服务的配置信息保存在/etc/ssh/sshd_config文件中；

| 参数 | 作用|
| --- | --- |
| Port 22 | 默认的sshd服务端口 |
| ListenAddress 0.0.0.0 | 设定sshd服务器监听的IP地址 |
| Protocol 2 | SSH协议的版本号 |
| HostKey /tc/ssh/ssh_host_key | SSH协议版本为1时，DES私钥存放的位置 |
| HostKey /etc/ssh/ssh_host_rsa_key | SSH协议版本为2时，RSA私钥存放的位置 |
| HostKey /etc/ssh/ssh_host_dsa_key | SSH协议版本为2时，DSA私钥存放的位置 |
| PermitRootLogin yes | 设定是否允许root管理员直接登录 |
| StrictModes yes | 当远程用户的私钥改变时直接拒绝连接 |
| MaxAuthTries 6 | 最大密码尝试次数 |
| MaxSessions 10 | 最大终端数 |
| PasswordAuthentication yes | 是否允许密码验证 |
| PermitEmptyPasswords no | 是否允许空密码登录（很不安全）|

	[root@linuxprobe ~]# ssh 192.168.10.10
	The authenticity of host '192.168.10.10 (192.168.10.10)' can't be established.
	ECDSA key fingerprint is 4f:a7:91:9e:8d:6f:b9:48:02:32:61:95:48:ed:1e:3f.
	Are you sure you want to continue connecting (yes/no)? yes
	Warning: Permanently added '192.168.10.10' (ECDSA) to the list of known hosts.
	root@192.168.10.10's password:此处输入远程主机root管理员的密码
	Last login: Wed Apr 15 15:54:21 2017 from 192.168.10.10
	[root@linuxprobe ~]# 
	[root@linuxprobe ~]# exit
	logout
	Connection to 192.168.10.10 closed.

禁止以root管理员的身份远程登录到服务器：

	[root@linuxprobe ~]# vim /etc/ssh/sshd_config 
 	………………省略部分输出信息………………
	 46 
 	47 #LoginGraceTime 2m
 	**48 PermitRootLogin no**
 	49 #StrictModes yes
 	50 #MaxAuthTries 6
 	51 #MaxSessions 10
 	52
 	………………省略部分输出信息………………

让新配置文件生效，则需要手动重启相应的服务程序；最好也将这个服务程序加入到开机启动项中：

	[root@linuxprobe ~]# systemctl restart sshd
	[root@linuxprobe ~]# systemctl enable sshd

## 2.2安全密钥验证

第1步：在客户端主机中生成“密钥对”。

	[root@linuxprobe ~]# ssh-keygen
	Generating public/private rsa key pair.
	Enter file in which to save the key (/root/.ssh/id_rsa):按回车键或设置密钥的存储路径
	Created directory '/root/.ssh'.
	Enter passphrase (empty for no passphrase):直接按回车键或设置密钥的密码
	Enter same passphrase again:再次按回车键或设置密钥的密码
	Your identification has been saved in /root/.ssh/id_rsa.
	Your public key has been saved in /root/.ssh/id_rsa.pub.
	Your identification has been saved in /root/.ssh/id_rsa.
	Your public key has been saved in /root/.ssh/id_rsa.pub.
	The key fingerprint is:
	40:32:48:18:e4:ac:c0:c3:c1:ba:7c:6c:3a:a8:b5:22 root@linuxprobe.com
	The key's randomart image is:
		+--[ RSA 2048]----+
		|+*..o .          |
		|*.o  +           |
		|o*    .          |
		|+ .    .         |
		|o..     S        |
		|.. +             |
		|. =              |
		|E+ .             |
		|+.o              |
		+-----------------+

第2步：把客户端主机中生成的公钥文件传送至远程主机：

	[root@linuxprobe ~]# ssh-copy-id 192.168.10.10
	The authenticity of host '192.168.10.20 (192.168.10.10)' can't be established.
	ECDSA key fingerprint is 4f:a7:91:9e:8d:6f:b9:48:02:32:61:95:48:ed:1e:3f.
	Are you sure you want to continue connecting (yes/no)? yes
	/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
	/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
	root@192.168.10.10's password:此处输入远程服务器密码
	Number of key(s) added: 1
	Now try logging into the machine, with: "ssh '192.168.10.10'"
	and check to make sure that only the key(s) you wanted were added.

第3步：对服务器进行设置，使其只允许密钥验证，拒绝传统的口令验证方式。记得在修改配置文件后保存并重启sshd服务程序。

	[root@linuxprobe ~]# vim /etc/ssh/sshd_config 
	 ………………省略部分输出信息………………
	 74 
	 75 # To disable tunneled clear text passwords, change to no here!
	 76 #PasswordAuthentication yes
	 77 #PermitEmptyPasswords no
	 **78 PasswordAuthentication no**
	 79 
	 ………………省略部分输出信息………………
	[root@linuxprobe ~]# systemctl restart sshd

第4步：在客户端尝试登录到服务器，此时无须输入密码也可成功登录。

	[root@linuxprobe ~]# ssh 192.168.10.10
	Last login: Mon Apr 13 19:34:13 2017

## 2.3远程传输命令

scp（secure copy）是一个基于SSH协议在网络之间进行安全传输的命令，其格式为“scp [参数] 本地文件 远程帐户@远程IP地址:远程目录；


| 参数 | 作用 |
| --- | --- |
| -v | 显示详细的连接进度 |
| -p | 指定远程主机的sshd端口号 |
| -r | 用于传送文件夹 |
| -6 | 使用IPv6协议 |

在使用scp命令把文件从本地复制到远程主机时，首先需要以绝对路径的形式写清本地文件的存放位置：

	[root@linuxprobe ~]# echo "Welcome to LinuxProbe.Com" > readme.txt
	[root@linuxprobe ~]# scp /root/readme.txt 192.168.10.20:/home
	root@192.168.10.20's password:此处输入远程服务器中root管理员的密码
	readme.txt 100% 26 0.0KB/s 00:00

使用scp命令把远程主机上的文件下载到本地主机，其命令格式为“scp [参数] 远程用户@远程IP地址:远程文件 本地目录”:

	[root@linuxprobe ~]# scp 192.168.10.20:/etc/redhat-release /root
	root@192.168.10.20's password:此处输入远程服务器中root管理员的密码
	redhat-release 100% 52 0.1KB/s 00:00 
	[root@linuxprobe ~]# cat redhat-release 
	Red Hat Enterprise Linux Server release 7.0 (Maipo)

# 3.不间断会话服务

screen是一款能够实现多窗口远程控制的开源服务程序，简单来说就是为了解决网络异常中断或为了同时控制多个远程终端窗口而设计的程序。用户还可以使用screen服务程序同时在多个远程会话中自由切换，能够做到实现如下功能。

	会话恢复：即便网络中断，也可让会话随时恢复，确保用户不会失去对远程会话的控制。
	
	多窗口：每个会话都是独立运行的，拥有各自独立的输入输出终端窗口，终端窗口内显示过的信息也将被分开隔离保存，以便下次使用时依然能看到之前的操作记录。
	
	会话共享：当多个用户同时登录到远程服务器时，便可以使用会话共享功能让用户之间的输入输出信息共享。

## 3.1管理远程会话

screen命令：


| 参数 | 作用 |
| --- | --- |
| -S | 创建会话窗口 |
| -d | 将指定会话进行离线处理 |
| -r | 恢复指定会话 |
| -x | 一次性恢复所有会话 |
| -ls | 显示当前已有会话 |
| -wipe | 将目前无法使用的会话删除 |

 














