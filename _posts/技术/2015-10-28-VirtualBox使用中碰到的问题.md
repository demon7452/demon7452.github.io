---
layout: post
title: VirtualBox使用中碰到的问题
category: 技术
tags: Linux
keywords: 
description: 
---
##启动时出现Kernel driver not installed (rc=-1908)
###Fedora环境下出现如下问题
```
Kernel driver not installed (rc=-1908)

The VirtualBox Linux kernel driver (vboxdrv) is either not loaded or there is a permission problem with /dev/vboxdrv. Re-setup the kernel module by executing

'/sbin/vboxconfig'

as root. Users of Ubuntu, Fedora or Mandriva should install the DKMS package first. This package keeps track of Linux kernel changes and recompiles the vboxdrv kernel module if necessary.

#切换到sbin文件夹下查看vboxconfig文件
lrwxrwxrwx. 1 root dev3 38 10月 28 09:07 vboxconfig -> /usr/lib/virtualbox/postinst-common.sh
#发现其是一个link文件，切换到 /usr/lib/virtualbox/文件夹下
[dev3@localhost virtualbox]$ ll vboxdrv.sh 
-rwxr-xr-x. 1 root root 12682 10月 15 23:24 vboxdrv.sh
#找到vboxdrv.sh文件，运行它
[dev3@localhost virtualbox]$ ./vboxdrv.sh 
Usage: ./vboxdrv.sh {start|stop|stop_vms|restart|force-reload|status|setup}
#使用root权限运行它，加上 setup指令，结果如下
[dev3@localhost virtualbox]$ sudo ./vboxdrv.sh setup
Stopping VirtualBox kernel modules                         [  确定  ]
Uninstalling old VirtualBox DKMS kernel modules            [  确定  ]
Trying to register the VirtualBox kernel modules using DKMS[  确定  ]
Starting VirtualBox kernel modules                         [  确定  ]
#ok,解决问题。蛋疼。
```
##参考文档
1.<a href="https://www.virtualbox.org/manual/UserManual.html" target="_blank">VirtualBox官方文档</a><br> 


