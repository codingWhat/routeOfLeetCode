---
title: 限流实战
date: 2024-07-09 17:42:14
tags:
- GO
- 可用性治理
- 限流
- 单机限流
- 集群限流
---

> 有经验的开发者都知道即便事前做了不同规模的容量模型，但是还是没办法准确预测未知的外部流量，因此服务必须得采取自保护策略，丢弃掉部分流量来保障服务的稳定性。

<!-- more -->
接下来我们会围绕静态、动态以及集群限流去讲解限流在不同场景下的工程实践。

# 静态限流
[标准库-令牌桶](golang.org/x/time/rate), 应对小规模突发流量;
[Uber-漏桶](https://github.com/uber-go/ratelimit), 匀速限流; 突发流量丢弃量多; !!这个库(v0.3.0)有bug[点击](https://colobu.com/2023/12/05/two-bugs-of-uber-ratelimit/)
滑动窗口, 精度高; 占用内存
固定窗口, 实现简单; 不精准，存在边界问题

总结:
- 实现简单
- 基于QPS限流静态限流, 无法根据服务的负载动态限流  
- 限流阈值不好配置(请求的处理成本不一致)  
- 节点扩缩, 需要重新设置


## 动手实践-令牌桶
核心逻辑源自标准库的rate包
```
type TokenBucket struct {
	rate       float64    // 令牌添加到桶中的速率。
	burst      int        // 桶的最大容量。
	tokens     float64    // 当前桶中的令牌数量。
	lastUpdate time.Time  // 上次更新令牌数量的时间。
	mu         sync.Mutex // 互斥锁，确保线程安全。
}

func (tb *TokenBucket) tokensFromDuration(d time.Duration) float64 {
	// Split the integer and fractional parts ourself to minimize rounding errors.
	// See golang.org/issues/34861.
	sec := float64(d/time.Second) * tb.rate
	nsec := float64(d%time.Second) * tb.rate
	return sec + nsec/1e9
}

// NewTokenBucket 创建一个新的令牌桶，给定令牌添加速率和桶的容量。
func NewTokenBucket(rate float64, b int) *TokenBucket {
	return &TokenBucket{
		rate:   rate,
		burst:  b,
		tokens: 0,
	}
}
func (tb *TokenBucket) durationFromTokens(tokens float64) time.Duration {
	seconds := tokens / tb.rate
	return time.Nanosecond * time.Duration(1e9*seconds)
}

// Allow 检查是否可以从桶中取出一个令牌。如果可以，它取出一个令牌并返回 true。
// 如果不可以，它返回 false。
func (tb *TokenBucket) Allow() bool {
	tb.mu.Lock()
	defer tb.mu.Unlock()

	now := time.Now()
	// 计算（可生成令牌数)所需要的时间，burst令牌桶容量，tokens: 当前存在的令牌个数
	maxElapsed := tb.durationFromTokens(float64(tb.burst) - tb.tokens)
	elapsed := now.Sub(tb.lastUpdate)
	if elapsed > maxElapsed {
		elapsed = maxElapsed
	}

	// 计算生成的令牌
	delta := tb.tokensFromDuration(elapsed)
	tokens := tb.tokens + delta
	if burst := float64(tb.burst); tokens > burst {
		tokens = burst
	}
	tokens--
	var waitDuration time.Duration
	if tokens < 0 {
		//说明取不到1个token, 那就计算取到1个token所需要的等待时间
		waitDuration = tb.durationFromTokens(-tokens)
	}
	ok := 1 <= tb.burst && waitDuration <= 0
	if ok {
		tb.lastUpdate = now
		tb.tokens = tokens
	}
	return ok
}

func main() {
	tokenBucket := NewTokenBucket(2.0, 1.0)
      success := 0
      reject := 0
	for {
		if tokenBucket.Allow() {
			fmt.Println(time.Now().Format("15:04:05"), ", 请求通过\n")
			success++
		}else {
		    reject++
		}
	}
	
	fmt.Println(success, "<======>", reject)
}
```
tips:
Sleep精准问题有兴趣可以看看[这篇文章](https://colobu.com/2023/12/07/more-precise-sleep/）

# 动态限流
通过实例的负载情况(采样窗口内的cpu使用率/load1)进行动态设置限流阈值，让服务保持高水位高效运行。

## 开源实现
[B站-BBR](https://github.com/go-kratos/aegis/tree/main/ratelimit/bbr)
[sentinel-go](https://github.com/alibaba/sentinel-golang/wiki/%E7%B3%BB%E7%BB%9F%E8%87%AA%E9%80%82%E5%BA%94%E6%B5%81%E6%8E%A7)
[Co-DEL](https://queue.acm.org/appendices/codel.html)
[B站-Codel](https://github.com/go-kratos/kratos/blob/4a93aa9b8d5dca550cc60a0c51c4726f83a2e6f8/pkg/container/queue/aqm/codel.go)

## 实现原理
1.B站-BBR: 使用滑动窗口统计成功数、响应时间；通过滑窗计算平均响应时间，根据利特尔法则计算QPS，当CPU使用率满足阈值时，动态设置限流阈值。
`QPS = (MaxPass(窗口内最大成功请求数) * MinRt(平均响应延时:ms) * BucketsPerSecond(1s的桶个数) /1000.0)`

2.Sentinel-go: 原理和B站类似，不过使用时load1(实时性较较差, 1分钟内的负载)
3.Co-DEL: 传统FIFO在海量请求场景下会出现大量请求“饿死”的情况, 而codel很好的规避了这个问题，codel会清理超时请求并且自动拒绝。[B站的实现](https://github.com/LyricTian/kratos/blob/bd2d576848f44f7bf4eb7c9420b36093fa4f8ef7/pkg/container/queue/aqm/codel.go)有2个容忍窗口, 容忍窗口期间请求还是会被放行, 超过窗口的才会被拒绝。

总结
- 精确限流, 动态调整阈值, 和服务负载正相关; 但是实现复杂，需要额外资源统计CPU使用率、QPS吞吐等; 限于接口调用，场景少
- 请求分优先级(用户纬度)，可按优先级丢弃、可以存在一定超卖。
- 拒绝请求也需要成本, cliet端需要截流(直接往上抛或者重试其他节点)


## 客户端节流
主要有以下两种场景  
1.用户客户端疯狂重试；客户端需要随机退避重试
2.下游过载, 返回"超出配额，拒绝请求"; 主调可以按概率拒绝请求; [算法](https://sre.google/sre-book/handling-overload/)
![自适应限流](/images/adaptive_throttling.png)


   
# 集群限流
为什么要用集群限流？在分布式场景下单机限流有2个缺陷：
- 当限流配额>节点数，单机限流就不能限制了；比如100个节点，50QPS，此时更适合集群限流
- 当流量不均时，单机限流会出现误限; 比如50个节点，100QPS，此时单节点2QPS，但如果流量不均，没达到阈值就拒绝请求了

## 限流模式
- 单次分配，即时消费即时结算
- 批次分配，先消费后结算
- 批次分配，预先分配消费


### <font color="green">单次分配</font> 即时消费即时结算 强一致
- 精准限流，会增加业务延迟
- 基于redis,sentinel实现
- 秒杀等对精准性要求较高的细粒度限流

### <font color="green">批次分配</font>  最终一致, 性能高，但准确性会降低
<font color="black">实现原理:</font>
<font color="orange">本地异步请求限流服务获取配额(quota)，本地采用静态限流算法</font>

一般都是客户端(LRU窗口)限流 + 客户端定期上报(ms级)配额到限流器 + 限流器响应客户端剩余配额 + 客户端重新计算限流额
- 预分配后消费; Youtube doorman; 本地限流，如果流量不均会有误限;适用服务级限流，读写分离的接口级限流
- 先消费后结算; 阿里AHAS; 客户端基于剩余整体配额进行扣除，不再进行均摊，解决误限问题，但可能会有超限; 服务/接口限流等允许一定误差的限流场景

先消费后结算:
1. client异步定期(30ms)同步限流server结算,
2. 请求一致性hash到对应的限流server上，
3. 限流server下发所有剩余配额

存在问题:
假设10个client,QPS限流1000， 每个节点QPS：100, 在30ms内消耗了100配额，实际放行请求: 10 * 1000个请求。
优化：
- 调整上报周期，降低周期+周期随机化(防止上报风暴)
- 每个窗口都单独上报, 性能有损, 对hash到同一节点的窗口合并批量上报
- 同步限流集群失败，降级为单机限流，总配额/客户端数(client)


[setinel](https://sentinelguard.io/zh-cn/docs/cluster-flow-control.html)集群限流(云上版本 AHAS Sentinel)

![集中式](/images/sentinel_limit_center.png)
![嵌入式](/images/sentinel_limit_embedded.png)

## 限流策略
- 多级限流(网关层、应用层、服务层、数据层)
- 动态阈值调整(负载高降低权重)
- 多级维度(ip,设备) + 业务侧规则(发评限制)

## 重要性-服务分级
在对服务进行限流时，可以引入更细的粒度-<strong>Criticality</strong>来按优先级丢弃流量,
CRITICAL_PLUS, 最高优先级，影响面:用户可见，严重；容量设置需充足
CRITICAL, 次优先级，影响面:用户可见，不如Plus严重；容量设置需充足
SHEDDABLE_PLUS, 异步任务，可定期重试
SHEDDABLE，最低优先级，接受不可用

<font color="red">Criticality 应该在服务调用链中逐级传递下去。</font>