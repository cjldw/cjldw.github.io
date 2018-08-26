---
title: MySQL单台服务器跑多个实例子
tags: []
date: 2015-12-22 16:18:00
---

单台数据服务器资源往往是跑不满的, 如何更有效的利用服务器资源? 可以当台数据服务器跑多个MySQL实例。

<!-- more -->

## mysqld_multi 命令实现管理 ##

1. 每个mysql实例都需要有独立的数据目录, socket链接, 端口. 所以, 为每个实例创建数据目录

    * 到mysql的数据目录下, 复制整个目录(有几个实例,就复制几份), 命名为新的实例名字(任意) (本例子拷贝了3份,分别放在/var/data/data{1,2,3}, Unix, Unix-like 注意文件权限)

    * 多个实例可以共享一份数据目录, 但强烈不建议, 会有意想不到的问到[mysql官方](http://dev.mysql.com/doc/refman/5.7/en/multiple-data-directories.html)

2. 修改my.cnf文件(本例子跑了3个实例子分别[1,2,3])

    ```sql
        [client]
        # 默认 使用/tmp/mysql.sock1
        socket = /tmp/mysql.sock1

        [mysqld1]
        socket     = /tmp/mysql.sock1
        port       = 3306
        pid-file   = /var/data/data1/data1.pid
        datadir    = /var/data/data1
        user       = mysql

        [mysqld2]
        #mysqld     = /usr/bin/mysqld_safe
        #mysqladmin = /usr/bin/mysqladmin
        socket     = /tmp/mysql.sock2
        port       = 3307
        pid-file   = /var/data/data2/data2.pid
        datadir    = /var/data/data2
        user	   = mysql

        [mysqld3]
        #mysqld     = /usr/bin/mysqld_safe
        #mysqladmin = /usr/bin/mysqladmin
        socket     = /tmp/mysql.sock3
        port       = 3308
        pid-file   = /var/data/data3/data3.pid
        datadir    = /var/data/data3
        user	   = mysql
    ```

3. 启动/停止

    * `mysqld_multi [OPTIONS] {start|stop|report} [GNR,GNR,GNR...]` **GNR** 表示组名, 本例子启动所有为`mysqld_multi start 1-3`, 启动单个 `mysqld_multi start 1`

    * 同理 停止 `mysqld_multi stop 1-3`

##  mysqld_safe 实现 ##

1. 这种方式实现很简单了, 直接使用命令

    ```sql
        mysqld_safe --defaults-file=/etc/my1.cnf --user=mysql &
        mysqld_safe --defaults-file=/etc/my2.cnf --user=mysql &
        mysqld_safe --datadir=/var/data/data3 --pid-file=/var/data/data3/data3.pid --socket=/tmp/mysql.sock3 --user=mysql &
    ```
