---
title: mysql过程循环结果集
date: 2017-11-13 23:30:36
desc: mysql过程中的循环
tags: mysql, procedure, loop
---

有个场景, 2张数据表, 表1模版表, 表2为具体实现表, 表2中有个字段是模版表的冗余字段, 现要求最快的方式实现表2的冗余字段和表1的数据同步!

<!--more-->

表1结构

```sql
    create table template (
        id int unsigned auto_increment,
        price decimal(8,2) not null default '0.0',

        primary key (id)
    ) engine=innodb charset=utf8 comment '模版表';

```

表2结构

```sql
    create table gifts (
        id int unsigned auto_increment,
        tid int unsigned not null default '0',
        price decimal(8,2) not null default '0.0',
        name varchar(32) not null default "" comment '名称',

        primary key (id)
    ) engine=innodb charset=utf8 comment "具体礼物表";
```

demo数据

![demo](/images/mysql_procedure.jpg)

gifts表的tid关联的是template表的id, 现在要根据tid去获取template的price 同步gifts表的price字段.

方案1 (使用脚本语言, 如`php`, `nodejs`, `python`...)实现

方案2 (使用mysql的过程实现). 本着最方便的原则

```
    drop procedure if exists sync_price;
    delimiter $$
    create procedure sync_price()
    begin
        declare giftId int;
        declare tplId int;
        declare tplPrice decimal(8,2);
        declare isDone int default 0;
        declare giftCursor cursor for select id, tid from gifts; -- 声明游标
        declare continue handler for not found set isDone = 1; -- 捕获 not found,设置isDone=1 表示循环完毕.
        open giftCursor;
        read_loop: loop -- 开始循环
            fetch giftCursor into giftId, tplId;
            if isDone then
                leave read_loop;
            end if;
            select price into tplPrice from template where id = tplId;
            select giftId, tplId, tplPrice; -- just for test
            update gifts set price = tplPrice where id = giftId;
        end loop;
        close giftCursor;
    end;
    $$
    delimiter ;
```

调用过程 `call sync_price()`

**注意: 过程的游标必须在所有变量声明后, 再声明, 否则出现错误**


