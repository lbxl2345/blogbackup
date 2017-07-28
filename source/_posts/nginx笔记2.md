---
title: nginx笔记2.md
date: 2016-03-14 17:40:00
tags:
- server
- nginx
- php

---
### nginx多站点配置
nginx提供了一个配置文件:./nginx/cosnf/nginx.conf。在启动时，nginx也只能使用一个配置文件。那么如何把多个站点的配置文件分开呢？对于不同的站点，可以对每个站点分别编写一个conf文件，然后在nginx.conf中，用**include**命令把它们包含起来。通常将其放在两个文件夹下:  

	mkdir ./nginx/sites-available/
	mkdir ./nginx/sites-enabled/
	cp ./nginx/conf/default.conf ./nginx/sites-available/<site>
	#link to the available site to enable it
	ln -s ./nginx/sites-available/<site> ./nginx/sites-enabled/<site>
	＃include enabled sites in nginx.conf
	include ./nginx/sites-enabled/*
	
实际上配置文件可以放在任何位置，只要合理的去include就可以了。

### php-fpm安装及配置  
php5和php5-fpm的安装：
	
	$ sudo apt-get install php5-cli
	$ sudo apt-get install php5-fpm

在与nginx配合使用时，需要对php-fpm的listen端口进行修改，这个值位于php5/fpm/pool.d/www.conf目录下，原始值为listen = /var/run/php5-fpm.sock。这里修改为listen = 127.0.0.1:9000。其实只要保证listen的值和nginx配置文件的地址:端口一致即可，也即在nginx.conf所包含的配置文件中，处理php的server及location中，设置fastcgi_pass php5-fpm.sock。  
配置完成后，启动php5-fpm，利用net-stat命令可以看到对目标地址:端口的监听是否存在，如果存在那么说明php5-fpm是正常工作的 

	$ php5-fpm
	$ net-stat -ano|grep 9000
	
### nginx + php配置
nginx和apache的不同之处在于，其本身不具有处理php的能力，它通过fastcgi来与php-fpm进行交互，来完成php文件的运行。因此为nginx和php设置好通讯端口，让两者能够正常交互是最基本的一步。  
在nginx.conf（及其所包含文件中），可以通过设置server和location，来处理URI中的php请求。
	
	server  {
		listen 80;
		server_name localhost;
		location ~ \.php${
			include fastcgi_params;
			root html;
			fastcgi_pass 127.0.0.1:9000;
			fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		}
	}
	
在配置完成之后，编写一个index.php文件进行测试。

	#index.php
	<?php
	phpinfo();
	?>
-------------
	./nginx -s reload
	php5-fpm
	curl localhost/index.php
	
如果能够显示php的版本信息，说明nginx+php的基本环境搭建成功了。  

### 可能的问题
不能正常处理路径时，修改：

	/etc/php5/fpm/php.ini中cgi.fix_pathinfo = 0;
php5-fpm组和拥护设置不正确：

	;/etc/php5/fpm/pool.d/www.conf
	user = www-data
	group = www-data
	
浏览器中以下载方式打开php文件时：

	/usr/local/nginx/conf/nginx.conf中default_type为text/html;

