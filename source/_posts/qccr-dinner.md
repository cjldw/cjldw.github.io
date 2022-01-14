---
title: qccr-dinner
date: 2016-10-24 00:14:00
desc:
tags:
---

汽车超人每天晚上如果需要加班, 可以免费订餐. 但是每次都下午4点半截至, 由于忙于码, 老是错过了. 此脚本添加到计划任务后, 实现每天定时订餐, 成功或失败, 发送通知邮件!

<!-- more -->


## 实现思路

* 读取配置`config.json`文件, 包含项目组名称,项目组密码(leader段), 发送邮箱助手邮箱配置(assistant段), 用户工号和通知邮箱地址,**可配置多个人**(members段)

    ```go
        {
            "leader": {
                "username": "WMS组",
                "password": "********"
            },
            "assistant": {
                "host": "smtp.exmail.qq.com:465",
                "email": "luowenhui@qccr.com",
                "password": "*******"
            },
            "members": [
                {
                    "username": "twl2798",
                    "email": "notify@qq.com"
                }
            ]
        }
    ```

* 首先请求以登入的页面, 会拿到一个服务端生成的一个`sessionId`, 之后通过模拟`ajax`(请求头中添加`loginReq.Header.Add("X-Requested-With", "XMLHttpRequest")`)请求post数据到登入地址, 会拿到cookie, 在把拿到的cookie放到请求头中, 去访问订餐接口, 实现订餐.

* 订餐成功后失败后, 通过`sendEmail`方法给配置的通知邮箱发送反馈邮件.

* 订餐成功会把当天的日期写如到`工号.lock`, 当脚本再执行后, 会读取`工号.lock`文件, 如果当前日期和lock的日期是相等, 则不在执行, 如果不等, 则表示新的一天, 则在执行订餐, 防止订餐成功后, 一直发送失败邮件.


