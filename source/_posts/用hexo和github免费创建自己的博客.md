---
title: 用hexo和github免费创建自己的博客
date: 2020-12-06 13:35:52
tags:
- 博客
categories:
- 博客
---

## 1.安装Nodejs 
官方网站: https://nodejs.org, 安装步骤非常简单，一直next，下一步就可以了，默认安装就行。
## 2.安装Git
官方网站：https://git-scm.com/downloads， 然后我们选择windows版本的下载，安装也是一直点下一步，安装官方默认的来就行，
tips：这个Git Bash下载下来就相当于Linux中的终端窗口了，以后我们就用这个东东来打开终端。
<!-- more -->
## 3.安装hexo
新建一个文件夹，比如我这里建了blog；打开你的文件夹，然后在空白处点鼠标的右键，选择 ==Git Bash Here==；
看看 node，npm是否安装成功，没有成功的就重新安装node：
node -v	#查看node版本
npm -v	#查看npm版本
我们需要先来安装个cnpm提高速度，以后下载什么东西都用cnpm，在上面终端继续输入：
npm install -g cnpm --registry=https://registry.npm.taobao.org
测试cnpm-成功:
cnpm -v	#查看cnpm版本
完成之后安装hexo：
cnpm install -g hexo-cli    #安装hexo框架
验证是否安装成功：
hexo -v	#查看hexo版本
用pwd命令查看当前路径，然后新建文件夹blog1：
mkdir blog1	#创建blog1目录
cd blog1	 #进入blog1目录目录
hexo init 	#生成博客 初始化博客（用Git Bash运行）
hexo s	#启动本地博客服务
http://localhost:4000/	#本地访问地址
hexo n "我的第一篇文章" #创建新的文章 
## 4.返回blog目录
hexo clean #清理
hexo g #生成
## 5.将博客部署到GitHub上
在Github创建一个新的仓库 YourGithubName.github.io
然后输入cnpm install --save hexo-deployer-git #在blog目录下安装git部署插件
## 6.配置_config.yml 
	# Deployment
	## Docs: https://hexo.io/docs/deployment.html
	deploy:
  		type: git
 		repo: https://github.com/YourGithubName/YourGithubName.github.io.git
  		branch: master
win10 记得在 hexo d 之前输入以下命令：
git config --global user.email "xxx"
git config --global user.name "xxx"
然后用hexo d命令将博客部署到Github仓库里
https://YourGithubName.github.io/  #访问这个地址可以查看博客
注：
如果在输入hexo d命令时，提示有以下报错：ERROR Deployer not found: git
则在hexo的文件目录内输入以下命令：npm install --save hexo-deployer-git，之后再次输入hexo d命令即可

## 7.安装主题
下载yilia主题到本地：git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia  
修改hexo根目录下的 _config.yml 文件 ： theme: yilia
hexo c	#清理一下
hexo g	#生成
hexo d	#部署到远程Github仓库
https://YourGithubName.github.io/  #查看博客