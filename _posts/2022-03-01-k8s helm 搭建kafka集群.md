---
layout:     post
title:      helm搭建kafka集群
subtitle:   helm初探
date:       2022-03-01
author:     savion
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- k8s
- kafka
---


## 目标
搭建一个kafka试验集群，然后部署kafka-ui

## 过程

首先去helm 寻找kfka的repo然后执行一下命令
1. `helm add repo bitnami/kafka`
2. `helm repo add bitnami https://charts.bitnami.com/bitnami `
3. `helm install my-release bitnami/kafka `

4. 接着`helm fetch kafka --untar` 下载yaml文件
5. 修改values.yaml 配置，修改pvc为集群内现有的storeclass
6. `helm upgrade --cleanup-on-fail --install kafka . --namespace=freefire`  安装

执行以上步骤后可以看到，相应的svc pvc po都已经起来了

紧接着使用helm安装`kaftop`，按照同样的步骤安装，修改配置文件。但是发现无法连上borker，使用podIP， svc Ip都不行。显示handshake 机制协商失败，
转而使用docker 安装kafka-ui, 由于单独使用docker，网络和集群不互通。所以修改kafka的svc从`clusterIP` 改成`NodePort`, 绑定宿主机的端口。docker启动的时候指定环境变量为
borker: 宿主机IP:暴露端口。启动完查看日志，提示无法和borker建立连接，在宿主机上nc-v 宿主机IP:暴露端口， 发现可以握手成功，随机分析是否有网络隔离（namespace)的问题。

使用docker top 查看kafka-ui容器的ip，查看进程号。nsenter -n -t pid进入到容器内进程的net namespace 使用nc -v , telent等命令测试端口连通性都发现无法路由到宿主机的IP,此时确定是由于网络问题无法握手成功

接着决定撰写yaml文件在集群内启动一个pod，环境变量borker地址填服务名:port，经由kafka的svc连接到borker， 成功启动。但是由于此时pod 只有一个pod IP，外界无法访问。
遂在本机采用port forward的方式临时调试连接pod，转发成功，成功访问

## 收获

- helm 的基本使用
- k8s yaml文件的编写，svc po pvc等概念的更深层次认识
- port foward, nodeport等方式的使用以及对比
- nsenter 命令的妙用

