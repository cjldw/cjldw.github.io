---
title: 政务云私有OSS解决方案
date: 2023-02-10 20:52:04
desc: 政务云私有OSS解决方案
tags: 政务云，私有OSS
---

政务云OSS为了安全考虑，BUCKET一股脑全部要求私有。这样直接导致了一些公开的资源被迫做鉴权处理了。然后分析业务，其实多此一举。
我理解的私有OSS只是对一些敏感的资源做存储，然后针对用户权限做开放处理，比如只有用于权限的用户，才能获取OSS的AccessKey, AccessSecret,生成图片签名,
预览下载资源文件。也就是说，业务上分2个BUCKET, 公开的传共有读OOS, 敏感的传私有OSS, 而不是以安全为理由，全部私有化。

<!-- more -->

得知这个消息，心中几万只cnm, 然而事情还是的做。应为已经有了历史文件，想到的办法就是通过nginx匹配路径做方向代理处理。

方案1： NGINX匹配上路径后，转发到后端的服务，有后端的服务生成代签名的URL, 并返回302做重定向。
方案2： NGINX匹配上路径后，转发到后端的服务，有后端的服务生成代签名的URL, 并获取内容，直接返回前端资源文件流。
方案3： NGINX匹配上路径后, 通过NGINX的LUA脚本实现签名生成，添加头，并方向代理到私有OSS地址。

综合考虑开发成本，选用第三种方案，对历史文件，新有的业务没有任何影响。方案1，2需要多出一个服务来。

代码如下简单搞定：

```nginx

    location ~* ^/assets {
        if ($request_method = 'OPTIONS') {
            return 204 ;
        }
        set_by_lua_block $gmt_time {
            local time = os.time() - 8 * 3600
            return os.date("%a, %d %b %Y %X GMT", time)
        }
        set_by_lua_block $signature {
            local accessKeyId = "KEYID"
            local accessKeySecret = "SECRET"
            local bucket = "BUCKET"
            local data = string.format("%s\n\n\n%s\n/%s%s", ngx.req.get_method(), ngx.var.gmt_time, bucket, ngx.var.uri)
            local signature = ngx.encode_base64(ngx.hmac_sha1(accessKeySecret, data))
            return string.format("OSS %s:%s", accessKeyId, signature)
        }
        proxy_set_header Date $gmt_time;
        proxy_set_header Authorization $signature;
        proxy_pass http://private-oss.internat.net;
    }


```

PS: 已经很长一段时间没有输出了，佛系更 :)
