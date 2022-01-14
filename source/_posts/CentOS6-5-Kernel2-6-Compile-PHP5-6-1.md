---
title: CentOS6.5(Kernel2.6) Compile PHP5.6
tags: []
date: 2015-08-30 00:15:00
---

今天手动编译php5.6, CentOS6.x yum源的php版本就的掉渣。 虽然网上教程多的很, 但自己还是记录下吧。

<!-- more -->


>CentOS的镜像源php版本是5.3,好旧，使用5.6

*  **到php官方网站下载最新的phpxxx.tar.bz2 [php official website](http://php.net),后使用命令**

      wget http://bg2.php.net/get/php-5.6.12.tar.bz2/from/this/mirror php-5.6.12.tar.bz2

*  **解压包**

      tar -jxvf php-5.6.12.tar.bz2 && cd php-5.6.12

* **安装编译php依赖**

      yum install gcc bison bison-devel zlib-devel libmcrypt-devel mcrypt mhash-devel openssl-devel libxml2-devel libcurl-devel bzip2-devel readline-devel libedit-devel sqlite-devel

* **编译**

      --prefix=/usr/local/php56 \
      -with-config-file-path=/usr/local/php56/etc \
        --enable-inline-optimization \
        --disable-debug \
        --disable-rpath \
        --enable-shared \
        --enable-opcache \
        --enable-fpm \
        --with-fpm-user=www \
        --with-fpm-group=www \
        --with-mysql=mysqlnd \
        --with-mysqli=mysqlnd \
        --with-pdo-mysql=mysqlnd \
        --with-gettext \
        --enable-mbstring \
        --with-iconv \
        --with-mcrypt \
        --with-mhash \
        --with-openssl \
        --enable-bcmath \
        --enable-soap \
        --with-libxml-dir \
        --enable-pcntl \
        --enable-shmop \
        --enable-sysvmsg \
        --enable-sysvsem \
        --enable-sysvshm \
        --enable-sockets \
        --with-curl \
        --with-zlib \
        --enable-zip \
        --with-bz2 \
        --with-readline   \
       --with-apxs2=/usr/sbin/apxs \ 

      (参数[**_--with-apxs2=FILE _**]php是以模块形式加入到httpd服务中运行需要家此参数, (nginx做web服务时候是不需要的) 如果没有apxs命令,使用yum install httpd-devel包)

* **httpd添加php模块支持,找多对应httpd配置文件,(etc. vim /etc/httpd/conf/httpd.conf)添加如下行**

      DirectoryIndex index.html index.html.var index.php

      LoadModule php5_module        /usr/lib64/httpd/modules/libphp5.so

      PHPIniDir /usr/local/php56/etc/php.ini

      AddType application/x-httpd-php .php   

* **测试是否成功**

      [sudo] service httpd start

      echo '<?php phpinfo();' > /var/www/html/index.php (ps: /var/www/html 为CentOS默认路径,根据自己配置的,对应配置即可)
