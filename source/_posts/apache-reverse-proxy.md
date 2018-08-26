---
title: apache2.4反向代理配置
date: 2017-06-01 21:31:51
desc: apache2.4反向代理实现
tags: Apache, Reverse, Proxy
---

今天是儿童节, 愿天下所有的宝宝都健康快乐的成长(我也是宝宝😄)....

回到正题: 假设你有多个网站, 但只有一个域名, 如何实现访问2个对应网站. 1, 通过方向代理实现了.

<!-- more -->

##  场景再现

1. 话不多说, 上一张图, 快速理解

    ![http reverse proxy](/images/httproxy/reverse_proxy.png)


2. 通过访问对应的路径, http代理过去

    ```bash
        Site: http://youdomain.com
        WebSite1: http://youdomain.com/app1 http://localhost/app1
        WebSite2: http://youdomain.com/app2 http://localhost/app2
    ```

## apache2.4配置

0.  开启`mod_proxy` `mod_proxy_http.so`

1. 编辑`http-vhost.conf`

    ```bash
        ProxyPreserveHost On
        ProxyPass /app1 http://127.0.0.1:8888/app1
    ```

## nginx1.x配置

1. 编辑`nginx.conf` 或对应vhost.conf文件

    ```bash
        location /app1 {
            proxy_pass http://127.0.0.1:8888
        }
    ```

## 创建8888端口监听http服务

1. 用golang快速搭建, 也可以使用nodejs, 都很方便

    ```golang

            package main

            import (
                "fmt"
                "net/http"
            )

            func UserInfo(w http.ResponseWriter, r *http.Request) {
                fmt.Fprintf(w, "Hello, I love, %s", r.URL.Path[1:])
            }

            func main() {
                http.HandleFunc("/app1/hello", UserInfo)
                http.ListenAndServe(":8888", nil)
            }
    ```

## 测试

1. 通过访问'http://youdomain.com/app1/hello', 会对应返回 Hello I love /app1/hello


## fastcgi代理(杂谈)

### apache fastcgi+php-fpm处理php脚本语言

1. 这个模式很少用, apache下使用php, 编译php时候, 指定`--with-apxs2` 直接编译mod_php5模块.性能, 配置各方面都方便.
尽作了解吧

2. 配置, 确保`mod_proxy.so`, `mod_proxy_fcgi.so`模块已经加载, document_root注意cgi变量

    ```bash
        ProxyPassMatch "^/app1/.*\.php(/.*)?$" "fcgi://localhost:9000/d:/workspace/" enablereuse=on
    ```

### nginx fastcgi代理php-fpm处理php脚本语言

1. nginx没有模块处理php, 只能通过php-fpm或php-cgi.exe处理php脚本文件

2. 基本配置信息

    ```bash
        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
    ```

**http代理和fastcgi代理是2个完全不同的机制, 注意区分**
**欢迎拍砖😄**
