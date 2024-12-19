---
title: golang读锁导致的死锁问题分析
date: 2024-12-19 12:09:42
tags:
---

> 本文来探讨下业务中错误使用sync.RWMutex的读锁而导致的死锁问题。
<!-- more -->

# 业务场景
为说明问题原因，这里简化业务逻辑，重点突出读锁重入。

如下是一个定时统计器，Incr会被外部并发调用，Run中会定期将data中热点数据拷贝到hot。
```golang

type Fake struct {
	data map[string]int
	hot []string
	mu sync.RWMutex
}

func (c *Fake) stat() {
	c.RLock()
	defer c.RUnLock()
	//copy data to hot slice
}
func (c *Fake) Incr(k string) {
c.Lock()
defer c.UnLock()
// c.data[k]++
}

func (c *Fake) Run() {
	
	ticker := time.NewTicker(2 * time.Second)
	defer ticker.Stop()
	for {
	    	select {
			   case <-ticker.C
                    c.RLock()
			        ....
                    c.stat()
                    .....
                    c.RUnLock()
			}
		
    }
	
}
```
在给出答案之前，先来了解下go读写锁的加锁/解锁的逻辑
# sync.RWMutex源码分析
此处go1.18

获取写锁
1. 写锁会先抢占互斥锁，
2. 接着readCount加rwmutexMaxReaders(超大负数)变为负数，记住这里设计比较巧妙！！<font color="red">只要readerCount为负数，说明上了写锁或者写锁在等待。</font>
3. 之后又加rwmutexMaxReaders，又可以复原读锁的协程数。
4. 尝试将当前读锁协程数赋值给readerWait
5. 查看是否没有释放读锁的协程，如果有，则直接进入等待休眠队列，等待被唤醒。
```golang
func (rw *RWMutex) Lock() {
	// 省略race检查
	// First, resolve competition with other writers.
	rw.w.Lock()
	// Announce to readers there is a pending writer.
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	// Wait for active readers.
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
    // 省略race检查
}
```
释放写锁
1. 通过 加rwmutexMaxReaders, 巧妙获取读协程个数
2. 挨个唤醒等待的读锁协程
```golang
func (rw *RWMutex) Unlock() {
	// 省略race检查

	// Announce to readers there is no active writer.
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}
	// Unblock blocked readers, if any.
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
    // 省略race检查
}

```

获取读锁
1. 加读锁，比较简单，知识统计读协程个数。
2. 若加后readerCount为负数，则说明已经上了写锁，只能休眠等待被唤醒; 否则直接返回上锁成功
```golang

func (rw *RWMutex) RLock() {
    // 省略race检查
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		// A writer is pending, wait for it.
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
    // 省略race检查
}


```
释放读锁
1. 释放读锁，也简单仅加-1。之后readerCount >=0，说明没有写锁，那就正常解锁就可以了。
2. 若<0, 说明还有写锁在等待
3. 若readerWait变为0，则说明所有读锁都执行完了，则唤醒写协程，大于0说明还有其他读锁，直接退出。
```golang
func (rw *RWMutex) RUnlock() {
	// 省略race检查
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		// Outlined slow-path to allow the fast-path to be inlined
		rw.rUnlockSlow(r)
	}

}
func (rw *RWMutex) rUnlockSlow(r int32) {
	//if r+1 == 0 || r+1 == -rwmutexMaxReaders {
	//	race.Enable()
	//	throw("sync: RUnlock of unlocked RWMutex")
	//}
	// A writer is pending.
	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		// The last reader unblocks the writer.
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}

```

# 问题定位

根据上述读写锁的原理分析，写锁时要等待读锁执行完之后，才能成功的上锁。在上述业务场景中，导致死锁的原因如下:

1. A协程执行Run.RLock
2. 此时并发协程B刚好执行Lock, 由于还有读锁，会直接进入休眠阻塞队列，此时readerWait=1
3. A会调用stat.RLock，readerCount <0, 说明有写锁，接下来执行计算，readerWait = 0，A进入休眠阻塞等待B唤醒。


# 总结
Go中要杜绝读锁重入，并且官方也提示不要重入读锁避免引起意外问题。
![avoid_rlock_reentrant](/images/avoid_rlock_reentrant.png)