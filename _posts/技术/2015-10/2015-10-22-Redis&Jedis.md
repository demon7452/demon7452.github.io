---
layout: post
title: Redis && Jedis
category: 技术
tags: 
keywords: 
description: 
---
## Redis服务器搭建

### 安装

在命令行执行下面的命令：

```
$ wget http://download.redis.io/releases/redis-3.0.5.tar.gz
$ tar xzf redis-3.0.5.tar.gz
$ cd redis-3.0.5
$ make
```

编译完成后,src文件夹中会包括以下的一些可执行文件：  
<font color=red>redis-server：</font>这个是redis的服务器   
<font color=red>redis-cli：</font>这个是redis的客户端   
<font color=red>redis-check-aof：</font>这个是检查AOF文件的工具   
<font color=red>redis-check-dump：</font>这个是本地数据检查工具   
<font color=red>redis-benchmark：</font>性能基准测试工具，安装完后可以测试一下当前Redis的性能   
<font color=red>redis-sentinel：</font>Redis监控工具，集群管理工具

### 配置文件
Redis的配置文件是：redis.conf 

### 常用配置项为: 
<font color=red>daemonize:</font> 是否以后台进程运行，默认为no </br>
<font color=red>pidfile /var/run/redis.pid: </font>pid文件路径 </br>
<font color=red>port 6379:</font> 监听端口 </br>
<font color=red>bind 127.0.0.1:</font>绑定主机ip</br> 
<font color=red>unixsocket /tmp/redis.sock：</font>sock文件路径 </br>
<font color=red>timeout 300：</font>超时时间，默认是300s </br>
<font color=red>loglevel verbose：</font>日志等级，可选项有debug:大量的信息，开发和测试有用；verbose：很多极其有用的信息，但是不像debug那么乱；notice：在生产环境中你想用的信息；warning：最关键、最重要的信息才打印。 默认是erbose </br>
<font color=red>logfile stdout：</font>日志记录方式，默认是stdout </br>
<font color=red>syslog-enabled no：</font>日志记录到系统日志中，默认是no </br>
<font color=red>syslog-ident redis：</font>指定系统日志标识 </br>
<font color=red>syslog-facility local0：</font>指定系统日志设备，必须是USER或者 LOCAL0~LOCAL7。 默认是local0 </br>
<font color=red>databases 16：</font>数据库的数量，默认的数据库是DB 0，你可以使用 SELECT 来选择不同的数据库。dbid的范围是0~(你设置的值-1)</br>
<font color=red>save \<seconds\> \<changes\>：</font>RDB在多长时间内，有多少次更新操作，就将数据同步到数据文件。 </br>
<font color=red>save 900 1：</font>15min内至少1个key被改变 </br>
<font color=red>save 300 10：</font>5min内至少有300个key被改变 </br>
<font color=red>save 60 10000：</font>60s内至少有10000个key被改变 </br>
<font color=red>rdbcompression yes：</font>存储至本地数据库时是否压缩数据，默认是yes </br>
<font color=red>dbfilename dump.rdb：</font>本地数据库文件名，默认是dump.rdb </br>
<font color=red>dir ./：</font>本地数据库存放路径，默认是./ </br>
<font color=red>slaveof <masterip> <masterport>：</font>当本机为从服务时，设置主服务的ip以及端口 </br>
<font color=red>masterauth <master-password>：</font>主服务的连接密码</br>

### 启动服务器
```
$ src/redis-server
```

### 启动客户端
```
$ src/redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
```

