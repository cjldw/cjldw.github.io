---
title: 批量删除redis制定键
date: 2016-08-15 21:42:40
desc:
tags:
---

有时候需要删除redis中的制定key, 使用redis-descktop桌面软件虽然挺好要, 但要批量删除, 并不是那么好用. so 使用脚本吧. 

<!-- more -->

### 脚本奉上, (欢迎拍砖)

```bash

    redis-cli -n {数据库} keys '{glob 匹配, 参考官方文档有惊喜}' | xargs -n 1 redis-cli del

```
