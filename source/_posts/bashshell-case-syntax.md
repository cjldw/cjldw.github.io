---
title: bash shell 获取命令行参数
tags: ["bash", "case", "command"]

date: 2016-07-12 10:42:00
---

场景: 写bash脚本经长要获取命令行参数, so...

<!-- more -->

#### bash 自动获取 本文参考[stackoverflow](http://stackoverflow.com/questions/192249/how-do-i-parse-command-line-arguments-in-bash)

    ```bash
        #!/bin/bash
        while [[ $# -gt 0 ]] # 判断有参数, 就获取
        do
          key=$1
          case $1 in
            -u|--user)
              echo $2 # get -u | --user value
              shift
              ;;
            -p|--password)
              echo $2 # get -p | --password value
              shift
              ;;
            -h|--help)
              echo "Use: (-u | --user set username) : (-p | --password set passwod)"  # get --help
              shift
              ;;
          esac
          shift
        done
    ```

ps: `$#` 获取参数个数. `$@` 获取所有参数. `$1`获取第1个参数. 脚本[shift](http://unix.stackexchange.com/questions/174566/what-is-the-purpose-of-using-shift-in-shell-scripts)关键字的作用就很好理解咯


