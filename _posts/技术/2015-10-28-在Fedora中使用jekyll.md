---
layout: post
title: 在Fedora中使用jekyll
category: 技术
tags: Linux
keywords: 
description: 
---
##安装ruby，RubyGems 和 jekyll
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
##问题解决：
###问题一：Gem::RemoteFetcher::FetchError
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
###问题二：ERROR: Failed to build gem native extension.
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
#这个错误是因为ruby-devel没有安装，使用如下命令应该就能解决了。
$ sudo yum install ruby-devel
```
##参考文档
1.<a href="https://rubygems.org/pages/download" target="_blank">RubyGems官方说明文档</a><br> 
2.<a href="https://rubygems.org/gems/jekyll/versions/2.5.3" target="_blank">RubyGems jekyll</a><br>
3.<a href="http://jekyll.bootcss.com/" target="_blank">jekyll 官方说明文档</a><br>



