---
title: python解码unicode字符
date: 2017-11-28 13:26:39
desc: python 解码\uxxxx字符
tags: python, decode
---

在终端下获取redis中的数据, 如果是中文汉字, 在终端下经常是`\uxxxx\uxxxx\uxxxx`等这种字符串形式. 平时使用, 都是拷贝出来, 然后通过浏览器的debug工具 console栏的alter(msg)来
获取其的数据。 这种方法可行, 但是感觉有点麻烦。今天介使用python来处理。

<!--more-->

上码:

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
# @author: luowen<bigpao.luo@gmail.com>
# @blogsite: https://vvotm.github.io
# @description: decode \uxxxx string tools
# @time: 2017-11-28 09:46:00

import sys;

def decoder():
    """ decoder func """
    ipt_len = len(sys.argv)
    if ipt_len < 2:
        help()
    ipt_str = " | ".join(sys.argv[1:])
    print(str.encode(ipt_str, "utf-8").decode("unicode-escape"))
    # print(ipt_str.decode("unicode-escpape"))

def help():
    """ Help Information """
    help = """
        Use: udecoder \\uxxx \\uxxx \\uxxx
    """
    print(help)
    exit(0)

if __name__ == '__main__':
    decoder()


```

使用:

```bash
    $ udecoder '\u767e\u53d8\u5a31\u4e50'
```

