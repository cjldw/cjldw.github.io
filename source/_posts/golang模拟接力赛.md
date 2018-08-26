---
title: golang模拟接力赛
date: 2017-06-03 16:04:54
desc: golang goroutine实现接力赛
tags: golang, goroutine
---

golang的协程不是很好理解, 今天看到一篇帖子从现实生活中去理解, 发现别有一番风味.

<!-- more -->


## 废话不多说, 上代码.

1. 重要的步骤添加了注释, 注意有个递归, 协程运行了协程.

    ```go
        package main
        import (
            "fmt"
            "sync"
            "time"
        )

        var wg sync.WaitGroup
        var track chan int

        func main() {
            wg.Add(1)
            track = make(chan int)
            go Runner(track)
            // 比赛开始, 一声抢响
            track <- 1
            wg.Wait()
        }

        func Runner(track chan int) {

            const maxExchange = 4 // 定义4个人赛跑
            var exchange int

            baton := <-track // 第一个听到抢响起跑, 后面要等到接力棒后在跑
            fmt.Printf("Runner %d Running To The Baton \n", baton)

            if baton < maxExchange { // 模拟其他4人在各自起跑线上
                exchange = baton + 1
                fmt.Printf("Runner %d To The Line \n", exchange)
                go Runner(track)
            }
            time.Sleep(100 * time.Millisecond) // 模拟在跑100毫秒

            if baton == maxExchange { // 赛道上有有定义的4个人在跑, 结束
                fmt.Printf("Runner %d Finished Race Over\n", baton)
                wg.Done()
                return
            }
            fmt.Printf("Runner %d Exchange With Runner %d\n", baton, exchange)
            track <- exchange // 模拟单个参赛者跑完了, 传递接力棒
        }
    ```

**有问题欢迎拍砖**
