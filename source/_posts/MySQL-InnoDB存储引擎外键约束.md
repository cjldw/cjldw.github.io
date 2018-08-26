---
title: MySQL InnoDB存储引擎外键约束
tags: []
date: 2015-08-14 11:22:00
---

## InnoDB存储引擎 外键约束 ##

语法:

```sql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name [CONSTRAINT [symbol]] FOREIGN KEY [index_name] (index_col_name,...)
REFERENCES tbl_name(index_col_name,...) ON DELETE  [RESTRICT | CASCADE | SET NULL | NO ACTION] ON UPDATE [RESTRICT | CASCADE | SET NULL | NO ACTION]
```
[官方文档参考][1]

本文讨论 ON {DELETE , UPDATE} [RESTRICT | CASCADE | SET DEFAULT | NO ACTION | SET NULL]

* ON DELETE | UPDATE 表示父表在发生删除, 更新事件的时候,子表对应的操作
* RESTRICT 表示当子表有数据关联到父表的数据时候, 不能删除父表的数据
* CASCADE  表示删除或更新父表数据的时候, 对应也会相应跟删除或更新子表的数据. Note: 到5.6版本为止, 还不支持事务
* SET DEFAULT MySQL占不支持InnoDB表
* NO ACTION MySQL中的NO ACTION可理解为RESTRICT, 有些数据库延迟检查功能, MySQL是即时检查,所以和RESTRICT一样的
* SET NULL 父表中的删除或更新数据, 子表中的有关联约束的字段设置为NULL

[官方参考文档][1]

[1]: http://dev.mysql.com/doc/refman/5.6/en/create-table-foreign-keys.html