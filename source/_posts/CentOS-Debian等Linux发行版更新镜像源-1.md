---
title: CentOS,Debian等Linux发行版更新镜像源
tags: []
date: 2015-07-25 11:31:00
---

安装后CentOS镜像源,默认使用国外的,很慢,你知道的, 国内更换163、aliyun (看了很多人的博客, 今天有空,自己写一篇)

<!-- more -->

## 找到想要更换的源的官方网站, 比如

* [网易163](http://mirrors.163.com/)页脚找到帮助中心 &nbsp;&nbsp;

* [阿里云](http://mirrors.aliyun.com/)找到对应的发行版本,后面有个**help**可以点击进去看帮助

* 以CentOS6.x做例子, 并使用163的源

    ```bash
        mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup # 备份
        cd /etc/yum.repos.d/
        wget http://mirrors.163.com/.help/CentOS6-Base-163.repo # 下载163源repo
        yum clean all  # 清除缓存
        yum makecache # 构建最新的缓存数
    ```
* 仅供参考, 欢迎拍砖:)
