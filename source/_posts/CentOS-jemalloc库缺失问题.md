---
title: CentOS jemalloc库缺失问题
tags: []
date: 2015-09-14 11:45:00
---

今天编译redis, 报jemalloc库缺失, 百度,google 得出一下解答, 亲测OK。

<!-- more -->

## jemalloc 库安装 ####

* 最近编译redis, varnish老薄jemalloc缺失.

* 到[网站](https://dl.fedoraproject.org/pub/epel/6/x86_64/repoview/jemalloc.html)下载对应的rpm包

    ```bash
       rpm -ivh jemalloc-3.6.0-1.el6.{i686 | X86_64}
    ```

* 在编译一次
