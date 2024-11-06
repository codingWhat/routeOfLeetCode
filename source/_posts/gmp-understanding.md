---
title: GMP学习总结
date: 2024-08-09 13:27:45
tags:
- GO
- GO-GMP
- Go调度原理
---
## 调度器前世今生
调度器核心职责就是通过复用线程来高效执行G, 目前GMP模型已经非常高效了，然而调度器也是经过一步一步演进来的，早期调度器只有G、M以及全局‘runq’，性能表现并不出色，尤其随着goroutine数增长，当m从全局队列消费G时会导致锁竞争非常严重，除此之外系统调用、阻塞等操作时M会休眠，此时M上的G得不到执行，严重影响性能。为了解决这些问题，Go开发者重新设计调度器，推出了当前“GMP”模型。

## 调度器核心概念

### Processor
职责:
1. 解决GM模型的全局锁问题
2. `runq`和P绑定(GM模型中，M包含`runq`)，当M在执行阻塞调用休眠时，M和P解绑，P可以去找空闲M继续执行`runq`中的G

P状态机:
![gmp_p_status](/images/gmp_p_status.png)

### Goroutine
Goroutine简化版三种状态:
- Waiting, 阻塞/系统调用中
- Executing, 在M中正在执行
- Runnable, 就绪状态，runq中

G生命周期：
- _GIdle(空闲链表中) -> _GDead(从链表中取出) -> _GRunnable(参数复制、入栈等) -> _GRunning
- _GSyscall(系统调用) -> _GRunning
- _GWaiting(阻塞) -> GRunnable

![g_status](/images/g_status.png)

G(用户)退出:
runtime·goexit1(mcall) -> goexit0
先切换到G0 -> 清空g的数据，解除和m的关系，g 的状态从 _Grunning 更新为 _Gdead，将g放入空闲队列

### G0,M0,P0,allgs,allm,allp
- M0:是启动程序后的编号为0的主线程，这个M对应的实例会在全局变量runtime.m0中，M0负责执⾏初始化操作和启动第⼀个G， 在之后M0就和其他的M⼀样了。
- G0: 是每次启动⼀个M都会第⼀个创建的gourtine，G0仅⽤于负责调度的G，G0不指向任何可执⾏的函数,每个M都会有⼀个⾃⼰的G0。在调度或系统调⽤时会使⽤G0的栈空间, 全局变量的G0是M0的G0。
- allgs: 记录所有的G
- allm: 记录所有的M
- allp: 记录所有的P
- sched: sched是调度器，这里记录着所有空闲的m,空闲的p,全局队列runq等等

![g0-p0-m0](/images/g0-p0-m0.png)


### Sysmon线程:
和P不需要的关联的m，循环执行，主要指责包含:
- netpoll(fd事件)
- retake (抢占)，抢占长时间运行的P
- forcegc(定期执行gc)
- scavenge heap (释放内存空间)


## 以Goroutine视角理解GMP
以Goroutine生命周期来看，更像是生产/消费模型，goroutine被创建之后，放到对应P的"队列"中, 之后被调度器消费执行。
### 启动流程
初始化m0和g0，m0和主线程绑定，之后按core核数初始化所有P，将第一个P(即P0)和m0、g0绑定，剩余的P放入空闲队列中。之后会创建g1(runtime.main)，放入P0的本地队列中,之后m0的g0开始执行调度逻辑。

### 生产goroutine
在生产阶段，底层对应`newproc`逻辑，通过`runqput`存储到P里。P有两个地方存储G, `runnenxt` 和 `runq`
-  容量: `runnext`1个G，`runq`256个G，因此P总共可以存储257个G
-  区别： `runnext`存储的是下一个要执行的G，`runq`是一个队列

`runnext`和`runq`可以整体看成一个FIFO队列，`runnext`指向队尾元素；不过需要注意的是，当队列满的时候，`runnext`还是会被新的G抢占，被强占的G和`runq`的前半部分的G会被合并放到`全局runq`中。

