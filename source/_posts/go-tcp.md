---
title: go-tcp服务
date: 2017-09-17 19:40:16
desc: 简单的一个gotcp服务, 用户连接上后, 发送数据, 服务端返回什么数据, 当发送字符"p", 的时候, 服务会广播消息
tags: go, tcp
---

最近又在看tcp服务的一些知识, 长连接编程还是有点意思, 今天试着写了个小玩意, 在服务端监听`8808`端口, 客户端连接上来后, 发送什么数据, 就返回什么数据.
如果用户发送的单个`p`字符, 服务端会给所有连接上来的用户广播消息`快跑, 警察来了!!!`, 有点意思.

<!--more-->

代码如下:


```go

package main

import (
    "net"
    "log"
    "sync"
    "os"
    "syscall"
    "os/signal"
    "fmt"
)

type Server  struct{
    closeChan chan  bool
    waitGroup *sync.WaitGroup
}

func NewServer() *Server {
    return &Server{
        make(chan bool),
        &sync.WaitGroup{},
    }
}

// ConnMgr connection manager object
type ConnMgr struct {
    ConnList map[string]*net.TCPConn
}

var (
    connmgr *ConnMgr
    connMgrOnce *sync.Once = &sync.Once{}
)
func NewConnMgr() *ConnMgr {
    if nil == connmgr {
        connMgrOnce.Do(func() {
            i := make(map[string]*net.TCPConn)
            log.Println(i)
            connmgr = &ConnMgr{i}
        })
    }
    return connmgr
}

// getConnList get all connection from connection manager
func (connMgr *ConnMgr) getConnList() map[string]*net.TCPConn {
    return connMgr.ConnList
}

// Add connection to connection manager
func (connMgr *ConnMgr) Add(conn *net.TCPConn)  {
    key := conn.RemoteAddr().String()
    connMgr.ConnList[key] = conn
}

// HandleConn process coming connection
func (srv *Server) HandleConn(conn *net.TCPConn)  {
    defer conn.Close()
    remoteAddr := conn.RemoteAddr().String()
    connmgr := NewConnMgr()
    connmgr.Add(conn)

    fmt.Printf("remoteAddress 【%s】 Connected !\n", remoteAddr)
    for  {
        select {
        case <-srv.closeChan:
            return
        default:
        }
        buf := make([]byte, 2014)
        l, err := conn.Read(buf)
        if nil != err {
            fmt.Printf("Read From Socket 【%v】 ! \n", err.Error())
            return
        }
        fmt.Printf("Read raw Data: length 【%d】data【%s】 !", l, buf[:l])
        reciveData := string(buf[:l])
        if reciveData == "p" { // if user send char 'p' then broadcast to all connected user
            go srv.Broadcast()
        }
        fmt.Printf("Read Data: 【%s】 !", reciveData)

        sentData := []byte("哈哈, 世界")
        data := fmt.Sprintf("【%s】===>【%s】\n", string(sentData), reciveData)
        fmt.Println(data)
        conn.Write([]byte(data))
    }
}

// Broadcast all of connected connection
func (srv *Server) Broadcast()  {
    connmgr := NewConnMgr()
    connList := connmgr.getConnList()

    for ip, conn := range connList{
        log.Println("broadcast to " + ip)
        conn.Write([]byte("快跑, 警察来了!!!"))
    }
}

func (srv *Server) Service(listener *net.TCPListener)  {
    for  {
        select {
        case <-srv.closeChan:
            return
        default:
        }

        conn, err := listener.AcceptTCP()
        if nil != err {
            fmt.Println(err)
            continue
        }

        srv.waitGroup.Add(1)
        go (func() {
            srv.HandleConn(conn)
            srv.waitGroup.Done()
        })()

    }


}

func (srv *Server) Stop()  {
    close(srv.closeChan)
    srv.waitGroup.Wait()
}


func main()  {

    network := "127.0.0.1:8808"
    addr, err := net.ResolveTCPAddr("tcp", network)
    if nil != err {
        log.Fatal(err)
    }
    listener, err := net.ListenTCP("tcp", addr)
    if nil != err {
        log.Fatal(err)
    }
    srv := NewServer()
    go srv.Service(listener)
    log.Println("Listen:"+network)

    stopChan := make(chan os.Signal)
    signal.Notify(stopChan, syscall.SIGINT, syscall.SIGTERM)
    <-stopChan

    srv.Stop()
}


```
