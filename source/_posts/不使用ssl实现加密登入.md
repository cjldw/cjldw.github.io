---
title: 不使用ssl实现加密登入
date: 2016-11-11 22:55:14
desc: 登入也密码加密传输
tags:
---

一般我们登入, 都想服务器post用户名, 用户密码, 然后服务端校验, 实现用户登入功能. 如果我们的网站没有使用ssl, 我们的信息就等于在网络中裸奔.
今天我们来实现密码加密登入(非ssl)


<!-- more -->

### 大致原理

1. 用户输入用户名, 用户密码, 我们先从服务端获取一个随机token, 类似验证码一样的东西

2. 我们同过服务端获取的验证码的东西, 然后和用户名加密

3. 将用户名, 和加密的密码post推送到服务端

4. 服务端拿到用户名, 先从数据库中根据用户名获取数据, 如果有, 获取数据库中的密码, 是用客户同样的加密策略, 加密, 然后校验是否相等, 如果相同, 则放行, 否则任务是错误的用户名.


### 代码实现

1. 服务端随机生成token, 发送到login.html

    ```php
        session_start();
        function randomString($length) {
            $chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
            $str = null;
            $i = 0;
            while ($i < $length) {
                $num = rand(0, 61);
                $tmp = substr($chars, $num, 1);
                $str .= $tmp;
                $i++;
            }
            return $str;
        }
        $token = $_SESSION['token'] = randomString(20); // 保存随即字符串
    ```

2. login.html结构[CryptoJS](https://github.com/brix/crypto-js)JavaScript加密库

    ```html
        <html>
            <head>
                <!-- <script src="https://raw.githubusercontent.com/brix/crypto-js/develop/src/core.js"></script>
                <script src="https://raw.githubusercontent.com/brix/crypto-js/develop/src/md5.js"></script> -->
                <script src="http://localhost/core.js"></script>
                <script src="http://localhost/md5.js"></script>
            </head>
            <body>
                <input type="hidden" id="token" value="<?=$token;?>" />
                <input type="text" id="username" value="" />
                <input type="password" id="password" value="" />
                <input type="button" id="submit" value="Submit" />
            </body>
            <script type="text/javascript">
                var request = { // 封装Requet对象
                    getXMLHttpRequest: function(){
                        return new XMLHttpRequest();
                    },
                    doGet: function(url, data, callback) {
                        var that = this;
                        var XMLHttpObj = that.getXMLHttpRequest();
                        var url = (function(url, data){
                            url = url.indexOf('?') >= 0 ? (url + '?_ok=true') : (url + '?');
                            for (var key in data) url += key + '=' + data[key];
                            return url;
                        })(url, data);

                        XMLHttpObj.open('GET', url);
                        XMLHttpObj.onreadystatechange = function() {
                            if(this.readyState == 4 && this.status == 200) {
                                callback(XMLHttpObj.responseText);
                            }
                        }
                        XMLHttpObj.send();
                    },
                    doPost: function(url, data, callback) {
                        var that = this;
                        var XMLHttpObj = that.getXMLHttpRequest();

                        var formData = new FormData();
                        for(var key in data) formData.append(key, data[key]);

                        XMLHttpObj.open("POST", url);
                        XMLHttpObj.onreadystatechange = function() {
                            if(this.readyState == 4 && this.status == 200) {
                                callback(XMLHttpObj.responseText);
                            }
                        }

                        XMLHttpObj.send(formData);
                    }
                };

                submit.onclick = function() {
                    var token = document.getElementById("token").value;
                    var username = document.getElementById("username").value;
                    var password = document.getElementById("password").value;
                    var submit = document.getElementById("submit")

                    var encryptPassword = CryptoJS.MD5(token + password).toString()

                    var postData = {
                        username: username,
                        password: encryptPassword
                    };
                    console.log(postData);
                    request.doPost('http://localhost/login.php', postData, function(response){
                        console.log(response)
                    });
                };
            </script>
        </html>
    ```

3. login.php校验, 获取用户post的字段, 和数据库中的字段校验

    ```php
        session_start();
        $token = $_SESSION['token'];
        $username = $_POST['username'];
        $password = $_POST['password'];

        $dbPassword = 'select password from tbl_user where username = $username'

        if($password == md5($token . $dbPassword))
            exit("loginOK");
        exit("loginError");
    ```

4. 就这么多了, 有问题, 欢迎拍砖.
