---
title: centos7编译apache2.4+php5.6+mysql5.7
date: 2017-05-14 10:37:37
desc: centos7编译apache2.4 php5.6 mysql5.7构建开发环境
tags:
---

先说点对本地开发环境新得. 对于开发环境, 我个人还是偏喜欢在本地环境搭建, 如果本地环境是window, 最需要的是
文件名大小写敏感问题. 不知不觉踩了好几次坑了. 也曾想过跑个虚拟机跑开发环境, 代码通过共享解决(virtualbox, vagrant...), 本地环境和
线上环境一致的问题. 但真实用起来还是不爽. 前几天有个网友让班忙做个镜像, 本着助人为乐的态度就弄了.

<!-- more -->

## 准备环境 (CentOS7 minimal IOS)

* 切换阿里云镜像源

    ```bash
        # mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
        # wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
        # yum makecache
    ```
如果没有安装`wget` 也可使用 `curl /etc/yum.repos.d/CentOS-Base.repo.backup -o /etc/yum.repos.d/CentOS-Base.repo`

* 安装必要的编译工具

    ```bash
        # yum groupinstall "Development Tools"
    ```

## 编译apache2.4

* 下载[apache2.4源码](http://apache.fayea.com//httpd/httpd-2.4.25.tar.bz2)

    ```bash
        # cd /opt/softs
        # tar -jxvf httpd-2.4.25.tar.bz2
    ```


* 下载[apr](http://mirrors.tuna.tsinghua.edu.cn/apache//apr/apr-1.5.2.tar.gz) 拷贝到 `path/to/apache2.4/srclib/apr`

    ```bash
        # cd /opt/softs
        # wget http://mirrors.tuna.tsinghua.edu.cn/apache//apr/apr-1.5.2.tar.gz
        # tar -zxvf apr-1.5.2.tar.gz
        # mv apr-1.5.2 /opt/softs/httpd-2.4.25/srclib/apr
    ```

* 下载[apr-util](http://mirrors.tuna.tsinghua.edu.cn/apache//apr/apr-util-1.5.4.tar.gz) 拷贝到 `/opt/apache2.4/srclib/apr-util`

    ```bash
        # cd /opt/softs
        # wget http://mirrors.tuna.tsinghua.edu.cn/apache//apr/apr-util-1.5.4.tar.gz
        # tar -zxvf apr-util-1.5.4.tar.gz
        # mv apr-util-1.5.4 /opt/softs/httpd-2.4.25/srclib/apr-util
    ```

* 下载[pcre](https://ftp.pcre.org/pub/pcre/pcre-8.00.tar.bz2) 编译安装

    ```bash
        # cd /opt/softs
        # wget https://ftp.pcre.org/pub/pcre/pcre-8.00.tar.bz2
        # tar -jxvf pcre-8.00.tar.bz2
        # cd pcre-8.00 
        # ./configure  --enable-utf8 --enable-unicode-properties
        # make && make install
    ```

* 编译apache2.4

    ```bash
        # cd /opt/softs/httpd-2.4.25
        # ./configure --prefix=/opt/apache2.4 --prefix=/opt/apache2.4 --with-inlucde-apr
        # make && make install
    ```

* 编写systemd启动脚本. SysV不适用, CentOS7默认启用了systemd管理服务. 可以使用

    ```bash
        # vi /etc/systemd/system/httpd.service

            [Unit]
            Description = apache2.4 service
            After = network.target

            [Service]
            Type = forking
            ExecStart = /opt/apache2.4/bin/apachectl -k start

            [Install]
            WantedBy = multi-user.taget

        # systemctl start http
    ```

* 检查是否服务启动

    ```bash
        # curl http://localhost
            <html><body><h1> It works! </h1></body></html>
    ```

* 注意: 如果不是本机访问不了, 查看是否是iptables关系. CentOS7-minimal IOS默认有防火墙策略

    1. 清空所有防火墙策略 `iptables -F` (开发环境推荐)
    2. 或添加一条规则 `iptables -t filter -I INPUT 1 -p tcp --dport=80 -j ACCEPT`. 所有到目标端口80的tcp协议全部放行.


## 安装mysql5.7

完全编译mysql5.7是个大工程, 我这篇幅写不完, 偷个懒

* 下载官方编译好的[MySQL5.7](https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.18-linux-glibc2.5-x86_64.tar.gz)二进制包解压即可.

    ```bash
        # cd /opt/softs/
        # wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.18-linux-glibc2.5-x86_64.tar.gz
        # tar -zxvf mysql-5.7.18-linux-glibc2.5-x86_64.tar.gz
        # mv mysql-5.7.18-linux-glibc2.5-x86_64 /opt/mysql5.7
    ```

* 初始化mysql

    ```bash
        # cd /opt/mysql5.7
        # useradd mysql -s /sbin/nologin
        # mkdir data && chown mysql:mysql data -R
        # bin/mysqld --initialize-insecure
    ```

* 配置`/etc/my.cnf`

    ```bash
        # vi /etc/my.cnf

            [client]
            socket=/tm/mysql.socket
            port=3306

            [mysqld]
            basedir=/opt/mysql5.7
            datadir=/opt/mysql5.7/data
            socket=/tmp/mysql.socket
            pid-file=/tmp/mysql.pid

            server-id=1
            user=mysql
            character-set-server=utf8
            collection_server=utf8_general_ci
            log_error=/opt/mysql5.7/data/error.log
            log_bin=/opt/mysql5.7/data/log-bin.log

    ```

* 编写systemd启动脚本. SysV不适用, CentOS7默认启用了systemd管理服务. 可以使用

    ```bash
        # vi /etc/systemd/system/mysql.service

            [Unit]
            Description = MySQL service
            After = network.target

            [Service]
            Type = forking
            ExecStart = /opt/mysql5.7/bin/mysqld --daemonize

            [Install]
            WantedBy = multi-user.taget

        # systemctl start mysql
    ```

 * 测试是否启动

     ```bash
        # /opt/mysql5.7/bin/mysql -uroot -p
     ```


## 编译php5.6

* 下载[php5.6](http://hk1.php.net/get/php-5.6.30.tar.bz2/from/this/mirror)

    ```bash
        # cd /opt/softs/
        # wget http://hk1.php.net/get/php-5.6.30.tar.bz2/from/this/mirror
        # tar -jxvf php-5.6.30.tar.gz2
        # cd php-5.6.30
        # ./configure --prefix=/opt/php5.6 \
            
    ```
* 安装必要的依赖

    ```bash
        # yum install libxml2-devel freetype-devel libpng-devel openssl-devel 
        # cd /opt/softs/php-5.6.30
        # ./configure --prefix=/opt/php5.6 --with-apxs2=/opt/apache2.4/bin/apxs \
            --with-zlib=/usr \
            --enable-bcmath \
            --enable-calendar \
            --enable-mbstring \
            --enable-mbregex \
            --with-mysql=mysqlnd
        # make && make install
    ```

* 添加mine类型到apache

    ```bash
        # vi /opt/apache2/conf/httpd.conf

            AddType text/html .php .phtml
            AddType application/x-httpd-php .php .phtml
    ```

* 测试是否ok

    ```bash
        # echo "<?php phpinfo(); " > /opt/apache2.4/htdocs/start.php
        # curl http://localhost/start.php
    ```



## 重启启动服务

* 启动mysql `systemctl start mysql`
* 启动apache `sysmtectl start httpd`
