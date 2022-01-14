---
title: go小工具实现翻墙
date: 2016-09-17 22:50:29
desc:
tags:
---

最近在学习go, go语言给我最爽的是build后成一个文件, 无需额外的环境以来, 这对发布简直就爽歪歪了。
边学边玩的态度, 写了一个go小工具, 基于[shawsocks](http://www.ishadowsocks.org/)页面实现自动翻墙。

<!-- more -->

## 原理分析

* go中`net/http`去获取`http://www.ishadowsocks.org/`页面内容。

* 使用`regexp`包正则匹配主机, 端口, 密码等信息。

* 构建shadowscoks配置文件结构, 自动写入到gui-config.json文件中。

* 重启shadowscoks, 是刚写好的gui-config.json生效。

* 以下是代码部分, 刚学, 多指教, 勿喷。
```go
    package main
    import (
        "encoding/json"
        "fmt"
        "io/ioutil"
        "log"
        "net/http"
        "os/exec"
        "regexp"
        "strconv"
    )

    type GuiConfigStruct struct { // 定义gui-config结构
        Configs                []ServerItemStruct `json:"configs"`
        Strategy               *bool              `json:"strategy"`
        Index                  int                `json:"index"`
        Global                 bool               `json:"global"`
        Enabled                bool               `json:"enabled"`
        ShareOverLan           bool               `json:"shareOverlan"`
        IsDefault              bool               `json:"isDefault"`
        LocalPort              int                `json:"localPort"`
        PacUrl                 *string            `json:"pacUrl"`
        UseOnlinePac           bool               `json:"useOnlinePac"`
        AvailabilityStatistics bool               `json:"availabilityStatistics"`
    }

    type ServerItemStruct struct {
        Server      string `json:"server"`
        Server_port int    `json:"server_port"`
        Password    string `json:"password"`
        Method      string `json:"method"`
        Remarks     string `json:"remarks"`
    }

    type RemoteHostPortPassword struct { // 匹配远程页面主机, 端口, 密码
        Host     string
        Port     int
        Password string
    }

    const SHADOWSOCKS_URL = "http://www.ishadowsocks.org/"

    func main() {
        //stopShadowsocks() // 关闭shadowscoks进程, 未成功.. 使用RunHiddenConsole.exe 执行start.bat来关闭
        var contents string = httpGetUrl(SHADOWSOCKS_URL) // 获取远程页面内容
        remoteHostPortPassword := matchHostAndPassword(contents) // 匹配主机,端口,密码
        fmt.Println(remoteHostPortPassword)
        defaultConfig := getGuiConfig() // 获取gui-config.json结构
        rebuildConfig(&remoteHostPortPassword, &defaultConfig) // 重新赋远程获取的值
        jsonBytes, err := json.Marshal(&defaultConfig)
        if err != nil {
            fmt.Println(err)
            return
        }
        ioutil.WriteFile("gui-config.json", jsonBytes, 0777) // 写入gui-config.json文件
        startShadowsocks() // 启动shadowocks进程
    }

    func stopShadowsocks() {
        command := exec.Command("taskkill /F /IM shadowsocks.exe")
        if err := command.Run(); err != nil {
            fmt.Println(err)
        }
    }

    func startShadowsocks() {
        command := exec.Command("cmd", "/C", "shadowsocks.exe")
        if err := command.Run(); err != nil {
            fmt.Println(err)
        }
    }

    func rebuildConfig(remoteHostPortPassword *RemoteHostPortPassword, guiConfig *GuiConfigStruct) {
        guiConfig.Configs[0].Server = remoteHostPortPassword.Host
        guiConfig.Configs[0].Server_port = remoteHostPortPassword.Port
        guiConfig.Configs[0].Password = remoteHostPortPassword.Password
    }

    func httpGetUrl(url string) string {

        response, error := http.Get(url)
        if error != nil {
            log.Fatal(error)
        }

        contents, error := ioutil.ReadAll(response.Body)
        response.Body.Close()

        if error != nil {
            log.Fatal(error)
        }
        return string(contents)
    }

    func matchHostAndPassword(body string) RemoteHostPortPassword {
        regexObj := regexp.MustCompile("C服务器地址:(?P<ip>\\w*\\.\\w*\\.\\w*).*\\n.*端口:(?P<port>\\d*).*\\n.*C密码:(?P<pwd>\\d*)")
        matchArray := regexObj.FindStringSubmatch(body)

        var matchLength int = len(matchArray)
        resultSet := new(RemoteHostPortPassword)
        if matchLength == 4 {
            resultSet.Host = matchArray[1]
            intPort, err := strconv.Atoi(matchArray[2])
            if err != nil {
                fmt.Println(err)
            }
            resultSet.Port = intPort
            resultSet.Password = matchArray[3]
        }

        return *resultSet
    }

    func getGuiConfig() GuiConfigStruct {

        /*
            c, error := ioutil.ReadFile("gui-config.json")

            if error != nil {
                log.Fatal(error)
            }
        */

        var defaultConfig = GuiConfigStruct{
            Strategy:               nil,
            Index:                  0,
            Global:                 false,
            Enabled:                true,
            ShareOverLan:           false,
            IsDefault:              false,
            LocalPort:              1080,
            PacUrl:                 nil,
            UseOnlinePac:           false,
            AvailabilityStatistics: false,
        }
        defaultConfig.Configs = make([]ServerItemStruct, 1)
        defaultConfig.Configs[0] = ServerItemStruct{
            Server:      "jp3.iss.tf",
            Server_port: 10000,
            Password:    "luowen",
            Method:      "aes-256-cfb",
            Remarks:     "",
        }
        return defaultConfig
    }
```


