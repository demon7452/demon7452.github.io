---
layout: post
title: zTree -- jQuery 树插件
category: 技术
tags: javascript
keywords: 
description: 
---
#zTree -- jQuery 树插件
##zTree 简介
zTree 是一个依靠 jQuery 实现的多功能 “树插件”。优异的性能、灵活的配置、多种功能的组合是 zTree 最大优点。
##zTree的简单使用过程
1、首先是导入 zTree的相关js库和css

```
<link rel="stylesheet" href="zTreeStyle.css" type="text/css">
<script type="text/javascript" src="jquery.ztree.core-3.5.js"></script>
<script type="text/javascript" src="jquery.ztree.excheck-3.5.js"></script>
```

2、在js中写好的树的相关配置

```
	var setting = {
		
		check: {
			enable: true,
			chkStyle: "checkbox",  //设置为checkbox类型的树
			chkDisabledInherit: false,
			chkboxType:{ "Y" : "s", "N" : "" } //设置勾选和取消勾选时的父子关系
		},
		data: {
			simpleData: {
				enable: true
			},
			key:
			{
				title:"id"
			}	
		}
	};
```

3、在js中设置树要显示的数据

```
#数据格式可以是Array:
	var zTreeNodes =[
				{ id:1, pId:0, name:"随意勾选 1", open:true},
				{ id:11, pId:1, name:"随意勾选 1-1", open:true}
				];
#或者是json:
	var zTreeNodes = [{"id":"1", "name":"test1","chkDisabled":true,open:true,"children":[{"id":"2","name":"test2",open:true}]}];
```
4、调用zTree的js库中的方法初始化树

```
#初始化树时必须有一个标签的class 为 ztree
	<ul id="treeDemo" class="ztree"></ul>
#使用之前创建的 配置setting 和 数据zTreeNodes 初始化树，且生成一个树对象
	var treeObj = $.fn.zTree.init($("#treeDemo"), setting, zTreeNodes);
```

5、获取在树中勾选的数据

```
#通过初始化树时生成的树对象treeObj，获取被勾选的数据
	function getTree()
	{
		//var treeObj = $.fn.zTree.getZTreeObj("#treeDemo");
		var nodes = treeObj.getCheckedNodes(true);
		var permissions="";
		for(i in nodes)
		{
			permissions += nodes[i].id + ";";
		}

		$("#list").text(permissions+"test");
	}
```
##参考文档
1.<a href="http://www.ztree.me/v3/main.php#_zTreeInfo" target="_blank">zTree -- jQuery 树插件</a><br> 
2.<a href="http://negativo17.org/steam/" target="_blank">Moving the Steam client installation</a><br>
3.<a href="http://askubuntu.com/questions/256628/steam-fatal-error-steam-needs-to-be-online-to-update-but-was-set-to-offline-mov" target="_blank">Steam fatal error steam needs to be online to update, but was set to offline movies</a><br>
