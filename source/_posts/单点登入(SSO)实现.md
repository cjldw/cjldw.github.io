---
title: 单点(SSO)登入实现
date: 2016-05-31 20:40:00
tags: [SSO, SESSION 共享, Cookie作用域]
---

场景: 一公司旗下有很多网站，要求在旗下任意一个网站登入后，其他网站都处于登入状态．
说明: 本帖子不涉及代码实现，之说明重要的点．(阐述不恰当的，欢迎拍砖)

<!-- more -->


### 理解登入原理

就我个人理解，网站登入两种实现．

* 基于session登入
* 基于token登入


#### 基于session登入原理

1. 用户访问网站的一个页面, 服务器端随机生成一个标识sessionId, 更具服务端session配置，session数据可以保存在文件中(默认),也可保存在数据库(关系性或nosql)中 同时并把sessionId的值添加到响应头中，让用户浏览器已cookie的方式保存．
2. 用户再次访问其他网页时候，请求头中就会带上上一步中sessionID, 服务端根据sessionId到取得对应用户的信息．判断是否登入.

3. session过期时间

    * 写入浏览器cookie的过期时间．当浏览器的cookie过期了，会导致sessionId丢失，倒是session失效．需要重新登入．
    * 服务端的session过期，当服务端session过期，就算cookie没有过期，session也失效．　需要重新登入．
    * 注意，每次请求，服务端session如果有存在值的话，需要更新服务端session过期时间，这些都session都是默认帮我们做了(一直访问，session不会失效)．

4. 几天免登入实现

    * 将session中的数据加密一起保存到cookie中，cookie的过期时间设置到对应几天失效，期间用户访问带上了cookie中的加密的值，实现免登入．


#### 基于token登入原理

* token验证有很多方式, (HTTP_BASIC Authorized, Auth0 auth2, jwt ...)，那jwt(json web token)说明．

    1. 用户请求认证，认证服务器发放token(token中包含了用户信息).
    2. 用户之后每个请求都在请求头中带上认证头信息.
    3. token过期，再次请求认证服务器，刷新token.



#### 单点(SSO)登入实现

* 根据session,token登入原理，很明显基于token的方式无需要任何而外的配置，就实现了单点登入功能了．
* session实现单点登入．
    * 保证服务端session是共享的，默认文件存储需要设置保存session的目录共享，推荐使用数据库存储．
    * 设置cookie作用域，域名，是每个域名下都能读取到保存sessionId的cookie



#### session vs token

* 实现复杂度　token &gt; session
* 性能　token &gt; session 
* 扩展性 token &gt; session



