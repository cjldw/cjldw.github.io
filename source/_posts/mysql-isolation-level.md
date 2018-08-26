---
title: mysql事务隔离级别
date: 2017-07-10 22:34:05
desc: 最近项目中有使用mysql的事务隔离级别, 回过头来仔细研究了一番.
tags: MySQL, ISOLATION LEVEL
---

一直对mysql的事务隔离级别搞的不是很清楚, 最近项目中有使用mysql的事务隔离级别, 
刚好, 趁这个机会, 好好研究一把. 回过头来仔细研究了一番. 发现有新的感悟, 写点东西记录一番

<!-- more -->

### MySQL的事务隔离级别简单介绍

MySQL默认事务有四中隔离级别, 分别是读未提交(READ UNCOMMITTED), 读提交(READ COMMITTED), 重复读(REPEACT READ), 序列化(SERIALIZE)
MySQL的默认隔离级别是重复读(REPEATABLE READ), 也就是在同一个事务中, 读取的结果是一致的. 查看事务隔离级别 `show variables like 'tx_isolation'`

```sql
    mysql> show variables like '%tx_iso%';

            +---------------+-----------------+
            | Variable_name | Value           |
            +---------------+-----------------+
            | tx_isolation  | REPEATABLE-READ |
            +---------------+-----------------+
```

