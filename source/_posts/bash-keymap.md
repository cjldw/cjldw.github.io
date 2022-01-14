---
title: bash中的快捷键使用指南
date: 2017-09-22 09:57:27
desc:
tags:
---

当你使用终端使用bash的时候, 发现写了一大长命令的是, 比如编译php, 突然发现中间个个单词写错了, 然后要跳到那块去修改, 没办法, 按着方向键不放, 慢慢移动到那个位置修改完后执行。 妈蛋, 发现又有一处写错了, 有要慢慢移啊移啊！ 是不是很尴尬。

<!--more-->

1. 解决方案一

bash自带有很多快捷键。 参考 [shell-tips](https://www.shell-tips.com/sheets/bash-help-sheet.pdf)

![BASH_HELP](/images/bash_shortkeymap.jpg)

2. 解决方案二

编辑`.bashrc` 使用vim模式, 这个需要了解vim的快捷键了。

```bash
    $ vim ~/.bashrc

        set -o vi
    $ source ~/.bashrc
```
