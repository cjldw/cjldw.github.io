---
title: 安装kubernetes
date: 2018-11-18 22:41:12
desc: linux 安装 kubernetes
tags: kubernetes, k8s
---

最近在学习 kubernetes, 发现这玩意学习曲线很陡啊， 困难重重, 一是很多概念, 二是在天朝, 妈蛋安装起来就是一道坎, 今天终于安装成功了, 马克下!

<!-- more -->

### 环境 (算是入门级服务器了)

 > CentOS6 折腾了一下, 放弃了,浪费时间基本玩不了。

 1. Ubuntu 16.04.1 LTS
 2. 2核2G


 ### 安装docker

 1. 这个按[官方文档](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1)来就可以安装了。
 2. 也可使用[阿里云](https://yq.aliyun.com/articles/110806?spm=5176.8351553.0.0.1d071991O6binf) 安装。


 ### 安装 kubeadm kubectl

 1. 官方文档方式, 很抱歉, 需要科学上网

 ```bash
 apt-get update && apt-get install -y apt-transport-https curl
 curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
 cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
 deb https://apt.kubernetes.io/ kubernetes-xenial main
 EOF
 apt-get update
 apt-get install -y kubelet kubeadm kubectl
 apt-mark hold kubelet kubeadm kubectl
 ```

 2. 使用阿里云镜像安装

 ```bash
 apt-get update && apt-get install -y apt-transport-https
 curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
 cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
 deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
 EOF
 apt-get update
 apt-get install -y kubelet kubeadm kubectl
 ```

 ### 初始化kubernetes集群

 1. 上面如果正常安装后, 可以初始化集群了。

 ```bash
    # kubeadmin init
 ```

 这个命令基本是不会成功的, 因为需要到gcr.io上去pull镜像下来, 先从阿里云下载下来, 然后在改名字吧。

 ```bash
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:v1.12.2
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:v1.12.2
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:v1.12.2
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:v1.12.2
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:3.2.24
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.2.2
 # 更名后, 在初始化
 # docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:v1.12.2 k8s.gcr.io/kube-apiserver:v1.12.2
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:v1.12.2 k8s.gcr.io/kube-controller-manager:v1.12.2
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:v1.12.2 k8s.gcr.io/kube-scheduler:v1.12.2
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:v1.12.2 k8s.gcr.io/kube-proxy:v1.12.2
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:3.2.24 k8s.gcr.io//etcd:3.2.24
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
 # docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.2.2 k8s.gcr.io/coredns:1.2.2

 ```
 再初始化集群了一次

 ```bash
    # kubeadmin init
 ```

出现 `Your Kubernetes master has initialized successfully!` 表明已经安装好了, 可以愉快的玩耍了。

 ### 配置 kubectl

 1. 默认kubctl 是和kube-apiserver 的8080通信的, 通过`kubeadmin init` 初始化后, 使用默认6443端口。

 ```bash
 mkdir -p $HOME/.kube
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 sudo chown $(id -u):$(id -g) $HOME/.kube/config
 ```

 其实就是更新端口, 和 kube-apiserver 通信。


 2. 校验是否成功

 ```bash
 kubectl get nodes
 NAME                STATUS     ROLES    AGE   VERSION
ubuntu               NotReady   master   8d    v1.12.2
 ```

 ps: 可以愉快的玩k8s了
