---
title: linux就该这么学-18：使用MariaDB数据库管理系统
date: 2021-02-24 21:36:15
tags:
- 第18章节：使用MariaDB数据库管理系统
categories:
- linux就该这么学

---

# 1.数据库管理系统

数据库是指按照某些特定结构来存储数据资料的数据仓库。

<!--more-->

数据库管理系统是一种能够对数据库中存放的数据进行建立、修改、删除、查找、维护等操作的软件程序。它通过把计算机中具体的物理数据转换成适合用户理解的抽象逻辑数据，有效地降低数据库管理的技术门槛。

# 2.初始化MariaDB服务

	[root@linuxprobe ~]# yum install mariadb mariadb-server
	Complete!
	[root@linuxprobe ~]# systemctl start mariadb 
	[root@linuxprobe ~]# systemctl enable mariadb 
	ln -s '/usr/lib/systemd/system/mariadb.service' '/etc/systemd/system/multi-user.target.wants/mariadb.service'

在确认MariaDB数据库软件程序安装完毕并成功启动后请不要立即使用。为了确保数据库的安全性和正常运转，需要先对数据库程序进行初始化操作。这个初始化操作涉及下面5个步骤：

1.设置root管理员在数据库中的密码值（注意，该密码并非root管理员在系统中的密码，这里的密码值2.默认应该为空，可直接按回车键）。
3.设置root管理员在数据库中的专有密码。
4.随后删除匿名账户，并使用root管理员从远程登录数据库，以确保数据库上运行的业务的安全性。
5.删除默认的测试数据库，取消测试数据库的一系列访问权限。
6.刷新授权列表，让初始化的设定立即生效。

	[root@linuxprobe ~]# mysql_secure_installation 
	/usr/bin/mysql_secure_installation: line 379: find_mysql_client: command not found
	NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
	      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!
	In order to log into MariaDB to secure it, we'll need the current
	password for the root user.  If you've just installed MariaDB, and
	you haven't set the root password yet, the password will be blank,
	so you should just press enter here.
	Enter current password for root (enter for none):  当前数据库密码为空，直接按回车键
	OK, successfully used password, moving on...
	Setting the root password ensures that nobody can log into the MariaDB
	root user without the proper authorisation.
	Set root password? [Y/n] y
	New password:输入要为root管理员设置的数据库密码
	Re-enter new password:再次输入密码
	Password updated successfully!
	Reloading privilege tables..
	 ... Success!
	By default, a MariaDB installation has an anonymous user, allowing anyone
	to log into MariaDB without having to have a user account created for
	them.  This is intended only for testing, and to make the installation
	go a bit smoother.  You should remove them before moving into a
	production environment.
	Remove anonymous users? [Y/n] y（删除匿名账户）
	... Success!
	Normally, root should only be allowed to connect from 'localhost'.  This
	ensures that someone cannot guess at the root password from the network.
	Disallow root login remotely? [Y/n] y（禁止root管理员从远程登录）
	 ... Success!
	By default, MariaDB comes with a database named 'test' that anyone can
	access.  This is also intended only for testing, and should be removed
	before moving into a production environment.
	Remove test database and access to it? [Y/n] y（删除test数据库并取消对它的访问权限）
	 - Dropping test database...
	 ... Success!
	 - Removing privileges on test database...
	 ... Success!
	Reloading the privilege tables will ensure that all changes made so far
	will take effect immediately.
	Reload privilege tables now? [Y/n] y（刷新授权表，让初始化后的设定立即生效）
	 ... Success!
	Cleaning up...
	All done!  If you've completed all of the above steps, your MariaDB
	installation should now be secure.
	Thanks for using MariaDB!

在很多生产环境中都需要使用站库分离的技术（即网站和数据库不在同一个服务器上），如果需要让root管理员远程访问数据库，可在上面的初始化操作中设置策略，以允许root管理员从远程访问。然后还需要设置防火墙，使其放行对数据库服务程序的访问请求，数据库服务程序默认会占用3306端口，在防火墙策略中服务名称统一叫作mysql：

	[root@linuxprobe ~]# firewall-cmd --permanent --add-service=mysql
	success
	[root@linuxprobe ~]# firewall-cmd --reload
	success

