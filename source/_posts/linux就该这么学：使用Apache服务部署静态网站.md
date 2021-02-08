---
title: linux就该这么学：使用Apache服务部署静态网站
date: 2021-02-07 22:29:13
tags:
- 第10章节：使用Apache服务部署静态网站
categories:
- linux就该这么学
---

# 1.网站服务程序

<!--more-->

第1步：把光盘设备中的系统镜像挂载到/media/cdrom目录。

	[root@linuxprobe ~]# mkdir -p /media/cdrom
	[root@linuxprobe ~]# mount /dev/cdrom /media/cdrom
	mount: /dev/sr0 is write-protected, mounting read-only

第2步：使用Vim文本编辑器创建Yum仓库的配置文件。

	[root@linuxprobe ~]# vim /etc/yum.repos.d/rhel7.repo
	[rhel7]
	name=rhel7
	baseurl=file:///media/cdrom
	enabled=1
	gpgcheck=0

第3步：动手安装Apache服务程序。注意，使用yum命令进行安装时，跟在命令后面的Apache服务的软件包名称为httpd。如果直接执行yum install apache命令，则系统会报错。

	[root@linuxprobe ~]# yum install httpd
	Loaded plugins: langpacks, product-id, subscription-manager
	………………省略部分输出信息………………
	Complete!

第4步：启用httpd服务程序并将其加入到开机启动项中，使其能够随系统开机而运行，从而持续为用户提供Web服务：

	[root@linuxprobe ~]# systemctl start httpd
	[root@linuxprobe ~]# systemctl enable httpd
	ln -s '/usr/lib/systemd/system/httpd.service' '/etc/systemd/system/multi-user.target.wants/httpd.service'

	[root@linuxprobe ~]# firefox

# 2.配置服务文件参数


| 作用 | 文件名称 |
| --- | --- |
| 服务目录 | /etc/httpd |
| 主配置文件 | /etc/httpd/conf/httpd.conf |
| 网站数据目录 | /var/www/html |
| 访问日志 | /var/log/httpd/access_log |
| 错误日志 | /var/log/httpd/error_log |

在httpd服务程序的主配置文件中，存在三种类型的信息：注释行信息、全局配置、区域配置；

 配置httpd服务程序时最常用的参数以及用途描述：


| 参数 | 作用 |
| --- | --- |
| ServerRoot | 服务目录 |
| ServerAdmin | 管理员邮箱 |
| User | 运行服务的用户 |
| Group | 运行服务的用户组 |
| ServerName | 网站服务器的域名 |
| DocumentRoot | 网站数据目录 |
| Listen | 监听的IP地址与端口号 |
| DirectoryIndex | 默认的索引页页面 |
| ErrorLog | 错误日志文件 |
| CustomLog | 访问日志文件 |
| Timeout | 网页超时时间，默认为300秒 |

在默认情况下，网站数据是保存在/var/www/html目录中，打开httpd服务程序的主配置文件，将约第119行用于定义网站数据保存路径的参数DocumentRoot修改为/home/wwwroot，同时还需要将约第124行用于定义目录权限的参数Directory后面的路径也修改为/home/wwwroot。配置文件修改完毕后即可保存并退出。

# 3.SELinux安全子系统

“SELinux域”和“SELinux安全上下文”称为是Linux系统中的双保险

SELinux服务有三种配置模式，具体如下：

	enforcing：强制启用安全策略模式，将拦截服务的不合法请求。
	
	permissive：遇到服务越权访问时，只发出警告而不强制拦截。
	
	disabled：对于越权的行为不警告也不拦截。

SELinux服务主配置文件目录：/etc/selinux/config；

getenforce：获取当前SELinux服务的运行模式； setenforce[0|1]：修改SELinux当前的运行模式（0为禁用，1为启用）；修改是暂时的，系统重启后失效。

