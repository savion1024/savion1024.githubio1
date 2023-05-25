---
layout:     post
title:      kafka的协调者
subtitle:   遇到一些问题
date:       2022-04-02
author:     savion
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- k8s
- kafka
- 问题排查
---

## 背景
kafka是k8s内部两个节点，分别是borker0，borker1。此外还有一个service 是Nodeport类型暴露出来。
按照基本的思路我们可以编写脚本 bootserver 填上 ip+端口（service）,然后尝试跑脚本

## 问题
### 1.生产者无法生产消息
代码报错，手动降低日志界别到debug,发现出现了k8s内部的dns pod地址，在集群外部是无法访问的。此时想到`port-forward`可以暂时解决这个问题，在本级开启port-forward 转发 kubectl port-forward kafka-0 9092:9092 ，且添加   hosts 127.0.0.1       kafka-0.kafka-headless.freefire.svc.cluster.local
但是由于端口只能转发一个，所以只转发了0号borker，然后bootserver 地址填上本机的回环地址127.0.0.1:9092。 尝试运行，结果可以生产消息
解决手段: 本机port-forward 直接连通pod 同时修改dns 直接转发到本机回环再到集群内 再到pod

### 2.消费者无法消费消息
按照之前的思路，同样开启消费者，发现日志打印出 部分错误 显示非组协调者错误，非组协调者的错误，查阅文档之后发现与group_id有关。因为`协调者`是一个不可或缺的角色，一个消费者组开启的时候必然要和协调者通信，因为协调者负责管理消费者组的分配以及位移的管理。由协调者帮助消费者组确定组内的每个消费者应该消费哪个分区。
ok 问题来了，协调者在每个boker上都有一个，那么消费者组怎么确定和哪个协调者通信呢，这就涉及到位移主题，位移主题默认情况下有50个分区  他们的分区leader分别散落在不同的borker上。每个消费者组都要有一个对应的唯一主题分区。此时borker会使用group_id哈希之后然后通过一些算法找到对应的唯一主题分区leader，分区leader所在的broker的协调者就是该消费者对应的协调者。
ok, 由于本机只开启了一个port-forward，两个节点的dns都被解析指向了本机回环的9092端口 
![hFbtW2Rdrc.png](https://cdn.nlark.com/yuque/0/2022/png/583261/1646475995447-12737f00-9fc3-4b19-a296-99306b642ec1.png#clientId=u3db09256-7217-4&from=paste&height=190&id=u7151d27c&name=hFbtW2Rdrc.png&originHeight=380&originWidth=966&originalType=binary&ratio=1&rotation=0&showTitle=false&size=181528&status=done&style=none&taskId=u4f169c05-62fa-4140-902a-31fd749b122&title=&width=483)
所以说即使找到了协调者，但是被转发去了错误的broker。所以才会报非组协调者的错误
![Y7yVVXLvj4.png](https://cdn.nlark.com/yuque/0/2022/png/583261/1646476056880-4840dae2-b3ba-4b4e-bdac-caf9ae31ea1e.png#clientId=u3db09256-7217-4&from=paste&height=139&id=ubc66077c&name=Y7yVVXLvj4.png&originHeight=278&originWidth=2514&originalType=binary&ratio=1&rotation=0&showTitle=false&size=170845&status=done&style=none&taskId=u9591d44b-fea5-4b83-94ce-fa560f2b52e&title=&width=1257)

解决手段：因为现在只能和一个broker通信，所以修改gorup_id让他哈希到能够通信的broker上去。
这样就只和一个broker通信了。

## 总结
- 分区由哪个消费者消费是由协调者决定的




