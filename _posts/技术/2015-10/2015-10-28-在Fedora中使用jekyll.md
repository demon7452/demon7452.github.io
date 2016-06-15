---
layout: post
title: 在Fedora中使用jekyll
category: 技术
tags: Linux
keywords: 
description: 
---
## 安装ruby，RubyGems 和 jekyll
```
#使用yum安装，这一般不会碰到问题
$ sudo yum install ruby
#ruby-devel可能不会默认安装，可以使用如下命令更新ruby-devel
$ sudo yum install ruby-devel
#安装ruby的过程中会自动安装RubyGems,可以使用如下命令更新RubyGems
$ gem update --system  
#安装 jekyll
$ sudo gem install jekyll

#安装过程中可能碰到如下问题
#问题一：
[dev3@localhost ~]$ gem update --system
ERROR:  While executing gem ... (Gem::RemoteFetcher::FetchError)
    Errno::ECONNRESET: Connection reset by peer - SSL_connect (https://api.rubygems.org/specs.4.8.gz)
#问题二：
[dev3@localhost ~]$ sudo gem install jekyll
Fetching: ffi-1.9.10.gem (100%)
Building native extensions.  This could take a while...
ERROR:  Error installing jekyll:
	ERROR: Failed to build gem native extension.

    /usr/bin/ruby -r ./siteconf20151028-8562-rmel60.rb extconf.rb
mkmf.rb can't find header files for ruby at /usr/share/include/ruby.h

extconf failed, exit code 1

Gem files will remain installed in /usr/local/share/gems/gems/ffi-1.9.10 for inspection.
Results logged to /usr/local/lib64/gems/ruby/ffi-1.9.10/gem_make.out
```

## 问题解决：

### 问题一：Gem::RemoteFetcher::FetchError
```
ERROR:  While executing gem ... (Gem::RemoteFetcher::FetchError)
    Errno::ECONNRESET: Connection reset by peer - SSL_connect (https://api.rubygems.org/specs.4.8.gz)
#问题原因应该是https://rubygems.org/无法访问，需要将https改为http,解决过程如下
#查看gem sources
$ gem sources --list 
*** CURRENT SOURCES ***

https://rubygems.org/
#将http://rubygems.org/加入sources
$ gem sources --add http://rubygems.org/
#将https://rubygems.org/移出sources
$ gem sources --remove https://rubygems.org/
#ok,问题应该解决了。
```

### 问题二：ERROR: Failed to build gem native extension.
```
Fetching: ffi-1.9.10.gem (100%)
Building native extensions.  This could take a while...
ERROR:  Error installing jekyll:
	ERROR: Failed to build gem native extension.
    /usr/bin/ruby -r ./siteconf20151028-8562-rmel60.rb extconf.rb
mkmf.rb can't find header files for ruby at /usr/share/include/ruby.h
extconf failed, exit code 1
Gem files will remain installed in /usr/local/share/gems/gems/ffi-1.9.10 for inspection.
Results logged to /usr/local/lib64/gems/ruby/ffi-1.9.10/gem_make.out

//这个错误是因为ruby-devel没有安装，使用如下命令应该就能解决了。
$ sudo yum install ruby-devel
```

## 运行 jekyll server 时碰到的错误
```
#错误描述：
Deprecation: You appear to have pagination turned on, but you haven't included the `jekyll-paginate` gem. Ensure you have `gems: [jekyll-paginate]` in your configuration file.
#解决方法：
$ gem install jekyll-paginate
#错误描述：
Yikes! It looks like you don't have redcarpet or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- redcarpet'
#解决方法：
$ gem install redcarpet
#错误描述：
Yikes! It looks like you don't have pygments or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- pygments'
#解决方法：
$ gem install pygments.rb
#总结：
#类似依赖不存在的话，https://rubygems.org/ 查找的该依赖，然后下载安装。

```

## 关于在windows中使用jekyll的说明
在Windows中安装jekyll可以参考文档<a href="http://blog.csdn.net/itmyhome1990/article/details/41982625" target="_blank">Windows上安装Jekyll</a><br> 
在运行 jekyll server 时我碰到的问题

### 问题一：

```
  Conversion error: Jekyll::Converters::Markdown encountered an error while conv
erting '_posts/技术/2014-09-12-Linux-Problems.md':
                    Failed to get header.
```

出现该问题的原因是：在windows中jekyll使用的python版本不对，安装的 python 必须是2.7版本。<br>
详细的说明可以参见<a href="https://teamtreehouse.com/community/error-running-jekyll-serve-liquid-exception-failed-to-get-header" target="_blank">Failed to get header."</a><br> 
部分原文：<br>
This error seems to be caused because of a component of jekyll that uses Python, not Ruby, and only works with Python 2.7 but not versions 3.+. See https://github.com/jekyll/jekyll/issues/1181&#13;

Python 2.7 is pre-installed by default on Linux and Mac OS X but not Windows. Go to https://www.python.org/downloads/ and download the latest 2.7 version (not 3+) and install making sure the Add to PATH option is selected.

This fixed the issue for me.

### 问题二:

```
jekyll 3.0.1 | Error:  Permission denied - bind(2) for 127.0.0.1:4000

#解决方法：说明端口被占有，不知为何，打开_config.yml 在最后加入一行 port: 5001 (其他也可)问题解决。
```

## 参考文档
1.<a href="https://rubygems.org/pages/download" target="_blank">RubyGems官方说明文档</a><br> 
2.<a href="https://rubygems.org/gems/jekyll/versions/2.5.3" target="_blank">RubyGems jekyll</a><br>
3.<a href="http://jekyll.bootcss.com/" target="_blank">jekyll 官方说明文档</a><br>



