---
title: go内存优化分析思路
date: 2024-12-18 16:01:01
tags:
---

> 本文假设读者了解Go内存空间、堆栈等概念，具备一定的go使用经验
<!-- more -->

# 优化原则
- 切勿过早优化！！！！
- 善用语言分析工具

# 优化思路？
内存优化的目标就是把<span style="color: red;">不合理的、冗余、低效</span>的内存使用逻辑变成<span style="color: green;">合理、紧凑、高效的</span>

而程序中使用到的内存不是在堆空间就是在栈空间，因此优化的核心就是这俩个内存段。
go针对上述两种提供了完整的工具链，来帮助开发者定位和分析内存问题，最终写出高质量代码。
- 栈空间，使用 `go build -gcflags="-m -l" 包名"` 分析内存逃逸
- 堆空间，使用go自带的`pprof`分析程序堆内存使用情况。
    

# 栈空间
优化思路： 尽可能将局部变量被分配到栈空间，减轻GC的扫描压力，减少逃逸的局部变量。
## 分析工具
go在编译时通过`gcflags`分析特定包下所有函数变量的逃逸情况。
```go
# -l 禁止编译器内联优化
go build -gcflags="-m -l"  package
```

## 逃逸场景
- 函数外引用, return 
- 局部变量太大
- 指针类型
- 接口类型，编译时无法确定大小，
- 反射

## 常见逃逸优化
- 局部变量slice/map，尽量在编译阶段确定大小
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

执行：go build -gcflags "-m -l" main.go

![go_mem_escape](/images/go_mem_escape.png)

# 堆空间

## pprof
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
## 问题分类
1. 内存泄漏
2. GC-STW耗时

### 内存泄漏
1. 临时性 (大量临时对象，gc还没来的即清理，影响新对象的申请)
2. 永久性 (资源未关闭/释放, 文件/连接未关闭, 协程未释放)

#### 临时对象泄漏

- 排查思路:
1. 通过prrof对cpu Profile 查看内存申请的调用占比.

```go
go tool pprof -http=:8883 http://目标机器:端口/debug/pprof/profile?seconds=采集周期
```

2. 提取**核心链路**中调用mallocgc的方法
- 常见解决方案:
    1. 一次性申请空间, 比如slice/map, 初始化时传具体大小参数，规避扩容(rehash/growslice)逻辑。
    2. 使用单例模式。一般服务都是分层的，如service/dao等，链路中会NewXXXService, 使用sync.Once避免创建大量临时对象。
    3. 去除不必要的数据结构。一般读接口会涉及到组装数据，通常会用map存储映射数据方便定位，不过可以去除这个map，直接用slice索引定位数据，能省下大量的map临时对象。
```go
伪代码
func getComments(commentIds []int) map[int]commentInfo {
     
     []commentsInfo  <= comments:=  loadDataFromDB(commentIds)
    
     var map[int]commentInfo //可以移除， 直接返回[]commentsInfo。外部组装时，直接用索引定位数据
     for _, comm :=range comments {
        ret map[comm.ID] = comm
     }
    return 
 }
```
4. 复用资源。常见的比如从连接中读取数据, 通常会创建 bytes.Buffer，可使用sync.Pool

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


#### 永久性对象泄漏
排查思路:
1. 协程泄漏

    1. 排查渠道: 
       1. 一般可以通过监控看到协程数暴涨 
       2. pprof heap的时候会看到 `runtime.malg` 调用量很多，这说明go创建了很多

    2. 查看goroutine堆栈信息
        ```
        # debug=0:可以看到goroutine总数; 1: 可以看到goroutine堆栈信息(死锁问题也可以用)
        go tool pprof -http=:8088 http://目标机器:端口/debug/pprof/groutine?debug=1
        ```
 2. http请求的响应，要么读完要么一定要Close,否则底层readloop协程会因为底层channel没收到退出信号一致阻塞导致协程泄漏。

### GC优化

#### 为什么要GC优化?

1. 服务耗时影响

GC并发扫描完之后会有STW，此时其他goroutine都是休眠的状态，即不执行任何逻辑。因此极端情况下一旦STW耗时变长，对时延敏感的服务，P99耗时可能会出现毛刺或者波动。

##### 影响STW有哪些因素?

1. 垃圾对象的数量
2. 清理垃圾对象的频率

##### GC时机?

- 主动执行

```go
runtime.GC()
```

- sysmon线程定期执行

```go
# 计算下次GC的内存阈值
NextGC = live data + GCPercent * live data
```

- 申请内存时执行, mallocgc

结论:

这里面看下来，最适合控制GC频率的就是GCPercent了。原因是我们服务中一般不会主动去执行GC 而mallocgc 无法手动干预，只能减少申请对象。


#### GC排查思路

排查工具: trace

```go
curl "http://目标机器:目标端口/debug/pprof/trace?seconds"  > trace.out

go tool trace -http=127.0.0.1:8129 trace.out
```

通过trace可以得知以下信息:

- GC频率，看是否太过频繁
- **Minimum mutator utilization**， mutator使用率越接近100%，GC影响越少
  ![mutator使用率](/images/gc_mutator.png)

tips:
1. 仅勾选”STW” ，mutator=0时，即为GC耗时
2. trace view中可以看到服务是不是并发的，具体来说看看服务协程是不是在同一时间端内跑



#### GC解决方案

所以GC优化方向一般就是通过调整GCPercent, 降低GC频率，不过这样内存占用就多了，本质还是空间换时间的思路。

这里需要注意！！！！为防止OOM, 需要设置:

```go
GOMEMLIMIT 如果超过，会强制执行GC，防止OOM 

如图 容器是3GB内存，GOMEMLIMIT=2750MiB, 会自动强制执行GC。
```
![memory_limit](/images/memory_limit.png)


# 《GO编码建议》
[跳转查看](https://dablelv.github.io/go-coding-advice/)