查看网站数据保存目录的SELinux安全上下文值：

	[root@linuxprobe ~]# setenforce 1
	[root@linuxprobe ~]# ls -Zd /var/www/html
	drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 /var/www/html
	[root@linuxprobe ~]# ls -Zd /home/wwwroot
	drwxrwxrwx. root root unconfined_u:object_r:home_root_t:s0 /home/wwwroot

在文件上设置的SELinux安全上下文是由用户段、角色段以及类型段等多个信息项共同组成的。其中，用户段system_u代表系统进程的身份，角色段object_r代表文件目录的角色，类型段httpd_sys_content_t代表网站服务的系统文件。

## 3.1 semanage命令

emanage命令用于管理SELinux的策略，格式为“semanage [选项] [文件]”，经常用到的几个参数及其功能如下所示：

		-l参数用于查询；	-a参数用于添加；	-m参数用于修改；	-d参数用于删除。

可以向新的网站数据目录中新添加一条SELinux安全上下文，让这个目录以及里面的所有文件能够被httpd服务程序所访问到：

	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/*
	
注意，执行上述设置之后，还无法立即访问网站，还需要使用restorecon命令将设置好的SELinux安全上下文立即生效。在使用restorecon命令时，可以加上-Rv参数对指定的目录进行递归操作，以及显示SELinux安全上下文的修改过程。最后，再次刷新页面，就可以正常看到网页内容了：

	[root@linuxprobe ~]# restorecon -Rv /home/wwwroot/
	restorecon reset /home/wwwroot context unconfined_u:object_r:home_root_t:s0->unconfined_u:object_r:httpd_sys_content_t:s0

再添加systemctl enable httpd命令，将所需服务添加到开机启动项中。

# 4.个人用户主页功能

第1步：在httpd服务程序中，默认没有开启个人用户主页功能。为此，我们需要编辑下面的配置文件，然后在第17行的UserDir disabled参数前面加上井号（#），表示让httpd服务程序开启个人用户主页功能；同时再把第24行的UserDir public_html参数前面的井号（#）去掉（UserDir参数表示网站数据在用户家目录中的保存目录名称，即public_html目录）。最后，在修改完毕后记得保存。

	 15 # permissions).
	 16 #
	 17 # UserDir disabled
	 18 
	 19 #
	 20 # To enable requests to /~user/ to serve the user's public_html
	 21 # directory, remove the "UserDir disabled" line above, and uncomment
	 22 # the following line instead:
	 23 # 
	 24   UserDir public_html
	 25 </IfModule>
	 …… …… …… ……

第2步：在用户家目录中建立用于保存网站数据的目录及首页面文件。另外，还需要把家目录的权限修改为755，保证其他人也有权限读取里面的内容。

	[root@linuxprobe home]# su - linuxprobe
	Last login: Fri May 22 13:17:37 CST 2017 on :0
	[linuxprobe@linuxprobe ~]$ mkdir public_html
	[linuxprobe@linuxprobe ~]$ echo "This is linuxprobe's website" > public_html/index.html
	[linuxprobe@linuxprobe ~]$ chmod -Rf 755 /home/linuxprobe

第3步：使用getsebool命令查询并过滤出所有与HTTP协议相关的安全策略。其中，off为禁止状态，on为允许状态。

	[root@linuxprobe ~]# getsebool -a | grep http
	httpd_anon_write --> off
	httpd_builtin_scripting --> on
	……
	**httpd_enable_homedirs --> off**
	……

第4步：用setsebool命令来修改SELinux策略中规则的布尔值，在setsebool命令后面加上-P参数，让修改后的SELinux策略规则永久生效且立即生效，并重新启动httpd服务程序：

	[root@linuxprobe ~]# setsebool -P httpd_enable_homedirs=on
	[root@linuxprobe ~]# systemctl restart httpd

第5步：在浏览器的地址栏中输入网址，其格式为“网址/~用户名”（其中的波浪号是必需的，而且网址、波浪号、用户名之间没有空格）如：127.0.0.1/~linuxprobe，就可以看到用户的个人网站了。

## 4.1在网页中添加口令功能

第1步：先使用htpasswd命令生成密码数据库。-c参数表示第一次生成；后面再分别添加密码数据库的存放文件，以及验证要用到的用户名称（该用户不必是系统中已有的本地账户）。

	[root@linuxprobe ~]# htpasswd -c /etc/httpd/passwd linuxprobe
	New password:此处输入用于网页验证的密码
	Re-type new password:再输入一遍进行确认
	Adding password for user linuxprobe

第2步：编辑个人用户主页功能的配置文件。把第31～35行的参数信息修改成下列内容，其中井号（#）开头的内容为刘遄老师添加的注释信息，可将其忽略。随后保存并退出配置文件，重启httpd服务程序即可生效。

	[root@linuxprobe ~]# vim /etc/httpd/conf.d/userdir.conf
	31 <Directory "/home/*/public_html">
	32 AllowOverride all
	#刚刚生成出来的密码验证文件保存路径
	33 authuserfile "/etc/httpd/passwd"
	#当用户尝试访问个人用户网站时的提示信息
	34 authname "My privately website"
	35 authtype basic
	#用户进行账户密码登录时需要验证的用户名称
	36 require user linuxprobe
	37 </Directory>
	[root@linuxprobe ~]# systemctl restart httpd

