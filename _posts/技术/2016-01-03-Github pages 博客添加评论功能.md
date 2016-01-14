---
layout: post
title: Github pages 博客添加评论功能
category: 技术
tags: github博客
keywords: 
description: 
---
##git学习总结
###一、git安装
git在Windows、Linux、Mac中的安装方法请参阅<a target="_blank" href="http://www.worldhello.net/">[Git权威指南]</a>,在这里就不一一赘述了；<br>
如果是在Windows中使用的话，可以使用Github for windows客户端。个人感觉还是比较好用的，值得一试；<br>

```
#git安装好后可以运行如下命令来查看git版本
$ git --version
git version 2.5.3.windows.1
```
###二、git config -- git配置
在开始使用git之前，需要设置一下git的配置变量，主要是配置提交者的email和name;<br>
git配置变量主要分为三个级别：版本库级、当前用户级、系统级（面向所有用户）；<br>
三者的优先级顺序为：版本库-->用户-->系统；<br>

```
#配置版本库级变量，需要在版本库目录下执行如下命令；
#该配置文件位于版本库目录下的 .git 目录中  .git/config
$ git config user.email "XXXX@XX.com"  //配置邮箱
$ git config user.name "XXX"  //配置用户名

#配置当前用户的git变量，执行以命令即可：
#该配置文件位于用户主目录下的 .gitconfig 文件中，linux中可以使用 cat ~/.gitconfig 查看
$ git config --global user.email "XXXX@XX.com"  //配置邮箱
$ git config --global user.name "XXX"  //配置用户名

#配置系统级的git变量，执行以命令即可：
#该配置文件位于 /etc/gitconfig 文件中;
$ git config --system user.email "XXXX@XX.com"  //配置邮箱
$ git config --system user.name "XXX"  //配置用户名
```

###三、git init -- 初始化版本库
主要是使用git init 创建版本库，使用方法如下：

```
#创建一个目录，进入该目录下初始化版本库
$ mkdir demo
$ cd demo
$ git init
初始化空的 Git 版本库于 /cygdrive/f/github/gittest/demo/.git/

#直接创建
$ git init demo
初始化空的 Git 版本库
```

###四、git clone -- 克隆版本库

```
#问题描述 ：Steam fatal error steam needs to be online to update, but was set to offline movies
#我直接运行如下命令就可以了
$ steam --reset
```

##参考文档
1.<a href="https://developer.valvesoftware.com/wiki/Steam_under_Linux" target="_blank">Steam under Linux</a><br> 
2.<a href="http://negativo17.org/steam/" target="_blank">Moving the Steam client installation</a><br>
3.<a href="http://askubuntu.com/questions/256628/steam-fatal-error-steam-needs-to-be-online-to-update-but-was-set-to-offline-mov" target="_blank">Steam fatal error steam needs to be online to update, but was set to offline movies</a><br>