### README,官方说明文档
```
/**
Where to find complete Redis documentation?
-------------------------------------------

This README is just a fast "quick start" document. You can find more detailed
documentation at http://redis.io

Building Redis
--------------

Redis can be compiled and used on Linux, OSX, OpenBSD, NetBSD, FreeBSD.
We support big endian and little endian architectures.

It may compile on Solaris derived systems (for instance SmartOS) but our
support for this platform is "best effort" and Redis is not guaranteed to
work as well as in Linux, OSX, and *BSD there.

It is as simple as: 
*/
##编译命令
    % make

##You can run a 32 bit Redis binary using: 对于32位系统

    % make 32bit

##After building Redis is a good idea to test it, using:
##编译完成后，测试运行

    % make test
/**
Fixing build problems with dependencies or cached build options
—--------
Redis has some dependencies which are included into the "deps" directory.
"make" does not rebuild dependencies automatically, even if something in the
source code of dependencies is changes.

When you update the source code with `git pull` or when code inside the
dependencies tree is modified in any other way, make sure to use the following
command in order to really clean everything and rebuild from scratch:
*/
    make distclean
/**
This will clean: jemalloc, lua, hiredis, linenoise.

Also if you force certain build options like 32bit target, no C compiler
optimizations (for debugging purposes), and other similar build time options,
those options are cached indefinitely until you issue a "make distclean"
command.

Fixing problems building 32 bit binaries
---------

If after building Redis with a 32 bit target you need to rebuild it
with a 64 bit target, or the other way around, you need to perform a
"make distclean" in the root directory of the Redis distribution.

In case of build errors when trying to build a 32 bit binary of Redis, try
the following steps:

* Install the packages libc6-dev-i386 (also try g++-multilib).
* Try using the following command line instead of "make 32bit":
*/
    make CFLAGS="-m32 -march=native" LDFLAGS="-m32"
/**
Allocator
---------

Selecting a non-default memory allocator when building Redis is done by setting
the `MALLOC` environment variable. Redis is compiled and linked against libc
malloc by default, with the exception of jemalloc being the default on Linux
systems. This default was picked because jemalloc has proven to have fewer
fragmentation problems than libc malloc.

To force compiling against libc malloc, use:
*/
    % make MALLOC=libc

##To compile against jemalloc on Mac OS X systems, use:

    % make MALLOC=jemalloc
/**
Verbose build
-------------

Redis will build with a user friendly colorized output by default.
If you want to see a more verbose output use the following:
*/
    % make V=1

##Running Redis 运行 Redis
-------------

##To run Redis with the default configuration just type:
##进入src文件夹，运行redis-server
    % cd src
    % ./redis-server
    
##If you want to provide your redis.conf, you have to run it using an additional parameter (the path of the configuration file):
##运行时带上你自己的配置文件 

    % cd src
    % ./redis-server /path/to/redis.conf

##It is possible to alter the Redis configuration passing parameters directly as options using the command line. Examples:
##运行时带上配置参数
    % ./redis-server --port 9999 --slaveof 127.0.0.1 6379
    % ./redis-server /etc/redis/6379.conf --loglevel debug

#All the options in redis.conf are also supported as options using the command line, with exactly the same name.

#Playing with Redis
------------------

#You can use redis-cli to play with Redis. Start a redis-server instance, then in another terminal try the following:
#启动redis服务器后，运行redis客户端，
    % cd src
    % ./redis-cli
    redis> ping
    PONG
    redis> set foo bar
    OK
    redis> get foo
    "bar"
    redis> incr mycounter
    (integer) 1
    redis> incr mycounter
    (integer) 2
    redis> 

#You can find the list of all the available commands here:
#命令说明网址
    http://redis.io/commands

#Installing Redis
#安装 redis
-----------------

#In order to install Redis binaries into /usr/local/bin just use:
#默认路径安装 /usr/local/bin
    % make install
	
#You can use "make PREFIX=/some/other/directory install" if you wish to use a
different destination.
#自定义路径安装
	% make PREFIX=/usr/lib/redis install
	
#Make install will just install binaries in your system, 
#Make install只会在你的系统中安装二进制文件
#but will not configure init scripts and configuration files in the appropriate place. 
#但不会在适当的位置配置用于初始化的脚本和配置文件
#This is not needed if you want just to play a bit with Redis, 
#如果只是想要试用Redis，下面的安装就不需要了；
#but if you are installing it the proper way for a production system,
#但是如果你想要为一个生产系统安装它
#we have a script doing this for Ubuntu and Debian systems:
#请运行以下脚本：

    % cd utils
    % ./install_server.sh

#The script will ask you a few questions and will setup everything you need to run Redis properly as a background daemon 
#安装过程中会询问你一些问题以正确的安装完成，并作为一个后台进程运行，
#that will start again on system reboots.
#重启后生效, 经测试，重启后Redis服务会自启动。

	$ sudo ./install_server.sh 
	Welcome to the redis service installer
	This script will help you easily set up a running redis server

	Please select the redis port for this instance: [6379] 
	Selecting default: 6379
	Please select the redis config file name [/etc/redis/6379.conf] 
	Selected default - /etc/redis/6379.conf
	Please select the redis log file name [/var/log/redis_6379.log] 
	Selected default - /var/log/redis_6379.log
	Please select the data directory for this instance [/var/lib/redis/6379] 
	Selected default - /var/lib/redis/6379
	Please select the redis executable path [] /usr/local/bin/redis-server
	Selected config:
	Port           : 6379
	Config file    : /etc/redis/6379.conf
	Log file       : /var/log/redis_6379.log
	Data dir       : /var/lib/redis/6379
	Executable     : /usr/local/bin/redis-server
	Cli Executable : /usr/local/bin/redis-cli
	Is this ok? Then press ENTER to go on or Ctrl-C to abort.
	Copied /tmp/6379.conf => /etc/init.d/redis_6379
	Installing service...
	Successfully added to chkconfig!
	Successfully added to runlevels 345!
	Starting Redis server...
	Installation successful!

#You'll be able to stop and start Redis using the script named
#你可以使用以下命令启动和停止Redis
	/etc/init.d/redis_<portnumber>
#for instance ,例如： 
	/etc/init.d/redis_6379

/**
Code contributions
---

Note: by contributing code to the Redis project in any form, including sending
a pull request via Github, a code fragment or patch via private email or
public discussion groups, you agree to release your code under the terms
of the BSD license that you can find in the COPYING file included in the Redis
source distribution.

Please see the CONTRIBUTING file in this source distribution for more
information.

Enjoy!
*/
```

### 资料链接
1.<a href="http://redis.io/download" target="_blank">[Redis]</a></br> 

2.<a href="http://my.oschina.net/gccr/blog/307725" target="_blank">[Redis服务器搭建/配置/及Jedis客户端的使用方法]</a>



