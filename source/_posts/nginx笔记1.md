---
title: nginx笔记1.md
date: 2016-03-11 17:40:00
tags:
- server
- nginx

---
### nginx 启动/终止/重载入
启动:运行主程序即启动  
控制:

	nginx -s [signal]
	signal:stop, quit, reload, ropen
	应用新的配置:
	nginx -s reload
	
### 配置文件
nginx包含由配置文件中的**directives**（simple, block）来控制的模块。包含其他directives的block被称为context。不被任何context包含的directives，被称为main context。例如：events和http在main context中，server在http中，location在server中。  

几个context包括有:  
通常一个配置文件会包含有数个通过ports和server name区分的**server**。对于一个request，nginx会先确定用哪一个server，然后测试其header中的URI是否与**server**中的**location**中项一致。  

	http{
		server{
			location / {
				root /data/www;
			}
			location /imamges/ {
				root /data;
			}
		}
	}
**location**指明了用来与URI比对的前缀，对于吻合的请求来说，URI会被添加上**root**中指定的根路径，来形成完整本地文件系统的路径。  

### FastCGI server
nginx能够讲requests导向FastCGI server。FastCGI server能够运行使用不同框架和编程语言（例如PHP）编写的程序。最为基本的的FastCGI server配置方法是使用fastcgi_pass命令，SCRIPT_FILENAME参数指定了脚本文件的名称，QUERY_STRING参数用来传递request parameters。

	server {
	    location / {
		        fastcgi_pass  localhost:9000;
		        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		        fastcgi_param QUERY_STRING    $query_string;
		    }
			
		    location ~ \.(gif|jpg|png)$ {
		        root /data/images;
	    	}
		}


### nginx如何处理request
nginx首先确定处理request的server，例如配置3个server:

	server {
    listen      80;
    server_name example.org www.example.org;
    ...
	}

	server {
    listen      80;
    server_name example.net www.example.net;
    ...
	}

	server {
    listen      80;
    server_name example.com www.example.com;
    ...
	}

nginx检测request的**header**域“Host”来确定它应该被指向哪一个server。如果request不吻合任何一个server name，或者不包含header，那么会被导向该端口默认的port。对于没有header的request，可以设置一个空的server来专门drop他们，例如，其中code 444表示关闭connection。

	server{
		listen 80;
		server_name "";
		return 444;
	}
	
在多个server监听不同IP地址的情况下，nginx首先会检测request的ip地址和port是否符合server中的listen命令。对于不同的ip，default server可以是不同的。

	server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    ...
	}

	server {
    listen      192.168.1.1:80;
    server_name example.net www.example.net;
    ...
	}

	server {
    listen      192.168.1.2:80;
    server_name example.com www.example.com;
    ...
	}

那么nginx如何选择处理request的location呢？以一个简单的php网站为例。nginx首先搜索和字符串最长匹配的前缀。在下面的设置中，唯一的前缀位置时"/"，由于它会匹配任何request，所以它会被视为最靠后的选择。然后nginx检查配置文件中，通过正则表达式给出的locations列表。第一次成功匹配就会终止搜索，nginx会使用这个location。nginx只会检查request的URI部分。举例说明:  

- request "/logo.gif"首先匹配了"/"，然后匹配了正则式"\.(git|jpg|png)$"，因此它会选择后一个location，通过指令"root /data/www"request会被映射到文件/data/www/logo.gif中。  
- reqeust "/index.php"首先匹配了"/"，然后匹配了正则式"\.(php)$"。因此request会被交给监听localhost:9000的FastCGI server。fastcgi_param命令将其脚本名设置"/data/www/index.php",随后FastCGI server执行这个文件。（$document_root变量和root命令指定的相同，$fastcgi_script_name与request的URI相同，也即/index.php）。  
- request ".about.html"只和前缀location"/"匹配，因此，它只会在这个location中被处理。使用“root /data/www”，request被映射到"data/www/about.html"，然后文件被发送到client。  
- request “/”的处理更为复杂，它只会匹配前缀location"/"，nginx会首先通过index指令来检测时候好存在index文件，如果/data/www/index.html不存在，但/data/www/index.php存在，那么FastCGI server会将其重定位到"index.php"，随后nginx会重新搜索这个location，就像这个request是被用户发送的一样。

-------- 
 


	server {
	    listen      80;
	    server_name example.org www.example.org;
	    root        /data/www;
	
	    location / {
	        index   index.html index.php;
	    }
	
	    location ~* \.(gif|jpg|png)$ {
	        expires 30d;
    	}

	    location ~ \.php$ {
	        fastcgi_pass  localhost:9000;
	        fastcgi_param SCRIPT_FILENAME
	                      $document_root$fastcgi_script_name;
	        include       fastcgi_params;
	    }
	}
	


