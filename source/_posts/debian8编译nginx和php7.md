---
title: debian8.2编译nginx和php7
date: 2017-03-26 22:34:30
desc:
tags:
---

每次编译php都会有新的问题出现, 好累, 暂时记录下. 希望下次编译会快点.
本帖写给自己看的, 写的很简陋, 😄

<!-- more -->

### 准备环境

1. 安装必要的编译器

    ```bash
        sudo apt-get install build-essential
    ```
### nginx

1. 安装pcre8, openssl-dev

2. 编译nginx

    ```bash
        # ./configure --prefix=/opt/app/nginx --with-pcre --with-openssl
        # make && make install
    ```

### mariadb

1. 偷个懒

    ```bash
        # apt-get  install mariadb-server-10.0
    ```

### php7

1. 安装 libjpeg, libpng, freetype libcurl-dev zlib2 libxml libmcrypt libbz2 libz2-dev 每种发行版的名称不一样, 对应搜索吧!! 😄, 没办法

2. 编译php7

    ```bash
        # ./configure --prefix=/opt/app/php7 --with-zlib-dir --with-freetype-dir --enable-mbstring --with-libxml-dir=/usr --enable-soap --enable-calendar --with-curl --with-mcrypt --with-zlib --with-gd --disable-rpath --enable-inline-optimization --with-bz2 --with-zlib --enable-sockets --enable-sysvsem --enable-sysvshm --enable-pcntl --enable-mbregex --enable-exif --enable-bcmath --with-mhash --enable-zip --with-pcre-regex --with-pdo-mysql --with-mysqli --with-mysql-sock=/var/run/mysqld/mysqld.sock --with-jpeg-dir=/usr --with-png-dir=/usr --enable-gd-native-ttf --with-openssl --with-fpm-user=www-data --with-fpm-group=www-data --with-libdir=/lib/x86_64-linux-gnu --enable-ftp  --with-kerberos --with-gettext --with-xmlrpc  --enable-opcache --enable-fpm
        # make && make install
    ```

3. 检验 `/opt/app/php7/bin/php -m`
