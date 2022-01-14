---
title: 同步hexo博客内容到公众号
date: 2020-01-09 21:49:56
desc: 再hexo中编写好的博客文件，实现自动同步到公众号中。
tags: blog, WeChat
image: source/imgs/hexo.jpg
---

最近想把hexo写的博客内容也同步到微信公众号中， 萌生了一个简单的项目，那就来搞它吧。

<!-- more -->

## 大致的思路

hexo 编写好的文档后， 推送到远程仓库， 仓库使用webhook通知服务器， 服务器脚本拉取最新代码后，

判断有没有新写的博客(难点， 怎么获取最近编辑的博客文件的markdown文件呢)， 有的话， 使用 `hexo deploy` 

发布部署后，再读取博客的标题，详情摘要, 有图片的话，再把图片对应生成微信的图文素材，调用微信开放接口,

同步到微信中。再调用群发接口，通知关注的用户(哈哈，好像很完美)

#### 解析md文件

hexo 博客markdown文件格式大致: 

```bash
---
title: 同步hexo博客内容到公众号
date: 2020-01-09 21:49:56
desc: 再hexo中编写好的博客文件，实现自动同步到公众号中。
tags: blog, WeChat
image: source/imgs/hexo.jpg
---

```
读取开头中的 title/公众号标题, date/原文http路径 , image/公众号的图片, 有的话。


### 推送微信

对应项目中的`net.go`


### hexo 发布

对应项目中的`hexo.go`

### 项目地址

包含了所有信息, 想想无所谓了,公众号有白名单。

```bash
    git clone https://github.com/loovien/vxgo.git
```

