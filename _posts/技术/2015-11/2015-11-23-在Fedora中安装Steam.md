---
layout: post
title: 在Fedora中安装Steam
category: 技术
tags: 
keywords: 
description: 
---
##在Fedora中安装Steam
1、在Fedora中配置Steam安装时需要的相关源

```
#运行如下命令
$ su -c 'dnf install http://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm http://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm'
```

2、在Fedora中使用dnf直接安装

```
#To install the repository on a supported Fedora 22+ distribution, run as root the following commands to install the client:
#配置steam的客户端的源
$ dnf config-manager --add-repo=http://negativo17.org/repos/fedora-steam.repo
$ dnf -y install steam
```

3、启动时碰到的问题

```
#问题描述 ：Steam fatal error steam needs to be online to update, but was set to offline movies
#我直接运行如下命令就可以了
$ steam --reset
```

##参考文档
1.<a href="https://developer.valvesoftware.com/wiki/Steam_under_Linux" target="_blank">Steam under Linux</a><br> 
2.<a href="http://negativo17.org/steam/" target="_blank">Moving the Steam client installation</a><br>
3.<a href="http://askubuntu.com/questions/256628/steam-fatal-error-steam-needs-to-be-online-to-update-but-was-set-to-offline-mov" target="_blank">Steam fatal error steam needs to be online to update, but was set to offline movies</a><br>