现在我们将首次登录MariaDB数据库。其中，-u参数用来指定以root管理员的身份登录，而-p参数用来验证该用户在数据库中的密码值。

	[root@linuxprobe ~]# mysql -u root -p
	Enter password: 此处输入root管理员在数据库中的密码
	Welcome to the MariaDB monitor. Commands end with ; or \g.
	Your MariaDB connection id is 5
	Server version: 5.5.35-MariaDB MariaDB Server
	Copyright (c) 2000, 2013, Oracle, Monty Program Ab and others.
	Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
	MariaDB [(none)]>

在登录MariaDB数据库后执行数据库命令时，都需要在命令后面用分号（;）结尾，这也是与Linux命令最显著的区别；

	MariaDB [(none)]> SHOW databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	+--------------------+
	3 rows in set (0.01 sec)

接下来使用数据库命令将root管理员在数据库管理系统中的密码值修改为linuxprobe。这样退出后再尝试登录，如果还坚持输入原先的密码，则将提示访问失败。

	MariaDB [(none)]> SET password = PASSWORD('linuxprobe');
	Query OK, 0 rows affected (0.00 sec)
	MariaDB [(none)]> exit
	Bye
	[root@linuxprobe ~]# mysql -u root -p
	Enter password:此处输入root管理员在数据库中的新密码
	ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)

# 3.管理用户以及授权

为了保障数据库系统的安全性，以及让其他用户协同管理数据库，我们可以在MariaDB数据库管理系统中为他们创建多个专用的数据库管理账户，然后再分配合理的权限，以满足他们的工作需求。为此，可使用root管理员登录数据库管理系统，然后按照“CREATE USER 用户名@主机名 IDENTIFIED BY '密码'; ”的格式创建数据库管理账户。再次提醒大家，一定不要忘记每条数据库命令后面的分号（;）。

	MariaDB [(none)]> CREATE USER luke@localhost IDENTIFIED BY 'linuxprobe';
	Query OK, 0 rows affected (0.00 sec)

创建的账户信息可以使用select命令语句来查询。下面命令查询的是账户luke的主机名称、账户名称以及经过加密的密码值信息：

	MariaDB [(none)]> use mysql
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A
	Database changed
	MariaDB [mysql]> SELECT HOST,USER,PASSWORD FROM user WHERE USER="luke";
	+-----------+------+-------------------------------------------+
	| host      | user | password                                  |
	+-----------+------+-------------------------------------------+
	| localhost | luke | *55D9962586BE75F4B7D421E6655973DB07D6869F |
	+-----------+------+-------------------------------------------+

不过，用户luke仅仅是一个普通账户，没有数据库的任何操作权限。不信的话，可以切换到luke账户来查询数据库管理系统中当前都有哪些数据库。可以发现，该账户甚至没法查看完整的数据库列表（刚才使用root账户时可以查看到3个数据库列表）：

	MariaDB [mysql]> exit
	Bye
	[root@linuxprobe ~]# mysql -u luke -p
	Enter password: 此处输入luke账户的数据库密码
	Welcome to the MariaDB monitor.  Commands end with ; or \g.
	Your MariaDB connection id is 6
	Server version: 5.5.35-MariaDB MariaDB Server
	Copyright (c) 2000, 2013, Oracle, Monty Program Ab and others.
	Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
	MariaDB [(none)]> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	+--------------------+
	1 row in set (0.03 sec)

GRANT命令的常见格式以及解释:


| 命令 | 作用 |
| --- | --- |
| GRANT 权限 ON 数据库.表单名称 TO 用户名@主机名 | 对某个特定数据库中的特定表单给予授权 |
| GRANT 权限 ON 数据库.* TO 用户名@主机名 | 对某个特定数据库中的所有表单给予授权 |
| GRANT 权限 ON *.* TO 用户名@主机名 | 对所有数据库及所有表单给予授权 |
| GRANT 权限1,权限2 ON 数据库.* TO 用户名@主机名 | 对某个数据库中的所有表单给予多个授权 |
| GRANT ALL PRIVILEGES ON *.* TO 用户名@主机名 | 对所有数据库及所有表单给予全部授权（需谨慎操作） |

