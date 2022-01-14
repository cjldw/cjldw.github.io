---
title: 记INodeClient VPN 安装出现的问题
date: 2020-02-10 23:13:20
desc: InodeClient failed to get scenario and connection information, please restart the client to try.
tags: vpn
---

今天在家办公， 需要vpn连接到办公室网络， 的用VPN， 找的运维哥哥要了安装包， 安装好了， 发现用不了。 一直出现 

**failed to get scenario and connection information, please restart the client to try**.

作为一个开发，这个问题必须解决它。

<!-- more -->

出现问题，本能的去查看安装，运行日志。能力不够， 看不懂。 于是又去查了下错误信息。搜索到了一篇文章

[解决方案](https://zhiliao.h3c.com/Theme/details/40161), 和我的情况一模一样。 

**经确定发现现场安装inode管理中心的操作系统是中文版，但是安装客户端的环境是英文版导致。 在英文版上安装iNode管理中心进行定制，定制出来客户端在英文版系统上安装，问题解决。**

上面说的很清楚了， 下载一个管理中心， 再定制一下就可以了。

[管理中心下载地址](http://www.h3c.com/cn/Service/Document_Software/Software_Download/IP_Management/iNode/iNode_PC/)

临时下载账号：yx800/01230123

打开inode管理中心， 点击 `client customization`, 选着一个场景， 默认不知道全部选择好了， 

注意： 选择了 `wireless access` 这个打包出来的客户端，安装后 windows 自带的WI-FI会被这个玩意控制， 需要先用

这个客户端连接WI-FI, 然后再释放控制权。我公司使用的是`SSL VPN`, 我就只勾选了这个场景了。 到目前为止， 搞好了。

管理中心(H3C-iNode-PC-7.3-E0548.zip)百度网盘提供下载： 链接: https://pan.baidu.com/s/1tv7UoEL7L1iAkt1GS2aKcw 提取码: nms2
