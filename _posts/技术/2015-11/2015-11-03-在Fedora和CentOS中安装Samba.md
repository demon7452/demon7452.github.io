---
layout: post
title: 在Fedora和CentOS中安装Samba
category: 技术
tags: Linux
keywords: 
description: 
---
##Samba简介
Samba是在Linux和UNIX系统上实现SMB协议的一个免费软件，由服务器及客户端程序构成。SMB（Server Messages Block，信息服务块）是一种在局域网上共享文件和打印机的一种通信协议，它为局域网内的不同计算机之间提供文件及打印机等资源的共享服务。SMB协议是客户机/服务器型协议，客户机通过该协议可以访问服务器上的共享文件系统、打印机及其他资源。通过设置“NetBIOS over TCP/IP”使得Samba不但能与局域网络主机分享资源，还能与全世界的电脑分享资源。  
如果您工作的环境中既有微软的Windows又有Linux，那么，一个共享文件及目录的方式便是通过一个跨平台网络文件共享协议：SMB/CIFS。Windows原生的支持SMB/CIFS，Linux也通过开源的软件Samba实现了SMB/CIFS协议。
##在Fedora和CentOS上安装Samba
首先，检验Samba是否已经安装在您的系统中：

```
$ rpm -q samba samba-common samba-client 
```

如果上面的命令没有任何输出，这意味着Samba并未安装。这时，应使用下面的命令来安装Samba。

```
$ sudo yum install samba samba-common samba-client 
#在CentOS中yum安装可能出错，可以根据出错信息自行下载相应版本的Samba rpm安装包。
```
##建立需要共享的文件夹
创建一个用于在网络中共享的本地文件夹。这个文件夹应该以Samba共享的方式导出到远程的用户。  
共享文件夹可以创建在home目录下，也可以在根目录下。

```
mkdir /shared   或者   /home/shared
```

通过getsebool命令我们可以查看samba在SELinux中的配置。

```
#查询命令
$ getsebool -a | grep samba
samba_create_home_dirs --> on
samba_domain_controller --> off
samba_enable_home_dirs --> off
samba_export_all_ro --> off
samba_export_all_rw --> off
samba_portmapper --> off
samba_run_unconfined --> off
samba_share_fusefs --> on
samba_share_nfs --> on
sanlock_use_samba --> off
use_samba_home_dirs --> on
virt_sandbox_use_samba --> on
virt_use_samba --> on
```

为了Samba共享在我们home文件夹内的文件夹，我们必须在SELinux中开启共享home文件夹的选项，该选项默认被关闭。下面的命令能达到该效果。

```
$ sudo setsebool -P samba_enable_home_dirs 1  
```

##为Samba配置SELinux
接下来，我们需要再次配置SELinux。在Fedora和CentOS发行版中SELinux是默认开启的。SELinux仅在正确的安全配置下才允许Samba读取和修改文件或文件夹。（例如，加上'sambasharet'属性标签）。

```
#下面的命令为文件的配置添加必要的标签：
$ sudo semanage fcontext -a -t samba_share_t "<directory>(/.*)?" 

#将替换为我们之前为Samba共享创建的本地文件夹（例如，/shared）：
$ sudo semanage fcontext -a -t samba_share_t "/shared(/.*)?" 

#我们必须执行restorecon命令来激活修改的标签，命令如下：
$ sudo restorecon -R -v /shared 
```

##为Samba配置防火墙
下面的命令用来打开防火墙中Samba共享所需的TCP/UDP端口。  
如果您在使用firewalld（例如，在Fedora和CentOS7下），接下来的命令将会永久的修改Samba相关的防火墙规则。

```
$ sudo firewall-cmd --permanent --add-service=samba 
```

如果您在防火墙中使用iptables（例如，CentOS6或者更早的版本），可以使用下面的命令来打开Samba必要的向外的端口。

```
$ sudo vi /etc/sysconfig/iptables 

#添加samba端口访问允许，注意一定要添加在ACCEPT行后面，不可添加在最后,应该是添加在COMMIT之前就行了。
-A INPUT -m state --state NEW -m tcp -p tcp --dport 139 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 445 -j ACCEPT
-A INPUT -p udp -m udp --dport 137 -j ACCEPT
-A INPUT -p udp -m udp --dport 138 -j ACCEPT

#重启iptables
/etc/rc.d/init.d/iptables restart

```

