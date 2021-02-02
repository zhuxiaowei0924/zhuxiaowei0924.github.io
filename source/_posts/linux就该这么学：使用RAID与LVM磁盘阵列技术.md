---
title: linux就该这么学：使用RAID与LVM磁盘阵列技术
date: 2021-02-02 21:17:08
tags:
- 第7章节：使用RAID与LVM磁盘阵列技术
categories:
- linux就该这么学

---

RAID：Redundant Array of Independent Disks，独立冗余磁盘阵列；
LVM：Logical Volume Manager，逻辑卷管理器；

<!--more-->

# 1.RAID 0

{% asset_img 1.png %}

# 2.RAID 1

{% asset_img 2.jpg %}

# 3.RAID 5

{% asset_img 3.png %}

# 4.RAID 10

{% asset_img 4.png %}

mdadm命令用于管理Linux系统中的软件RAID硬盘阵列，格式为“mdadm [模式] <RAID设备名称> [选项] [成员设备名称]”.


| 参数 | 作用 |
| --- | --- |
| -a | 检测设备名称 |
| -n | 指定设备数量 |
| -l | 指定RAID级别 |
| -C | 创建 |
| -v | 显示过程 |
| -D | 查看详细信息 |
| -r | 移除设备 |
| -S | 停止RAID磁盘阵列 |

-C参数代表创建一个RAID阵列卡；-v参数显示创建的过程，同时在后面追加一个设备名称/dev/md0，这样/dev/md0就是创建后的RAID磁盘阵列的名称；-a yes参数代表自动创建设备文件；-n 4参数代表使用4块硬盘来部署这个RAID磁盘阵列；而-l 10参数则代表RAID 10方案；最后再加上4块硬盘设备的名称就搞定了。

	[root@linuxprobe ~]# mdadm -Cv /dev/md0 -a yes -n 4 -l 10 /dev/sdb /dev/sdc /dev/sdd /dev/sde

之后格式化mkfs.ext4，创建挂载点mkdir /RAID，查看详细信息mdadm -D /dev/md0，并把挂载信息写入配置文件。

# 5.磁盘阵列+备份盘

在下面的命令中，参数-n 3代表创建这个RAID 5磁盘阵列所需的硬盘数，参数-l 5代表RAID的级别，而参数-x 1则代表有一块备份盘。当查看/dev/md0（即RAID 5磁盘阵列的名称）磁盘阵列的时候就能看到有一块备份盘在等待中了：

	mdadm -Cv /dev/md0 -n 3 -l 5 -x 1 /dev/sdb /dev/sdc /dev/sdd /dev/sde

# 6.部署逻辑卷

{% asset_img 5.jpg %}

	[root@linuxprobe ~]# pvcreate /dev/sdb /dev/sdc
	[root@linuxprobe ~]# vgcreate storage /dev/sdb /dev/sdc
	[root@linuxprobe ~]# lvcreate -n vo -l 37 storage（在对逻辑卷进行切割时有两种计量单位。第一种是以容量为单位，所使用的参数为-L。例如，使用-L 150M生成一个大小为150MB的逻辑卷。另外一种是以基本单元的个数为单位，所使用的参数为-l。每个基本单元的大小默认为4MB。例如，使用-l 37可以生成一个大小为37×4MB=148MB的逻辑卷。）

之后再格式化，挂载，写入配置文件中即可！

# 7.扩容逻辑卷

扩展前请一定要记得卸载设备和挂载点的关联。

	[root@linuxprobe ~]# lvextend -L 290M /dev/storage/vo （逻辑卷vo扩展至290MB）
	[root@linuxprobe ~]# e2fsck -f /dev/storage/vo （检查硬盘完整性）
	[root@linuxprobe ~]# resize2fs /dev/storage/vo （重置硬盘容量）

之后mount -a挂载硬盘设备即可！

# 8.缩小逻辑卷

扩展前请一定要记得卸载设备和挂载点的关联。

	[root@linuxprobe ~]# e2fsck -f /dev/storage/vo
	[root@linuxprobe ~]# resize2fs /dev/storage/vo 120M
	[root@linuxprobe ~]# lvreduce -L 120M /dev/storage/vo

之后mount -a挂载硬盘设备即可！

# 9.逻辑卷快照

LVM的快照卷功能有两个特点：

	快照卷的容量必须等同于逻辑卷的容量；

	快照卷仅一次有效，一旦执行还原操作后则会被立即自动删除。

	[root@linuxprobe ~]#  lvcreate -L 120M -s -n SNAP /dev/storage/vo （使用-s参数生成一个快照卷，使用-L参数指定切割的大小。另外，还需要在命令后面写上是针对哪个逻辑卷执行的快照操作。）
	[root@linuxprobe ~]# umount /linuxprobe （记得先卸载掉逻辑卷设备与目录的挂载。）
	[root@linuxprobe ~]# lvconvert --merge /dev/storage/SNAP
	[root@linuxprobe ~]# mount -a

# 10.删除逻辑卷

	[root@linuxprobe ~]# umount /linuxprobe （取消逻辑卷与目录的挂载关联）
	[root@linuxprobe ~]# vim /etc/fstab （删除配置文件中永久生效的设备参数）
	[root@linuxprobe ~]# lvremove /dev/storage/vo （删除逻辑卷设备，需要输入y来确认操作）
	[root@linuxprobe ~]# vgremove storage （删除卷组，此处只写卷组名称即可，不需要设备的绝对路径）
	[root@linuxprobe ~]# pvremove /dev/sdb /dev/sdc （删除物理卷设备）








