---
title: mysqlbinlog恢复数据实战
date: 2016-09-24 22:52:25
desc: 数据异常删除, 如何恢复
tags: MySQL
---

前几天, 快凌晨接到总监电话 说线上出现问题了, 感觉大不妙. 排查问题, 发现程序bug, 把不该删除的数据, 删的一干二净, 关键还不是软删除, 头大了. 如何恢复.
登入线上服务器, 发现有备份包, 感觉不错, 很快发现备份文件只到了月份, 感觉不好了. 在看下, 发现有binlog日志, 感觉有希望了. 今天在本地重现一下情况. 

<!-- more -->


### 数据构造

* 创建一个测试库、表

    ```sql
        create database data_backup default character utf8 default collate utf8_unicode_ci;
        create table goods_feature(
        id int unsigned auto_increment,
        parent_id int unsigned not null default 0,
        main_feature varchar(32) not null default '',
        sub_feature varchar(32) not null default '',
        primary key (id)
        ) engine innodb default charset utf8 default collate utf8_unicode_ci;

    ```
* 编写存储过程, 执行`call buildTestData("what you want", "how many you need create")` 插入你想要的数据.

    ```sql
        delimiter //

        create procedure buildTestData(in feature VARCHAR(32), in subTimes int)
        begin
            declare pid int unsigned default 0;
            declare pfeature varchar(32) default '';
            declare insertId int unsigned default 0;

            insert into goods_feature (main_feature) values (feature);
            select last_insert_id() into insertId;
            select main_feature into pfeature from goods_feature where id = insertId;

            SUBSECTION: loop
            if subTimes <= 0 then
                leave SUBSECTION;
            end if;
            insert into goods_feature (parent_id, sub_feature) values (insertId, concat(pfeature, '-', subTimes));
            set subTimes = subTimes - 1;
            end loop SUBSECTION;
        end; //

        delimiter ;

    ```

### 模拟线上数据流

