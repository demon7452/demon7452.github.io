---
layout: post
title: Github pages 博客添加分享功能
category: 技术
tags: git
keywords: 
description: 
---

## Github pages 博客添加分享功能
使用百度分享为博客添加分享功能将十分简单，只需在页面中添加一段script即可，如下所示：
<pre>
	script
		window._bd_share_config={
			"common":{
				"bdSnsKey":{},
				"bdText":"",
				"bdMini":"1",
				"bdMiniList":["qzone","weixin","sqq","tsina","douban","fbook","twi","tieba","linkedin","youdao","print"],
				"bdPic":"",
				"bdStyle":"0",
				"bdSize":"16"
			},
			"slide":{
				"type":"slide",
				"bdImg":"4",
				"bdPos":"right",
				"bdTop":"65.5"
			},
			"image":{
				"viewList":["qzone","sqq","tsina","weixin"],
				"viewText":"分享到：",
				"viewSize":"16"
			}
		};
		with(document)0[(getElementsByTagName('head')[0]||body).appendChild(createElement('script')).src='http://bdimg.share.baidu.com/static/api/js/share.js?v=89860593.js?cdnversion='+~(-new Date()/36e5)];
	/script
</pre>


进入到<a href="http://share.baidu.com/code" target="_blank">百度分享</a>页面中，<br>
简单的话可以使用**自由选择版**，按照提示一步步定制，最后获得script代码；<br>
**Tips:**建议不要选择**划词分享功能**功能，不怎么好用。<br>
也可以选择**专业开发版**，可以自由定制。<br>

## 参考文档
1.<a href="http://share.baidu.com/code" target="_blank">百度分享</a><br> 
2.<a href="http://www.smslit.top/jekyll/2015/07/19/jekyllShare.html#no1" target="_blank">Jekyll博客添加分享功能</a><br>