---
layout: post
title: Ubuntu安装SecureCRT
category: 技术
tags: Linux
keywords: linux SecureCRT
description:
---

### 参考文档
* [Ubuntu16.04下安装破解secureCRT和secureFX的操作记录](https://www.cnblogs.com/kevingrace/p/9353963.html)

### problems

Question:

```bash
root@lubuntu:~# SecureCRT 
SecureCRT: error while loading shared libraries: libpng12.so.0: cannot open shared object file: No such file or directory
```

Answer:

This is a missing dependency issue, and solution to this is very simple and quick. Just follow:
 Add repository for libpng12-0 in sources.list via terminal

```bash
 sudo vim /etc/apt/sources.list
```

 at the bottom of the file add

```bash
## Manually Added sources
## source for libpng12-0 package
deb http://mirrors.kernel.org/ubuntu/ xenial main
```
 
 now to update the package list
 
 ```bash
 sudo apt update
 sudo apt install libpng12-0
 ```