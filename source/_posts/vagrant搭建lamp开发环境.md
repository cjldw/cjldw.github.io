---
title: 使用vagrant快速搭建分享开发环境 
tags: []
date: 2016-05-09 13:20:00
---

开发往往有这么一个问题, 自己本地环境和线上的环境不一致, 本地好的代码上到服务器上后, 各种问题。
有没有一个好的方法能让本地环境和线上环境一样呢或是尽可能一样呢。其实是有的vagrant是一个很好的解决方案,
最近火的docker也是可以实现的。今天主要将vagrant.

<!-- more -->


### 本案列操作环境

1. windows7 vagrant1.8.1 virtualbox4.3

### 安装virtualbox4.x(或更高)

1. virtualbox, win7下傻瓜式安装, 无需配置, 略过.  

### 安装vagrant

1. 下载vagrant软件包 [地址](https://www.vagrantup.com/downloads.html), 安装(windows傻瓜式安装, 不多说).  

2. 创建vagrant工程目录, 初始化Vagrantfile.  

        F:\> mkdir vagrant
        F:\> cd vagrant
        F:\vagrant>vagrant.exe init
            A `Vagrantfile` has been placed in this directory. You are now
            ready to `vagrant up` your first virtual environment! Please read
            the comments in the Vagrantfile as well as documentation on
            `vagrantup.com` for more information on using Vagrant.

3. 添加box(类似docker的容器), 到[官方源](https://atlas.hashicorp.com/boxes/search)下载需要的box, 本案例下载(puppetlabs/centos-6.6-64-nocm)(https://atlas.hashicorp.com/puppetlabs/boxes/centos-6.6-64-nocm),公司服务器是centos6.x.  

        F:\vagrant>vagrant box add puppetlabs/centos-6.6-64-nocm
        ==> box: Loading metadata for box 'puppetlabs/centos-6.6-64-nocm'
            box: URL: https://atlas.hashicorp.com/puppetlabs/centos-6.6-64-nocm
        This box can work with multiple providers! The providers that it
        can work with are listed below. Please review the list and choose
        the provider you will be working with.

        1) virtualbox
        2) vmware_desktop

        Enter your choice: 1
        ==> box: Adding box 'puppetlabs/centos-6.6-64-nocm' (v1.0.3) for provider: virtualbox
            box: Downloading: https://atlas.hashicorp.com/puppetlabs/boxes/centos-6.6-64-nocm/versions/1.0.3/providers/virtualbox.box
            box: Progress: 0% (Rate: 27052/s, Estimated time remaining: 3:23:32)==> box: Waiting for cleanup before exiting...

            box: Progress: 0% (Rate: 23813/s, Estimated time remaining: 3:26:33)
        ==> box: Box download was interrupted. Exiting.

    ps: 慢的有的一B啊, 果断复制链接, 迅雷下载, 安装*.box文件. 速度还能接受.  
        ![download](http://images2015.cnblogs.com/blog/451327/201605/451327-20160509131846359-688138673.png)

    box命令略解: `vagrant box add [可选参数] <官方源名称, url链接地址, 下载后的*.box文件>`   

    安装box文件(秒干):  

            F:\vagrant>vagrant box add --name puppetlabs/centos-6.6-64-nocm  F:\boxes\virtualbox.box
            ==> box: Box file was not detected as metadata. Adding it directly...
            ==> box: Adding box 'puppetlabs/centos-6.6-64-nocm' (v0) for provider:
                box: Unpacking necessary files from: file://F:/boxes/virtualbox.box
                box: Progress: 100% (Rate: 658M/s, Estimated time remaining: --:--:--)
            ==> box: Successfully added box 'puppetlabs/centos-6.6-64-nocm' (v0) for 'virtualbox'!

4. 使用刚下在的box

    1. 编辑vagrant工程目录下的Vagrantfile文件  

            15行: config.vm.box = "puppetlabs/centos-6.6-64-nocm"

    2. 启动vagrant(此处会启动virtualbox, 预先安装virtualbox, win7安装有点小问题, 请自行google或百度解决)  

            vagrant up

    3. ssh上去(默认F:/vagrant 自动挂载到/vagrant)上  

            F:\vagrant>vagrant ssh
            Last login: Fri May  6 02:33:11 2016 from 10.0.2.2
            [vagrant@localhost ~]$ ls /
            bin  boot  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  sbin  selinux  srv  sys  tmp  usr  vagrant  var
            [vagrant@localhost ~]$

    4. 在/vagrant创建的文件, 会同步到 F:\>vagrant下.  

### Centos Box 安装lamp环境

> 本案列使用 apache2.4, php5.6.12 mysql5.7 编译安装, 源文件都一打包好, 有需要可以找我要<loovien@163.com>.  

1. 默认所有服务安装到/usr/local/webserver目录下.  

2. 安装必要的依赖.  

        $ sudo yum groupinstall "Development tools"

3. apache2.4 编译.  

    1. http2.4.x源码包中不含有apr apr-util [下载它们](http://apr.apache.org/download.cgi), 解压到http2.4.x/srclib下.  
    2. 下载pcre-8.3.x.tar.gz, 解压 执行 `./configure && make && make install`.  
    3. 编译apache2.4.x 定位到httpd源码目录, 执行编译`./configure --enable-file-cache --enable-cache --enable-disk-cache --enable-mem-cache --enable-expires --enable-headers --enable-usertrack  --enable-cgi --enable-vhost-alias --enable-rewrite --enable-so --with-include-apr --prefix=/usr/local/webserver`, `make && make install`
    4. 配置httpd DocumentRoot =\> /vagrant/www目录下.  

4. 编译php

    1. 安装依赖, 没办法, 编译就是这样, 少了什么依赖就安装什么依赖, 编译安装.  

        $ sudo yum install libxml2 libxml-devel
        $ ./configure --prefix=/usr/local/webserver/php --with-mysql
        $ make && make install

5. 编译mysql  

    1. 完全编译mysql5.5工程量比较大, 本例下载二进制包, [下载](http://dev.mysql.com/downloads/mysql/).  
    2. 解压后查看README文档安装即可

6. 运行项目可能需要编译php扩展, 自行安装即可, 使用 `phpize`.  

7. 在/etc/rc.local, 添加apache2, mysql 启动命令, 是开机启动.  

### 开发

1. 后期自动把项目挂载F:\vagrant下, 让虚拟box的http Document目录软链接到 /vagrant目录即可.
