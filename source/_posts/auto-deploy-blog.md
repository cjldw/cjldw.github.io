---
title: 使用github webhook 自动发布博客
date: 2019-04-07 09:57:27
desc:  使用github webhook 自动发布博客
tags: github,webhook
---

最近电脑重穿了系统, 发现自己的博客环境缺失了， 索性本地就不要这个node环境了， 放到服务器上搞吧。原理很简单， 每次我的blog-code有代码提交, github的webhook自动回调我的服务器上一个接口， 那个接口处理把代码拉去下来， 更新， 并且生成静态文件， 发布。

<!-- more -->

>本着怎么简单怎么来的原则， 更新代码使用一个bash脚本实现。

### 代码更新发布

```bash

#!/bin/bash
# @title: update blog-code
# @desc: use go exec.Command don't working, use bash script do it
# @author: luowen<loovien@163.com>

repoName=blog-code
git=/usr/bin/git
repoURL=https://github.com/loovien/blog-code.git
hexo=/opt/nodejs/bin/hexo
npm=/opt/nodejs/bin/npm
workDir=/root/luowen

cd $workDir

if [ ! -d ${workDir}/${repoName} ]; then
    cmd="${git} clone ${repoURL}"
    eval $cmd
    echo "git clone ${repoURL} status: $?"
fi

cd ${repoName}

cmd="${git} pull"
eval $cmd
echo "git pull staus: $?"

cmd="${hexo} generate -d"
eval $cmd
echo "hexo generate -d status: $?"

```


## 服务端使用go来实现一个服务接口

```golang
package main

import (
	"bytes"
	"errors"
	"flag"
	"fmt"
	"net/http"
	"os"
	"os/exec"
	"os/signal"
	"strings"
	"syscall"
	"time"
)

const (
	bash        = "/bin/bash"
	script      = "/root/luowen/bloggem/bloggem.sh"
	logfileName = "app.log"
	httpEntry   = "/path/to/deploy.json"
)

var (
	logfile *os.File
	mode    string
)

func init() {
	flag.StringVar(&mode, "mode", "http", "run console or http mode.")
	flag.Parse()
}

func main() {
	defer closelogfile()
	openlogfile()
	switch strings.ToLower(mode) {
	case "http":
		runHttpMode()
	case "console":
		runConsoleMode()
	default:
		flag.PrintDefaults()
	}
}

func runConsoleMode() string {

	logsave("info", "console run start.")
	var buf = &bytes.Buffer{}
	cmd := exec.Cmd{
		Path: bash,
		Args: []string{"-c", script},
		Stdout: buf,
		Stderr: buf,
	}
	err := cmd.Run()
	if err != nil {
		fmt.Println(err)
		return err.Error()
	}
	result := buf.String()
	fmt.Println(result)
	logsave("info", result)
	logsave("info", "console run completed.")
	return result
}

func runHttpMode() {
	logsave("info", "open logfile: app.log success!")
	go httpListen()
	logsave("info", "http server listen: 20000 success!")
	sigChan := make(chan os.Signal)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGHUP, syscall.SIGKILL)
	sig := <-sigChan
	logsave("info", "receive terminal signal:", sig)
}

// httpListen start http server.
func httpListen() {
	mux := http.NewServeMux()
	mux.HandleFunc(httpEntry, deploy)
	_ = http.ListenAndServe(":20000", mux)
}

// deploy blog code
func deploy(w http.ResponseWriter, r *http.Request) {
	result := runConsoleMode()
	_, _ = w.Write([]byte(result))
}

// openlogfile open a file for logging
func openlogfile() {
	logfile, _ = os.OpenFile(logfileName, os.O_CREATE|os.O_WRONLY|os.O_APPEND, os.ModePerm)
}

// closelogfile close the opening log file
func closelogfile() {
	if logfile != nil {
		_ = logfile.Close()
	}
}

// loginfo write application log into logfile
func logsave(typ string, logmesg ...interface{}) {
	if logfile == nil {
		panic(errors.New("logfile don't open yet!"))
	}
	// [info]:2019-04-06 12:00:12: something has wrong!
	fmtMesg := fmt.Sprintf("[%s]:%s: %v\n", typ, time.Now().Format("2006-01-02 15:04:05"), logmesg)
	fmt.Print(fmtMesg)
	_, _ = logfile.WriteString(fmtMesg);
}

```

### 通过nginx的代理【可选】

```nginx

server {
    listen      80;
    server_name localhost;
    charset     utf-8;
    access_log  logs/default.log  main;
    error_log   logs/default.log;
    error_page  404              /404.html;
    error_page  500 502 503 504  /50x.html;

    location / {
        return 403;
    }

    location = /50x.html {
        root   html;
    }

    location ~ /\.ht {
        deny  all;
    }

    location = /path/to/deploy.json {
        proxy_pass http://127.0.0.1:20000;
        proxy_set_header Host $host;
        proxy_set_header X-Forword-Ip $remote_addr;
        proxy_timeout 300;
    }
}

```

