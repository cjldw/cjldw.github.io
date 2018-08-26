---
title: 理解linux-sed 命令
date: 2017-11-22 22:36:22
desc: linux sed命令
tags: linux, sed
---

一直不是和明白linux的sed(stream editor)命令用法, 每次要用的时候, 脑子一篇空白, 有的查一下. 来来去去搞了很多次. 今天总结下常用的用法, 加深下记忆吧.

<!--more-->

linux sed是linux下一个很强的文本处理工具, 能轻松处理上G的文本文件

* 替换: 将制定的问

    ```bash
        $ sed 's/hello/world' file 
    ```

* 多个匹配规则可使用`-e`

    ```bash
        $ sed -e "s/hello/world; s/bash/sh" file
    ```

* 替换的几个flag `s/search/replace/{flag}`,

    - g 全局替换 `sed 's/old/new/g' file` 全局替换
    - 数字, 制定替换几次 `sed 's/old/new/10' file` 替换10次
    - p 打印原始行 `sed 's/old/new/p' file` 
    - w 替换后,写入到源文件(慎用) `sed 's/old/new/w' file`

* 替换制定行

    ```bash
        $ sed '10s/old/new/p' file # 匹配前10行
        $ sed '1, 100s/old/new/p' file # 替换1~100行的数据
        $ sed '2, $s/old/new/p' file 替换2到末尾行的文本
    ```

* 查找替换

    ```bash
        $ sed '/old/s/hello/world/' file # 查找有old词的行,并替换此行hello->world
    ```

* 删除行

    ```bash
        $ sed '3d' file # 　删除3行开始以后的行
        $ sed '3, 10d' file # 删除3~10行的数据
        $ sed '3, $d' file # 删除3~末行的数据
    ```

* 查找删除

    ```bash
        $ sed '/old/d' file # 匹配old的行删除
        $ sed '/old/, /hello/d' file # 匹配old或hello的行删除
    ```

* 插入文本

    ```bash
        $ cat file
            111111111
            222222222
            333333333
        $ sed 'i\luowen' file
            luowen
            111111111
            luowen
            222222222
            luowen
            333333333
        $ sed 'a\luowen' file
            111111111
            luowen
            222222222
            luowen
            333333333
            luowen
        $ sed `2i\luowen` file
            111111111
            luowen
            222222222
            333333333
        $ sed `2a\luowen` file
            111111111
            222222222
            luowen
            333333333
    ```

* 修改行

    ```bash
        $ cat file
            111111111
            222222222
            333333333
        $ sed 'c\luowen' file
            luowen
            luowen
            luowen
    ```

* 字符替换

    ```bash
        cat file
            111111
            222222
            333333
        $ sed 'y/123/456/' file
            444444
            555555
            666666
    ```

* 打印行号

    ```bash
        $ sed '=' file
        $ sed -n '/test/=' file
    ```

**欢迎补充**



