---
title: windows 生成sshkey 实现github git协议克隆提交代码
date: 2017-03-15 21:41:14
desc:
tags:
---

一般的, 我们从github, 或是私有仓库gitlab中克隆代码, 都是使用 git clone https://github.com/u/xxx.git, https协议相比git, (git@github.com:u:/xx.git) 会慢很多, 而且要保存帐号密码到本地,才能实现免帐号密码登入. 使用ssh协议就方便很多了.

<!-- more -->

### 废话不多说, 开搞

1. 生成密钥,公钥对. 随便找一台linux机器, 执行如下命令

    ```bash
        $ ssh-genkey -t rsa -f test // 生成test test.pub 密钥公钥对
        $ sz test*  // 下载密钥公钥对到本地 (rz命令不存在, 安装lrzsz)
    ```

2. 复制`test.pub`中的内容, 贴到你的github中的sshkey管理.具体导航如下

    - (确认您登入了啊)点击github右上角的您的头像, 找到`settings`
    - 左边导航菜单招到`SSH and GPG keys`
    - 点击右上角 `New SSH key`
    - 随便输入标题
    - 把复制的`test.pub`内容贴到文本框中.

3. 本地配置ssh

    - 找到你的用户目录, 一般在 `C:\Users\您的名字`, 也可以使用下面步骤打开
        1. `win+r` 打开运行, 输入`cmd`, 此时打开了黑窗口
        2. 输入`explorer .` 此时打开的就是你的用户目录了
    - 在用户下新建`.ssh`目录, windows不能新建, 可在刚打开的黑窗口中输入 `mkdir .ssh`
    - 复制第一步生成的`test test.pub` 文件到此目录
    - 新建 `config` 文件, 并复制如下内容

        ```bash
            # 对应github地址
            Host github.com
            IdentityFile ~/.ssh/test
        ```

4. 随便找自己一个仓库地址, 到黑窗口执行

    ```bash
        git clone git@github.com:your-name/your-repo.git
    ```
5. 是不是稍微变快了点😄, 随便改点东西, commit push 发现是不是不用密码了? 😄
