---
title: 搭建 microk8s
date: 2019-07-03 09:40:16
desc: microk8s, k8s
tags: k8s, kubernetes
---

安装一个kubernetes集群学习， 还是又点门槛的， 试过各种方案， 今天搞个microk8s来玩玩。也许是最简单的一中.

<!-- more -->

### 环境

1. Ubuntu Server 18.04.1 LTS 64位 (腾讯云: 1vCPU/1Gb/50G)

### [安装](https://microk8s.io/)

1. 安装 microk8s

```bash
    sudo snap install microk8s --classic
```

2. 查看 microk8s 信息

```bash
    sudo snap info microk8s
```