账户的授权工作肯定是需要数据库管理员来执行的。下面以root管理员的身份登录到数据库管理系统中，针对mysql数据库中的user表单向账户luke授予查询、更新、删除以及插入等权限。

	[root@linuxprobe ~]# mysql -u root -p
	Enter password:此处输入root管理员在数据库中的密码
	MariaDB [(none)]> use mysql;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A
	Database changed
	MariaDB [mysql]> GRANT SELECT,UPDATE,DELETE,INSERT ON mysql.user TO luke@localhost;
	Query OK, 0 rows affected (0.00 sec)
	在执行完上述授权操作之后，我们再查看一下账户luke的权限：
	
	MariaDB [(none)]>  SHOW GRANTS FOR luke@localhost;
	+-------------------------------------------------------------------------------------------------------------+
	| Grants for luke@localhost |
	+-------------------------------------------------------------------------------------------------------------+
	| GRANT USAGE ON *.* TO 'luke'@'localhost' IDENTIFIED BY PASSWORD '*55D9962586BE75F4B7D421E6655973DB07D6869F' |
	| GRANT SELECT, INSERT, UPDATE, DELETE ON `mysql`.`user` TO 'luke'@'localhost' |
	+-------------------------------------------------------------------------------------------------------------+
	2 rows in set (0.00 sec)

上面输出信息中显示账户luke已经拥有了针对mysql数据库中user表单的一系列权限了。这时我们再切换到账户luke，此时就能够看到mysql数据库了，而且还能看到表单user（其余表单会因无权限而被继续隐藏）：

	[root@linuxprobe ~]# mysql -u luke -p
	Enter password:此处输入luke用户在数据库中的密码
	MariaDB [(none)]> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	+--------------------+
	2 rows in set (0.01 sec)
	MariaDB [(none)]> use mysql
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A
	Database changed
	MariaDB [mysql]> SHOW TABLES;
	+-----------------+
	| Tables_in_mysql |
	+-----------------+
	| user            |
	+-----------------+
	1 row in set (0.01 sec)
	MariaDB [mysql]> exit
	Bye

当前，先切换回root账户，移除刚才的授权。

	[root@linuxprobe ~]# mysql -u root -p
	Enter password:此处输入root管理员在数据库中的密码
	MariaDB [(none)]> use mysql;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A
	Database changed
	MariaDB [(none)]> REVOKE SELECT,UPDATE,DELETE,INSERT ON mysql.user FROM luke@localhost;
	Query OK, 0 rows affected (0.00 sec)

可以看到，除了移除授权的命令（revoke）与授权命令（grant）不同之外，其余部分都是一致的。这不仅好记而且也容易理解。执行移除授权命令后，再来查看账户luke的信息：

	MariaDB [(none)]> SHOW GRANTS FOR luke@localhost;
	+-------------------------------------------------------------------------------------------------------------+
	| Grants for luke@localhost |
	+-------------------------------------------------------------------------------------------------------------+
	| GRANT USAGE ON *.* TO 'luke'@'localhost' IDENTIFIED BY PASSWORD '*55D9962586BE75F4B7D421E6655973DB07D6869F' |
	+-------------------------------------------------------------------------------------------------------------+
	1 row in set (0.00 sec)

# 4.创建数据库与表单

在MariaDB数据库管理系统中，一个数据库可以存放多个数据表，数据表单是数据库中最重要最核心的内容。

用于创建数据库的命令以及作用:


| 用法 | 作用 |
| --- | --- |
| CREATE database 数据库名称。 | 创建新的数据库 |
| DESCRIBE 表单名称; | 描述表单 |
| UPDATE 表单名称 SET attribute=新值 WHERE attribute > 原始值;	 | 更新表单中的数据 |
| USE 数据库名称;	 | 指定使用的数据库 |
| SHOW databases; | 显示当前已有的数据库 |
| SHOW tables; | 显示当前数据库中的表单 |
| SELECT * FROM 表单名称; | 从表单中选中某个记录值 |
| DELETE FROM 表单名 WHERE attribute=值; | 从表单中删除某个记录值 |

现在尝试创建一个名为linuxprobe的数据库，然后再查看数据库列表，此时就能看到它了：

	MariaDB [(none)]> CREATE DATABASE linuxprobe;
	Query OK, 1 row affected (0.00 sec)
	MariaDB [(none)]> SHOW databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| linuxprobe         |
	| mysql              |
	| performance_schema |
	+--------------------+
	4 rows in set (0.04 sec)