* 源码贴出, 在windows or linux环境下是用`go build twldinner.go` 生成执行文件. `config.json`必须在生成二进制文件同级目录下.

    ```go
        package main

        import (
            "bytes"
            "crypto/tls"
            "encoding/json"
            "fmt"
            "io/ioutil"
            "log"
            "net"
            "net/http"
            "net/mail"
            "net/smtp"
            "net/url"
            "os"
            "regexp"
            "strconv"
            "time"
        )

        type member struct {
            Username string `json: "username"`
            Email    string `json: "email"`
        }

        type leader struct {
            Username string `json: "username"`
            Password string `json: "password"`
        }

        type assistant struct {
            Host     string `json: "host"`
            Email    string `json: "email"`
            Password string `json: "password"`
        }

        type config struct {
            Leader    leader    `json: "leader"`
            Assistant assistant `json: "assistant"`
            Members   []member  `json: "members"`
        }

        func main() {
            jsonConfig := getConfig()
            doLogin(&jsonConfig)
        }

        func sendEmail(memberConfig *member, errorMsg string, assistant *assistant) {
            from := mail.Address{"", assistant.Email}
            to := mail.Address{"", memberConfig.Email}
            subj := "User: [ " + memberConfig.Username + " ] Order Dinner Status"
            body := "User: [ " + memberConfig.Username + "  ] => " + errorMsg

            // Setup headers
            headers := make(map[string]string)
            headers["From"] = from.String()
            headers["To"] = to.String()
            headers["Subject"] = subj

            // Setup message
            message := ""
            for k, v := range headers {
                message += fmt.Sprintf("%s: %s\r\n", k, v)
            }
            message += "\r\n" + body
            // Connect to the SMTP Server
            servername := assistant.Host

            host, _, _ := net.SplitHostPort(servername)

            auth := smtp.PlainAuth("", assistant.Email, assistant.Password, host)

            // TLS config
            tlsconfig := &tls.Config{
                InsecureSkipVerify: true,
                ServerName:         host,
            }

            // Here is the key, you need to call tls.Dial instead of smtp.Dial
            // for smtp servers running on 465 that require an ssl connection
            // from the very beginning (no starttls)
            conn, err := tls.Dial("tcp", servername, tlsconfig)
            if err != nil {
                log.Panic(err)
            }

            c, err := smtp.NewClient(conn, host)
            if err != nil {
                log.Panic(err)
            }

            // Auth
            if err = c.Auth(auth); err != nil {
                log.Panic(err)
            }

            // To && From
            if err = c.Mail(from.Address); err != nil {
                log.Panic(err)
            }

            if err = c.Rcpt(to.Address); err != nil {
                log.Panic(err)
            }

            // Data
            w, err := c.Data()
            if err != nil {
                log.Panic(err)
            }

            _, err = w.Write([]byte(message))
            if err != nil {
                log.Panic(err)
            }

            err = w.Close()
            if err != nil {
                log.Panic(err)
            }

            c.Quit()

        }

        func getConfig() config {
            s, err := ioutil.ReadFile("config.json")
            if err != nil {
                fmt.Println(err)
            }
            structJson := config{}
            json.Unmarshal(s, &structJson)

            return structJson

        }

        func doLogin(jsonConfig *config) {

            var welcomeUrl string = "http://dc.kfqccr.com:80/meal/to"
            firstRquest, error := http.NewRequest("GET", welcomeUrl, nil)
            client := &http.Client{}
            welcomeResp, error := client.Do(firstRquest)
            cookies := welcomeResp.Cookies()

            defer welcomeResp.Body.Close()

            _, err := ioutil.ReadAll(welcomeResp.Body)
            if err != nil {
                fmt.Println(err)
            }
            var loginUrl string = "http://dc.kfqccr.com:80/meal/login"
            postData := url.Values{"username": {jsonConfig.Leader.Username}, "password": {jsonConfig.Leader.Password}}
            loginReq, error := http.NewRequest("POST", loginUrl, bytes.NewBufferString(postData.Encode()))
            loginReq.Header.Add("Origin", "http://dc.kfqccr.com")
            loginReq.Header.Add("Host", "dc.kfqccr.com")
            loginReq.Header.Add("Content-Type", "application/x-www-form-urlencoded; charset=UTF-8")
            loginReq.Header.Add("Content-Length", strconv.Itoa(len(postData.Encode())))
            loginReq.Header.Add("X-Requested-With", "XMLHttpRequest")
            loginReq.Header.Add("Referer", "http://dc.kfqccr.com/login.jsp")
            loginReq.Header.Add("User-Agent", "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.116 Safari/537.36")

            for _, cookie := range cookies {
                loginReq.AddCookie(cookie)
            }

            loginResp, error := client.Do(loginReq)
            if error != nil {
                fmt.Println(error)
                os.Exit(1)
            }
            defer loginResp.Body.Close()

            request, error := http.NewRequest("GET", welcomeUrl, nil)
            for _, cookie := range cookies {
                request.AddCookie(cookie)
            }
            secondWelcomeResp, error := client.Do(request)

            defer secondWelcomeResp.Body.Close()

            // dingfan
            var dinnerUrl string = "http://dc.kfqccr.com/meal/set"
            for _, member := range jsonConfig.Members {
                fmt.Println(member.Username)
                if !isOrdered(member.Username) {
                    postIdData := url.Values{"suggest": {member.Username}}
                    dinnerReq, error := http.NewRequest("POST", dinnerUrl, bytes.NewBufferString(postIdData.Encode()))
                    if error != nil {
                        fmt.Println(error)
                        sendEmail(&member, "Order Request Error", &jsonConfig.Assistant)
                    }
                    dinnerReq.Header.Add("Origin", "http://dc.kfqccr.com")
                    dinnerReq.Header.Add("Host", "dc.kfqccr.com")
                    dinnerReq.Header.Add("Content-Type", "application/x-www-form-urlencoded; charset=UTF-8")
                    dinnerReq.Header.Add("Content-Length", strconv.Itoa(len(postData.Encode())))
                    dinnerReq.Header.Add("X-Requested-With", "XMLHttpRequest")
                    dinnerReq.Header.Add("Referer", "http://dc.kfqccr.com/meal/to")
                    dinnerReq.Header.Add("User-Agent", "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.116 Safari/537.36")

                    for _, cookie := range cookies {
                        dinnerReq.AddCookie(cookie)
                    }
                    dinnerResp, error := client.Do(dinnerReq)
                    defer dinnerResp.Body.Close()
                    dinnerResult, error := ioutil.ReadAll(dinnerResp.Body)
                    var resultMsg string = string(dinnerResult)
                    matched, err := regexp.MatchString("订饭成功.*", resultMsg)
                    againMatched, err := regexp.MatchString("重复订饭.*", resultMsg)
                    if err != nil {
                        sendEmail(&member, resultMsg, &jsonConfig.Assistant)
                    }
                    if matched || againMatched {
                        markAsSuccess(member.Username)
                    }
                    sendEmail(&member, resultMsg, &jsonConfig.Assistant)
                } else {
                    fmt.Println("User: [ " + member.Username + " ] 难道你想变成胖子？")
                }
            }
        }

        func isOrdered(username string) bool {
            var lockFile string = username + ".lock"
            var today int = time.Now().Day()
            if _, err := os.Stat(lockFile); err != nil { // 文件不存在
                if os.IsNotExist(err) {
                    return false
                }
            }
            oldDay, err := ioutil.ReadFile(lockFile)
            oldDayInt, err := strconv.Atoi(string(oldDay))

            if err != nil {
                fmt.Println(err)
            }

            fmt.Println(today)
            if today != oldDayInt {
                return false
            }
            return true
        }

        func markAsSuccess(username string) {
            var today int = time.Now().Day()
            var lockFile string = username + ".lock"
            ioutil.WriteFile(lockFile, []byte(strconv.Itoa(today)), 0777)
        }
    ```

* **刚学go语言, 有问题欢迎拍砖**

