---
layout: post
title: Github pages 博客添加评论功能
category: 技术
tags: git
keywords: 
description: Github pages 博客添加评论功能
---
## Github pages 博客添加评论功能
由于被墙的原因，所以国内一般推荐使用类似 DISQUS 的 ”多说“ 评论插件

### 注册多说用户
在使用多说评论插件之前，需要先前往<a href="http://duoshuo.com/" target="_blank">多说官网</a>注册一个帐号，<br>
将自己的博客网站与多说帐号绑定，并获得一个短域名（short_name）。

### 添加评论框的标签

```
 <div class="ds-thread" data-thread-key="文章在原站点中的id或其他唯一标识" data-title="您的文章标题" ></div>
 #例如：
 <div class="ds-thread" data-thread-key="{{ page.title }}" data-title="{{ page.title }}" data-url="{{ page.url }}"></div>
```

### 添加多说的js

```
<!-- 多说公共JS代码 start (一个网页只需插入一次) -->
<script type="text/javascript">
	var duoshuoQuery = {short_name:"填写你的短域名"};
	(function() {
		var ds = document.createElement('script');
		ds.type = 'text/javascript';ds.async = false;
		ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.js';
		ds.charset = 'UTF-8';
		(document.getElementsByTagName('head')[0] 
		 || document.getElementsByTagName('body')[0]).appendChild(ds);
	})();
</script>
<!-- 多说公共JS代码 end -->
```

### 自己修改评论框样式（可选）
添加css样式，将评论头像修改为圆形，并添加旋转效果。

```
#ds-reset .ds-avatar img {
  width: 54px !important;
  height: 54px !important;
  -webkit-border-radius: 27px !important;
  -moz-border-radius: 27px !important;
  border-radius: 27px !important;
  -webkit-transition: -webkit-transform 0.4s ease-out;
  -moz-transition: -moz-transform 0.4s ease-out;
  transition: transform 0.4s ease-out;
}
#ds-reset .ds-avatar img:hover {
  -webkit-transform: rotateZ(360deg);
  -moz-transform: rotateZ(360deg);
  transform: rotateZ(360deg);
}
#ds-reset .ds-powered-by {
  display: none;
}
```



## 参考文档
1.<a href="http://duoshuo.com/" target="_blank">多说官网</a><br> 
2.<a href="https://disqus.com/profile/login/" target="_blank">DISQUS官网</a><br>
3.<a href="http://havee.me/internet/2013-07/add-duoshuo-commemt-system-into-jekyll.html" target="_blank">为 Jekyll 添加多说评论系统</a><br>