要想创建数据表单，需要先切换到某个指定的数据库中。比如在新建的linuxprobe数据库中创建表单mybook，然后进行表单的初始化，即定义存储数据内容的结构。我们分别定义3个字段项，其中，长度为15个字符的字符型字段name用来存放图书名称，整型字段price和pages分别存储图书的价格和页数。当执行完下述命令之后，就可以看到表单的结构信息了：

	MariaDB [(none)]> use linuxprobe;
	Database changed
	MariaDB [linuxprobe]> CREATE TABLE mybook (name char(15),price int,pages int);
	Query OK, 0 rows affected (0.16 sec)
	MariaDB [linuxprobe]> DESCRIBE mybook;
	+-------+----------+------+-----+---------+-------+
	| Field | Type     | Null | Key | Default | Extra |
	+-------+----------+------+-----+---------+-------+
	| name  | char(15) | YES  |     | NULL    |       |
	| price | int(11)  | YES  |     | NULL    |       |
	| pages | int(11)  | YES  |     | NULL    |       |
	+-------+----------+------+-----+---------+-------+
	3 rows in set (0.02 sec)

# 5.管理表单及数据

下面我们使用该命令插入一条图书信息，其中书名为linuxprobe，价格和页数分别是60元和518页。在命令执行后也就意味着图书信息已经成功写入到数据表单中，然后就可以查询表单中的内容了。我们在使用select命令查询表单内容时，需要加上想要查询的字段；如果想查看表单中的所有内容，则可以使用星号（*）通配符来显示：

	MariaDB [linuxprobe]> INSERT INTO mybook(name,price,pages) VALUES('linuxprobe','60', '518');
	Query OK, 1 row affected (0.00 sec)
	MariaDB [linuxprobe]> select * from mybook;
	+------------+-------+-------+
	| name       | price | pages |
	+------------+-------+-------+
	| linuxprobe |    60 |   518 |
	+------------+-------+-------+
	1 rows in set (0.01 sec)

对数据库运维人员来讲，需要做好四门功课—增、删、改、查。这意味着创建数据表单并在其中插入内容仅仅是第一步，还需要掌握数据表单内容的修改方法。例如，我们可以使用update命令将刚才插入的linuxprobe图书信息的价格修改为55元，然后在使用select命令查看该图书的名称和定价信息。注意，因为这里只查看图书的名称和定价，而不涉及页码，所以无须再用星号通配符来显示所有内容。

	MariaDB [linuxprobe]> UPDATE mybook SET price=55 ;
	Query OK, 1 row affected (0.00 sec)
	Rows matched: 1  Changed: 1  Warnings: 0
	MariaDB [linuxprobe]> SELECT name,price FROM mybook;
	+------------+-------+
	| name       | price |
	+------------+-------+
	| linuxprobe |    55 |
	+------------+-------+
	1 row in set (0.00 sec)

我们还可以使用delete命令删除某个数据表单中的内容。下面我们使用delete命令删除数据表单mybook中的所有内容，然后再查看该表单中的内容，可以发现该表单内容为空了。

	MariaDB [linuxprobe]> DELETE FROM mybook;
	Query OK, 1 row affected (0.01 sec)
	MariaDB [linuxprobe]> SELECT * FROM mybook;
	Empty set (0.00 sec)

一般来讲，数据表单中会存放成千上万条数据信息。比如我们刚刚创建的用于保存图书信息的mybook表单，随着时间的推移，里面的图书信息也会越来越多。在这样的情况下，如果我们只想查看其价格大于某个数值的图书时，又该如何定义查询语句呢？

下面先使用insert插入命令依次插入4条图书信息：

	MariaDB [linuxprobe]> INSERT INTO mybook(name,price,pages) VALUES('linuxprobe1','30','518');
	Query OK, 1 row affected (0.05 sec)
	MariaDB [linuxprobe]> INSERT INTO mybook(name,price,pages) VALUES('linuxprobe2','50','518');
	Query OK, 1 row affected (0.05 sec)
	MariaDB [linuxprobe]> INSERT INTO mybook(name,price,pages) VALUES('linuxprobe3','80','518');
	Query OK, 1 row affected (0.01 sec)
	MariaDB [linuxprobe]> INSERT INTO mybook(name,price,pages) VALUES('linuxprobe4','100','518');
	Query OK, 1 row affected (0.00 sec)