此后，当用户再想访问某个用户的个人网站时，就必须要输入账户和密码才能正常访问了。另外，验证时使用的账户和密码是用htpasswd命令生成的专门用于网站登录的口令密码，而不是系统中的用户密码，请不要搞错了。

## 4.3虚拟网站主机功能

利用虚拟主机功能，可以把一台处于运行状态的物理服务器分割成多个“虚拟的服务器”。但是，该技术无法实现目前云主机技术的硬件资源隔离，让这些虚拟的服务器共同使用物理服务器的硬件资源，供应商只能限制硬盘的使用空间大小。

Apache的虚拟主机功能是服务器基于用户请求的不同IP地址、主机域名或端口号，实现提供多个网站同时为外部提供访问服务的技术，用户请求的资源不同，最终获取到的网页内容也各不相同。

### 4.3.1基于IP地址

第1步：使用nmtui命令，创建三个IP地址，如：192.168.10.10、192.168.10.20、192.168.10.30；

{% asset_img 1.png %}

第2步：分别在/home/wwwroot中创建用于保存不同网站数据的3个目录，并向其中分别写入网站的首页文件。每个首页文件中应有明确区分不同网站内容的信息，方便我们稍后能更直观地检查效果。

{% asset_img 2.png %}

	[root@linuxprobe ~]# mkdir -p /home/wwwroot/10
	[root@linuxprobe ~]# mkdir -p /home/wwwroot/20
	[root@linuxprobe ~]# mkdir -p /home/wwwroot/30
	[root@linuxprobe ~]# echo "IP:192.168.10.10" > /home/wwwroot/10/index.html
	[root@linuxprobe ~]# echo "IP:192.168.10.20" > /home/wwwroot/20/index.html
	[root@linuxprobe ~]# echo "IP:192.168.10.30" > /home/wwwroot/30/index.html

第3步：在httpd服务的配置文件中大约113行处开始，分别追加写入三个基于IP地址的虚拟主机网站参数，然后保存并退出。记得需要重启httpd服务，这些配置才生效。

	[root@linuxprobe ~]# vim /etc/httpd/conf/httpd.conf
	………………省略部分输出信息………………
	113 <VirtualHost 192.168.10.10>
	114 DocumentRoot /home/wwwroot/10
	115 ServerName www.linuxprobe.com
	116 <Directory /home/wwwroot/10 >
	117 AllowOverride None
	118 Require all granted
	119 </Directory>
	120 </VirtualHost>
	121 <VirtualHost 192.168.10.20>
	122 DocumentRoot /home/wwwroot/20
	123 ServerName bbs.linuxprobe.com
	124 <Directory /home/wwwroot/20 >
	125 AllowOverride None
	126 Require all granted
	127 </Directory>
	128 </VirtualHost>
	129 <VirtualHost 192.168.10.30>
	130 DocumentRoot /home/wwwroot/30
	131 ServerName tech.linuxprobe.com
	132 <Directory /home/wwwroot/30 >
	133 AllowOverride None
	134 Require all granted
	135 </Directory>
	136 </VirtualHost>
	………………省略部分输出信息………………
	[root@linuxprobe ~]# systemctl restart httpd

