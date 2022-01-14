---
title: 来个markdown出个博客系统
tags: [github, workflow, hexo]
date: 2022-01-14 21:32:00
---

简单记录下博客更新过程(很水)。最终效果是只要写个markdown文件，提交到github, 自动构建出blog系统。

<!-- more -->

## 原由

有很长一段时间没有更新自己的博客了，发现原先的hexo版本已经更新很高的版本了，然后我的一砣安全漏洞需要修复。今天有空就更新一把了。

## 原先的更新方式

原先的方式是使用hexo的本地的策略从创建，到生成，到发布。 对应命令

```bash
    $ hexo new post xxx # 新建blog。
    $ vim _source/_post/xxx.md # 编辑blog。
    $ hexo generate # 生成静态文件。
    $ hexo deploy # 发布到对应 username.github.io 仓库。
```

优点：

1. 本地完全可以看到生成后的效果。

缺点：

1. 本地需要要nodejs环境。
2. 本地需要安装项目依赖的npm包。
3. 需要有2个项目来支持，A: 源文件仓库。 B：github Pages 仓库（username.github.io）。


## 最新的方式（github Actions）

原理单纯的就使用hexo的骨架，在源码目录下上对应的博客（markdown文件）。然后提交到github仓库。使用github workflow 自动构建。

[官方文档](https://hexo.io/docs/gitlab-pages)

1. 克隆hexo博客系统的骨架, 可以按这个项目在目录创建对应的文件目录。（.git, .github 目录除外)。

```bash
    $ git clone https://github.com/hexojs/hexo-starter.git
```

2. 克隆下来后，删除对应目录（.git .github), 根据自己喜欢， 重名了hexo-starter新的名称。

```
    $ rm -rf hexo-starter/.git hexo-starter/.github
    $ mv hexo-starter blog-code
```

3. 初始化当前目录为自己的博客项目, 添加远程仓库地址。

```bash
    $ cd blog-code
    $ git init
    $ git remote add origin git@github.com:username/username.github.io.git
```

4. 创建github workflows 流程。

    * 项目下创建目录`.github/workflows/`
    * 新建workflow脚本（类似.gitlab-ci.yml)

```yaml
name: document-build
on: [push]
jobs:
  use-hexo-build-website:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2 # 克隆代码
      - uses: actions/setup-node@v2 # 使用node环境
        with:
          node-version: "14" # node 版本
      - run: npm install hexo-cli -g # 安装hexo-ci
      - run: npm install # 安装依赖
      - run: npm run build # 构建项目
      - uses: peaceiris/actions-gh-pages@v3 # 使用action-gh-pages 发布项目
        with:
          github_token: ${{ secrets.token }} # github登录TOKEN
          publish_dir: ./public
          publish_branch: gh-pages # 发布到gh-pages分支。

```

PS: github私有仓库发布需要有token，[创建地址](https://github.com/settings/tokens), 项目workflows添加secrets `https://github.com/loovien/yourname.github.io/settings/secrets/actions`，对应替换地址中的yourname. 添加好了后（TOKEN是添加好的名称），${{ screts.TOKEN }} 就可以使用它的值了。

[参考文档](https://docs.github.com/en/actions/learn-github-actions)


5. 都处理好了，现在直接在`_source/_post`下写对应的markdown文件， 就可以愉快的玩耍了。

```markdown
---
title: 来个markdown出个博客系统
tags: [github, workflow, hexo]
date: 2022-01-14 21:32:00
---

description section

<!-- more -->

body section

balabala ....

```

6. 代码提交github, 完成发布。

```bash
    $ git add . && git commit . -m "deploy new BLOG"
    $ git push
```