* 一下登入到终端操作模拟成线上数据流.

    ```sql
        mysql> show binary logs;
        +------------------+-----------+
        | Log_name         | File_size |
        +------------------+-----------+
        | mysql-bin.000001 |       177 |
        | mysql-bin.000002 |     12757 |
        +------------------+-----------+
        2 rows in set (0.00 sec)

        mysql> show procedure status where db='data_backup';
        +-------------+---------------+-----------+----------------+---------------------+---------------------+---------------+---------+----------------------+----------------------+--------------------+
        | Db          | Name          | Type      | Definer        | Modified            | Created             | Security_type | Comment | character_set_client | collation_connection | Database Collation |
        +-------------+---------------+-----------+----------------+---------------------+---------------------+---------------+---------+----------------------+----------------------+--------------------+
        | data_backup | buildTestData | PROCEDURE | root@localhost | 2016-09-24 23:53:33 | 2016-09-24 23:53:33 | DEFINER       |         | utf8mb4              | utf8mb4_general_ci   | utf8_unicode_ci    |
        +-------------+---------------+-----------+----------------+---------------------+---------------------+---------------+---------+----------------------+----------------------+--------------------+
        1 row in set (0.01 sec)

        mysql> call buildTestData("鞋子", 3);
        Query OK, 1 row affected (0.63 sec)

        mysql> select * from goods_feature;
        +----+-----------+--------------+-------------+
        | id | parent_id | main_feature | sub_feature |
        +----+-----------+--------------+-------------+
        |  1 |         0 | 鞋子         |             |
        |  2 |         1 |              | 鞋子-3      |
        |  3 |         1 |              | 鞋子-2      |
        |  4 |         1 |              | 鞋子-1      |
        +----+-----------+--------------+-------------+
        4 rows in set (0.00 sec)

        mysql> call buildTestData("袜子", 1);
        Query OK, 1 row affected (0.29 sec)

        mysql> select * from goods_feature;
        +----+-----------+--------------+-------------+
        | id | parent_id | main_feature | sub_feature |
        +----+-----------+--------------+-------------+
        |  1 |         0 | 鞋子         |             |
        |  2 |         1 |              | 鞋子-3      |
        |  3 |         1 |              | 鞋子-2      |
        |  4 |         1 |              | 鞋子-1      |
        |  5 |         0 | 袜子         |             |
        |  6 |         5 |              | 袜子-1      |
        +----+-----------+--------------+-------------+
        6 rows in set (0.00 sec)

        mysql> call buildTestData("外套", 4);
        Query OK, 1 row affected (0.70 sec)

        mysql> call buildTestData("秋裤", 3);
        Query OK, 1 row affected (0.57 sec)

        mysql> call buildTestData("篮球", 5);
        Query OK, 1 row affected (0.64 sec)

        mysql> call buildTestData("足球", 3);
        Query OK, 1 row affected (0.31 sec)

        mysql> call buildTestData("排球", 3);
        Query OK, 1 row affected (0.38 sec)

        mysql> select * from goods_feature;
        +----+-----------+--------------+-------------+
        | id | parent_id | main_feature | sub_feature |
        +----+-----------+--------------+-------------+
        |  1 |         0 | 鞋子         |             |
        |  2 |         1 |              | 鞋子-3      |
        |  3 |         1 |              | 鞋子-2      |
        |  4 |         1 |              | 鞋子-1      |
        |  5 |         0 | 袜子         |             |
        |  6 |         5 |              | 袜子-1      |
        |  7 |         0 | 外套         |             |
        |  8 |         7 |              | 外套-4      |
        |  9 |         7 |              | 外套-3      |
        | 10 |         7 |              | 外套-2      |
        | 11 |         7 |              | 外套-1      |
        | 12 |         0 | 秋裤         |             |
        | 13 |        12 |              | 秋裤-3      |
        | 14 |        12 |              | 秋裤-2      |
        | 15 |        12 |              | 秋裤-1      |
        | 16 |         0 | 篮球         |             |
        | 17 |        16 |              | 篮球-5      |
        | 18 |        16 |              | 篮球-4      |
        | 19 |        16 |              | 篮球-3      |
        | 20 |        16 |              | 篮球-2      |
        | 21 |        16 |              | 篮球-1      |
        | 22 |         0 | 足球         |             |
        | 23 |        22 |              | 足球-3      |
        | 24 |        22 |              | 足球-2      |
        | 25 |        22 |              | 足球-1      |
        | 26 |         0 | 排球         |             |
        | 27 |        26 |              | 排球-3      |
        | 28 |        26 |              | 排球-2      |
        | 29 |        26 |              | 排球-1      |
        +----+-----------+--------------+-------------+
        29 rows in set (0.00 sec)

        mysql> delete from goods_feature where parent_id=0;
        Query OK, 7 rows affected (0.14 sec)

        mysql> select * from goods_feature;
        +----+-----------+--------------+-------------+
        | id | parent_id | main_feature | sub_feature |
        +----+-----------+--------------+-------------+
        |  2 |         1 |              | 鞋子-3      |
        |  3 |         1 |              | 鞋子-2      |
        |  4 |         1 |              | 鞋子-1      |
        |  6 |         5 |              | 袜子-1      |
        |  8 |         7 |              | 外套-4      |
        |  9 |         7 |              | 外套-3      |
        | 10 |         7 |              | 外套-2      |
        | 11 |         7 |              | 外套-1      |
        | 13 |        12 |              | 秋裤-3      |
        | 14 |        12 |              | 秋裤-2      |
        | 15 |        12 |              | 秋裤-1      |
        | 17 |        16 |              | 篮球-5      |
        | 18 |        16 |              | 篮球-4      |
        | 19 |        16 |              | 篮球-3      |
        | 20 |        16 |              | 篮球-2      |
        | 21 |        16 |              | 篮球-1      |
        | 23 |        22 |              | 足球-3      |
        | 24 |        22 |              | 足球-2      |
        | 25 |        22 |              | 足球-1      |
        | 27 |        26 |              | 排球-3      |
        | 28 |        26 |              | 排球-2      |
        | 29 |        26 |              | 排球-1      |
        +----+-----------+--------------+-------------+
        22 rows in set (0.00 sec)

        mysql> call buildTestData("第二天商品", 2);
        Query OK, 1 row affected (0.20 sec)

        mysql> call buildTestData("第二天面膜", 2);
        Query OK, 1 row affected (0.22 sec)

        mysql> call buildTestData("第二天手机", 2);
        Query OK, 1 row affected (0.29 sec)

        mysql> select * from goods_feature;
        +----+-----------+-----------------+-------------------+
        | id | parent_id | main_feature    | sub_feature       |
        +----+-----------+-----------------+-------------------+
        |  2 |         1 |                 | 鞋子-3            |
        |  3 |         1 |                 | 鞋子-2            |
        |  4 |         1 |                 | 鞋子-1            |
        |  6 |         5 |                 | 袜子-1            |
        |  8 |         7 |                 | 外套-4            |
        |  9 |         7 |                 | 外套-3            |
        | 10 |         7 |                 | 外套-2            |
        | 11 |         7 |                 | 外套-1            |
        | 13 |        12 |                 | 秋裤-3            |
        | 14 |        12 |                 | 秋裤-2            |
        | 15 |        12 |                 | 秋裤-1            |
        | 17 |        16 |                 | 篮球-5            |
        | 18 |        16 |                 | 篮球-4            |
        | 19 |        16 |                 | 篮球-3            |
        | 20 |        16 |                 | 篮球-2            |
        | 21 |        16 |                 | 篮球-1            |
        | 23 |        22 |                 | 足球-3            |
        | 24 |        22 |                 | 足球-2            |
        | 25 |        22 |                 | 足球-1            |
        | 27 |        26 |                 | 排球-3            |
        | 28 |        26 |                 | 排球-2            |
        | 29 |        26 |                 | 排球-1            |
        | 30 |         0 | 第二天商品      |                   |
        | 31 |        30 |                 | 第二天商品-2      |
        | 32 |        30 |                 | 第二天商品-1      |
        | 33 |         0 | 第二天面膜      |                   |
        | 34 |        33 |                 | 第二天面膜-2      |
        | 35 |        33 |                 | 第二天面膜-1      |
        | 36 |         0 | 第二天手机      |                   |
        | 37 |        36 |                 | 第二天手机-2      |
        | 38 |        36 |                 | 第二天手机-1      |
        +----+-----------+-----------------+-------------------+
        31 rows in set (0.00 sec)

        mysql> delete from goods_feature where parent_id=0;
        Query OK, 3 rows affected (0.17 sec)

        mysql> call buildTestData("第三天火锅底料", 2);
        Query OK, 1 row affected (0.27 sec)

        mysql> call buildTestData("第三天苹果", 2);
        Query OK, 1 row affected (0.26 sec)

        mysql> call buildTestData("第三天凤梨", 2);
        Query OK, 1 row affected (0.24 sec)

        mysql> call buildTestData("第三天香蕉", 2);
        Query OK, 1 row affected (0.26 sec)

        mysql> select * from goods_feature;
        +----+-----------+-----------------------+-------------------------+
        | id | parent_id | main_feature          | sub_feature             |
        +----+-----------+-----------------------+-------------------------+
        |  2 |         1 |                       | 鞋子-3                  |
        |  3 |         1 |                       | 鞋子-2                  |
        |  4 |         1 |                       | 鞋子-1                  |
        |  6 |         5 |                       | 袜子-1                  |
        |  8 |         7 |                       | 外套-4                  |
        |  9 |         7 |                       | 外套-3                  |
        | 10 |         7 |                       | 外套-2                  |
        | 11 |         7 |                       | 外套-1                  |
        | 13 |        12 |                       | 秋裤-3                  |
        | 14 |        12 |                       | 秋裤-2                  |
        | 15 |        12 |                       | 秋裤-1                  |
        | 17 |        16 |                       | 篮球-5                  |
        | 18 |        16 |                       | 篮球-4                  |
        | 19 |        16 |                       | 篮球-3                  |
        | 20 |        16 |                       | 篮球-2                  |
        | 21 |        16 |                       | 篮球-1                  |
        | 23 |        22 |                       | 足球-3                  |
        | 24 |        22 |                       | 足球-2                  |
        | 25 |        22 |                       | 足球-1                  |
        | 27 |        26 |                       | 排球-3                  |
        | 28 |        26 |                       | 排球-2                  |
        | 29 |        26 |                       | 排球-1                  |
        | 31 |        30 |                       | 第二天商品-2            |
        | 32 |        30 |                       | 第二天商品-1            |
        | 34 |        33 |                       | 第二天面膜-2            |
        | 35 |        33 |                       | 第二天面膜-1            |
        | 37 |        36 |                       | 第二天手机-2            |
        | 38 |        36 |                       | 第二天手机-1            |
        | 39 |         0 | 第三天火锅底料        |                         |
        | 40 |        39 |                       | 第三天火锅底料-2        |
        | 41 |        39 |                       | 第三天火锅底料-1        |
        | 42 |         0 | 第三天苹果            |                         |
        | 43 |        42 |                       | 第三天苹果-2            |
        | 44 |        42 |                       | 第三天苹果-1            |
        | 45 |         0 | 第三天凤梨            |                         |
        | 46 |        45 |                       | 第三天凤梨-2            |
        | 47 |        45 |                       | 第三天凤梨-1            |
        | 48 |         0 | 第三天香蕉            |                         |
        | 49 |        48 |                       | 第三天香蕉-2            |
        | 50 |        48 |                       | 第三天香蕉-1            |
        +----+-----------+-----------------------+-------------------------+
        40 rows in set (0.00 sec)

        mysql> delete from goods_feature where parent_id = 0;
        Query OK, 4 rows affected (0.19 sec)

        mysql> delete from goods_feature where id > 0;

    ```

