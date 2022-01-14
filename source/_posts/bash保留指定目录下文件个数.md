---
title: BASH保留指定目录下指定个数文件
date: 2017-07-12 22:31:51
desc: 某个目录至保存指定个数文件, 可以根据脚本删除多余的文件
tags: BASH, BACKUP
---

线上有一台服务器, 每天凌晨3点做数据库的全量备份, 然后打包到指定目录, 前几天, 磁盘下满了, 查看该目录下
文件多达2个多月了, 不行, 要定期清理.

<!--more-->

### 获取该目录下需要被删除的文件列表

由于该目录下备份文件有规则, 为`database_YYYYMMDD.tgz`, 因此列出需要删除的文件 `ls | sort -r | sed -n '30,$p'`

`sort -r` 逆向排序, 默认会根据`YYYYMMDD`来循序排列
`sed -n '30,$p'` sed将30行到地步的行打印出来
`rm file` 最后删除列出的文件名称.

```bash

    #!/bin/bash
    # Author: luowen
    # Email: bigpao.luo@gmail.com
    # Description: only keep one mouth db backup file
    # Time: 2017-07-12 22:00:00

    DB_BACKUP_DIR=/var/data/mysqldb

    for expire_file in `ls $DB_BACKUP_DIR | sort -r | sed -n '30,$p'`
    do
        rm "$DB_BACKUP_DIR/$expire_file"
    done;
```


**这是最简单的方法, 还有就是当前时间-mtime, 如果时间大于一个月的, 就删除.**
**方法还很多, 详细就不列出来了**

