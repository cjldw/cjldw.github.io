---
title: gotcp服务
date: 2017-08-07 08:23:52
desc:
tags: go, tcp
---

golang实现tcp非常简单, 加上goroutine, 跟好理解, 一下代码就是一个tcp连接过来后, 使用一个goroutine处理,
通过client对象包装后, 每个client有开启两个goroutine, 一个负责循环读取客户端发送过来的数据, 一个负责循环往
客户端连接socket中写数据, 中间通过一个chan实现数据交互. 当chan中没有数据的时候, 会阻塞client的读写goroutine.

<!-- more -->

```go

    package main
    import (
        "bufio"
        "log"
        "net"
        "strconv"
        "strings"
    )

    type Client struct {
        // incoming chan string
        outgoing chan string
        reader   *bufio.Reader
        writer   *bufio.Writer
        conn     net.Conn
    }

    func (client *Client) Read() {
        for {
            log.Println("readConnection")
            line, err := client.reader.ReadString('\n')
            if err != nil {
                log.Println("read from tcp connection error" + err.Error())
                client.conn.Close()
            }
            trimLine := strings.Trim(line, "\r\n")
            length := len(line)
            resp := "readData->" + trimLine + " AND length is ->" + strconv.Itoa(length)
            log.Println(resp)
            client.outgoing <- (resp + "\r\n")
            if trimLine == "close" {
                break
            }
        }
        client.conn.Close()
    }

    func (client *Client) Write() {
        for data := range client.outgoing {
            client.writer.WriteString(data)
            client.writer.Flush()
        }
    }

    func HandleConn(connection net.Conn) {
        writer := bufio.NewWriter(connection)
        reader := bufio.NewReader(connection)

        client := &Client{
            outgoing: make(chan string),
            conn:     connection,
            reader:   reader,
            writer:   writer,
        }
        go client.Read()
        go client.Write()

    }

    func main() {
        listener, _ := net.Listen("tcp", ":8080")
        for {
            conn, err := listener.Accept()
            if err != nil {
                log.Println(err.Error())
            }
            go HandleConn(conn)
        }
    }

```