##修改改Samba配置文件smb.conf

```
#文件编辑器打开Samba配置文件
$ sudo vim /etc/samba/smb.conf 

#将下面的行添加到文件的末尾。
[shared]
comment=shared files
path=/shared  或者是 /home/shared
public=yes
writeable=yes

#默认配置中会共享home文件夹，如果不想共享home，可以将下面配置注解掉。
[homes]
	comment = Home Directories
	browseable = no
	writable = yes
;	valid users = %S
;	valid users = MYDOMAIN\%S
```

##创建Samba用户帐户
创建Samba用户帐户，这是挂载和导出Samba文件系统所必须的。我们可以使用smbpasswd工具来创建一个Samba用户。注意，Samba用户帐户必须是已有的Linux用户。如果您尝试使用smbpasswd添加一个不存在的用户，它会返回一个错误的消息。  
如果您不想使用任何已存在的Linux用户作为Samba用户，您可以在您的系统中创建一个新的用户。为安全起见，设置新用户的登录脚本为/sbin/nologin，并且不创建该用户的home文件夹。

```
#我们也可以使用系统已有的用户。
#在这个例子中，我们创建了一个名叫"sambaguest"的用户
$ sudo useradd -M -s /sbin/nologin sambaguest
$ sudo passwd sambaguest 

#在创建一个新用户后，使用smbpasswd命令添加Samba用户。当这个命令询问一个密码时，您可以键入一个与其用户密码不同的密码。
$ sudo smbpasswd -a sambaguest
```

##共享文件的权限问题
创建Samba用户帐户后，我们需要对之前创建的共享文件夹shared进行权限设置。

```
#如果需要设定为只能通过 sambaguest账户名密码才能访问，可以将shared用户组设为 sambaguest,并为其加入rwx权限。
drwxrwx---.  2 root root  4096 11月  2 16:55 shared
$ sudo chgrp sambaguest shared/
drwxrwx---.  2 root sambaguest  4096 11月  2 16:55 shared

#如果设定为任何人都可以匿名访问的话，可以为others用户组加入 rwx权限
$ sudo chmod o+rwx shared/
drwxrwxrwx.  2 root dev3  4096 11月  2 16:55 shared
```

##启动Samba服务
激活Samba服务，并检测Samba服务是否在运行。

```
#在Fedora中可以通过如下方式启动
$ sudo systemctl enable smb.service 
$ sudo systemctl start smb.service 
$ sudo systemctl is-active smb
#在CentOS中使用如下命令启动
$ /etc/rc.d/init.d/smb start 或者重新启动 restart

#将samba加入默认启动项，默认为2-5开启
$ chkconfig smb on
#检查开启情况
$ chkconfig --list smb
```
##访问 Samba
在Fedora中可以 smb://<samba-server-IP-address> 连接共享文件夹  
在windows中可以在‘运行’中输入 \\<samba-server-IP-address> 连接共享文件夹

##碰到的问题
1、访问samba服务器错误："您可能没有权限使用网络资源"的问题  
可能是因为SELinux。SELinux(Security-Enhanced Linux) 是美国国家安全局（NSA）对于强制访问控制的实现，是 Linux&reg; 上最杰出的新安全子系统。  
尝试解决办法：

```
#要是想让共享目录能访问，可以使用命令暂时停掉SELinux
$ setenforce 0

#启用SELinux
$ setenforce 1
```

##参考文档
1.<a href="https://linux.cn/article-5547-1.html" target="_blank">如何在Fedora或CentOS上使用Samba共享文件夹</a><br> 
2.<a href="http://www.cnblogs.com/ginoz/archive/2012/07/31/2616760.html" target="_blank">CentOS 6.2 安装 Samba</a><br>
3.<a href="http://www.jeepshoe.org/808959594.htm" target="_blank">Fedora访问samba服务器错误："您可能没有权限使用网络资源"的问题</a><br>
