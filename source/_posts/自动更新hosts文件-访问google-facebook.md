---
title: '自动更新hosts文件, 访问google, facebook'
tags: []
date: 2015-12-28 16:39:00
---

此脚本自动抓去racaljk的repository中hosts文件, 替换本地host, 实现翻墙功能。
hosts访问只限于查询资料, 涉及国外视频网站可能不给力。

<!-- more -->

## 脚本自动获取 [racaljk project](https://github.com/racaljk/hosts) hosts文件(racaljk自带更新脚本, 个人嫌复杂, 写此简单能用版本), 和本地hosts文件合并, 使用--start, --line 保留本地hosts文件行数 ##

## 基本要求系统 ##

* Python3.x.x
* 应hosts文件权限关系
* windows需要使用administrator 权限
* linux需要使用root权限

## 加入计划任务(windows) ##

* 打开开始菜单, 搜索框输入**计划任务** 或快捷键[win + r] 粘贴 `%windir%\system32\taskschd.msc /s`

![](http://images2015.cnblogs.com/blog/451327/201512/451327-20151228170213417-181074786.png)

* 右击 [任务计划库] -> [创建任务]

![](http://images2015.cnblogs.com/blog/451327/201512/451327-20151228170224932-659219118.png)

* 常规选项卡, 添加名称, 描述

![](http://images2015.cnblogs.com/blog/451327/201512/451327-20151228170321089-2002023170.png)

* 点击触发器选项卡 [新建] 添加触发器, 比如多久运行, 什么时候执行

![](http://images2015.cnblogs.com/blog/451327/201512/451327-20151228165952292-1171750287.png)

* 点击操作选项卡   [新建] 添加脚本执行

![](http://images2015.cnblogs.com/blog/451327/201512/451327-20151228170603885-700230869.png)

## Linux安装方法 ##

* 添加执行权限. `shell> chmod +x UpdateHosts.py`

* 添加计划任务. `shell> echo ' 0 09 * * * python /path/to/UpdateHosts.py --line 60 --os linux 2>&1 >/dev/null' >> /var/spool/cron/root`
* 每天9点半更新, 根据自己时间更新

## UpdateHosts.py 代码如下(欢迎拍砖) ##

    ```python
        #!/bin/python env
        import argparse
        from urllib import request
        """
            The Script get from github racaljk's hosts project and merge localhost host file.
            Used: python UPdateHosts.py --help get detail information.
            Author: Gawim <bigpao.luo@gmail.com>
            Time: 2015-12-28
            Require: python3.x.x
            Notes: please use administrator execute it. before execute script remember backup your hosts file.
            in the first line of hosts, you can set how many line you don't want replace. for example 
            ```hosts
                # 11
                # Copyright (c) 1993-2009 Microsoft Corp.
                #
                # This is a sample HOSTS file used by Microsoft TCP/IP for Windows.
            ```
            that will keep 11 line
        """

        WINDOWS_HOST = "C:\Windows\System32\drivers\etc\hosts"
        LINUX_HOST = "/etc/hosts"
        GITHUB_HOSTS = "https://raw.githubusercontent.com/racaljk/hosts/master/hosts"  # copy github racaljk hosts project

        def getParam():
            "Obtain The Extran Parameters"
            parser = argparse.ArgumentParser()
            parser.add_argument("--start", help="Input The Line Which You Need Not To Replace", default="0", type=int)
            parser.add_argument("--line", help="Input The Line Which You Need Not To Replace", default="50", type=int)
            parser.add_argument("--os", help="What System You Use", default="windows", choices=["linux", "window", "OSX"])

            return parser.parse_args()

        def getHostsNeedContent(hostsPath, lineStart, lineOffset):

            needHostDomain = []
            with open(hostsPath, mode="r+", encoding="utf-8") as fileHandle:
                customerLineNumStr = fileHandle.readline().strip("#").strip();
                try:
                    customerLineNum = int(customerLineNumStr)
                except ValueError:
                    customerLineNum = lineOffset
                fileHandle.seek(lineStart)
                i = 0
                while i < customerLineNum:
                    needHostDomain.append(fileHandle.readline())
                    i += 1
                return "".join(needHostDomain)

        def getGitHubHostsAndWriteNewHots(needHostContents, hostsPath):
            fileHandle = open(hostsPath, mode="w", encoding="utf-8")
            fileHandle.write(needHostContents)
            with request.urlopen(GITHUB_HOSTS) as socketHandle:
                for contents in socketHandle:
                    fileHandle.write(contents.decode("utf-8"))
            fileHandle.close()
            return True

        def main():
            options = getParam()
            if options.os == "windows":
                hostsPath = WINDOWS_HOST
            elif options.os == "linux":
                hostsPath = LINUX_HOST
            elif options.os == "OSX":
                hostsPath = LINUX_HOST
            else:
                raise ValueError("Please Tell Me What System You Used")
            lineStart = options.start
            lineOffset = options.line

            needHostContents = getHostsNeedContent(hostsPath, lineStart, lineOffset)  # get localhost hosts line need merge default start 0 line 50

            resultSet = getGitHubHostsAndWriteNewHots(needHostContents, hostsPath)  # get github hosts and merge localhost hosts
            print(resultSet);

        if __name__ == "__main__":
            main()

    ```
