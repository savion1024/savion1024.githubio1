---
layout:     post
title:      groupcache如何避免缓存击穿
subtitle:   从源码开始
date:       2022-11-01
author:     savion
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- Go
---


## 介绍一下groupcache

groupcache是一款由go编写的分基于内存的分布式KV型存储库, 代码量不多。是一个比较有特色的缓存组件,目前在github上有着12K的star。
它有以下几个特点

1. 不可以删除缓存
2. 值不能修改
3. 只能获取值(get方法)


## 假设一个场景

例如有N个客户端同时请求缓存的同一个key,发现都miss,即缓存里没有对应的值,这时候就会都透过缓存去后端拉数据。
导致下游压力暴涨。其实这种情况下只需要一个请求去拉数据就可以,其他的原地等待。
当然，我们可以通过锁+二次校验来解决这个问题,但是groupcache的解决办法更加优雅


## singlefilght

源码位置 groupcache/singlefilght/singlefilght.go

先来看一个Group结构体 里面有一把锁和一个字典m, 这个m就是数据实体

``` go
// Group represents a class of work and forms a namespace in which
// units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}
```
字典的key是一个string, 而value是一个结构体,这个Call结构体长下面这样。真正的value是里面的val字段，而wg是用来实现同步的。err用来存储错误。

``` go
// call is an in-flight or completed Do call
type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}
```
OK,看到这里应该大概能猜到大概实现,但还是要仔细阅读一下整体逻辑,整体的逻辑在do方法.可以看到方法的接受者是一个Group结构体，
然后参数是key和一个fn方法。这个fn方法就是最后去获取value的方法。
如果缓存被穿透，就要执行fn方法获取最终的值。同时这个方法会返回一个error。
``` go
// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)   //懒加载
	}
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```

我把它翻译成以下几个逻辑。
1. 开启锁
2. 通过懒加载的方式生成m
3. 检查m里面是否已经存在了key，如果存在。等待wg归零,归零后return
4. 如果m里面没有key,new一个call。同时add 一下call里面的wg,然后设置m,标志着已经有人进入到获取value的逻辑了,然后解锁。解锁完毕之后如果有其他人进入就会走到上一步的等待逻辑。
5. 执行fn方法获取值,然后wg -1。wg归零之后其他人就能拿到值return
6. 加锁 删除m里面对应的key和value后返回, singlefilght只是用来做数据获取时的竞争保护。真正的值不在这里存储所以没必要保留

通过wg的使用就能轻松实现一个竞争等待的逻辑。不可谓不优雅。


## 引用

- <https://liqingqiya.github.io/groupcache/golang/%E7%BC%93%E5%AD%98/2020/05/10/groupcache.html>
- <https://github.com/golang/groupcache>

