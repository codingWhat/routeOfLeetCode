---
title: 服务可用性治理-实践篇
date: 2024-07-17 16:41:53
tags:
- GO
- 微服务
- 服务可用性
- 可用性治理
---
>  本文将围绕节点、架构以及外部工具三个方面介绍服务治理。通过理论结合实践的方式帮助读者逐步掌握相关概念和技能，并能将其应用于实际工作中。本文内容是基于个人学习和实践的总结，如果有任何错误或不准确之处，欢迎指正。

<!-- more -->

# 单节点
想象一下如果你的服务只部署在一个节点上，并且你的系统恰好面临以下问题，作为服务owner你会怎么应对呢?
- 如果有异常流量怎么办？(超过平时流量的数倍)
- 如果依赖的服务挂了怎么办？
- 如果服务过载了怎么办？

## 限流
![限流模型](/images/limiter.png)
我们知道单节点的处理能力是有限的，所以对于异常流量第一步我们要做的就是限流保护系统。限流主要有分为客户端限流、服务端限流以及分布式限流。这几种限流场景我在另一篇[《限流实战》](https://codingwhat.github.io/2024/07/09/limiter-in-action/)做了详细分析，所以这里不再重复了。



## 熔断
熔断为什么能提高可用性?   
1.当依赖服务/组件发生故障时，断路器会打开，此时无需请求下游，业务要么降级要么fast-fail，有效的避免了资源浪费以及资源堆积引发的雪崩效应。
2.断路器会通过探针探测服务是否恢复，如果发现服务恢复了，会继续放行请求，提升可用性。

<strong>断路器分类</strong>
- 传统断路器，[hystrix-go](https://github.com/afex/hystrix-go)/[sentinel-go](https://github.com/alibaba/sentinel-golang/tree/master/core/circuitbreaker)
- 自适应熔断, google sre-breaker

### 传统断路器
![传统断路器](/images/circuit_breaker.png)
网上介绍断路器的文章很多, 本文偏实战这里就不详细介绍了, 我这里挑重点介绍
<strong>状态机原理:</strong>  
它是一个状态机模型，通过状态切换处理故障减少对主调的影响，主要包含三种状态:打开(Open)、半打开(Half-Open)、关闭(Closed)
以下是断路器的基本逻辑：
1.初始状态下，断路器处于关闭状态（Closed）。
2.当下游服务出现故障时，断路器会统计相应的指标，如错误率、错误数或慢调用。如果这些指标超过了用户定义的阈值（比如错误数的静默数等），则断路器进入打开状态（Open）。
3.在打开状态下，断路器会等待一个用户定义的时间窗口，然后进入半打开状态（Half Open）。
4.在半打开状态下，断路器会放行一定数量的探针请求，并根据探针的结果进行判断（用户定义探针数和探针成功的比例）。
5.如果所有的探针请求都成功，说明下游服务已经恢复正常，断路器将重新进入关闭状态（Closed）。
6.如果探针请求失败或未达到成功比例的要求，断路器将继续保持打开状态（Open），并继续进行探测。

断路器的优点在于它提供了丰富的配置选项，可以根据具体需求来设置错误率、慢调用比例、错误数等指标。然而，由于配置项较多，准确地配置这些值可能会有一定的挑战。

<details>
<summary> hystrix-go实现</summary>
```golang
package main

import (
	"fmt"
	"net/http"
	"time"

	"github.com/afex/hystrix-go/hystrix"
)

func main() {
	// 设置一个命令名为"callOutRPC"的断路器
	hystrix.ConfigureCommand("callOutRPC", hystrix.CommandConfig{
		Timeout:                int(3 * time.Second), // rpc调用超时时间
		MaxConcurrentRequests:  10,                   // 并发请求10个，用chanel控制
		SleepWindow:            5000,                 //单位ms, open->half open 睡眠窗口
		RequestVolumeThreshold: 10,                   // 静默数，这里就是错误数必须要>=10个
		ErrorPercentThreshold:  30,                   //错误率阈值
	})

	_ = hystrix.Do("callOutRPC", func() error {
		// 尝试调用远端服务
		_, err := http.Get("https://www.1baidu.com")
		if err != nil {
			return err
		}
		return nil
	}, func(err error) error {
		// 快速失败时的回调函数
		fmt.Println("call rpc failed. now calling fallback logic")
		return nil
	})
}
```
</details>
<details>
<summary>sentinel-go实现</summary>
```golang

func main () {
    if err := InitCircuitBreaker(); err != nil {
        panic(err)
    }
    
	e, b := sentinel.Entry("calleeSrv")
	if b != nil {
	    // 触发熔断
	    // metric上报
		return ret, b
	}
	err := callOutRpc()
	e.Exit(base.WithError(err))
}

func callOutRpc() error {
    time.Sleep(1 * time.Second)
    return errors.New("happend error")
}
// InitCircuitBreaker 初始化断路器
func InitCircuitBreaker() error {
	err := sentinel.InitDefault()
	if err != nil {
		return err
	}
	defaultRules := []*circuitbreaker.Rule{
		{
			Resource:                     "calleeSrv",                  // 名字
			Strategy:                     circuitbreaker.SlowRequestRatio, // 慢查询
			RetryTimeoutMs:               5000,                            // 5s后尝试恢复，进入half状态
			MinRequestAmount:             100,                             // 静默数 Open的前置条件, 100，主要针对热点
			StatIntervalMs:               2000,                            // 2s钟慢查询比例不超过0.4
			StatSlidingWindowBucketCount: 100,                             // 每个格子 20ms
			MaxAllowedRtMs:               130,                             // (120 + 10(buffer)))毫秒以外算慢查询
			Threshold:                    0.5,                             // 5s钟慢查询比例不超过0.4
			ProbeNum:                     10,
		},
	}
	circuitbreaker.RegisterStateChangeListeners(&stateChangeTestListener{})
	_, err = circuitbreaker.LoadRules(defaultRules)
	return err
}

type stateChangeTestListener struct {
}

// OnTransformToClosed 转换至关闭状态回调函数
func (s *stateChangeTestListener) OnTransformToClosed(prev circuitbreaker.State, rule circuitbreaker.Rule) {
	CircuitBreakerClosed.Inc()
	log.Infof("rule.strategy: %+v, From %s to Closed, time: %v\n", rule.Strategy, prev.String(),
		util.FormatTimeMillis(util.CurrentTimeMillis()))

}

// OnTransformToOpen 转换至开启状态回调函数
func (s *stateChangeTestListener) OnTransformToOpen(prev circuitbreaker.State, rule circuitbreaker.Rule,
	snapshot interface{}) {
	CircuitBreakerOpen.Inc()
	log.Infof("rule.strategy: %+v, From %s to Open, snapshot: %.2f, time: %v\n", rule.Strategy, prev.String(),
		snapshot, util.FormatTimeMillis(util.CurrentTimeMillis()))
}

// OnTransformToHalfOpen 转换至半开状态回调函数
func (s *stateChangeTestListener) OnTransformToHalfOpen(prev circuitbreaker.State, rule circuitbreaker.Rule) {
	CircuitBreakerHalfOpen.Inc()
	log.Infof("rule.strategy: %+v, From %s to Half-Open, time: %v\n", rule.Strategy, prev.String(),
		util.FormatTimeMillis(util.CurrentTimeMillis()))
}
```
</details>

### 自适应断路器
<strong>google sre-breaker</strong> 出自书籍《Google SRE》中Handling Overload节

![谷歌自适应断路器-核心算法](/images/sre_breaker.png)
传统断路器在Open状态时有固定的等待窗口，窗口内会拒绝所有请求，以此来保障服务的稳定性，不过在短暂抖动的场景下，这种策略反而会降低服务的可用性(比如等待窗口10s, 可是服务3s就恢复了，还得等待7s才会进入half-open), GoogleSRE提供了一个解决方案-自适应客户端限流，通过滑窗口统计成功请求数、请求总数采用上图算法决策是否放行探针，较传统方式能更快的感知到下游服务的恢复情况
算法: f(x) = max(0, (requests - K * accepts)/(requests+1))

<strong>算法剖析：</strong>
初始状态: requests == accepts,假设K=0.5,
无故障时: f(x) >  0, 当故障开始时, requests变大, accepts保持不变, f(x)一直 > 0
假设K=1
无故障时: f(x) == 0, 当故障开始时, requests变大, accepts保持不变，当requests > 1 * accepts时 f(x) > 0
假设K=2
无故障时: f(x) == 0, 当故障开始时, requests变大, accepts保持不变, 当requests > 2 * accepts时, f(x) > 0

综上，K是调节熔断刚性的因子，当K>=1,偏柔性，比如K=1, 此时相当于能容忍accepts个请求通过，当K<1, 则偏刚性，直接拒绝了。


<strong>总结:</strong>
- 少了很多自定义配置，开发只需要调节K这个变量; K越小越激进
- 实时性更好点，不会有固定的等待窗口


<strong>代码实现</strong>  
可以参考[B站实现](https://github.com/go-kratos/kratos/blob/v1.0.x/pkg/net/netutil/breaker/sre_breaker.go)

![B站使用效果](/images/bilibili_sre.png)

## 超时控制
<strong>超时控制的必要性</strong>
- 防止资源浪费，不能及时释放资源，满足fast-fail
- 避免雪崩效应，故障扩散

### 超时策略
- 固定超时
- EMA动态超时

### 固定超时
- 链路超时
- 服务内超时

<strong>链路超时控制:</strong>  
假设链路调用关系: A(300ms)->B(200ms)->C(100ms), 链路总超时为1s
链路超时传递为: A(used:300ms, left:700) -> B (used:200ms, left:500ms) -> C (used:100ms, left:400ms)

![链路超时传递](/images/timeout_propagation.png)


<strong>如何传递?</strong>
- grpc中是通过http2的HEADERS Frame透传， `grpc-timeout` 字段

<strong>服务内超时控制:</strong>
假设服务超时为600ms，有三个串行的RPC调用[A(500ms), B(300ms), C(100ms)]，在调用B时会等待300ms后会触发超时。很明显这里超时是可以优化的, timeout=min(timeout, time.Util(deadline)), 比如deadline剩10ms，rpc超时为100ms，那就没必要等100ms。(http请求，go已经帮我们做好了)
<strong>如何传递?</strong>
<details>
  <summary> 利用context.WithTimeout 实现</summary>


```
package main

import (
	"context"
	"fmt"
	"log"
	"time"
)

func main() {
	// 创建一个上下文，并设置总超时时间为600毫秒
	ctx, cancel := context.WithTimeout(context.Background(), 600*time.Millisecond)
	defer cancel()

	// 启动A、B、C三个调用，并传递父上下文
	callA(ctx)
	callB(ctx)
	callC(ctx)

	// 等待1秒钟，等待所有调用完成
	time.Sleep(time.Second)
}

func callA(parentCtx context.Context) {
	// 根据父上下文的截止时间计算A调用的超时时间
	deadline, ok := parentCtx.Deadline()
	if !ok {
		log.Println("Parent context does not have a deadline")
		return
	}
	timeout := 500 * time.Millisecond
	if timeout > time.Until(deadline) && time.Now().Before(deadline) {
		timeout = time.Until(deadline)
	}
	fmt.Println("callA--->", time.Until(deadline))

	// 创建一个子上下文，并设置A调用的超时时间
	ctx, cancel := context.WithTimeout(parentCtx, timeout)
	defer cancel()

	select {
	case <-time.After(500 * time.Millisecond):
		log.Println("Call A completed")
	case <-ctx.Done():
		log.Println("Call A timed out")
	}
}

func callB(parentCtx context.Context) {
	// 根据父上下文的截止时间计算B调用的超时时间
	deadline, ok := parentCtx.Deadline()
	if !ok {
		log.Println("Parent context does not have a deadline")
		return
	}
	fmt.Println("callB--->", time.Until(deadline))
	timeout := 300 * time.Millisecond
	if timeout > time.Until(deadline) && time.Now().Before(deadline) {
		timeout = time.Until(deadline)
	}

	// 创建一个子上下文，并设置B调用的超时时间
	ctx, cancel := context.WithTimeout(parentCtx, timeout)
	defer cancel()

	select {
	case <-time.After(300 * time.Millisecond):
		log.Println("Call B completed")
	case <-ctx.Done():
		log.Println("Call B timed out")
	}
}

func callC(parentCtx context.Context) {
	// 根据父上下文的截止时间计算C调用的超时时间
	deadline, ok := parentCtx.Deadline()
	if !ok {
		log.Println("Parent context does not have a deadline")
		return
	}

	timeout := 100 * time.Millisecond
	if timeout > time.Until(deadline) && time.Now().Before(deadline) {
		timeout = time.Until(deadline)
	}
	// 创建一个子上下文，并设置C调用的超时时间
	ctx, cancel := context.WithTimeout(parentCtx, timeout)
	defer cancel()

	select {
	case <-time.After(100 * time.Millisecond):
		log.Println("Call C completed")
	case <-ctx.Done():
		log.Println("Call C timed out")
	}
}
```
</details>

### 动态超时
传统的超时控制一般会根据下游服务p90、p95超时时间静态设置，然而，当网络出现短暂的抖动时，会导致一些请求的响应时间变得异常长，从而产生长尾请求。为了解决这个问题，可以采用动态超时的方法，通过统计历史数据（包括成功和失败的请求数）来动态调整超时时间。
[ema算法原理](https://github.com/fefeding/ema-timeout):
当平均响应时间(EMA)大于超时时间(Thwm)说明网络环境比较差，动态超时时长(Tdto)就会趋向于超时时长(Thwm), 缩短超时时长。当平均响应时间(EMA)小于超时时间(Thwm),说明平均情况表现很好，动态超时时长就可以适当超出超时时间(Thwm),但要小于最大弹性时间(Tmax).
![EMA动态超时控制算法](/images/ema.png)

代码实现:
```golang
package main

import (
	"fmt"
	"math"
	"math/rand"
)

type Ema struct {
	options map[string]float64
	ema     float64
	r       float64
}

/*
*      Tavg: 最低响应时间， 一般用平均响应时间替代 (ms)
*      Thwm：超时时间限制， 确保最坏的时候，所有请求能处理。正常时正确处理的成功率满足需求。 (ms)
*      Tmax: 最大弹性时间 (ms)
*      N: 平滑指数， 平滑因子决定了最新数据的权重，越大，最新数据的权重越高，EMA对数据的变化更加敏感。而旧数据的权重则通过(1-α)进行衰减，随着时间的推移，旧数据的影响逐渐减小。
*/
func NewEma() *Ema {
	options = map[string]float64{
		"Tavg": 60,
		"Thwm": 250, //超时时间
		"Tmax": 500, //最大超时时间
		"N":    50,
	}
	return &Ema{
		options: options,
		ema:     0, //平均响应时间
		r:       2 / (options["N"] + 1),
	}
}

func (e *Ema) Update(x float64) float64 {
	// 满足指数滑动平均值
	ema := x*e.r + e.ema*(1-e.r)
	e.ema = ema
	return ema
}

func (e *Ema) Get() float64 {
	var tdto float64
	if e.ema <= e.options["Tavg"] {
		tdto = e.options["Tmax"]
	} else if e.ema >= e.options["Thwm"] {
		tdto = e.options["Thwm"]
	} else {
		p := (e.options["Thwm"] - e.ema) / (e.options["Thwm"] - e.options["Tavg"])
		tdto = e.options["Thwm"] + p*(e.options["Tmax"]-e.options["Thwm"])
	}
	return math.Abs(tdto)
}

func main() {
	ema := NewEma()

	for i := 0; i < 100; i++ {
		a := rand.Float64() * 200
		e := ema.Update(a)
		t := ema.Get()
		fmt.Println(a, e, t)
	}

	for i := 0; i < 100; i++ {
		a := rand.Float64()*200 + 500
		e := ema.Update(a)
		t := ema.Get()
		fmt.Println(a, e, t)
	}
}
```

使用方法:
- 非关键链路
缩短超时时间(Thwm), 相对小, 降低非核心链路耗时以及资源消耗
- 关键链路
延长超时时间(Thwm), 能有效缓解短暂网络抖动导致的长尾请求

### 超时时间选择
- 存量服务可以选用p99、p95作为超时时间

## 降级
降级一般有以下几种策略
- 一致性降级，强一致变弱一致
- 功能降级，下线非核心功能
- 用户体验降级, 不展示用户标签、个性化信息等
- 同步转异步，同步逻辑转化为异步，会有些延迟

降级一般都和限流、熔断放在一起讨论，适合具体问题具体分析，本质是提供有损服务。这里就不多介绍理论内容，我给大家举几个实际场景，感受下即可。
1. 双11为了节省资源，tb或pdd会暂时关闭退货功能
2. 视频平台推荐页会缓存首页的数据，防止进来就是白页
3. 评论列表里有用户的各种信息，比如勋章等身份信息，如果获取失败这里返回空
4. 还有一些计数场景，app评论/点赞，如果是同步操作，很容易因为网络问题直接报错体验不好。一般都是异步静默提交，页面做假显。

## 重试

### 重试识别
可以通过http staus code识别错误类型，比如4xx类型明显就是请求有问题就别重试了；还有些情况可能需要根据响应中code码去识别，比如参数错误、鉴权失败等也不应该重试。
### 重试策略
确认重试之后, 首先要限制重试的比例，其次重点关注重试次数和重试间隔，重试间隔我们可以采用以下策略:
- 固定间隔, interval: base; 实现简单但是这种策略很容易出现重试波峰
- 随机间隔, interval: base + rand; 打散重试时间，减少重试波峰；虽然每个请求重试时间不一样，但是下游如果短时间内不能恢复，就会收到大量请求可能会造成服务雪崩。
- 随机 + 指数退避, interval: (exp)^retryNum + rand; 减少了重试波峰以及对下游的重试压力；超时配置需要注意，不要影响核心链路的耗时

```golang 

type RetryStrategy int

const (
Fixed  RetryStrategy = 0 // 固定值, n, n, n...
Linear RetryStrategy = 1 // 线性, n, 2n, 3n...
Exp    RetryStrategy = 2 // 指数, n, 2n, 4n, 8n...
Rand   RetryStrategy = 3 // 随机, [n, 2n]
)

func sleep(i, milliSec int, s RetryStrategy) time.Duration {
	n := milliSec
	switch s {
	case Linear:
		n = i*milliSec + milliSec
	case Exp:
		n = int(math.Pow(2, float64(i))) * milliSec
	case Rand:
		n = rand.Intn(milliSec+1) + milliSec
	default:
	}
	return time.Millisecond * time.Duration(n)
}
```

### 对冲策略
这个概念源自GRPC, 是指在不等待响应的情况下主调主动发送多个请求，本质是更加激进的重试。 适用于一些流量不大的场景，可以缓解短暂网络抖动导致的长尾请求，不过一定确认好重试对下游负载的影响。
如下图，假设主调和被调超时时间为60ms，第一个请求发出之后会触发一个10ms定时器, 假设主调在10ms内没有收到响应，定时器就会触发立即发送重试请求，如果重试请求响应先返回了，就会立即返回，第一个请求的响应会被主调丢弃。
![对冲模型](/images/hedging.png)

<details> <summary>对冲模拟实现</summary>
```golang
func main() {

	request, err := http.NewRequest("Get", "http://www.baidu.com", nil)
	if err != nil {
		panic(err)
	}
	hedged, err := retryHedged(request, 3, 10*time.Millisecond, 10*time.Second, Backoff)
	fmt.Println(hedged, err)
}

type RetryStrategy func(int) time.Duration

func Backoff(retryNum int) time.Duration {
	return time.Duration(retryNum*2+2) * time.Millisecond
}

func retryHedged(req *http.Request, maxRetries int, hedgeDelay time.Duration, reqTimeout time.Duration, rs RetryStrategy) (*http.Response, error) {
	var (
		originalBody []byte
		err          error
	)
	if req != nil && req.Body != nil {
		originalBody, err = copyBody(req.Body)
	}
	if err != nil {
		return nil, err
	}

	AttemptLimit := maxRetries
	if AttemptLimit <= 0 {
		AttemptLimit = 1
	}

	client := http.Client{
		Timeout: reqTimeout,
	}

	// 每次请求copy新的request
	copyRequest := func() (request *http.Request) {
		request = req.Clone(req.Context())
		if request.Body != nil {
			resetBody(request, originalBody)
		}
		return
	}

	multiplexCh := make(chan struct {
		resp  *http.Response
		err   error
		retry int
	})

	totalSentRequests := &sync.WaitGroup{}
	allRequestsBackCh := make(chan struct{})
	go func() {
		totalSentRequests.Wait()
		close(allRequestsBackCh)
	}()
	var resp *http.Response

	var (
		canHedge   uint32
		readyHedge = make(chan struct{})
	)
	for i := 0; i < AttemptLimit; i++ {
		totalSentRequests.Add(1)

		go func(i int) {
			if atomic.CompareAndSwapUint32(&canHedge, 0, 1) {
				go func() {
					<-time.After(hedgeDelay)
					readyHedge <- struct{}{}
				}()
			} else {
				<-readyHedge
				time.Sleep(rs(i))
			}
			// 标记已经执行完
			defer totalSentRequests.Done()
			req = copyRequest()
			resp, err = client.Do(req)
			if err != nil {
				fmt.Printf("error sending the first time: %v\n", err)
			}
			// 重试 500 以上的错误码
			if err == nil && resp.StatusCode < 500 {
				multiplexCh <- struct {
					resp  *http.Response
					err   error
					retry int
				}{resp: resp, err: err, retry: i}
				return
			}
			// 如果正在重试，那么释放fd
			if resp != nil {
				resp.Body.Close()
			}
			// 重置body
			if req.Body != nil {
				resetBody(req, originalBody)
			}
		}(i)
	}

	select {
	case res := <-multiplexCh:
		return res.resp, res.err
	case <-allRequestsBackCh:
		// 到这里，说明全部的 goroutine 都执行完毕，但是都请求失败了
		return nil, errors.New("all req finish，but all fail")
	}
}
func copyBody(src io.ReadCloser) ([]byte, error) {
	b, err := io.ReadAll(src)
	if err != nil {
		return nil, err
	}
	src.Close()
	return b, nil
}

func resetBody(request *http.Request, originalBody []byte) {
	request.Body = io.NopCloser(bytes.NewBuffer(originalBody))
	request.GetBody = func() (io.ReadCloser, error) {
		return io.NopCloser(bytes.NewBuffer(originalBody)), nil
	}
}
```
</details>

### 重试总结
1. 明确好哪些情况下才能重试
2. <font color="red"> 重试只在当前层. </font> 当重试失败时，应该约定全局错误码，“no need retry” 避免及联重试
3. 一定注意<font color="red">随机化重试间隔时间</font>，避免重试波峰
4. 下游一定是幂等的，不能产生副作用

# 架构
## 冗余架构
- 同城灾备
- 同城双活
- 两地三中心
- 异地双活
这部分可以直接看[这篇文章](http://kaito-kidd.com/2021/10/15/what-is-the-multi-site-high-availability-design/)写的非常好

这里以双中心架构为例(不过同城双活、两地三中心更常见)
![双中心架构](/images/two_idc.png)
整体架构通过GSLB将流量划分为北京和广州，覆盖接入层、服务层、存储层，形成异地"双活"(一般都是读双活，写双活成本高，一致性容易出问题) 
- 服务层做读写分离，以上图为例, 北京(读写), 广州(读), 广州的写请求路由到北京;
- 存储也是各自一套，以mysql为例, master节点在北京(因为北京负责写)，广州为备份节点，自动从北京master异步同步，保持最终一致。

用户请求链路:
用户(北京、电信用户) 请求经过DNS(GSLB)域名解析返回北京电信sgtw的IP，之后请求会转发到stgw，由sgtw通过内网隧道转发到API网关。
- sgtw是我司收敛不同运营商的网关(七层流量负载均衡; 解决跨运营商访问延时大的问题)


### 单元化架构
单元化架构是将系统划分成不同的业务单元，将同一单元的业务部署在一个机房，在单元内实现业务自包含。如下图服务的上下游组成独立、完整的单元并且部署在一起，链路调用绝对不会产生跨机房延时问题。
![单元化架构](/images/set_arch.png),

<strong>单元架构分类</strong>
这里以阿里云的介绍为例：
RZone（Region Zone）：部署按用户维度拆分的关键业务系统。核心业务和数据单元化拆分，拆分后分片均衡，单元内尽量自包含（调用封闭），拥有自己的数据，能完成所有业务。一个可用区可以有多个 RZone。
GZone（Global Zone）：部署未按用户维度拆分的系统，被 RZone 依赖，提供不可拆分的数据和服务，如配置型的业务。数据库可以和 RZone 共享，多租户隔离，全局只有一组，可以配置流量权重。
CZone（City Zone）：部署未按用户维度拆分的系统，被 RZone 高频访问 ，解决跨域通信延时问题。为了解决异地延迟问题而特别设计，适合读多写少且不可拆分的业务。 一般每个城市一套应用和数据，是 GZone 的快照。

![单元化架构-单元类型](/images/logic_set_type.png)


## 自动故障转移
- API网关故障转移
- 客户端故障转移
- 自适应重试
- 负载均衡

### API网关故障转移 
如果网关发现本地AZ不可用时，会自动故障转移，路由到其他地域的AZ解决故障。为了防止重试流量压垮异地，需要控制重试次数
![网关故障转移](/images/api_gateway_failover.png)

### 客户端故障转移
客户端就近访问本地网关时，如果发现本地网关不可用，可以重试异地网关解决故障。

### 自适应重试
<strong>设置合理的重试窗口</strong>
实现思路: 依旧请求结果[成功率]来动态调整重试窗口。若成功率满足阈值，当前重试次数超过窗口，则适当增加窗口，否则保持窗口大小；若不满足成功率,窗口阈值=max(1, num/2)
```golang
package main

import (
	"fmt"
	"math/rand"
)

type RetryLimiter struct {
	CurRetryWindowSize int //重试窗口
	CurUsedQuota       int
}

// GetRetryQuota 获取重试配额
// succRate 滑窗统计最近成功率，比如最近5s
// retryProbeNum: 重试次数
// reqIdx: 本地请求总次数
func (l *RetryLimiter) GetRetryQuota(succRate float64, retryProbeNum int, reqIdx int) int {
	if succRate > 0.9 {
		if retryProbeNum >= l.CurRetryWindowSize {
			// 取当前请求流量1%作为增量，同时min函数确保窗口调整的增量不超过当前窗口大小，保持调整的平稳性
			l.CurRetryWindowSize = l.CurRetryWindowSize + max(min(1*reqIdx/100, l.CurRetryWindowSize), 1)
		}
	} else {
		l.CurRetryWindowSize = max(1, l.CurRetryWindowSize/2)
	}
	
	return l.CurRetryWindowSize
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func main() {

	l := RetryLimiter{
		CurRetryWindowSize: 10,
	}

	for i := 1; i < 100; i++ {
		succRate := float64(i) * 0.1
		if i > 50 {
			succRate *= 0.1
		}
		//retryNum := rand.Int() % 10
		retryProbeNum := rand.Int() % 40
		fmt.Println("req:", i, ", succRate:", succRate, ", get retry quota:", l.GetRetryQuota(succRate, retryProbeNum, i))
	}
}
```
## 负载均衡
### 前端负载均衡
这部分借鉴自《Google SRE》，主要是通过DNS和Maglev集群去实现分流, 简单来说请求先通过DNS拿到接入层外网ip, 之后发起VIP请求到Maglev节点上(VIP基于keepalive), Maglev也是4层软件负载和LVS类似,有兴趣可以看下[这篇文章](https://www.manjusaka.blog/posts/2020/05/22/a-simple-introduction-about-maglev/index.html)
![Google-maglev负载均衡](/images/maglev.png)

国内用lvs居多，大体也类似:
![前端负载均衡](/images/fe_lb.png)

### 数据中心内负载均衡
<strong>Subset(子集算法限制海量连接)</strong>
在微服务架构下，服务之间不仅会有“正常的”rpc调用，也会有心跳请求探测依赖服务的存活。问题来了假设当前服务依赖的下游服务很多，并且如果下游又是冗余了多个集群，那么势必需要建立大量的tcp连接(连接数=clients*backends)，再加上后续需要会有大量的心跳包，占用了大量cpu资源，面对海量连接client该如何处理?
![子集算法](/images/google_subset.png)

<strong>常见策略</strong>
- 轮训
- 最少连接数(inflight)
- 轮训加权,(成功+，失败-) + cpu使用率
- [the choice of two] (https://medium.com/the-intuition-project/load-balancing-the-intuition-behind-the-power-of-two-random-choices-6de2e139ac2f)

<strong>轮训:</strong>
理想情况下流量被平均分配之后，下游节点之间的cpu负载差异应该都不相上下，可是实际情况是节点之间的负载差异可能会很大，导致很多资源被浪费，原因如下:
- 请求处理成本不一致
- 机器资源/配置不一致
- 性能因素: GC
因此轮训在生产环境很少会使用，毕竟真实环境的请求处理成本一定是不均衡的。

<strong>最少连接数(inflight)</strong>
统计每个连接的inflight请求数, 请求转发到请求最少的节点上。但还是存在请求处理成本的问题，虽然某些节点连接数少，但是万一有个请求成本很高，还是导致负载不均衡。

<strong>加权轮训</strong>
以上两种负载均衡都是从client端出发，没有从下游负载去考虑，导致下游负载不均。所以轮训加权的实现思路是依据请求<strong>响应结果</strong>[成功/失败]以及下游服务<strong>cpu使用率</strong>来动态控制节点权重(cpu使用率是通过rpc回报获取)。

<strong>best of two random choices</strong>
加权轮训的设计由于“信息滞后”存在“羊群效应”问题，原因有2点, 第一client至少需要1个RTT才能拿到cpu使用率，存在网络、先后请求延迟。第二“定期"更新节点权重。因此client以为拿到了最优节点，但实际请求的是“已经从不饱和变饱和”的节点，导致大量请求超时/拒绝。
best of two random choices，则采用了带时间衰减的指数衰减(exponentially weighted moving average)[带系数的指数衰减]，引入了inflight，lag作为负载均衡的参考

![two_of_random_choices](/images/two_of_random_choices.png)
<strong>算法实现</strong>
[B站实现](https://github.com/go-kratos/kratos/blob/4a93aa9b8d5dca550cc60a0c51c4726f83a2e6f8/pkg/net/rpc/warden/balancer/p2c/p2c.go)
![算法实现](/images/two_of_random_choices_algo.png)


## 分布式限流
- 即时消费即时结算
- 先消费后结算
- 预分配
这部分内容就不重复了，直接看[限流实战](https://codingwhat.github.io/2024/07/09/limiter-in-action/)

## 隔离
- 动静隔离
- 线程隔离
- 进程隔离(容器部署)
- 租户隔离
- 核心隔离
- 读写隔离
- 热点隔离
- 集群隔离
### 动静隔离
- 静态资源, CDN缓存html、css等静态资源
- 动态资源，接口获取

### 线程隔离
- java会通过不同线程池处理请求，划分cpu资源
- Go不适用，Go调度模型就会复用线程，无法做隔离，只能控制goroutine个数

### 进程隔离
- 目前微服务架构基于容器部署，都是独立进程、cpu、内存资源互不影响

### 租户隔离
- 不同租户请求的不同服务/存储

### 核心隔离
核心隔离通常是指将资源按照 `核心业务` 与 `非核心业务` 进行划分，优先保障 `核心业务` 的稳定运行
核心/非核心故障域的差异隔离（机器资源、依赖资源）  

核心业务可以搭建多集群通过冗余资源来提升吞吐和容灾能力

按照服务的核心程度进行分级  
1级：系统中最关键的服务，如果出现故障会导致用户或业务产生重大损失  
2级：对于业务非常重要，如果出现故障会导致用户体验受到影响，但不会导致系统完全无法使用  
3级：会对用户造成较小的影响，不容易注意或很难发现  
4级：即使失败，也不会对用户体验造成影响  

### 读写隔离
- 存储读写分离(redis/mysql/es)
- 应用层读写分离，CQRS
- 事件驱动，写操作之后发布事件，读服务监听修改


### 热点隔离
- 实时统计 + 热点识别 + 多级缓存 
- 热点监控

### 集群隔离
每个服务部署独立的集群


# 外部工具
## 混沌工程
通过注入cpu高负载、网络超时等故障，主动找出系统中薄弱环节
## 全链路压测
生产环境模拟真实用户压测, 提前发现隐藏问题。
压测过程：压测流量构建 -> 流量染色(隔离, 影子库(redis/mysql)) -> 压测引擎启动  -> 可视化监控

如果你司没有自研平台，也可以用云厂商提供的。
<strong>故障演练</strong>
[腾讯云混沌工程](https://cloud.tencent.com/product/cfg)
[阿里云Chaos](https://www.aliyun.com/product/aliware/ahas/chaos?spm=5176.21213303.J_qCOwPWspKEuWcmp8qiZNQ.17.7da82f3dazso4O&scm=20140722.S_product@@%E4%BA%91%E4%BA%A7%E5%93%81@@1157223._.ID_product@@%E4%BA%91%E4%BA%A7%E5%93%81@@1157223-RL_%E6%B7%B7%E6%B2%8C%E5%B7%A5%E7%A8%8B-LOC_llm-OR_ser-V_3-RE_new4@@cardOld-P0_1)

<strong>全链路压测</strong>
[腾讯云-PTS](https://cloud.tencent.com/product/pts)
[阿里云-PTS](https://www.aliyun.com/product/pts?spm=5176.21213303.J_qCOwPWspKEuWcmp8qiZNQ.20.13fc2f3dkx6uFr&scm=20140722.S_card%40%40%E4%BA%A7%E5%93%81%40%40596788.S_card0.ID_card%40%40%E4%BA%A7%E5%93%81%40%40596788-RL_%E5%85%A8%E9%93%BE%E8%B7%AF%E5%8E%8B%E6%B5%8B-LOC_search%7EUND%7Ecard%7EUND%7Eitem-OR_ser-V_3-RE_cardOld-P0_0)
