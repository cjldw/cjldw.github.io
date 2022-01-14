---
title: git一键发布
date: 2018-12-06 00:22:58
desc: git 一键发布
tags: git
---

一直使用命令行操作git, 发现处理发布流程还是贼费劲, 今天写个`bash`版的一键发布, 解放下`git checkout/merge/tag`。

<!-- more -->

bash版本

```bash
#!/bin/bash
# @Author luowen<loovien@163.com>
# @time: 2018-12-05
# @desc: 批处理发布流程

help() {
  echo -e "\t Used: gitman branch environment\n"
  echo -e "\t branch: feature/xxx"
  echo -e "\t environment: [beta/online/gray]"
  exit 1
}
if [ ! $# -eq 2 ]; then
  help
fi

feature_branch=$1
environment=$2

if [ ! -z "${feature_branch##*/*}" ];then
  echo "branch must be feature/xxx format!"
  exit 1
fi

case $environment in
  "beta")
    # merge and push to beta
    beta="git fetch && git checkout $feature_branch && git pull && git checkout beta && git pull && git merge $feature_branch && git push"
    eval $beta
    ;;
  "online")

    ######################################### create release branch ###################################

    relese_branch="release/"$(echo $feature_branch | cut -d "/" -f2)
    while ((1))
    do
      read -r -p "use $release_branch? [Y/N]" ok
      case $ok in 
        "N")
          read -r  -p "Please input your release branch name: " release_branch
          ;;
        "Y")
          break
          ;;
      esac
    done

    ######################################## merge code ################################################

    merge_cmd="git fetch && git checkout $feature_branch && git pull && git checkout develop && git pull && git merge $feature_branch && git checkout -b $release_branch && git checkout master && git pull && git merge $release_branch && git push"
    eval $merge_cmd
    if [ ! $? -eq 0 ];then
      echo "merge code failture."
      exit 1
    fi

    #################################### read tag ######################################################
    declare deploy_tag
    last_tag=$("git tag | grep v | sort -r | sed -n '1p'")
    echo
    echo "current tag: $last_tag"
    while ((1))
    do
      read -r -p "Please input your tag:" deploy_tag
      echo "confirm $deploy_tag"
      read -r -p "use this tag? [Y/N]" confirm
      case $confirm in 
        Y)
          break
      esac
    done
    echo "this deploy use $deploy_tag"
    echo 
    tag_cmd="git tag $deploy_tag -am `date '+%F:%T'` && git push origin $deploy_tag"
    eval $tag_cmd

    if [ ! $? -eq 0 ];then
      echo "build tag failture."
      exit 1
    fi

    ################################# delete branch ###################################################
    read -r -p "delete local and remote branch" ok
    case $ok in 
      Y)
        delete_cmd="git branch -d $release_branch && git branch -d $feature_branch && git push origin $feature_branch --delete"
        eval $delete_cmd
        ;;
    esac
    ;;
  "gray")
    ######################################### create release branch ###################################

    relese_branch="release/"$(echo $feature_branch | cut -d "/" -f2)
    echo "release branch: $release_branch"
    echo
    while ((1))
    do
      read -r -p "use $release_branch? [Y/N]" ok
      case $ok in 
        N)
          read -r  -p "Please input your release branch name!" release_branch
          ;;
        Y)
          break
          ;;
      esac
    done

    ######################################## merge code ################################################

    merge_cmd="git fetch && git checkout $feature_branch && git pull && git checkout master && git pull && git branch -D huidu && git checkout huidu && git pull && git merge $feature_branch && git push "
    eval $merge_cmd
    if [ ! $? -eq 0 ];then
      echo "merge code failture."
      exit 1
    fi

    #################################### read tag ######################################################
    declare deploy_tag
    last_tag=$("git tag | grep x | sort -r | sed -n '1p'")
    echo
    echo "current tag: $last_tag"
    while ((1))
    do
      read -r -p "Please input your tag:" deploy_tag
      echo "confirm $deploy_tag"
      read -r -p "use this tag? [Y/N]" confirm
      case $confirm in 
        Y)
          break
      esac
    done
    echo "this deploy use $deploy_tag"
    echo 
    tag_cmd="git tag $read -am `date '+%F:%T'`"
    eval $tag_cmd

    if [ ! $? -eq 0 ];then
      echo "build tag failture."
      exit 1
    fi

    ################################# delete branch ###################################################

    delete_cmd="git branch -d $release_branch && git branch -d $feature_branch && git push origin $feature_branch --delete"
    eval $delete_cmd
    ;;
  *)
    help
    ;;
esac

```

