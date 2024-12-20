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


### 负载均衡 Work-Stealing机制
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


## Go调度策略
### 协作式 vs 抢占式
协作式主动让出CPU，抢占式被动让出CPU

![coop_vs_retake](/images/coop_vs_retake.png)

总结: 
1. 协作式调度任务执行耗时较抢占式短，由于抢占任务任务耗时变长。
2. 抢占的中断次数多
3. 抢占式导致耗时长的任务，耗时更加延长了，不过耗时短的被及时执行了，

### Sysmon线程
- netpoll(获取fd事件)
- retake(抢占)
- forcegc(定期执行gc)
- scavenge heap(清理内存)


#### Sysmon抢占G

##### 抢占条件
sysmon遍历所有P，针对(`_Prunning` 和 `_Psyscall`)状态的P, 如果发现同时满足以下条件就抢占
1. 从上一次监控线程观察到 p对应的m处于系统调用/运行时间已经超过10ms，
2. p的运行队列里面有等待运行的goroutine
3. 没有“无所事事”的 p; 这就意味着没有“找工 作”的 M，也没有空闲的 P，大家都在“忙”，可能有很多工作要做,因此要抢占当前的 P，让它来承担 一部分工作。

##### 抢占逻辑
1. 设置协程抢占标记，在`preemptone (_p_ *p)`方法中设置抢占标志位，`gp.stackguard0 = stackPreempt` 
2. 对处于`_Psyscall`的P还会执行`handoff`。

```golang

func retake(now int64) uint32 {
    n := 0
    lock(&allpLock)
    // 遍历所有的 p
    for i := int32(0); i < gomaxprocs; i++ {
    _p_ := allp[i]
    if _p_ == nil {
    continue
    }
    // 用于 sysmon 线程记录被监控 p 的系统调用时间和运行时间
    pd := &_p_.sysmontick
    // p 的状态
    s := _p_.status
    sysretake := false
    if s == _Prunning || s == _Psyscall {
    // p 处于运行状态，检查是否运行得太久了
    // 每发生一次调度，调度器 ++ 该值
        t := int64(_p_.schedtick)
        if int64(pd.schedtick) != t {
            pd.schedtick = uint32(t)
            pd.schedwhen = now
        } else if pd.schedwhen+forcePreemptNS <= now {
            // pd.schedtick == t 说明(pd.schedwhen ～ now)这段时间未发生过调度
            // 这段时间是同一个goroutine 一直在运行，检查是否连续运行超过了 10ms
            preemptone(_p_) /
            sysretake = true 
        }
    }
        if s == _Psyscall {
            // _p_.syscalltick 用于记录系统调用的次数，在完成系统调用之后加 1
            t := int64(_p_.syscalltick)
            // pd.syscalltick != _p_.syscalltick，说明已经不是上次观察到的系统调用了，
            // 而是另外一次系统调用，所以需要重新记录 tick 和 when 值
            if !sysretake && int64(pd.syscalltick) != t {
                pd.syscalltick = uint32(t)
                pd.syscallwhen = now
                continue
            }
        
            // 只要满足下面三个条件中的任意一个，则抢占该 p，否则不抢占
            // 1. p 的运行队列里面有等待运行的 goroutine
            // 2. 没有“无所事事”的 p
            // 3. 从上一次监控线程观察到 p 对应的 m 处于系统调用之中到现在已经超过 10ms
            if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+
            10*1000*1000 > now {
            continue
            }
            unlock(&allpLock)
            incidlelocked(-1)
            if atomic.Cas(&_p_.status, s, _Pidle) {
                if trace.enabled {
                    traceGoSysBlock(_p_)
                    traceProcStop(_p_)
                }
                n++
                _p_.syscalltick++
                // 寻找一新的 m 接管 p
                handoffp(_p_)   
            }
            incidlelocked(1)
            lock(&allpLock)
		}  
    }
    unlock(&allpLock)
    return uint32(n)
}


func preemptone(_p_ *p) bool {
    mp := _p_.m.ptr()
    if mp == nil || mp == getg().m {
      return false
    }
    gp := mp.curg
    if gp == nil || gp == mp.g0 {
        return false
    }
    
    gp.preempt = true
    
    // Every call in a goroutine checks for stack overflow by
    // comparing the current stack pointer to gp->stackguard0.
    // Setting gp->stackguard0 to StackPreempt folds
    // preemption into the normal stack overflow check.
    gp.stackguard0 = stackPreempt  //！！！！！这里设置为可抢占了！！！
    
    // Request an async preemption of this P.
    if preemptMSupported && debug.asyncpreemptoff == 0 {
        _p_.preempt = true
        preemptM(mp)
    }
    
    return true
}

```

