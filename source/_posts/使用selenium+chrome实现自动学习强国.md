---
title: 使用seleium+chrome实现自动学习
date: 2020-08-21 14:17:59
desc: 学习强国每日30分
tags: selenium chromedriver
---

老婆的学习强国账号，每日要学满30分。 每天也是一个任务，作为一个开发者，本着学习的态度， 实现它(包含学习文章， 学习视频， 每日答题)。

<!-- more -->

大致的实现逻辑也很简单，就是使用selenium+chromedriver 自动化测试工具，完整的代替手工的点来点去。

为了防检测， 也把对应人chromedriver的标示， window navigator 的标示清除了。

具体的效果如下：

例子中先答题，后学习文章3篇， 文章学习时长10秒， 视频学习3个 视频学习时间10秒， 图片过大，截了一段。

![案例](/images/videos/xuexi1.gif)


PS: 学习官方说会有检测机制，虽然把driver的标示去掉了，浏览器的标示也去掉了， 具体也不知道是怎么检测的，所以谨慎使用。

需要学习源码可以加微信，获取。

![超级罗大文](/images/weixin1.png)
