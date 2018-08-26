---
title: windows10下使用mintty代替cmd仿linux终端
date: 2016-11-09 00:40:15
desc: windowns10 安装cgywin实现linux终端
tags:
---

大部分的开发者都都是在windows下开发的(包括本人), 但开发的很多时候, 我们都希望我们windows的cmd窗口能想unix-like系统那样功能强大. 怎么办呢? 今天来实现它.

<!-- more -->

#### 来一发!

* 废话不多说, 来一张预览图

    ![预览](/images/tty/1.png)

    **还不错吧**


#### 如何搭建

* 到[官方网站](http://www.cygwin.com/install.html) 下载安装

* 下载安装后, 将安装目录下的`bin, sbin`到环境变量中(*_非必须_*), 这样是为了cmd下也可以使用linux命令.

* 将bin目录下的mintty.exe 穿件快捷方式, 移动到`c:\windows\system32\cmmd` 以后启动直接使用`win+r` 输入cmmd即可.

* 字体设置, 窗口透明, 右击打开的窗口, 自行设置即可.

* 是不是很简单, 很实用, 逼格一下上升了很多了吧