第3步：手动把新的网站数据目录的SELinux安全上下文设置正确，并使用restorecon命令让新设置的SELinux安全上下文立即生效;

	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/10
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/10/*
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/20
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/20/*
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/30
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/30/*
	[root@linuxprobe ~]# restorecon -Rv /home/wwwroot

{% asset_img 3.png %}

### 4.3.2基于主机域名

当服务器无法为每个网站都分配一个独立IP地址的时候，可以尝试让Apache自动识别用户请求的域名，从而根据不同的域名请求来传输不同的内容。在这种情况下的配置更加简单，只需要保证位于生产环境中的服务器上有一个可用的IP地址（这里以192.168.10.10为例）就可以了。

/etc/hosts是Linux系统中用于强制把某个主机域名解析到指定IP地址的配置文件。

第1步：手工定义IP地址与域名之间对应关系的配置文件，保存并退出后会立即生效。可以通过分别ping这些域名来验证域名是否已经成功解析为IP地址。

	[root@linuxprobe ~]# vim /etc/hosts
	127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
	::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
	192.168.10.10 www.linuxprobe.com bbs.linuxprobe.com tech.linuxprobe.com
	[root@linuxprobe ~]# ping -c 4 www.linuxprobe.com
	PING www.linuxprobe.com (192.168.10.10) 56(84) bytes of data.
	64 bytes from www.linuxprobe.com (192.168.10.10): icmp_seq=1 ttl=64 time=0.070 ms

第2步：分别在/home/wwwroot中创建用于保存不同网站数据的三个目录，并向其中分别写入网站的首页文件。每个首页文件中应有明确区分不同网站内容的信息，方便我们稍后能更直观地检查效果。

	[root@linuxprobe ~]# mkdir -p /home/wwwroot/www
	[root@linuxprobe ~]# mkdir -p /home/wwwroot/bbs
	[root@linuxprobe ~]# mkdir -p /home/wwwroot/tech
	[root@linuxprobe ~]# echo "WWW.linuxprobe.com" > /home/wwwroot/www/index.html
	[root@linuxprobe ~]# echo "BBS.linuxprobe.com" > /home/wwwroot/bbs/index.html
	[root@linuxprobe ~]# echo "TECH.linuxprobe.com" > /home/wwwroot/tech/index.html

第3步：在httpd服务的配置文件中大约113行处开始，分别追加写入三个基于主机名的虚拟主机网站参数，然后保存并退出。记得需要重启httpd服务，这些配置才生效。

	[root@linuxprobe ~]# vim /etc/httpd/conf/httpd.conf
	………………省略部分输出信息………………
	113 <VirtualHost 192.168.10.10>
	114 DocumentRoot "/home/wwwroot/www"
	115 ServerName "www.linuxprobe.com"
	116 <Directory "/home/wwwroot/www">
	117 AllowOverride None
	118 Require all granted
	119 </directory> 
	120 </VirtualHost>
	121 <VirtualHost 192.168.10.10>
	122 DocumentRoot "/home/wwwroot/bbs"
	123 ServerName "bbs.linuxprobe.com"
	124 <Directory "/home/wwwroot/bbs">
	125 AllowOverride None
	126 Require all granted
	127 </Directory>
	128 </VirtualHost>
	129 <VirtualHost 192.168.10.10>
	130 DocumentRoot "/home/wwwroot/tech"
	131 ServerName "tech.linuxprobe.com"
	132 <Directory "/home/wwwroot/tech">
	133 AllowOverride None
	134 Require all granted
	135 </directory>
	136 </VirtualHost>

第4步：因为当前的网站数据目录还是在/home/wwwroot目录中，因此还是必须要正确设置网站数据目录文件的SELinux安全上下文，使其与网站服务功能相吻合。最后记得用restorecon命令让新配置的SELinux安全上下文立即生效，这样就可以立即访问到虚拟主机网站了。

	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/www
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/www/*
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/bbs
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/bbs/*
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/tech
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/tech/*
	[root@linuxprobe ~]# restorecon -Rv /home/wwwroot

{% asset_img 4.png %}

### 4.3.3基于端口号

基于端口号的虚拟主机功能可以让用户通过指定的端口号来访问服务器上的网站资源。一般来说，使用80、443、8080等端口号来提供网站访问服务是比较合理的，如果使用其他端口号则会受到SELinux服务的限制。

在接下来的实验中，我们不但要考虑到目录上应用的SELinux安全上下文的限制，还需要考虑SELinux域对httpd服务程序的管控。

第1步：分别在/home/wwwroot中创建用于保存不同网站数据的两个目录，并向其中分别写入网站的首页文件。每个首页文件中应有明确区分不同网站内容的信息，方便我们稍后能更直观地检查效果。

	[root@linuxprobe ~]# mkdir -p /home/wwwroot/6111
	[root@linuxprobe ~]# mkdir -p /home/wwwroot/6222
	[root@linuxprobe ~]# echo "port:6111" > /home/wwwroot/6111/index.html
	[root@linuxprobe ~]# echo "port:6222" > /home/wwwroot/6222/index.html

第2步：在httpd服务配置文件的第43行和第44行分别添加用于监听6111和6222端口的参数。

	[root@linuxprobe ~]# vim /etc/httpd/conf/httpd.conf 
	………………省略部分输出信息……………… 
	 33 #
	 34 # Listen: Allows you to bind Apache to specific IP addresses and/or
	 35 # ports, instead of the default. See also the <VirtualHost>
	 36 # directive.
	 37 #
	 38 # Change this to Listen on specific IP addresses as shown below to 
	 39 # prevent Apache from glomming onto all bound IP addresses.
	 40 #
	 41 #Listen 12.34.56.78:80
	 42 Listen 80
	 43 Listen 6111
	 44 Listen 6222
	………………省略部分输出信息……………… 

第3步：在httpd服务的配置文件中大约113行处开始，分别追加写入两个基于端口号的虚拟主机网站参数，然后保存并退出。记得需要重启httpd服务，这些配置才生效。

	[root@linuxprobe ~]# vim /etc/httpd/conf/httpd.conf
	………………省略部分输出信息……………… 
	113 <VirtualHost 192.168.10.10:6111>
	114 DocumentRoot "/home/wwwroot/6111"
	115 ServerName www.linuxprobe.com
	116 <Directory "/home/wwwroot/6111">
	117 AllowOverride None
	118 Require all granted
	119 </Directory> 
	120 </VirtualHost>
	121 <VirtualHost 192.168.10.10:6222>
	122 DocumentRoot "/home/wwwroot/6222"
	123 ServerName bbs.linuxprobe.com
	124 <Directory "/home/wwwroot/6222">
	125 AllowOverride None
	126 Require all granted
	127 </Directory>
	128 </VirtualHost>
	………………省略部分输出信息………………

第4步：因为我们把网站数据目录存放在/home/wwwroot目录中，因此还是必须要正确设置网站数据目录文件的SELinux安全上下文，使其与网站服务功能相吻合。最后记得用restorecon命令让新配置的SELinux安全上下文立即生效。

	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/6111
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/6111/*
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/6222
	[root@linuxprobe ~]# semanage fcontext -a -t httpd_sys_content_t /home/wwwroot/6222/*
	[root@linuxprobe ~]# restorecon -Rv /home/wwwroot/

第5步：用semanage命令查询并过滤出所有与HTTP协议相关且SELinux服务允许的端口列表。

	[root@linuxprobe ~]# semanage port -l | grep http
	http_cache_port_t tcp 8080, 8118, 8123, 10001-10010
	http_cache_port_t udp 3130
	http_port_t tcp 80, 81, 443, 488, 8008, 8009, 8443, 9000
	pegasus_http_port_t tcp 5988
	pegasus_https_port_t tcp 5989

第6步：SELinux允许的与HTTP协议相关的端口号中默认没有包含6111和6222，因此需要将这两个端口号手动添加进去。该操作会立即生效，而且在系统重启过后依然有效。设置好后再重启httpd服务程序

	[root@linuxprobe ~]# semanage port -a -t http_port_t -p tcp 6111
	[root@linuxprobe ~]# semanage port -a -t http_port_t -p tcp 6222
	[root@linuxprobe ~]# semanage port -l| grep http
	http_cache_port_t tcp 8080, 8118, 8123, 10001-10010
	http_cache_port_t udp 3130
	http_port_t tcp  6222, 6111, 80, 81, 443, 488, 8008, 8009, 8443, 9000
	pegasus_http_port_t tcp 5988
	pegasus_https_port_t tcp 5989
	[root@linuxprobe ~]# systemctl restart httpd

{% asset_img 5.png %}

# 5.Apache的访问控制

Apache通过Allow指令允许某个主机访问服务器上的网站资源，通过Deny指令实现禁止访问。在允许或禁止访问网站资源时，还会用到Order指令，这个指令用来定义Allow或Deny指令起作用的顺序，其匹配原则是按照顺序进行匹配，若匹配成功则执行后面的默认指令。比如“Order Allow, Deny”表示先将源主机与允许规则进行匹配，若匹配成功则允许访问请求，反之则拒绝访问请求。

第1步：先在服务器上的网站数据目录中新建一个子目录，并在这个子目录中创建一个包含Successful单词的首页文件；

	[root@linuxprobe ~]# mkdir /var/www/html/server
	[root@linuxprobe ~]# echo "Successful" > /var/www/html/server/index.html

第2步：打开httpd服务的配置文件，在第129行后面添加下述规则来限制源主机的访问。这段规则的含义是允许使用Firefox浏览器的主机访问服务器上的首页文件，除此之外的所有请求都将被拒绝。

	[root@linuxprobe ~]# vim /etc/httpd/conf/httpd.conf
	………………省略部分输出信息………………
	129 <Directory "/var/www/html/server">
	130 SetEnvIf User-Agent "Firefox" ff=1
	131 Order allow,deny
	132 Allow from env=ff
	133 </Directory>
	………………省略部分输出信息………………
	[root@linuxprobe ~]# systemctl restart httpd
	[root@linuxprobe ~]# firefox

{% asset_img 6.png %}

除了匹配源主机的浏览器特征之外，还可以通过匹配源主机的IP地址进行访问控制。例如，我们只允许IP地址为192.168.10.20的主机访问网站资源，那么就可以在httpd服务配置文件的第129行后面添加下述规则。这样在重启httpd服务程序后再用本机（即服务器，其IP地址为192.168.10.10）来访问网站的首页面时就会提示访问被拒绝了。

	[root@linuxprobe ~]# vim /etc/httpd/conf/httpd.conf
	………………省略部分输出信息………………
	129 <Directory "/var/www/html/server">
	130 Order allow,deny 
	131 Allow from 192.168.10.20
	132 </Directory>
	………………省略部分输出信息………………
	[root@linuxprobe ~]# systemctl restart httpd
	[root@linuxprobe ~]# firefox

{% asset_img 7.jpg %}




