要想让查询结果更加精准，就需要结合使用select与where命令了。其中，where命令是在数据库中进行匹配查询的条件命令。通过设置查询条件，就可以仅查找出符合该条件的数据。

 where命令中使用的参数以及作用：


| 参数 | 作用 |
| --- | --- |
| = | 相等 |
| <>或!= | 不相等 |
| > | 大于 |
| < | 小于 |
| >= | 大于或等于 |
| <= | 小于或等于 |
| BETWEEN | 在某个范围内 |
| LIKE | 搜索一个例子 |
| IN | 在列中搜索多个值 |

现在进入动手环节。分别在mybook表单中查找出价格大于75元或价格不等于80元的图书，其对应的命令如下所示。在熟悉了这两个查询条件之后，大家可以自行尝试精确查找图书名为linuxprobe2的图书信息。

	MariaDB [linuxprobe]> SELECT * FROM mybook WHERE price>75;
	+-------------+-------+-------+
	| name        | price | pages |
	+-------------+-------+-------+
	| linuxprobe3 |    80 |   518 |
	| linuxprobe4 |   100 |   518 |
	+-------------+-------+-------+
	2 rows in set (0.06 sec)
	MariaDB [linuxprobe]> SELECT * FROM mybook WHERE price!=80;
	+-------------+-------+-------+
	| name | price | pages        |
	+-------------+-------+-------+
	| linuxprobe1  | 30  | 518    |
	| linuxprobe2  | 50  | 518    |
	| linuxprobe4  | 100 | 518    |
	+-------------+-------+-------+
	3 rows in set (0.01 sec)
	MariaDB [mysql]> exit
	Bye

# 6.数据库的备份及恢复

mysqldump命令用于备份数据库数据，格式为“mysqldump [参数] [数据库名称]”。其中参数与mysql命令大致相同，-u参数用于定义登录数据库的账户名称，-p参数代表密码提示符。下面将linuxprobe数据库中的内容导出成一个文件，并保存到root管理员的家目录中：

	[root@linuxprobe ~]# mysqldump -u root -p linuxprobe > /root/linuxprobeDB.dump
	Enter password:此处输入root管理员在数据库中的密码

然后进入MariaDB数据库管理系统，彻底删除linuxprobe数据库，这样mybook数据表单也将被彻底删除。然后重新建立linuxprobe数据库：

	MariaDB [(none)]> DROP DATABASE linuxprobe;
	Query OK, 1 row affected (0.04 sec)
	MariaDB [(none)]> SHOW databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	+--------------------+
	3 rows in set (0.02 sec)
	MariaDB [(none)]> CREATE DATABASE linuxprobe;
	Query OK, 1 row affected (0.00 sec)

接下来是见证数据恢复效果的时刻！使用输入重定向符把刚刚备份的数据库文件导入到mysql命令中，然后执行该命令。接下来登录到MariaDB数据库，就又能看到linuxprobe数据库以及mybook数据表单了。数据库恢复成功！

	[root@linuxprobe ~]# mysql -u root -p linuxprobe < /root/linuxprobeDB.dump 
	Enter password: 此处输入root管理员在数据库中的密码值
	[root@linuxprobe ~]# mysql -u root -p
	Enter password: 此处输入root管理员在数据库中的密码值
	MariaDB [(none)]> use linuxprobe;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A
	Database changed
	MariaDB [linuxprobe]> SHOW tables;
	+----------------------+
	| Tables_in_linuxprobe |
	+----------------------+
	| mybook               |
	+----------------------+
	1 row in set (0.05 sec)
	MariaDB [linuxprobe]> DESCRIBE mybook;
	+-------+----------+------+-----+---------+-------+
	| Field | Type     | Null | Key | Default | Extra |
	+-------+----------+------+-----+---------+-------+
	| name  | char(15) | YES  |     | NULL    |       |
	| price | int(11)  | YES  |     | NULL    |       |
	| pages | int(11)  | YES  |     | NULL    |       |
	+-------+----------+------+-----+---------+-------+
	3 rows in set (0.02 sec)





















