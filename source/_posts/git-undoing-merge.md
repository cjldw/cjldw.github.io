---
title: git合错分支解决方案
date: 2017-10-31 20:51:14
desc: git reset vs git revert. git和错了分支, 如何回退呢?
tags: git, git reset, git revert
---

使用版本控制git工作流的时候, 不可避免的使用多个分支, 合并, 删除。 使用一款好用的git管理工具(如sourceTree, github-desktop等等)可以很方便的避免一些问题。
当很多场景下也是不可避免的要使用终端工具。 个人感觉使用终端工具熟练后, 可以做到比管理工具更快捷的操作[一键发布](#onekeydeploy)。 但是使用终端不可避免的增加了你的犯错几率。 比如和错了分支, 再错误的分支开发啊等等。 这个帖子见到那的说明下合错了分支的处理方式。

<!-- more -->

一步一步来实验吧

```bash
    $ 初始化仓库 ctrl+c
    $ git init 

    $ touch master.txt 
    $ git add . && git commit . -m "initialize project"
    $ 创建`master`, `develop`分支 ctrl+c
    $ git checkout -b develop

    $ 创建feautre/1分支 ctrl+c
    $ git checkout -b feature/1 
    $ touch 1.txt
    $ git add . && git commit . -m "feature/1"

    $ 创建features/2分支 ctrl+c
    $ git checkout develop 
    $ git checkout -b feature/2
    $ touch 2.txt
    $ git add . && git commit . -m "feature/2"

    $ 创建features/3分支 ctrl+c
    $ git checkout develop
    $ git checkout -b feature/3
    $ touch 3.txt
    $ git add . && git commit . -m "feature/3"

    $ feature/1功能开发完毕. 合并feature/1分支到develop ctrl+c
    $ git checkout develop
    $ git merge feature/1

    $ 创建feature/4分支 ctrl+c
    $ git checkout develop
    $ git checkout -b feature/4
    $ touch 4.txt
    $ git add . && git commit . -m "feature/4"

    $ 创建feature/5分支
    $ git checkout develop
    $ git checkout -b feature/5
    $ touch 5.txt
    $ git add . && git commit . -m "feature/5"

    $ feature/4功能开发完毕. 合并feature/4的功能发布
    $ git checkout develop
    $ git merge feature/4

    $ 此时feature/2功能开发完毕, 合并develop发布
    $ git checkout develop
    $ git merge feature/2

    $ 此时feature/5功能开发完毕, 合并develop发布
    $ git checkout develop
    $ git merge feature/5

    $ 此时feature/3功能开发完毕, 合并develop发布
    $ git checkout develop
    $ git merge feature/3

```

感觉很绕, 截一张git的log图看看

![gitlog](/images/git/gitlog.png)

What the fuck!! 是不是看的很懵逼。 **记住一个原则, 从下往上看, 同一个颜色的分支表示在同一时期创建的, 因为合并到develop的分支的时间不同, 就导致了这么的分叉**
拿最下面的绿色线条来说吧, `feature/1`, `feature/2`, `feature/3` 是在同一时期创建的, 由于他们合并的`develop`的分支的时间不同, 就导致了那么丑难以理解的图。

回到正题, 假设我现在`feature/2`开发的功能由于有缺陷, 直接导致系统崩溃。 现在要把`feature/2`的功能剔除掉!, 怎么搞呢???

- 方案1 使用`git reset` 回到`feature/2`代码提交的上一个点, 在把新的功能合并过来

    ```bash
    $ 我们回到合并feature/2功能的上一个提交点, 也就是 8162972
    $ git reset --hard 8162972

    $ 再把feature/3 feature/5的功能合并过来, 再提交
    $ git merge feature/3
    $ git merge feature/5

    $ 强制提交到远程
    $ git push --force
    ```
**缺点: 此方法会把提交历史记录丢失, 一旦分支多了，合并新开发的分支也是很痛苦的，　一旦强制push到远程仓库, 其他协作者的本地分支如果有跟踪远程的分支, 那么提交就出现问题了**

- 方案2 使用更加优雅的`git revert` 直接把`feature/2`提交的代码剔除, 在生成一个新的提交记录

    ```bash
    $ 现还原最后的提交记录
    $ git reset --hard 9c9b941

    $ -m 1 表示merge分支的父节点, 注意, 122c518 是develop合并 feature/2分支后的在develop分支提交点。 如果不是merge的, 如不要分支`feature/1`的功能, 可直接 git revert b42a249 即可
    $ git revert -m 1 122c518
    $ git push
    ```
**有没有发现,全是优点!**

**<a name="onekeydeploy">一键发布分支</a>**
```bash
$ git add . && git commit . -m "deploy" && git checkout develop && git pull && git merge feature/1 && git push
```

本文参考原帖[Undoing Merges](https://git-scm.com/blog/2010/03/02/undoing-merges.html)
