---
title: swoole-webapp-environment-build
date: 2017-02-20 16:00:40
desc:
tags:
---

准备做一个基于swoole的webapp框架项目, 但是swoole在windows机器上是跑不起来的, 思想向后, 用vagrant包装一个
自己的box,后期重用度更高点.

<!-- more -->

# 使用预安装好的box

1. 下载安装virtualbox[地址](https://www.virtualbox.org/wiki/Downloads)
2. 下载安装vagrant[地址](https://www.vagrantup.com/)
3. vagrant init luowen/swoole-webapp; vagrant up --provider=virtualbox

-----

# 一下是安装过程, 可忽略

## 下载php7.1 编译安装

1. 安装必要的依赖 `pcre freetype jpeg png xml curl mcrypt`(太多了, 不写)
2. 编译php

    ```bash
    /configure --prefix=/usr/local/php7 \
        --with-config-file=/usr/local/php7/etc/php.ini \
        --with-config-scan-dir=/usr/local/php7/etc/conf.d \
        --enable-bcmath \
        --with-curl \
        --with-gd \
        --enable-filter \
        --enable-fpm \
        --enable-gd-native-ttf \
        --with-freetype-dir \
        --with-jpeg-dir \
        --with-png-dir \
        --with-intl \
        --enable-mbstring \
        --with-mcrypt \
        --enable-mysqlnd \
        --with-mysqli=mysqlnd \
        --with-pdo-mysql=mysqlnd \
        --with-openssl \
        --enable-zip \
        --enable-zlib \
        --enable-xmlreader \
        --enable-xmlwriter
    ```
3. 下载[swoole包](https://github.com/swoole/swoole-src/archive/v2.0.6.tar.gz), 安装php swoole支持 (google phpize)

    ```bash
        phpize \
       ./configure --with-php-config=/usr/local/php7/bin/php-config \
       --enable-coroutine \
       --enable-thread \
       --enable-openssl \
       --enable-swoole
    ```


4. 编辑php.ini 添加模块查看模块

    ```bash
        /usr/local/php7/bin/php -m | grep swoole
    ```

5. 加载了swoole后,明天继续, 累了!
