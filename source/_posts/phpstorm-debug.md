---
title: phpstorm-debug
date: 2016-10-31 23:59:31
desc:
tags:
---

phpstorm 是一个php开发利器,本博客简单的描述如何使用phpstrom2016.2实现phpweb应用debug. 简单单个php脚本不做解释.
英文好的同学直接阅读[官方文档](https://www.jetbrains.com/help/phpstorm/2016.2/configuring-xdebug.html)会更加详细

<!-- more -->


### 前提

* 下载xdebug扩展包, (windows下载对应的xdebug.dll), unix-like下载xdebug.so或自行编译(phpize)不多介绍

* 配置xdebug注入到php引擎中, 实现断点拦截调试

* 配置守护进程模式

* 配置运行时(just in time) 模式


####  下载debug

* 到[xdebug](https://xdebug.org/)下载对应的版本到ext目录下

* 编辑`php.ini`文件, 找到对应的[xdebug]端, 加入xdebug扩展

    ```php
        [xdebug]
        zend_extension=php_xdebug.dll
        xdebug.remote_enable=1
        xdebug.remote_connect_back=1
        xdebug.remote_host=127.0.0.1
        xdebug.remote_port=9999
        debug.show_local_vars=0
        xdebug.var_display_max_data=10000
        xdebug.var_display_max_depth=20
        xdebug.show_exception_trace=0
    ```

    **注意`xdebug_remote_port=9999` 默认是9000, 由于我本地php-cgi跑的端口是9000, 所以修改了, 后面对应的phpstrom也要修改默认9000 =&gt; 9999端口.**

* 重启服务, apache(mod)形式重启apache, nginx+fpm, 重启fpm


#### 配置ide注入

* 打开`File|Settings`对话框, 选择`Languages&Frameworks.`, 在php页面选择`interpreter`, 选择本地安装的php脚本.

    ![xdebug interpreter](/images/debug/1.png)

* 如果有修改默认`xdebug_remote_port`, 修改默认phpstorm默认debug端口

    ![phpstrom default debug port](/images/debug/2.png)

* 添加默认服务地址

    ![default php host](/images/debug/3.png)

* 新建调试页面(一个服务地址下, 可以配置多个页面调试)

    ![create debug page](/images/debug/4.png)


* 在对应的php文件打上断点后, 点击debug按钮后, 会打开默认浏览器访问上一部配置的地址, 然后再回到phpstrom, php已经停止在断点处了. 之后慢慢调试咯. 效果如图

    ![OK](/images/debug/5.png)

    **这样就实现了phpweb应用的调试了, 不用var_dump($var);exit;调试了**


PS: 有问题欢迎拍砖, 本人亲测没有问题.
