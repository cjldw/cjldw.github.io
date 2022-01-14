---
title: goproxy.io的使用
date: 2019-10-09 21:40:16
desc: go1.13后支持的go.mod
tags: golang, goproxy
---

go1.11后支持的module再天朝使用起来，并不是那么爽，各种限制.因此[goproxy](https://goproxy.io/)就出来了.

<!-- more -->

1. linux or MacOS 使用

```bash
# Enable the go modules feature
export GO111MODULE=on
# Set the GOPROXY environment variable
export GOPROXY=https://goproxy.io

```
2. windows PowerShell 使用

```bash
# Enable the go modules feature
$env:GO111MODULE="on"
# Set the GOPROXY environment variable
$env:GOPROXY="https://goproxy.io"
```

3. golang的版本再&gt;=1.13的时候, 可以直接使用命令

```bash
go env -w GOPROXY=https://goproxy.io,direct
```

4. 再公司内部搭建的仓库, 不经过代理, 可以使用.

```bash
# Set environment variable allow bypassing the proxy for selected modules
go env -w GOPRIVATE=*.corp.example.com
```

PS: 权威参考[goproxy.io](https://goproxy.io)
