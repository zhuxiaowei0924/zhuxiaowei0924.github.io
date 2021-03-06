---
title: linux就该这么学：管道符、重定向与环境变量
date: 2021-01-09 15:30:48
tags:
- 第3章节：管道符、重定向与环境变量
categories:
- linux就该这么学

---
# 1.输入输出重定向

输入重定向：是指把文件导入到命令中； 输出重定向：是指把原本要输出到屏幕的数据写入到指定文件中。输出重定向又分为标准输出重定向和错误输出重定向。
输入重定向中用到的符号及其作用：
<!--more-->


| 符号 | 作用|
| --- | --- |
| 命令 < 文件 | 将文件作为命令的标准输入 |
| 命令 << 分界符（tag） | 将开始标记 tag 和结束标记 tag 之间的内容作为输入 |

输入重定向中用到的符号及其作用：


| 符号 | 作用|
| --- | --- |
| 命令 > 文件 |	将输出重定向到文件（会清空原有数据） |
| 命令 >> 文件 |	将输出以追加的方式重定向到文件 |
| n > 文件 |	将文件描述符为 n 的文件重定向到文件 |
| n >> 文件 |	将文件描述符为 n 的文件以追加的方式重定向到文件 |
| n >& m |	将输出文件 m 和 n 合并 |
| n <& m |	将输入文件 m 和 n 合并 |
| << tag |	将开始标记 tag 和结束标记 tag 之间的内容作为输入 |

注意：文件描述符 0 通常是标准输入（STDIN），1 是标准输出（STDOUT），2 是标准错误输出（STDERR）。

# 2. 管道命令符

命令符：可以把前一个命令原本要输出到屏幕的数据当作是后一个命令的标准输入；

# 3.命令行的通配符

星号（*）：代表匹配零个或多个字符； 问号（？）：代表匹配单个字符； [0-9]:代表匹配0-9之间的单个数字； [abc]:代表匹配a、b、c三个字符中的任意一个；
[^...]和[!...]表示匹配不在方括号里面的字符（不包括空字符）；这两种写法是等价的。

# 4.转义字符

反斜杠（\）：使反斜杠后面变量称为单纯的字符串；
单引号（''）：转义其中所有的变量为单纯的字符串；
双引号（""）:保留其中的变量属性，不尽兴转义处理；
反引号（``）:把其中的命令执行后返回结果。

# 5.环境变量

变量是计算机系统用于保存可变值得数据类型，变量名称一般都是大写的；环境变量是用来定义系统运行环境的一些参数，比如每个用户不同的家目录、邮件存放位置等。

前面讲过，在 Linux 系统中“一切皆文件”，Linux 命令也不例外。那么，当编辑完成 Linux 命令并回车后，系统底层到底发生了什么事情呢？

简单来说，Linux 命令的执行过程分为如下 4 个步骤。

	1) 判断路径
		判断用户是否以绝对路径或相对路径的方式输入命令（如 /bin/ls），如果是的话直接执行。

	2) 检查别名
		Linux 系统会检查用户输入的命令是否为“别名命令”。要知道，通过 alias 命令是可以给现有命令自定义别名的，即用一个自定义的命令名称来替换原本的命令名称。

		例如，我们经常使用的 rm 命令，其实就是 rm -i 这个整体的别名：
		[root@localhost ~]# alias rm
		alias rm='rm -i'

		这使得当使用 rm 命令删除指定文件时，Linux 系统会要求我们再次确认是否执行删除操作。例如：
		[root@localhost ~]# rm a.txt <-- 假定当前目录中已经存在 a.txt 文件
		rm: remove regular file 'a.txt'? y  <-- 手动输入 y，即确定删除
		[root@localhost ~]#

		这里可以使用 unalias 命令，将 Linux 系统设置的 rm 别名删除掉，执行命令如下：
		[root@localhost ~]# alias rm
		alias rm='rm -i'
		[root@localhost ~]# unalias rm
		[root@localhost ~]# rm a.txt
		[root@localhost ~]#  <--直接删除，不再询问


		注意，这里仅是为了演示 unalisa 的用法，建议读者删除 rm 别名之后，再手动添加到系统中，执行如下命令即可再次成功添加：
		[root@localhost ~]# alias rm='rm -i'

	3) 判断是内部命令还是外部命令
		Linux命令行解释器（又称为 Shell）会判断用户输入的命令是内部命令还是外部命令。其中，内部命令指的是解释器内部的命令，会被直接执行；而用户通常输入的命令都是外部命令，这些命令交给步骤四继续处理。
		内部命令由 Shell 自带，会随着系统启动，可以直接从内存中读取；而外部命令仅是在系统中有对应的可执行文件，执行时需要读取该文件。

		判断一个命令属于内部命令还是外部命令，可以使用 type 命令实现。例如：
		[root@localhost ~]# type pwd
		pwd is a shell builtin  <-- pwd是内部命令
		[root@localhost ~]# type top
		top is /usr/bin/top  <-- top是外部命令

	4) 查找外部命令对应的可执行文件
		当用户执行的是外部命令时，系统会在指定的多个路径中查找该命令的可执行文件，而定义这些路径的变量，就称为 PATH 环境变量，其作用就是告诉 Shell 待执行命令的可执行文件可能存放的位置，也就是说，Shell 会在 PATH 变量包含的多个路径中逐个查找，直到找到为止（如果找不到，Shell 会提供用户“找不到此命令”）

自己创建的变量不具有全局性，作用范围有限，默认情况下不能被其他用户使用，因此，可通过export命令（如：export WORKDIR）将其提升为全局变量，这样其他用户也可以使用了！