* 数据本来运行好好的, 但是中途出现删除**parent_id=0**的删除语句, 这样导致数据关联失效了.


### 解决方案

* 锁表, 防止二次伤害

    ```sql
        lock table goods_feature write;
    ```
* 找到出现问题的起始时间, 和结束时间, 来到处sql恢复数据, (当然也可以根据binglog日志的position来判断, 此处不讨论, 自己研究)

    ```sql
        mysqlbinlog --start-datetime="2016-09-24 00:00:00" --end-datetime="2016-09-25 59:59:59" --verbose mysql-bin.000002 > backup.sql
    ```
    **`--verbose`** 参数很重要, 他能找到对应的binlog是删除, 还是更新, 还是插入. 编辑backup.sql找到类似这块的语句, 将BINLOG 'xxxxxxxxuasdlfjsluuuuuuuuwo==' ~ COMMIT这段删除.

        ```sql
            BINLOG '
            maTmVxMBAAAARgAAAHxUAAAAAHIAAAAAAAEAC2RhdGFfYmFja3VwAA1nb29kc19mZWF0dXJlAAQD
            Aw8PBGAAYAAAbhCKAA==
            maTmVyABAAAAmgAAABZVAAAAAHIAAAAAAAEAAgAE//ABAAAAAAAAAAbpnovlrZAA8AUAAAAAAAAA
            BuiinOWtkADwBwAAAAAAAAAG5aSW5aWXAPAMAAAAAAAAAAbnp4voo6QA8BAAAAAAAAAABuevrueQ
            gwDwFgAAAAAAAAAG6Laz55CDAPAaAAAAAAAAAAbmjpLnkIMAZm+dHA==
            '/*!*/;
            ### DELETE FROM `data_backup`.`goods_feature`
            ### WHERE
            ###   @1=1
            ###   @2=0
            ###   @3='鞋子'
            ###   @4=''
            ### DELETE FROM `data_backup`.`goods_feature`
            ### WHERE
            ###   @1=5
            ###   @2=0
            ###   @3='袜子'
            ###   @4=''
            ### DELETE FROM `data_backup`.`goods_feature`
            ### WHERE
            ###   @1=7
            ###   @2=0
            ###   @3='外套'
            ###   @4=''
            ### DELETE FROM `data_backup`.`goods_feature`
            ### WHERE
            ###   @1=12
            ###   @2=0
            ###   @3='秋裤'
            ###   @4=''
            ### DELETE FROM `data_backup`.`goods_feature`
            ### WHERE
            ###   @1=16
            ###   @2=0
            ###   @3='篮球'
            ###   @4=''
            ### DELETE FROM `data_backup`.`goods_feature`
            ### WHERE
            ###   @1=22
            ###   @2=0
            ###   @3='足球'
            ###   @4=''
            ### DELETE FROM `data_backup`.`goods_feature`
            ### WHERE
            ###   @1=26
            ###   @2=0
            ###   @3='排球'
            ###   @4=''
            # at 21782
            #160925  0:06:49 server id 1  end_log_pos 21813 CRC32 0x83e74fc6 	Xid = 451
            COMMIT/*!*/;
        ```

    **最后登入mysql, 执行`source /path/to/backup.sql`, 执行ok后, `unlock tables`**

* 不出意外, 数据全回来了.


* 遗留的问题, 当数据文件过大时, 类似好好几个G, 普通文本编辑器是没法打开的怎么办呢?