细节交互:
![Goroutine和P交互细节](/images/g_to_p.png)


### 消费goroutine
go1.20.11版本  
调度逻辑: `runtime/proc.go`中的`schedule()`的`findRunnable()`方法
1. 保证公平，防止全局`runq`中的goroutine饿死， 按概率(`runtime.SchedTick%61==0`)从全局runq中获取G
2. 从本地`runnext`和`runq`中获取
3. 从全局`runq`中获取
3. 从netpoll中获取G, 返回第一个，剩下的直接追加到全局`runq`的尾部
4. 从其他P那偷取一半G

> 在P都很繁忙的场景下，`全局runq`中的G可能迟迟得不到调度，为了公平起见，调度器会统计`runtime.SchedTick`，每次调度都会++, 当`runtime.SchedTick%61==0`时,会从`全局runq`中获取G来执行(高优先级)。

runtime/proc.go
> 按'概率'从全局`runq`中获取  
![gmp_global_runq_probability](/images/gmp_global_runq_random.png)    
从本地`runq`中获取    
![get from local runq](/images/gmp_local_runq.png)  
从全局`runq`中获取  
![get_from_global_runq](/images/get_from_global_runq.png)  
从`netpoll`中获取  
![get_form_netpoll](/images/get_form_netpoll.png)  
从其他P中获取一半G  
![steal_from_other_p](/images/steal_from_other_p.png)


## Go调度器策略
### steal-work机制
尝试4次, 每次随机选择一个P，尝试从"适合的"P中获取一半G
>![stealwork](/images/stealwork.png)

高效获取一半G:  
从'pp'(受害者P)的`runq`中将其一半g取出写入batch(施害者P的`runq`)，更新`pp`(受害者)的`runq`头指针。
```golang
batchHead := pp.runqtail
batch = &pp.runq

func runqgrab(pp *p // 受害者P, batch *[256]guintptr //施害者P的runq , batchHead uint32 //施害者P的队尾指针, stealRunNextG bool) uint32 {
	for {
		h := atomic.LoadAcq(&pp.runqhead) // load-acquire, synchronize with other consumers
		t := atomic.LoadAcq(&pp.runqtail) // load-acquire, synchronize with the producer
		n := t - h
		n = n - n/2
		
        ....省略非核心...
		//取出前半部分G
		for i := uint32(0); i < n; i++ {
			g := pp.runq[(h+i)%uint32(len(pp.runq))]
			batch[(batchHead+i)%uint32(len(batch))] = g
		}
		
		if atomic.CasRel(&pp.runqhead, h, h+n) { // cas-release, commits consume
			return n
		}
	}
}

```

### handoff机制
当前发现正在执行的G发生阻塞、系统调用、或者需要被抢占时，P和M会解绑，会给P绑定"可用的"M,继续执行。而M会休眠，当调用结束之后，找一个空闲的P，若找不到P，M会变成休眠状态，进入空闲队列，G会被加入到全局`runq`

### 抢占(retake)
遍历所有P，针对(_Prunning 和 _Psyscall)状态的P, 如果发现同时满足以下条件就抢占
![preempt_cond](/images/preempt_cond.png)
1. 从上一次监控线程观察到 p对应的m处于系统调用/运行时间已经超过10ms
2. p的运行队列里面有等待运行的goroutine
3. 没有“无所事事”的 p; 这就意味着没有“找工 作”的 M，也没有空闲的 P，大家都在“忙”，可能有很多工作要做,因此要抢占当前的 P，让它来承担 一部分工作。

抢占的实现:
- 信号机制，注册`runtime.sighandler`,接收到信号`SIGURG`时，由操作系统中断转入内核空间，而后将所中断线程的执行上下文参数(例如寄存器 rip、rep 等)传递给处理函数。如果在 runtime.sighandler 中修改了这个上下文参数，操作系统则会根据修 改后的上下文信息恢复执行