#### handoff机制
当前发现正在执行的G发生阻塞、系统调用、或者需要被抢占时，P和M会解绑，会给P绑定"可用的"M,继续执行。而M会休眠，当调用结束之后，找一个空闲的P，若找不到P，M会变成休眠状态，进入空闲队列，G会被加入到全局`runq`

```golang
func handoffp(_p_ *p) {
    // 如果 p 本地有工作或者全局有工作，需要绑定一个 m
    if !runqempty(_p_) || sched.runqsize != 0 {
        startm(_p_, false)
        return
    }
    // ……………………
    // 所有其他 p 都在运行 goroutine，说明系统比较忙，需要启动 m
    if atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) == 0 && atomic.Cas(&sched.nmspinning, 0, 1) { // TODO:
        fast atomic
        // p 没有本地工作，启动一个自旋 m 来找工作
        startm(_p_, true)
        return
    }
    lock(&sched.lock)
    // ……………………
    // 全局队列有工作
    if sched.runqsize != 0 {
        unlock(&sched.lock)
        startm(_p_, false)
        return
    }
    // ……………………
    // 没有工作要处理，把 p 放入全局空闲队列
    pidleput(_p_)
    unlock(&sched.lock)
}
```

### 基于协作的抢占式调度
#### 触发时机
- sysmon超时检测，触发抢占G


#### 如何让goroutine主动让出CPU？

实现思路：编译器会在每个函数插入一段检查是否栈扩容的逻辑(`runtime.morestack`)。
在morestack中判断Goroutine状态是否为`stackPreempt`，如果可抢占，则执行`gopreempt_m(gp)`,类似runtime.GoSched()


go1.18 asm_amd64.s
```
TEXT runtime·morestack(SB),NOSPLIT,$0-0
	// Cannot grow scheduler stack (m->g0).
	get_tls(CX)
	MOVQ	g(CX), BX
	MOVQ	g_m(BX), BX
	MOVQ	m_g0(BX), SI
	CMPQ	g(CX), SI
	JNE	3(PC)
	CALL	runtime·badmorestackg0(SB)
	CALL	runtime·abort(SB)
     .....
	// Call newstack on m->g0's stack.
	MOVQ	m_g0(BX), BX
	MOVQ	BX, g(CX)
	MOVQ	(g_sched+gobuf_sp)(BX), SP
	....  创建栈
	CALL	runtime·newstack(SB)
	.....
	CALL	runtime·abort(SB)	// crash if newstack returns
	RET

```
runtime.newstack()
```golang
func newstack() {
thisg := getg()

gp := thisg.m.curg
... 省略

morebuf := thisg.m.morebuf
thisg.m.morebuf.pc = 0
thisg.m.morebuf.lr = 0
thisg.m.morebuf.sp = 0
thisg.m.morebuf.g = 0

// NOTE: stackguard0 may change underfoot, if another thread
// is about to try to preempt gp. Read it just once and use that same
// value now and below.
// 检查 g.stackguard0 是否被设置成抢占
stackguard0 := atomic.Loaduintptr(&gp.stackguard0)

// Be conservative about where we preempt.
// We are interested in preempting user Go code, not runtime code.
// If we're holding locks, mallocing, or preemption is disabled, don't
// preempt.
// This check is very early in newstack so that even the status change
// from Grunning to Gwaiting and back doesn't happen in this case.
// That status change by itself can be viewed as a small preemption,
// because the GC might change Gwaiting to Gscanwaiting, and then
// this goroutine has to wait for the GC to finish before continuing.
// If the GC is in some way dependent on this goroutine (for example,
// it needs a lock held by the goroutine), that small preemption turns
// into a real deadlock.
preempt := stackguard0 == stackPreempt
if preempt {
if !canPreemptM(thisg.m) {
// Let the goroutine keep running for now.
// gp->preempt is set, so it will be preempted next time.
gp.stackguard0 = gp.stack.lo + _StackGuard
gogo(&gp.sched) // never return
}
}

... 省略
// 判断是否抢占
if preempt {
if gp == thisg.m.g0 {
throw("runtime: preempt g0")
}
if thisg.m.p == 0 && thisg.m.locks == 0 {
throw("runtime: g is running but p is not")
}

if gp.preemptShrink {
// We're at a synchronous safe point now, so
// do the pending stack shrink.
gp.preemptShrink = false
shrinkstack(gp)
}

if gp.preemptStop {
preemptPark(gp) // never returns
}

// Act like goroutine called runtime.Gosched. 直接让出CPU，类似runtime.GoSched()
gopreempt_m(gp) // never return
}
... 省略
}
```



