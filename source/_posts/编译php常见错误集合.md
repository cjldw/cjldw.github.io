---
title: 编译php常见错误集合
date: 2018-04-01 09:38:58
desc: 编译php常见的各种错误, 看着头痛, 今天整理下
tags: php compile
---

编译php7的扩展插件, 依赖的东西还蛮多的, 编译起来总是不那么丝滑顺畅, 今天把常见的问题整理下。

<!-- more -->

#### 环境:

- 系统 centos7
- php7.2

#### openSSL 扩展

编译添加openSSL扩展

```bash
    yum install openssl openssl-devel
```

#### bz2 扩展

编译打开了bz2扩展会碰到
```bash
    checking for BZip2 support… yes
    checking for BZip2 in default path… not found
    configure: error: Please reinstall the BZip2 distribution
```
解决方案
```bash
    yun install bzip2 bzip2-devel
```

#### curl 扩展

编译打开了curl扩展会碰上个
```bash
    checking for cURL support… yes
    checking if we should use cURL for url streams… no
    checking for cURL in default path… not found
    configure: error: Please reinstall the libcurl distribution -
    easy.h should be in /include/curl/
```
解决方案
```bash
    yun install curl curl-devel
```

#### gd 扩展

编译开启了gd扩展会碰上
```bash
    checking for fabsf… yes
    checking for floorf… yes
    configure: error: jpeglib.h not found.
    checking for fabsf… yes
    checking for floorf… yes
    checking for jpeg_read_header in -ljpeg… yes
    configure: error: png.h not found.
```
解决方案
```bash
    yum install libpng libpng-devel
```

#### freetype 字体扩展

系统没有 freetype 库
```bash
    checking for png_write_image in -lpng… yes
    If configure fails try –with-xpm-dir=
    configure: error: freetype.h not found.
```
解决方案
```bash
    $ wget https://download.savannah.gnu.org/releases/freetype/freetype-2.4.0.tar.bz2
    $ tar -jxvf freetype-2.4.0.tar.bz2
    $ cd freetype-2.4.0
    $ ./configure
    $ make && make install
```

#### libxml 扩展

系统缺少 libxml 库支持
```bash
    configure: error: xml2-config not found. Please check your libxml2 installation.
```
解决方案
```bash
    yum install libxml libxml-devel
```

#### pcre 类库

系统缺少 pcre 库支持
```bash
    checking for PCRE headers location… configure: error: Could not find pcre.h in /usr
```
解决方案
```bash
    yum install pcre pcre-devel
```

#### t1lib支持

系统缺少t1lib支持
```bash
    configure: error: Your t1lib distribution is not installed correctly. Please reinstall it.
```
解决
```bash
    yum install t1lib-devel.x86_64
```

#### mcrypt 加解密类库

系统缺少 mcrypt 库
```bash
    configure: error: mcrypt.h not found. Please reinstall libmcrypt.
```
解决
```bash
    yum install libmcrypt libmcrypt-devel
```

#### mysql 相关的错误

出现mysql相关的错误总结一点

1. mysql-devel 包需要安装, 基本不会有问题。


