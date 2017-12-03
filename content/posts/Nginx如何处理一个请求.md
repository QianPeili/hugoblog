---
title: "【翻译】 Nginx 如何处理一个请求"
date: 2017-06-09T20:12:05+08:00
category: 
 - 文档
tags: 
 - nginx
---

原文：[How nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)

网上不乏好的翻译，我再做一次翻译，只是为了让自己更好地理解，也方便以后自己翻阅。

## Name-based virtual servers

nginx 首先需要判断哪个 Server 来处理这个请求。让我们来看一个简单的配置文件，里面定义了3个虚拟服务器，并且都监听了80端口。

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


这份配置文件测试的情况是，nginx 根据请求头中的 Host 字段，选择将请求导向哪个 Server。如果Host的值不匹配所有 server\_name,或者值为空，那么 nginx 会将请求导向监听这个端口的默认虚拟 Server。如上的配置文件，nginx 会默认把第一个 Server 选为默认 Server。当然也可以通过配置 listen 指令的 `default_server` 参数，来指定默认 Server。


	server {
	    listen      80 default_server;
	    server_name example.net www.example.net;
	    ...
	}


## How to prevent processing requests with undefined server names

如果想要阻止没有Host头字段的请求，可以通过配置一个如下Server来实现：

	server {
	    listen      80;
	    server_name "";
	    return      444;
	}

在这里，`server_name` 字段被设置为空字符串，没有Host头字段的请求会被匹配到这个 Server，然后返回一个nginx非标准的 `444code` 用来关闭连接。

> 从 0.8.48 版本以后，server\_name 的默认设置就是空字符串，所以 server\_name "" 这一行配置可以省略。在之前的版本，server\_name 会使用主机名来作为默认值。

## Mixed name-based and IP-based virtual servers

下面来看一个更复杂一点的例子，里面的Server监听了不同的地址。

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

在这个配置中，nginx 首先测试请求的 ip 地址和端口, 与配置中的服务所监听的地址和端口进行对比，找到匹配的服务后，再判断 Host 头字段的值与 server\_name 字段是否匹配。如果没有 server\_name 与其匹配，就会将请求导向默认服务器。举个例子，一个被 192.168.1.1：80 端口接收的，访问 `www.example.com` 的请求，会被监听在 192.168.1.1:80 端口的默认服务器，即上述第一个 Server，所接收。因为192.168.1.1:80 端口的服务中，没有一个 server\_name 是`www.example.com`。

正如之前所说，默认服务是所监听端口的属性，所以不同的监听端口，可以设置不同的默认服务。

	server {
	    listen      192.168.1.1:80;
	    server_name example.org www.example.org;
	    ...
	}
	
	server {
	    listen      192.168.1.1:80 default_server;
	    server_name example.net www.example.net;
	    ...
	}
	
	server {
	    listen      192.168.1.2:80 default_server;
	    server_name example.com www.example.com;
	    ...
	}

## A simple PHP site configuration

现在，通过一个典型的小型PHP站点配置，我们来观察一下nginx是如何选择一个location来处理request的。

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

nginx 首先会从被列出的 location中，根据正则找到前缀字符串匹配度最精确的 location。在如上配置中，前缀只包含了一个 `"/"` 的location可以匹配任何请求，但优先度也是最低的。 nginx 会根据配置中列举的顺序用正则表达式检查 locations。一旦 nginx 找到第一个完全匹配的表达式，就会停止查找然后使用这个 location。如果没有找到完全匹配的表达式，nginx会使用之前找到的匹配度最高的 location。

注意，这些匹配规则只测试请求的 URI 部分，不包括参数部分。这是因为请求参数在可以有多种组合方式，比如：

	/index.php?user=john&page=1
	/index.php?page=1&user=john

另外请求参数是可以存在任意内容的

	/index.php?page=1&something+else&user=john

现在我们来看一下，用以上的配置，请求会被如何处理。

- `"/log.gif"` 请求一开始匹配location "/", 然后正则匹配于 `"\.(gif|jpg|png)$"`，因此，它会被后一个处理。通过 "root /data/www" 命令，请求被映射到 /data/www/logo.gif 文件，服务器会被这个文件发送给客户端。
- “/index.php”请求同样先匹配location “/” 然后匹配 `“\.(php)$”` 。因此，它会被后一个 location 处理，请求被传到监听在 localhost:9000 的 FastCGI 服务上。 `fastcgi_param` 命令将 FastCGI 参数 `SCRIPT_FILENAME` 的值设置为 “/data/www/index.php”, FastCG I会执行这个文件. `$document_root` 变量的值等同于 root 命令的值，$fastcgi_script_name的值等同于请求所指向的资源, 即 “/index.php”.

- `“/about.html”` 请求只匹配location “/” ，因此, 它会被第一个location处理。“root /data/www”指令会是请求被映射到文件——/data/www/about.html, 文件被发到客户端.

- `“/”` 请求会处理的负责一点。它只匹配 location “/”, 所以它被第一个location处理。 然后 `index` 指令会尝试寻找在 “root /data/www” 所设置的目录是否存在这些文件。 如果文件 /data/www/index.html 不存在， 但是 /data/www/index.php 文件存在，这个指令就会将请求重定向到 “/index.php”， 接着 nginx 重新为这个请求匹配对应的 location，就向客户端直接发送了这个请求。正如前面所看到的，这个请求会被导向FastCGI server。