### 基于信号量的抢占式调度
#### 实现原理
注册到统一的信号处理函数`runtime.sighandler`,接收到信号`SIGURG`时，转交给`doSigPreemt`抢占函数执行，由操作系统中断转入内核空间，而后将所中断线程的执行上下文参数(例如寄存器 rip、rep 等)传递给处理函数。如果在 runtime.sighandler 中修改了这个上下文参数，操作系统则会根据修 改后的上下文信息恢复执行

抢占信号处理函数:
```golang
// doSigPreempt handles a preemption signal on gp.
func doSigPreempt(gp *g, ctxt *sigctxt) {
	// Check if this G wants to be preempted and is safe to
	// preempt.
	if wantAsyncPreempt(gp) {
		if ok, newpc := isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()); ok {
			// Adjust the PC and inject a call to asyncPreempt.
			
			ctxt.pushCall(abi.FuncPCABI0(asyncPreempt), newpc)
		}
	}

	// Acknowledge the preemption.
	atomic.Xadd(&gp.m.preemptGen, 1)
	atomic.Store(&gp.m.signalPending, 0)

	if GOOS == "darwin" || GOOS == "ios" {
		atomic.Xadd(&pendingPreemptSignals, -1)
	}
}

// asyncPreempt saves all user registers and calls asyncPreempt2.
//
// When stack scanning encounters an asyncPreempt frame, it scans that
// frame and its parent frame conservatively.
//
// asyncPreempt is implemented in assembly.
func asyncPreempt()

//go:nosplit
func asyncPreempt2() {
    gp := getg()
    gp.asyncSafePoint = true
    if gp.preemptStop {
		//mcall 底层从当前gp切换到g0，之后执行preemptPark -> 
    mcall(preemptPark)  //mcall switches from the g to the g0 stack and invokes `preemptPark`,
	    // parks gp and puts it in _Gpreempted.
    } else {
    mcall(gopreempt_m) // 将上下文中的G写入调度队列中，等待被调度
    }
    gp.asyncSafePoint = false
}

```

#### 抢占触发时机
- GC标记阶段
- sysmon超时检测，触发抢占G

runtime/mgcmark.go
```golang
func markroot(gcw *gcWork, i uint32, flushBgCredit bool) int64 {
	        
	        ....
			// TODO: suspendG blocks (and spins) until gp
			// stops, which may take a while for
			// running goroutines. Consider doing this in
			// two phases where the first is non-blocking:
			// we scan the stacks we can and ask running
			// goroutines to scan themselves; and the
			// second blocks.
			stopped := suspendG(gp)
			if stopped.dead {
				gp.gcscandone = true
				return
			}
	        
			.....
	return workDone
}

// runtime/preempt.go
func suspendG(gp *g) suspendGState {
	
    ....
	case _Grunning:
	// Optimization: if there is already a pending preemption request
	// (from the previous loop iteration), don't bother with the atomics.
	if gp.preemptStop && gp.preempt && gp.stackguard0 == stackPreempt && asyncM == gp.m && atomic.Load(&asyncM.preemptGen) == asyncGen {
	break
	}

	// Temporarily block state transitions.
	if !castogscanstatus(gp, _Grunning, _Gscanrunning) {
	break
	}

	// Request synchronous preemption.
    gp.preemptStop = true
    gp.preempt = true
    gp.stackguard0 = stackPreempt

    // Prepare for asynchronous preemption.
    asyncM2 := gp.m
    asyncGen2 := atomic.Load(&asyncM2.preemptGen)
    needAsync := asyncM != asyncM2 || asyncGen != asyncGen2
    asyncM = asyncM2
    asyncGen = asyncGen2

    casfrom_Gscanstatus(gp, _Gscanrunning, _Grunning)
    
    // Send asynchronous preemption. We do this
    // after CASing the G back to _Grunning
    // because preemptM may be synchronous and we
    // don't want to catch the G just spinning on
    // its status.
    if preemptMSupported && debug.asyncpreemptoff == 0 && needAsync {
    // Rate limit preemptM calls. This is
    // particularly important on Windows
    // where preemptM is actually
    // synchronous and the spin loop here
    // can lead to live-lock.
    now := nanotime()
     if now >= nextPreemptM {
        nextPreemptM = now + yieldDelay/2
        preemptM(asyncM)
     }
    }		

	...	
}
```

参考资料:
[了解go在协程调度上的改进](https://cloud.tencent.com/developer/article/1938510)