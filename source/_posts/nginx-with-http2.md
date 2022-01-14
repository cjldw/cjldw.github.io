---
title: nginx-with-http2
date: 2018-07-11 21:31:37
desc: nginx 服务支持http2 协议
tags: nginx, http2
---

现在主流的浏览器都支持了http2, 我们也有必要让我们的服务器支持下http2协议了。

<!-- more -->

### http/2 主要的优势

1. 所有的请求都是并发的, 而不是队列的。
2. http头部是压缩过的, 高效快速。
3. 网页传输是二进制的, 不再是文本, 更加高效。
4. 在用户不请求的情况下, 服务端可以push数据下来。（这里的不请求是指， 用户和服务端有连接的情况下）

ps: http2的安全问题是基于https上的。


### nginx 如何配置

1. 检测您的nginx是否支持了 http2

    ```bash
        $ nginx -V | grep --with-http_v2_module
    ```

2. 没有的话, 需要从新编译 (非 root 用户使用 sudo)

    ```bash
        # cd /path/to/nginx/source/code/dir
        # ./configure --prefix=/usr/local/nginx --with-pcre-jit --with-ipv6 --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --with-http_addition_module --with-http_dav_module --with-http_geoip_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_image_filter_module --with-http_v2_module --with-http_sub_module --with-http_xslt_module --with-stream --with-stream_ssl_module --with-mail --with-mail_ssl_module --with-threads
        # make && make install
    ```

3. 配置nginx支持http2

    ```bash
        # vim /path/to/nginx/install/dir/conf/default

        server {
            listen       80;
            server_name yourdomain.com;
            add_header Strict-Transport-Security "max-age=31536000" always;
            return 301 https://$server_name$request_uri;
        }
        server {
            listen       443 ssl http2;
            listen [::]:443 ssl http2;
            server_name youdomain.com;
            root /data/www/;

            ssl_certificate /path/to/ssl/yourdomain.crt;
            ssl_certificate_key /path/to/ssl/yourdomain.key;
            ssl_session_cache    shared:SSL:1m;
            ssl_session_timeout  5m;
            ssl_ciphers  HIGH:!aNULL:!MD5;
            ssl_prefer_server_ciphers  on;
            # add Strict-Transport-Security to prevent man in the middle attacks
            index index.html index.htm index.php;
            location / {
                proxy_pass http://localhost:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-forwarded-for $remote_addr;
            }
            location = /favicon.ico { access_log off; log_not_found off; }
            location = /robots.txt  { access_log off; log_not_found off; }
            sendfile on;
            client_max_body_size 100m;
        }
    ```

3. 启动nginx

    ```bash
       # /path/to/nginx/install/dir/sbin/nginx -t
       # /path/to/nginx/install/dir/sbin/nginx
    ```

### 验证是是否OK

打开chrome F12 查看 `h2`表示已经使用了http2了

![http2协议支持](/images/http2.jpg)

