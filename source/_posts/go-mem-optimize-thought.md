---
title: go内存优化分析思路
date: 2024-12-18 16:01:01
tags:
---

# 优化原则

- 切勿过早优化
- 善用语言分析工具

# 优化思路

1. 优先分配到栈空间
2. 一次性申请空间
3. 复用空间
4. 减少指针。指针会被编译器加入(= nil panic逻辑， 较值类型多判断逻辑)

**内存优化本质就是对栈空间和堆空间的优化**

# 编码阶段-栈空间

尽可能控制局部变量被分配到栈空间，减轻GC的扫描压力

- 逃逸分析

```go
go build -gcflags="-m"  package
```

常见逃逸场景:

- 函数外引用
- 局部变量太大
- 指针类型
- 接口类型，编译时无法确定大小，

常见规避逃逸场景:

- 字符串拼接，使用strings.Builder
- 局部变量map/slice，使用固定大小(非依赖外部参数场景)
- 慎用 time.Format(). 底层中[]byte会逃逸，使用time.AppendFormat(使用已知大小的byte)

模拟标准库`time.Now.Format`为例

```go
func Format(layout string) string {
	const bufSize = 64
	var b []byte
	max := len(layout) + 10
	if max < bufSize {
		var buf [bufSize]byte
		b = buf[:0]
	} else {
		b = make([]byte, 0, max)
	}
	b = AppendFormat(b, layout) //这里简化，只为了说明buf内存逃逸
	return string(b)
}

func AppendFormat(b []byte, c string) []byte {
	return []byte{}
}
```

执行：go build -gcflags "-m" main.go

![image.png](/images/go_mem_escape.png)

# 运行阶段-堆空间

go提供了强大的性能分析工具pprof，通常生产环境会以服务的形式打开pprof, 可以通过以下命令分析。

```go
go tool pprof http://目标机器:端口/debug/pprof/heap?seconds=采集周期

# 创建本地web服务，访问火焰图
go tool pprof -http=:8884 pprof文件 
```

上述两者也可以合并:

```go
go tool pprof -http=:8884  http://目标机器:端口/debug/pprof/heap?seconds=采集周期
```

## 内存泄漏问题

- 排查思路:

基础的内存泄漏，可以通过火焰图识别解决。

- 解决方案:

及时释放/或者LRU淘汰

## 临时对象优化

- 排查思路:
1. 通过prrof对cpu Profile 查看内存申请的调用占比.

```go
go tool pprof -http=:8883 http://目标机器:端口/debug/pprof/profile?seconds=采集周期
```

1. 提取**核心链路**中调用mallocgc的方法
- 常见解决方案:
1. 一次性申请空间, 比如slice/map, 初始化时传具体大小参数。并且像map，如果传了大小，可以避免rehash。
2. 使用单例模式。一般服务都是分层的，如service/dao等，链路中会NewXXXService, 使用sync.Once避免创建大量临时对象。
3. 去除不必要的数据结构。一般读接口会涉及到组装接口，都会用map存储映射数据，可以去除这个map，直接用slice 索引定位数据，能省下大量的map临时对象。
4. 复用资源。常见的比如从连接中读取数据, 通常会创建 bytes.Buffer，可使用sync.Pool复用

```go
var buffPool10K = sync.Pool{
	New: func() interface{} { return make([]byte, 10240) },
}

func GetBuffer() *bytes.Buffer {
	 return buffPool10K.Get().(*bytes.Buffer)
}

func PutBuffer(buff *bytes.Buffer) {
	buff.Reset()
	buffPool10K.Put(buff)
}
```

## GC优化

### GC优化的背景是什么?

1. 服务耗时影响
   GoGC三色标记之后会有STW，此时除了扫描协程外其他goroutine都是休眠的状态，即不执行任务逻辑。因此极端情况下一旦STW耗时变长，对时延敏感的服务，P99耗时可能会出现毛刺或者波动。

### 影响STW有哪些因素?

1. 清理垃圾对象的数量
2. 清理垃圾对象频率

### GC时机?

- 主动执行

```go
runtime.GC()
```

- sysmon线程定期执行

```go
NextGC = live data + GCPercent * live data
```

- 申请内存时执行, mallocgc

结论:

这里面看下来，最适合控制GC频率的就是GCPercent了。原因是我们服务中一般不会主动去执行GC 而mallocgc 无法干预。

### GC解决方案

所以GC优化方向一般就是通过调整GCPercent, 降低GC频率，这样内存占用就多了。本质还是空间换时间的思路。

不过！！这里需要注意，为防止OOM, 需要设置:

```go
GOMEMLIMIT 如果超过，会强制执行GC，防止OOM 
```

### GC排查思路

排查工具: trace

```go
curl "http://目标机器:目标端口/debug/pprof/trace?seconds"  > trace.out

go tool trace -http=127.0.0.1:8129 trace.out
```

通过trace可以得知以下信息:

- GC频率，看是否太过频繁
- **Minimum mutator utilization**，

mutator使用率越接近100%，GC影响越少，

![image.png](/images/gc_mutator.png)

tips:
1. 仅勾选”STW” ，mutator=0时，即为GC耗时
2. trace view中可以看到服务是不是并发的，具体来说看看服务协程是不是在同一时间端内跑