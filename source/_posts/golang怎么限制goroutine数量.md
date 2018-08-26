---
title: golang怎么限制goroutine数量
date: 2017-10-20 00:16:29
desc: 限制goroutine数量
tags: golang, goroutine
---

当你用golang写并发脚本时候, 你使用goroutine, 但是goroutine也不能无限制的开启, 毕竟cpu数量有限. 多个goroutine处理任务调度也是有消耗的. 此帖子列出限制goroutine数量的代码. 初学golang, 代码有问题 欢迎拍砖!

<!-- more -->

1. 使用chan的缓冲区来控制goroutine的数量, 当最后一批数据没处理, 强行插入空起始位置.

    ```golang 

        maxGoRoutineNum := 100000000
        valveChan := make(chan int, maxGoRoutineNum)
        for index := 0; index < maxGoRoutineNum; index++ {
            valveChan <- index
            go func() {
                fmt.Println("自己的业务代码")
                <- valveChan
            }()
        }
        fmt.Println("所有操作执行完毕")
        bufferSize := cap(valveChan)
        for i := 0; i < bufferSize; i++ {
            valveChan <- i
        }

    ```

1. 使用chan的缓冲区来控制goroutine的数量.

    ```golang
        waitGroup := &sync.WaitGroup{}
        valveChan := make(chan int, 1000) // 限制1000个goroutine
        maxGoRoutineNum := 100000000
        waitGroup.Add(maxGoRoutineNum)
        for index := 0; index < maxGoRoutineNum; index++ {
            valveChan <- index
            go func() {
                fmt.Println("自己的业务代码")
                <-valveChan // 消费valveChan的数据
                waitGroup.Done()
            }()
        }
        waitGroup.Wait() // 等待所有的gorontine执行完毕
        fmt.Println("所有操作执行完毕")
    ```

1. 通过chan数据交换实现

    ```golang
        func main()  {

            maxGoroutine := 10
            waitGroup := &sync.WaitGroup{}
            input := make(chan int, 10)
            output := make(chan int, 10)
            waitGroup.Add(maxGoroutine)
            for index := 0; index < maxGoroutine; index++ {
                go func() {
                    Bridge(input, output)
                    waitGroup.Done()
                }()
            }
            go func() {
                for index := 0; index < maxGoroutine ; index++ {
                    input <- index
                }
                close(input)
            }()
            go func() {
                waitGroup.Wait()
                close(output)
            }()
            for item := range output{
                fmt.Println(item)
            }

        }

        func Bridge(input <-chan int, output chan <- int){
            for item := range input {
                output <- item
            }
        }

    ```

仅供参考
