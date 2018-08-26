---
title: bash脚本定期创建MySQL增量表
date: 2017-12-07 23:18:18
desc: bash根据如期创建日期增量表
tags: bash, mysql
---

在日常业务中, 会将mysql表根据日期创建分表, 最简单的方法就是写个脚本每个月定时跑一遍创建下一个月的增量表。编写脚本有很多，这里就介绍用bash来实现。个人认为这是最快， 代码量、依赖最少的方法。

<!--more-->

bash优点是代码量少，简单。但依赖了`mysql`客户端(并不是要求安装MySQL服务，简单的方法就是从安装好MySQL服务的服务器，将`msyql`文件拷贝下来就好)。

废话不多说了, 贴码

```bash
#!/bin/bash
# @author: luowen<bigpao.luo@gmail.com>
# @desc: 自动创建数据库日期增量表
# @time: 2017-12-06

MYSQL_HOST=127.0.0.1
MYSQL_USER=root
MYSQL_PWD=111111
MYSQL_PORT=3306

TABLE_LIST=($1) # 以参数的形式接受表名字
echo $TABLE_LIST

resultCode=0 # 是否成功标识
for table_prefix in ${TABLE_LIST[*]}
do
    pre_table=$table_prefix$(date "+%Y%m")
    new_table=$table_prefix$(date -d "+1 month" "+%Y%m")
    sql="mysql -u${MYSQL_USER} -h${MYSQL_HOST} -P${MYSQL_PORT} -p${MYSQL_PWD} -e 'create table ${new_table} like ${pre_table}'"
    echo $sql
    #eval $sql
    if [ $? -gt 0 ]
    then
        $resultCode=$?
    fi
done
exit $resultCode
```

执行:

```bash
    $ bash table-generate.sh "log.userlogin_log_ log.userip_log_" # 参数已空格隔开
```

**PS:bash的数组没有关联数组**
