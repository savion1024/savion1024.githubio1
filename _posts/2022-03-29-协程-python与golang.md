---
layout:     post
title:      协程-go与python的对比
subtitle:   探究一下协程
date:       2022-03-29
author:     savion
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- Go
- Python
---

## python的协程
- `单核并发`，遇到IO可以让出cpu，协程之间执行任务按照一定的顺序交替执行。
- 协程安全, 因为cpu打断不会影响值的正确读取，单线程内的并发操作
- 遇到while循环 不会让出cpu

``` python
import asyncio
import time
import requests
import aiohttp

count = 0


# 同步方法
async def test_requests(i):
    print(f'任务{i}开始')
    resp = requests.get('http://www.baidu.com')
    print(resp.status_code)
    print(f'任务{i}结束')


    # 异步方法
async def test_aiohttp(i):
    print(f'任务{i}开始')
    async with aiohttp.request('GET', url='http://www.baidu.com') as resp:
        result = await resp.text()
        print('done')
        # resp = requests.get('http://www.baidu.com')
        # print(resp.text)
    print(f'任务{i}结束')
    # a = 0
    # while True:
    #     a = a + 1
    #     a = a - 1


    # 测试协程安全
async def test_count(i):
    global count
    for item in range(10000):
        count = count + 1


if __name__ == '__main__':
    start_time = time.time()
    loop = asyncio.get_event_loop()
    # tasks = [test_requests(i) for i in range(10)]
    tasks = [test_aiohttp(i) for i in range(10)]
    # tasks = [test_count(i) for i in range(10)]
    loop.run_until_complete(asyncio.wait(tasks))
    loop.close()
    print(count)

    print(f'cost time {time.time() - start_time}')
```

## Golang的协程

- `多核并行`  参考GMP模型
- 协程不安全
- 抢占式调度，不会因为while循环而占住cpu


``` go

package main

import (
    "fmt"
    "sync"
)

var love int32 = 10000 //公共资源love
var wg sync.WaitGroup

func getLove() {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
       love = love - 1
   }
    fmt.Println("love left:", love)
}
func main() {
    for children := 0; children < 10; children++ {
       wg.Add(1)
       go getLove()
   }
    wg.Wait()
    fmt.Println("love is:", love)
}


// 以上代码开启了一个开启了十个协程对公有变量做一千次减法
// wg wait结束之后发现公有变量其实不受保护
```
``` go

package main

import (
   "fmt"
   "log"
   "net/http"
   "runtime"
   "sync"
   "time"
)

var mwg sync.WaitGroup
var count int32 = 0

func httpRemote(url int, httpclient *http.Client) {
   defer mwg.Done()  // -1
   log.Println("开始", url)
   resp, err := httpclient.Get("http://www.baidu.com")
   if err != nil {
      fmt.Println(err)
   }
   fmt.Println(resp.Status)
   log.Println("结束", url)
   for {
      count = count + 1
      count = count - 1
   }
}
func main() {
   runtime.GOMAXPROCS(5)
   startTime := time.Now()
   var httpClient = &http.Client{}
   mwg.Add(10)
   for i := 0; i < 10; i++ {
      go httpRemote(i, httpClient)
   }
   costTime := time.Since(startTime)
   mwg.Wait()
   fmt.Println(costTime)
}
```
## 对比
- Python 协程在io密集型的任务下可以释放出最佳性能
- Golang 协程在io密集，cpu密集型的任务下都有很好的表现


## 参考资料
- [https://go.cyub.vip/gmp/gmp-model.html](https://go.cyub.vip/gmp/gmp-model.html)         
- [https://segmentfault.com/a/1190000038241863](https://segmentfault.com/a/1190000038241863)
- <https://gfw.go101.org/article/control-flows-more.html>

