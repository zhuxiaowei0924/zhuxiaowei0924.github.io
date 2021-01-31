---
title: linux就该这么学：用户身份与文件权限
date: 2021-01-25 21:17:51
tags:
- 第5章节：用户身份与文件权限
categories:
- linux就该这么学
---
# 1.用户身份与能力

管理员UID为0：系统的管理员用户。

系统用户UID为1～999： Linux系统为了避免因某个服务程序出现漏洞而被黑客提权至整台服务器，默认服务程序会有独立的系统用户负责运行，进而有效控制被破坏范围。

普通用户UID从1000开始：是由管理员创建的用于日常工作的用户。

<!--more-->
一个用户只有一个基本用户组，但是可以有多个扩展用户组，从而满足日常的工作需要。

（1）useradd：用于创建新的用户；默认的用户家目录会被存放在/home目录中，默认的Shell解释器为/bin/bash，而且默认会创建一个与该用户同名的基本用户组。


| 参数 | 作用 |
| --- | --- |
| -d |	指定用户的家目录 |
| -u |	指定该用户的默认UID |
| -g |	指定一个初始的用户基本组（必须已存在） |
| -G |	指定一个或多个扩展用户组 |
| -s |	指定该用户的默认Shell解释器 |

/sbin/nologin，它是终端解释器中的一员，与Bash解释器有着天壤之别。一旦用户的解释器被设置为nologin，则代表该用户不能登录到系统中：

（2）groupadd:用于创建用户组；

（3）usermod：用于修改用户属性；用户的信息保存在/etc/passwd文件中，可以直接用文本编辑器来修改其中的用户参数项目，也可以用usermod命令修改已经创建的用户信息，诸如用户的UID、基本/扩展用户组、默认终端等。  


| 参数 | 作用 |
| --- | --- |
| -d -m |	参数-m与参数-d连用，可重新指定用户的家目录并自动把旧的数据转移过去 |
| -g |	变更所属用户组 |
| -G |	变更扩展用户组 |
| -L |	锁定用户禁止其登录系统 |
| -U |	解锁用户，允许其登陆系统 |
| -s |	变更默认终端 |

（4）passwd：用于修改用户密码、过期时间、认证信息等；


| 参数 | 作用 |
| --- | --- |
| -l |	锁定用户禁止其登录系统 |
| -u |	解锁用户，允许其登陆系统 |
| --stdin |	允许通过标准输入修改用户密码，如echo "NewPassWord" | passwd --stdin Username |
| -d |	使该用户可用空密码登录系统 |
| -e |	强制用户在下次登录时修改密码 |
| -S |	显示用户的密码是否被锁定，以及密码所采用的加密算法名称 |

（5）userdel：用于删除用户；


| 参数 | 作用 |
| --- | --- |
| -f |	强制删除用户 |
| -r |	同时删除用户及用户家目录 |

# 2.文件权限与归属

 -：普通文件；d：目录文件；l：链接文件；b：块设备文件；c：字符设备文件；p：管道文件

对于一般文件来说，权限比较容易理解：“可读”表示能够读取文件的实际内容；“可写”表示能够编辑、新增、修改、删除文件的实际内容；“可执行”则表示能够运行一个脚本程序。

对目录文件来说，“可读”表示能够读取目录内的文件列表；“可写”表示能够在目录内新增、删除、重命名文件；而“可执行”则表示能够进入该目录。

## 2.1 文件的特殊权限

详细内容可参考：https://www.cnblogs.com/sparkdev/p/9651622.html；

（1）SUID：让执行者临时获得命令的所有者的权限；

（2）SGID：目录内新文件所有组，继承原有目录所有组的名称！

1.SUID
在 Linux 中，所有账号的密码记录在 /etc/shadow 这个文件中，并且只有 root 可以读写入这个文件：

{% asset_p1.jpg %}

如果另一个普通账号 tester 需要修改自己的密码，就要访问 /etc/shadow 这个文件。但是明明只有 root 才能访问 /etc/shadow 这个文件，这究竟是如何做到的呢？事实上，tester 用户是可以修改 /etc/shadow 这个文件内的密码的，就是通过 SUID 的功能。让我们看看 passwd 程序文件的权限信息：

{% asset_p2.jpg %}

上图红框中的权限信息有些奇怪，owner 的信息为 rws 而不是 rwx。当 s 出现在文件拥有者的 x 权限上时，就被称为 SETUID BITS 或 SETUID ，其特点如下：

-SUID 权限仅对二进制可执行文件有效
-如果执行者对于该二进制可执行文件具有 x 的权限，执行者将具有该文件的所有者的权限
-本权限仅在执行该二进制可执行文件的过程中有效

下面我们来看 tester 用户是如何利用 SUID 权限完成密码修改的：
	(1)tester 用户对于 /usr/bin/passwd 这个程序具有执行权限，因此可以执行 passwd 程序
	(2)passwd 程序的所有者为 root
	(3)tester 用户执行 passwd 程序的过程中会暂时获得 root 权限
	(4)因此 tester 用户在执行 passwd 程序的过程中可以修改 /etc/shadow 文件

