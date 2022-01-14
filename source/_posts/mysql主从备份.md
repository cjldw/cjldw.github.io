---
title: mysql主从备份
date: 2016-09-15 13:45:37
desc:
tags:
---

MySQL的主从复制配置很简单, 帖子已经很多了, 再不厌其烦的写主要是为了下篇配合zabbix做监控存活做准备.

<!-- more -->

### 先说mysql备份策略

* 冷备份(不能实时备份)

    1. 将mysql的dataDir目录打包scp/rsync 到备份服务器. 优点: 简单暴力. 冷备的缺点, mysql版本一致, 后期数据量大了, 备份越来越久.
    2. 使用mysqldump, (mysqlcopy, mysqlbackup企业版) 自带工具备份, 有点: 简单, 一条命令就ok, mysql版本能向上兼容, 数据量大, 备份慢.


* 热备份(及时备份)

    1. mysql主从复制, 主mysql开启bin-log, 从mysql读取主bin-log文件名和位置, 写入到从库中(mysql版本需要一致)
    2. inotify+rsync inotify检测文件是否发生变动, 如变动使用rsync备份到远程.(使用所有备份, 不只是mysql业务)
    3. drbd (分布式块设备) 即时写入.(使用所有备份, 不只是mysql业务)


### 配置mysql主从配置.

* 英文好的同学, 直接参照[官方文档](https://dev.mysql.com/doc/refman/5.6/en/replication.html)

* 默认环境

    ```sql
        192.168.1.100 主mysql5.6
        192.168.1.101 从mysql5.6
    ```

* 主mysql配置

    1. 编辑my.cnf. (参考多目录下my.cnf的优先级)

        ```sql
            bind-address=192.168.1.100
            server-id=1
            bin_log=/var/log/mysql/mysql-bin.log
            binlog_do_db=newdatabase # 需要同步的数据库, 不写, 表示全部库
        ```
    2. 添加具有复制权限的用户

        ```sql
            GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%' IDENTIFIED BY 'slave_password';
            flush privileges; # 刷新权限
        ```

    3. 重启动mysql服务 `service mysqld restart | systemctl restart mysqld`

    4. 锁定当前主库的数据状态, 将当前数据状态备份到从库中去.

        ```sql
            USE DATABASE;
            FLUSH TABLES WITH READ LOCK;

            SHOW MASTER STATUS;
            +------------------+----------+--------------+------------------+
            | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
            +------------------+----------+--------------+------------------+
            | mysql-bin.000001 |      107 | newdatabase  |                  |
            +------------------+----------+--------------+------------------+

            # 此窗口不要动了(如果有改动, 会自动解除锁), 新开一个窗口, 将数据库导出
            # 新窗口mysql回话
            mysqldump -uUsername -pPassword newdatabase > newdatabase-dump.sql

            exit; # 主配置完成, 关闭退出.
        ```

* 从mysql配置

    1. 导入主mysql数据

        ```sql
            mysql -uUser -pPassword newdatabase < newdatabase-dump.sql
        ```

    2. 编辑从my.cnf

        ```sql
            server-id=2
            relay-log=/var/log/mysql/mysql-relay-bin.log
            log_bin=/var/log/mysql/mysql-bin.log
            binlog_do_db=newdatabase
        ```

    3. 重启从mysql服务 `service mysqld restart | systemctl restart mysqld`

    4. 设置从mysql连接主mysql

        ```sql
            CHANGE MASTER TO MASTER_HOST='192.168.1.100',
            MASTER_USER='slave',
            MASTER_PASSWORD='slavepassword',
            MASTER_LOG_FILE='mysql-bin.000001',
            MASTER_LOG_POS=  107;

            start slave;
        ```

    5. 查看是否成功

        ```sql
            SHOW SLAVE STATUS\G

            # 出现一下 表明已经ok了
            Slave_IO_Running: Yes
            Slave_SQL_Running: Yes

        ```

* 仅作参考, 有问题欢迎拍砖. :)
