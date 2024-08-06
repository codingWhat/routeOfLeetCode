---
title: Single-flight 核心逻辑拆解
date: 2024-07-17 16:19:03
tags:
- GO
- 缓存击穿
- 缓存问题
---

业务场景中经常会有缓存的身影，虽然缓存给我们带来了诸多好处，但是缓存带来的问题却不容小觑，常见的有缓存雪崩、缓存穿透、缓存击穿。 今天来说说缓存击穿及其解决方案。

## 问题场景
当发生缓存击穿时，瞬时流量会涌入下游服务或者存储造成极大的冲击甚至打挂，此时业务应该如何应对？

<!-- more -->
## 解决方案:
singleflight, 主要解决了:
1. 流量合并，将N个请求->1个请求
2. 流量拦截，如果发现已经有inflight请求，会阻塞等待inflight请求返回结果

### 核心逻辑
- 抽象同类请求，利用wg去控制阻塞
```
type call struct {
	wg sync.WaitGroup //利用其Wait 阻塞请求

	val interface{} // 返回结果，被阻塞请求需要

    ## 省略非核心字段
}

```
- 保存全局瞬时请求
```
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // 保存全局请求，lazily initialized
}
```

- 核心函数Do
```
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		g.mu.Unlock()
		## 一旦发现有请求，就在这阻塞，注意使用了wg
		c.wg.Wait()

		#if e, ok := c.err.(*panicError); ok {
		#	panic(e)
		#} else if c.err == errGoexit {
		#	runtime.Goexit()
		#}
		return c.val, c.err, true
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}

func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	// use double-defer to distinguish panic from runtime.Goexit,
	// more details see https://golang.org/cl/134395
	defer func() {
		// the given function invoked runtime.Goexit
		if !normalReturn && !recovered {
			c.err = errGoexit
		}

		g.mu.Lock()
		defer g.mu.Unlock()
		c.wg.Done()
		if g.m[key] == c {
			delete(g.m, key)
		}
        .... 省略panic/channel相关处理
	}()
    .... 省略非核心代码
		c.val, c.err = fn()
    ...  省略非核心代码

	if !normalReturn {
		recovered = true
	}
}
```

### 自己动手实践

tips:
为了理解singleflight的设计思想，在实践过程中省去了非核心逻辑, 只关注核心数据结构。
```
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

type HandleFn func() (interface{}, error)

type call struct {
	sync.WaitGroup
	val interface{}
	err error
}

var (
	groups = make(map[string]*call)
	mu     sync.RWMutex
)

func main() {
	var wg sync.WaitGroup
	num := 5
	wg.Add(num)
	for i := 0; i < num; i++ {
		go func(gid int) {
			defer wg.Done()
			v, err := Do("key1", func() (interface{}, error) {
				queryDB(gid)
				return time.Now().Unix(), nil
			})
			fmt.Println("Goroutine:", gid, "----> get data ", v, err)
		}(i)
	}
	wg.Wait()
}

func queryDB(gid int) {
	// 模拟查询DB
	time.Sleep(1 * time.Second)
	fmt.Println("Goroutine:", gid, "---> querying DB .... ")
}

func Do(key string, fn HandleFn) (interface{}, error) {
	mu.Lock()
	w, ok := groups[key]
	if ok {
		mu.Unlock()
		w.Wait()
		return w.val, w.err
	}
	c := new(call)
	c.Add(1)
	groups[key] = c
	mu.Unlock()

	fmt.Println("--->call")
	c.val, c.err = fn()

	mu.Lock()
	c.Done()
	delete(groups, key)
	mu.Unlock()

	return c.val, c.err
}

```
输出结果:
```
--->call
Goroutine: 0 ---> querying DB .... 
Goroutine: 2 ----> get data  1721205160 <nil>
Goroutine: 0 ----> get data  1721205160 <nil>
Goroutine: 1 ----> get data  1721205160 <nil>
Goroutine: 4 ----> get data  1721205160 <nil>
Goroutine: 3 ----> get data  1721205160 <nil>
```