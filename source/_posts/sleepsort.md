---
title: 睡眠排序
date: 2017-07-06 07:45:33
desc: wtf
tags: sort, javascript, 
---

有一种拙劣的排序叫睡眠排序

<!-- more -->

# 睡眠排序是啥?

感觉很神秘? 上码... 😂😂😂

```javascript
    var data = [1,23,2,3, 20, 11, 24, 100];
    var sorteddata = [];
    data.map((item) => {
        setTimeout(() => {
            console.log(item)
        }, item)
    });
```
