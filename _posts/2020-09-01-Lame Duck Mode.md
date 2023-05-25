---
layout:     post
title:      Lame Duck Mode
subtitle:   优雅关闭
date:       2020-09-01
author:     savion
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- linux
---


## 什么是 Lame Duck Mode

`lameduckmode`是一种模式，就是我们常常听到的优雅关闭，用于在服务器关闭前允许一段时间处理未完成的请求。
当服务器决定关闭时，它会切换到`lameduckmode`，这意味着它不再接受新的请求，而是继续处理已经接收的请求，直到所有请求都完成或超时。
在`lameduckmode`中，服务器会发送信号给客户端，通知它们服务器即将关闭，并且不再接受新的请求。
这使得客户端有机会逐渐停止向服务器发送请求，而不会丢失已经发送的请求。
`lameduckmode`对于一些场景很有用，例如在进行服务器维护或升级时，希望平滑地关闭服务器而不中断正在进行的请求。
它允许服务器在关闭前优雅地完成已接收的任务，从而提供更好的用户体验和系统可靠性。


## 实现原理

在`linux`操作系统上我们一般通过`监听信号`实现，windows系统上没有信号这个概念需要依赖于别的手段。

以python语言为例我们实现一个 优雅关闭
``` python

import signal
import time
import sys

def lameduckmode(signum, frame):
    # 进入 lameduckmode 后的逻辑处理
    print("Entering lameduckmode...")
    # 这里可以进行一些清理工作或日志记录

    # 模拟处理时间
    time.sleep(5)

    # 关闭服务器或终止进程等其他操作

def main():
    # 注册终止信号处理函数
    signal.signal(signal.SIGINT, lameduckmode)
    signal.signal(signal.SIGTERM, lameduckmode)

    try:
        # 正常运行的主逻辑
        print("Server is running...")
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        # 处理键盘中断（Ctrl+C）
        pass

    # 清理工作或其他终止逻辑
    print("Server shutdown completed.")

if __name__ == "__main__":
    main()

```


以go语言为例我们实现一个 优雅关闭

``` go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	// 创建一个带有超时的上下文
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// 创建一个 HTTP 服务器
	server := &http.Server{
		Addr: ":8080",
	}

	// 启动 HTTP 服务器
	go func() {
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("HTTP server error: %v", err)
		}
	}()

	// 监听操作系统信号
	signalCh := make(chan os.Signal, 1)
	signal.Notify(signalCh, syscall.SIGINT, syscall.SIGTERM)    // INT TERM 信号

	select {
	case <-ctx.Done():
		// 在 lameduckmode 中继续处理已接收的请求
		fmt.Println("Entering lameduckmode...")
		// 这里可以进行一些清理工作或日志记录

	case sig := <-signalCh:
		// 收到终止信号时，进入 lameduckmode
		fmt.Printf("Received signal: %v\n", sig)
		fmt.Println("Entering lameduckmode...")
		// 这里可以进行一些清理工作或日志记录
	}

	// 关闭 HTTP 服务器
	shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer shutdownCancel()

	if err := server.Shutdown(shutdownCtx); err != nil {
		log.Fatalf("HTTP server shutdown error: %v", err)
	}

	fmt.Println("Server shutdown completed.")
}

```





