---
title: php-fpm 调试
date: 2019-07-11 13:57:27
desc: 直接调试php-fpm
tags: php-fpm
---

遇到一个问题， nginx代理php请求到php-fpm, 发现不知道是nginx的问题， 还是php-fpm的问题. 想通过bash直接调试php-fpm.

<!-- more -->

查了下, 发现是可以。记录下。

debian/ubuntu 下安装

```bash
    # apt-get install libfcgi0ldbl
```

centos/redhat 下安装

```bash
    #  yum --enablerepo=epel install fcgi
```

编写脚本测试

```bash
    # cat > php-fpm.sh <<EOF
        #!/bin/bash
        SCRIPT_FILENAME=/var/www/html/site/index.php \
        REQUEST_URI=/api/user/wealth.json \
        QUERY_STRING=uid=121923823&_=1828188212 \
        REQUEST_METHOD=GET \
        cgi-fcgi -bind -connect php-fpm-host:9000
      EOF
```

测试

```bash
    # bash -x php-fpm.sh
```

这个就是nginx代理过去的请求是原理是一样的, nginx传的变量会更多点。

```nginx
fastcgi_param	QUERY_STRING		$query_string;
fastcgi_param	REQUEST_METHOD		$request_method;
fastcgi_param	CONTENT_TYPE		$content_type;
fastcgi_param	CONTENT_LENGTH		$content_length;

fastcgi_param	SCRIPT_FILENAME		$request_filename;
fastcgi_param	SCRIPT_NAME		$fastcgi_script_name;
fastcgi_param	REQUEST_URI		$request_uri;
fastcgi_param	DOCUMENT_URI		$document_uri;
fastcgi_param	DOCUMENT_ROOT		$document_root;
fastcgi_param	SERVER_PROTOCOL		$server_protocol;

fastcgi_param	GATEWAY_INTERFACE	CGI/1.1;
fastcgi_param	SERVER_SOFTWARE		nginx/$nginx_version;

fastcgi_param	REMOTE_ADDR		$remote_addr;
fastcgi_param	REMOTE_PORT		$remote_port;
fastcgi_param	SERVER_ADDR		$server_addr;
fastcgi_param	SERVER_PORT		$server_port;
fastcgi_param	SERVER_NAME		$server_name;

fastcgi_param	HTTPS			$https if_not_empty;

```
