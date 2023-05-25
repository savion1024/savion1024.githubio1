---
layout:     post
title:      认识TIME_WAIT和CLOSE_WAIT 
subtitle:   有何不同？
date:       2022-01-14
author:     savion
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- 计算机网络
- 思考
---


## 都叫WAIT,有何不同
最近在看wireshark的两本书，重新温习了一遍计算机网络，温习到很多网络相关的知识。

现在假设有一个客户端，一个服务端。客户端发起断开连接的请求，由此展开了经典的四次挥手。
在命令行上敲下 netstat -ant 我们可以观察到服务器上所有的连接情况。最后一列就是这个连接的状态。
`TIME_WAIT`出现在客户端，客户端在收到服务端的第二个`FIN`包同时发出最后一个`ACK`包之后进入`TIME_WAIT`状态，并且保持2个`MSL`时间
而`CLOSE_WAIT`出现在服务端，服务端在第一次收到客户端的`FIN`包同时发出ACK进入到`CLOSE_WAIT`状态

## 哪个才是危险的
这两个状态都是正常状态，但如果很多`socket`都处于这两个状态呢，是否是异常且危险的

首先讲讲`TIME_WAIT`过多，出现`time_wait`是正常的 正常客户端会维持在此状态2个`MSL`，`MSL`即一个包在网络上能生存的最大时间。
过多`TIME_WAIT`有可能是短时间内过多断开的连接，由此引发的问题就是 可能会导致服务器句柄不够用,造成后面的进程无法申请句柄。
此时一般可以开启内核参数 `tcp_tw_reuse`,使得 `TIME_WAIT`状态下的socket快速投入使用。

`CLOSE_WAIT`出现在服务端发出第一个`ACK`包之后。在这段时间内服务端会继续发送报文。收尾完毕后会发出`FIN`包请求客户端断开连接，
然后服务端进入到`last_check`状态 。一般原因就是 服务端没有正确close掉句柄。状态没有迁移。`last_check`状态会自动迁移但是。
`CLOSE_WAIT`不会，所以当服务上的socket出现大量`CLOSE_WAIT`的时候，就需要引起警惕。检查是否没有正确关闭连接。


## 为什么TIME_WAIT需要等待2个MSL

确保被动关闭TCP连接的一端能收到第四次挥手的ACK. 避免上一次TCP连接的数据包影响到下一次的TCP连接。
这2个MSL中的第一个MSL是为了等自己发出去的最后一个ACK从网络中消失，
而第二MSL是为了等在对端收到ACK之前的一刹那可能重传的FIN报文从网络中消失。


## 总结
- `TIME_WAIT`过多是正常的，这个状态下的socket会被自动回收
- `CLOSE_WAIT`过多危险的，因为这个状态的socket不会自动迁移，此时需要检查代码是否有不正确关闭的情况




