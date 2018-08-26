---
title: SELECT ... FOR UPDATE
date: 2017-08-19 08:23:32
desc: mysql select for update 详解
tags: mysql, SQL
---

这几天遇到一个问题, 关于购票库存问题. 如何能够在并发情况下减去正确的库存量. 如A请求过来, 查询库存100,
判断大于购买的量10(假设), 可以购买.  B请求过来查询库存也是100, 然后也可以购买. 然后A请求减去了库存10,
数据库更新成功.就在同时, B请求也减去了库存10, 这样就问题就出来了。

<!-- more -->

总体的思路是要让操纵库存的部分串行处理.

**方案1** 没次下单请求过来, 加入到队列中, 每次减库存的时候, 判断下队列中的请求数量超过库存量的时候, 不接受其他下单请求.
这个方案php实现起来不是很容易, php没发记录请求数. 能想到的使用swoole来做.

**方案2** 在数据库层串行化, 使用`SELECT .... FOR UPDATE`实现,(前提MYSQL是InnoDB引擎, 并且每个请求都在一个独立的事务中)
原理是 A请求过来查询库存 `SELECT * FROM goods WHERE id = 1 FOR UPDATE`;
此时ID = 1的数据行已经加了共享锁, _此所不阻塞普通的`SELECT`_, 另外B请求过来 同样执行了`SELECT * FROM goods WHERE id = 1 FOR UPDATE`;
此时, B请求会被阻塞, 知道A请求操作完成, 事务提交后, 才能拿到锁, 执行下面的操作. 举个🌰

```sql

    A请求                                                  B请求
    mysql> set autocommit = 0;                             mysql> set autocommit = 0;
    Query OK, 0 rows affected (0.00 sec)                   Query OK, 0 rows affected (0.00 sec)
    mysql> select * from tx_test;                          mysql> select * from tx_text;
    +------+------+                                        +------+------+
    | a    | b    |                                        | a    | b    |
    +------+------+                                        +------+------+
    | 9999 |    2 |                                        | 9999 |    2 |
    | 8888 |    3 |                                        | 8888 |    3 |
    | 9999 |    2 |                                        | 9999 |    2 |
    |    4 | 1000 |                                        |    4 | 1000 |
    | 9999 |    2 |                                        | 9999 |    2 |
    +------+------+                                        +------+------+
    5 rows in set (0.00 sec)                               5 rows in set (0.00 sec)

    mysql> select * from tx_test where a = 4 for update;   mysql> select * from tx_test where a = 4 for update;
    +---+------+
    | a | b    |
    +---+------+
    | 4 | 1000 |
    +---+------+
    1 row in set (0.00 sec)

    mysql> update tx_test set b = 1 where a = 4;
    Query OK, 1 row affected (0.00 sec)
    Rows matched: 1  Changed: 1  Warnings: 0
    mysql> commit;                                         +---+------+
    Query OK, 0 rows affected (0.09 sec)                   | a | b    |
                                                           +---+------+
                                                           | 4 |    1 |
                                                           +---+------+
                                                           1 row in set (24.81 sec)
```

看到B请求最后的查询, 会阻塞, 知道A请求更新玩数据后, 并且提交后, 才能拿到结果。
总的来说, `SELECT ... FOR UPDATE` 能确保获取的最新的更新后的数据。
