---
title: go-http-server
date: 2017-06-18 23:34:54
desc: golang 实现简单的web服务器
tags: golang, http, server
---

对于web开发者来收, 需要快速搭建一个web服务, 之前都在使用 nodejs实现的 [http-server](https://www.npmjs.com/package/http-server)
今天发现golang可以30行代码搞定类似`http-server`的功能, golang着实强大.

<!-- more -->

## 简单说明, 和代码

`flag`包处理命令行的参数, `fmt`包不多说, `net/http`负责http的服务 `log`负责打印点日志

```go

package main

import (
    "flag"
    "fmt"
    "log"
    "net/http"
)

var (
    port int
    host string
    path string
)

func init() {
    flag.IntVar(&port, "port", 8000, "server port listen")
    flag.StringVar(&host, "host", "0.0.0.0", "host ip address")
    flag.StringVar(&path, "path", "./", "server document root")
    flag.Parse()
}

func main() {
    log.Printf("server listen at [%v]:[%v] and document root is [%v]\n", host, port, path)
    log.Println(path)
    fs := http.FileServer(http.Dir(path))
    http.Handle("/", fs)
    http.ListenAndServe(fmt.Sprintf("%v:%v", host, port), nil)
}

```

## 使用

查看帮助 `gohttpserver.exe -h`

```bash
    $ gohttpserver.exe -h
        Usage of gohttpserver.exe:
          -host string
                host ip address (default "0.0.0.0")
          -path string
                server document root (default "./")
          -port int
                server port listen (default 8000)
```

启动 `gohttpserver.exe --host=127.0.0.1 --port=9999 --path=d:/workspaces`

**有问题, 欢迎拍砖**


