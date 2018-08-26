---
title: golang 反射调用结构体方法
date: 2018-03-05 21:43:46
desc: 使用 golang 调用结构体方法
tags: golang, reflect
---

在 TCP 编程中, 很常见要根据客户端发送过来的自定义协议包, 通过解析协议包的指定部分信息, (一般为字符串) 来调用指定包中的指定方法。如何优雅的调用呢?

<!-- more -->

使用 golang 反射实现它！

```golang

// {GOPATH}/handles/chat.go

package handle

import "fmt"

type PingReq struct {
    Message string
    SendAt int
}

type Handle struct {
}

func New() *Handle {
    return new(Handle)
}

func (h *Handle) Chat (pr PingReq) {
    fmt.Println(pr)
}



// {GOPATH}/labcode/main.go

package main

import (
    "labcode/handles"
    "reflect"
)

func main() {
    hd := handle.New()
    hdv := reflect.ValueOf(hd)
    chatMethed := hdv.MethodByName("Chat") // 反射出方法
    if !chatMethed.IsValid() {
        return
    }
    abc := handle.PingReq{
        Message: "Wen Lo",
        SendAt: 121212,
    }
    param := []reflect.Value{reflect.ValueOf(abc)} // 构造参数
    chatMethed.Call(param) // 调用参数
}


```

**这样就能根据 `Chat` 这个字符串调用上了 Chat 方法
