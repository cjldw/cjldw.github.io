---
title: 使用猴油插件看vip电影
date: 2020-08-29 14:17:59
desc: 使用猴油插件看各在视频网-站上的vip电影
tags: tampermonkey
---

使用猴油插件的vip视频破解，看vip的电影有一段时间了， 发现用起来还不错，有些还可以看vvip的电影资源。简单分享下。

<!-- more -->


原理其实很简单，就是市面上有很多这个服务，他们先用自己的vip帐号去把视频下载下来， 然后上传到自己的服务器上，再次提供免费的服务。

猴油插件其实就是把这提供的地址做下管理 (脚本检测到你打开的是视频网站， 就在页面上显示一个按钮， 让你选择解析方式)，下面贴出来一些地址出来。

这些地址怎么来的，[脚本中心](https://greasyfork.org/zh-CN/scripts?q=%E8%A7%86%E9%A2%91), 搜索视频破解， 然后找个脚本，查看下源码就可以看了。

直接使用猴油插件更好用了。(会用很多广告)。

[视频教程](/images/videos/vip.mp4)

资源地址: https://pan.baidu.com/s/1L-ymBQ1lXsjo_qEKgyyf3A 提取码: fcvi

拿到这些地址， 就可以直接使用了。

```json
    {name:"618G",url:"http://jx.618g.com/?url=",title:"618G",intab:1},
    {name:"玩的嗨",url:"http://tv.wandhi.com/go.html?url=",title:"",intab:0},
    {name:"最惠买",url:"http://www.zhmdy.top/index.php?zhm_jx=",title:"",intab:0},
    {name:"噗噗电影",url:"http://www.pupudy.com/splay.php?play=",title:"",intab:0},
    {name:"搜你妹",url:"http://www.sonimei.cn/?url=",title:"",intab:0},
    {name:"石头解析",url:"https://jiexi.071811.cc/jx.php?url=",title:"",intab:1},
    {name:"乐乐云",url:"https://660e.com/?url=",title:"",intab:1},
    {name:"石头解析",url:"https://jiexi.071811.cc/jx.php?url=",title:"",intab:1},
    {name:"无名小站",url:"http://www.sfsft.com/admin.php?url=",title:"",intab:1},
    {name:"VIP看看",url:"http://q.z.vip.totv.72du.com/?url=",title:"",intab:1},
    {name:"无名小站2",url:"http://www.wmxz.wang/video.php?url=",title:"",intab:1},
    {name:"人人发布",url:"http://v.renrenfabu.com/jiexi.php?url=",title:"",intab:0}
```

举个例子：比如想看腾讯视频的[《黑金》https://v.qq.com/x/cover/smp6o8ceqbfu4bb.html](https://v.qq.com/x/cover/smp6o8ceqbfu4bb.html)

[直接用 `http://jx.618g.com/?url=https://v.qq.com/x/cover/smp6o8ceqbfu4bb.html`](http://jx.618g.com/?url=https://v.qq.com/x/cover/smp6o8ceqbfu4bb.html)

观看就好了。