但是如果由 tester 用户执行 cat 命令去读取 /etc/shadow 文件确是不行的：

{% asset_p3.jpg %}

原因很清楚，tester 用户没有读 /etc/shadow 文件的权限，同时 cat 程序也没有被设置 SUID。我们可以通过下图来理解这两种情况：

{% asset_p4.jpg %}

如果想让任意用户通过 cat 命令读取 /etc/shadow 文件的内容也是非常容易的，给它设置 SUID 权限就可以了：

$ sudo chmod 4755 /bin/cat

{% asset_p5.jpg %}

现在 cat 已经具有了 SUID 权限，试试看，是不是已经可以 cat 到 /etc/shadow 的内容了。因为这样做非常不安全，所以赶快通过下面的命令把 cat 的 SUID 权限移除掉：

$ sudo chmod 755 /bin/cat

2.SGID

当 s 标志出现在用户组的 x 权限时称为 SGID。SGID 的特点与 SUID 相同，我们通过 /usr/bin/mlocate 程序来演示其用法。mlocate 程序通过查询数据库文件 /var/lib/mlocate/mlocate.db 实现快速的文件查找。 mlocate 程序的权限如下图所示：

{% asset_p6.jpg %}

很明显，它被设置了 SGID 权限。下面是数据库文件 /var/lib/mlocate/mlocate.db 的权限信息：很明显，它被设置了 SGID 权限。下面是数据库文件 /var/lib/mlocate/mlocate.db 的权限信息：

{% asset_p7.jpg %}

普通用户 tester 执行 mlocate 命令时，tester 就会获得用户组 mlocate 的执行权限，又由于用户组 mlocate 对 mlocate.db 具有读权限，所以 tester 就可以读取 mlocate.db 了。程序的执行过程如下图所示：

{% asset_p8.jpg %}

除二进制程序外，SGID 也可以用在目录上。当一个目录设置了 SGID 权限后，它具有如下功能：

	(1)用户若对此目录具有 r 和 x 权限，该用户能够进入该目录
	(2)用户在此目录下的有效用户组将变成该目录的用户组
	(3)若用户在此目录下拥有 w 权限，则用户所创建的新文件的用户组与该目录的用户组相同

下面看个例子，创建 testdir 目录，目录的权限设置如下：

{% asset_p9.jpg %}

此时目录 testdir 的 owner 是 nick，所属的 group 为 tester。
先创建一个名为 nickfile 的文件：

{% asset_p10.jpg %}

这个文件的权限看起来没有什么特别的。然后给 testdir 目录设置 SGID 权限：

$ sudo chmod 2775 testdir

{% asset_p11.jpg %}

然后再创建一个文件 nickfile2：

{% asset_p12.jpg %}

新建的文件所属的组为 tester！

总结一下，当 SGID 作用于普通文件时，和 SUID 类似，在执行该文件时，用户将获得该文件所属组的权限。当 SGID 作用于目录时，意义就非常重大了。当用户对某一目录有写和执行权限时，该用户就可以在该目录下建立文件，如果该目录用 SGID 修饰，则该用户在这个目录下建立的文件都是属于这个目录所属的组。

SBIT
其实 SBIT 与 SUID 和 SGID 的关系并不大。
SBIT 是 the  restricted  deletion  flag  or  sticky  bit 的简称。
SBIT 目前只对目录有效，用来阻止非文件的所有者删除文件。比较常见的例子就是 /tmp 目录：

{% asset_p13.jpg %}

权限信息中最后一位 t 表明该目录被设置了 SBIT 权限。SBIT 对目录的作用是：当用户在该目录下创建新文件或目录时，仅有自己和 root 才有权力删除。

设置 SUID、SGID、SBIT 权限
以数字的方式设置权限
SUID、SGID、SBIT 权限对应的数字如下：

SUID->4
SGID->2
SBIT->1
所以如果要为一个文件权限为 "-rwxr-xr-x" 的文件设置 SUID 权限，需要在原先的 755 前面加上 4，也就是 4755：

$ chmod 4755 filename

同样，可以用 2 和 1 来设置 SGID 和 SBIT 权限。设置完成后分别会用 s, s, t 代替文件权限中的 x。

其实，还可能出现 S 和 T 的情况。S 和 t 是替代 x 这个权限的，但是，如果它本身没有 x 这个权限，添加 SUID、SGID、SBIT 权限后就会显示为大写 S 或大写 T。比如我们为一个权限为 666 的文件添加 SUID、SGID、SBIT 权限：

{% asset_p14.jpg %}

执行 chmod 7666 nickfile，因为 666 表示 "-rw-rw-rw"，均没有 x 权限，所以最后变成了 "-rwSrwSrwT"。

通过符号类型改变权限

除了使用数字来修改权限，还可以使用符号：

$ chmod u+s testfile # 为 testfile 文件加上 SUID 权限。
$ chmod g+s testdir  # 为 testdir 目录加上 SGID 权限。
$ chmod o+t testdir  # 为 testdir 目录加上 SBIT 权限。

总结
SUID、SGID、SBIT 权限都是为了实现特殊功能而设计的，其目的是弥补 ugo 权限无法实现的一些使用场景。









