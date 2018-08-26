---
title: 简单的mysql过程实现数据行累加
date: 2016-09-10 13:04:37
desc:
tags:
---

今天碰到mysql的一个问题, 下一行的一个字段需要累加上一行的字段的值

    ```sql
        +----+------+-----+           +----+------+--------------+
        | id | name | age |           | id | name | age          |
        +----+------+-----+           +----+------+--------------+
        | 1  | lily | 10  |           | 1  | lily | 10           |
        +----+------+-----+    =>     +----+------+--------------+
        | 2  | lucy | 28  |           | 2  | lucy | 28 +10       |
        +----+------+-----+           +----+------+--------------+
        | 3  | tom  | 80  |           | 3  | tom  | 80 + 28 + 10 |
        +----+------+-----+           +----+------+--------------+
    ```


<!-- more -->


### 如何实现它

* 可以使用java, php, python等脚本语言轻松实现, 不讨论.

* 今天讨论用sql语言实现它. 要实现这个东东, 第一感觉想到的是过程

    1. 编写过程

            ```sql

                delimiter ;;

                create procedure demo()

                begin

                declare done int default 0;
                declare prevalue int default 0;
                declare vid int;
                declare currentvalue int;

                -- 声明游标

                declare demo_cursor cursor for select id, age from demo;
                declare continue handler for not found set done = 1;

                -- 打开游标

                open demo_cursor;

                -- 循环操作
                read_loop: LOOP

                    fetch invest into vid, currentvalue;
                    set prevalue := prevalue+currentvalue;

                    if done then
                        leave read_loop;
                    end if;

                    update demo set age = prevalue where id = id;

                end LOOP;
                CLOSE demo_cursor;

                end;
                ;;

                delimiter ;
            ```
=======

    1. 编写过程

        ```sql

            delimiter ;;

            create procedure demo()

            begin

            declare done int default 0;
            declare prevalue int default 0;
            declare vid int;
            declare currentvalue int;

            -- 声明游标

            declare demo_cursor cursor for select id, age from demo;
            declare continue handler for not found set done = 1;

            -- 打开游标

            open demo_cursor;

            -- 循环操作
            read_loop: LOOP

                fetch invest into vid, currentvalue;
                set prevalue := prevalue+currentvalue;

                if done then
                    leave read_loop;
                end if;

                update demo set age = prevalue where id = id;

            end LOOP;
            CLOSE demo_cursor;

            end;
            ;;

            delimiter ;
        ```

    2. 执行过程 `call demo();`

    3. 删除过程 `drop procedure demo;`



就这样实现了, 欢迎拍砖.