每个SESSION可以[修改事务的隔离级别](https://dev.mysql.com/doc/refman/5.7/en/set-transaction.html)

```sql
    mysql> SET GLOBAL tx_isolation='REPEATABLE-READ';
    mysql> SET SESSION tx_isolation='SERIALIZABLE';
```

- **读未提交(READ UNCOMMITTED)**

    举个🌰, 如果同时开启A事务, 和B事务, A事务更新一个test表的数据, 没有提交, 此时在B事务中就可以读取到
    A事务修改的结果, 这个隔离级别, 基本不用, 读的数据都不对, 脏读.

    ```SQL
        A事务                                           B事务
        mysql> set session transaction isolation \      mysql> set session transaction isolation \
                level READ UNCOMMITTED                          level READ UNCOMMITTED
        mysql> set autocommit = 0;                      mysql> set autocommit = 0;

        mysql> select * from test;                      mysql> select * from test

            +--------+------+                                   +--------+------+
            | a      | b    |                                   | a      | b    |
            +--------+------+                                   +--------+------+
            | 888888 |    2 |                                   | 888888 |    2 |
            |   8888 |    3 |                                   |   8888 |    3 |
            | 888888 |    2 |                                   | 888888 |    2 |
            |      4 |    3 |                                   |      4 |    3 |
            | 888888 |    2 |                                   | 888888 |    2 |
            +--------+------+                                   +--------+------+

                                                        mysql> update test set a = 1111 where b = 2;

                                                        mysql> select * from test;

                                                                +------+------+
                                                                | a    | b    |
                                                                +------+------+
                                                                |  111 |    2 |
                                                                | 8888 |    3 |
                                                                |  111 |    2 |
                                                                |    4 |    3 |
                                                                |  111 |    2 |
                                                                +------+------+
         mysql> select * from test;

         mysql> -- 事务A尽然读取到了事务B跟新的数据
         mysql> -- 此时事务B还没有提交呢!

            +------+------+
            | a    | b    |
            +------+------+
            |  111 |    2 |
            | 8888 |    3 |
            |  111 |    2 |
            |    4 |    3 |
            |  111 |    2 |
            +------+------+


        ```

- **读取提交(READ COMMITED)**

    读提交, 是指B事务修改了行, 并且B事务提交了, 此时A事务可以读取到B事务修改的内容

    ```SQL
        A事务                                           B事务
        mysql> set session transaction isolation \      mysql> set session transaction isolation \
                level READ COMMITTED                          level READ COMMITTED
        mysql> set autocommit = 0;                      mysql> set autocommit = 0;

        mysql> select * from test;                      mysql> select * from test

            +--------+------+                                   +--------+------+
            | a      | b    |                                   | a      | b    |
            +--------+------+                                   +--------+------+
            | 888888 |    2 |                                   | 888888 |    2 |
            |   8888 |    3 |                                   |   8888 |    3 |
            | 888888 |    2 |                                   | 888888 |    2 |
            |      4 |    3 |                                   |      4 |    3 |
            | 888888 |    2 |                                   | 888888 |    2 |
            +--------+------+                                   +--------+------+

                                                        mysql> update test set a = 1111 where b = 2;

                                                        mysql> select * from test;

                                                                +------+------+
                                                                | a    | b    |
                                                                +------+------+
                                                                |  111 |    2 |
                                                                | 8888 |    3 |
                                                                |  111 |    2 |
                                                                |    4 |    3 |
                                                                |  111 |    2 |
                                                                +------+------+
         mysql> select * from test;

         mysql> -- 事务A读取的还是A事务的数据
         mysql> -- 此时事务B还没有提交呢

            +--------+------+
            | a      | b    |
            +--------+------+
            | 888888 |    2 |
            |   8888 |    3 |
            | 888888 |    2 |
            |      4 |    3 |
            | 888888 |    2 |
            +--------+------+
                                                        mysql> -- 提交事务
                                                        mysql> commit;


        mysql> --事务B提交后, 能够读取到B事务更改
        mysql> --内容了

        mysql> select * from test;

            +------+------+
            | a    | b    |
            +------+------+
            |  111 |    2 |
            | 8888 |    3 |
            |  111 |    2 |
            |    4 |    3 |
            |  111 |    2 |
            +------+------+


        ```
- **重复读 (REPEATABLE READ)**

    重复读是指. B事务修改了行, 并且B事务提交了, 此时A事务还是不能读取到B事务修改的内容
    只有A事务也提交后, 才能读取到B事务修改的行数.

    ```SQL
        A事务                                           B事务
        mysql> set session transaction isolation \      mysql> set session transaction isolation \
                level REPEATABLE READ
        mysql> set autocommit = 0;                      mysql> set autocommit = 0;

        mysql> select * from test;                      mysql> select * from test

            +--------+------+                                   +--------+------+
            | a      | b    |                                   | a      | b    |
            +--------+------+                                   +--------+------+
            | 888888 |    2 |                                   | 888888 |    2 |
            |   8888 |    3 |                                   |   8888 |    3 |
            | 888888 |    2 |                                   | 888888 |    2 |
            |      4 |    3 |                                   |      4 |    3 |
            | 888888 |    2 |                                   | 888888 |    2 |
            +--------+------+                                   +--------+------+

                                                        mysql> update test set a = 1111 where b = 2;

                                                        mysql> select * from test;

                                                                +------+------+
                                                                | a    | b    |
                                                                +------+------+
                                                                |  111 |    2 |
                                                                | 8888 |    3 |
                                                                |  111 |    2 |
                                                                |    4 |    3 |
                                                                |  111 |    2 |
                                                                +------+------+
         mysql> select * from test;

         mysql> -- 事务A读取的还是A事务的数据
         mysql> -- 此时事务B还没有提交呢

            +--------+------+
            | a      | b    |
            +--------+------+
            | 888888 |    2 |
            |   8888 |    3 |
            | 888888 |    2 |
            |      4 |    3 |
            | 888888 |    2 |
            +--------+------+
                                                        mysql> -- 提交事务
                                                        mysql> commit;


        mysql> --事务B提交后, 还是读取A事务的内容

        mysql> select * from test;

             +--------+------+
             | a      | b    |
             +--------+------+
             | 888888 |    2 |
             |   8888 |    3 |
             | 888888 |    2 |
             |      4 |    3 |
             | 888888 |    2 |
             +--------+------+


        mysql> commit;
        mysql> -- 此时可以获取到B事务的修改数据
        mysql> select * from test;

            +------+------+
            | a    | b    |
            +------+------+
            |  111 |    2 |
            | 8888 |    3 |
            |  111 |    2 |
            |    4 |    3 |
            |  111 |    2 |
            +------+------+


        ```


- **序列化 (SERIALIZE) 最高的隔离级别, 代价很大, 很少使用**

    这个级别和重复读(REPEATABLE READ)类似, 唯一不同的是 innodb的SELECT语句, 此隔离级别会显性的将所有的
    SELECT 语句转成 SELECT ..... LOCK IN SHARE 在AUTOCOMMIT 关闭的情况下, 在AUTOCOMMIT 开启模式下
    SELECT 语句在各自的事务中, 读取不会阻塞其他事务, 如果其它事务有对SELECT 行修改, 此时会阻塞SELECT



**READ COMMITTED, 和REPEATABLE READ 的读一致性是[快照模式](https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html), 这样可以防止并发锁导致的死锁问题**




