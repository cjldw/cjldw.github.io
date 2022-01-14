---
title: express+phantomjs构建截图服务
date: 2017-09-25 22:49:07
desc: express, phatomjs构建截图服务
tags: express, phatomjs
---

最近有个需求需要把长的网页截图后, 用于微信分享。 google了一番, 发现了集中解决方案。

- 使用python的`selenium`库实现
- 使用`CutyCapt`实现截图
- 使用nodejs中的`pagers`或`webshot`实现

<!-- more -->

### 对比

python的`selenium`库是基于浏览器的, 线上服务器, 那里来的浏览器, 要知道去线上服务器装一个浏览器有多么复杂啊. (X服务, QT, 巴拉巴拉省略好几百个依赖)， 太吓人了， 果断放弃

`CutyCapt`不需要装浏览器, 但它的原理是将页面转成PDF在有PDF转成图片， 我在想页面边PDF色彩会不会丢失, 会不会出现字体样式的问题。 没试， 还是放弃了。依赖的东西也不少。

nodejs`webshot2` 一个简单的页面截图, 依赖少的可怜, 装好nodejs就可以了, 直接抓去页面, 或直接渲染html, 截图。一步到位, 毫无悬念, 就它了。

### 展示下效果图

1. 截图传递参数详情参见[webshot2](https://www.npmjs.com/package/webshot2)

    ![展示1](/images/webshot/1.jpg)

2. 截图后的效果

    ![展示2](/images/webshot/2.jpg)

由于是后台业务, 无需考虑大并发业务场景。[代码](https://github.com/vvotm/webshot)


