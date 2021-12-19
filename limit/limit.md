## 限流，熔断和降级

### 服务雪崩

#### 什么是服务雪崩

服务雪崩效应是一种”服务提供者的不可用“，导致”服务调用这不可用“，并将不可用逐渐放大的现象。

#### 形成原因

雪崩的雪崩过程可以分为三个阶段：

1. 服务提供不可用
2. 重试增加大量请求
3. 服务调用这不可用

福泵的每个阶段都可能由不同的原因造成：

- 服务不可用
  - 硬件故障
  - 程序 Bug
  - 缓存击穿
  - 用户大量请求
- 重试增加大量请求
  - 用户重试
  - 代码逻辑重复
- 服务调用这不可用，同步等待造成的资源耗尽

#### 应对策略

- 应用扩容：增加机器数量，升级硬件规格
- 流量控制：限流，关闭重试
- 缓存：缓存预加载
- 服务降级
  - 服务接口拒绝服务
  - 页面拒绝服务
  - 延迟持久化
  - 随机拒绝服务
- 服务熔断

一开始你的服务能考虑抗住高并发吗？

成本的增加：开发成本和硬件成本

如果平时流量和高峰流量差异很大该怎么办？

我们现在考虑到我们今年可能出现一次高并发 1W，我们全年的服务都部署成可以抗住 1w 并发，但是平时可能只有 500，在这情况下，可能只考虑 1000 的并发，但是某个时候出现流量的猛增，到了 2k 了，系统极容易就搞崩溃。

实际上在最开始的时候就需要考虑限流。流量现在是 2k，我们只能支持 1k，多出来的流量，我们可以**拒绝，排队**。

**用户体验不好，体验降级**

熔断： 比如A 服务，访问 B 服务，这个时候 B 服务很慢，B 服务压力过大，导致了出现了不少请求错误的情况，调用方很容出现问题：每次调用都超时 2k，结果这个时候数据库出现问题，超时重试-2k 的流量突然变成了 3k，折让原本就满负荷的 B 服务，雪上加霜。

如果这个时候服务 A 发现

- 发现大部分请求慢了 50%
- 50%都错误了
- 1 秒出现了 20 个错误。

现实世界的熔断

- 保险丝，空开
- 股市熔断

# sentinel

快速开始：[quick-start (sentinelguard.io)](https://sentinelguard.io/zh-cn/docs/quick-start.html)

## Sentinel 限流

```go
go get "github.com/alibaba/sentinel-golang"
```

```golang
package main

import (
	"fmt"
	sentinel "github.com/alibaba/sentinel-golang/api"
	"github.com/alibaba/sentinel-golang/core/base"
	"github.com/alibaba/sentinel-golang/core/config"
	"github.com/alibaba/sentinel-golang/core/flow"
	"github.com/alibaba/sentinel-golang/logging"
	"log"
)

var resName string = "test"

func main() {
	conf := config.NewDefaultConfig()
	// for testing, logging output to console
	conf.Sentinel.Log.Logger = logging.NewConsoleLogger()
	err := sentinel.InitWithConfig(conf)
	if err != nil {
		log.Fatal(err)
	}

	//配置限流规则
	_, err = flow.LoadRules([]*flow.Rule{
		{
			Resource:               resName,
			TokenCalculateStrategy: flow.Direct,
			ControlBehavior:        flow.Reject, //直接拒绝
			Threshold:              10,          //10 个
			StatIntervalInMs:       1000,        //1 秒为统计周期
		},
	})
	if err != nil {
		log.Fatalf("Unexpected error: %+v", err)
		return
	}

	for i := 0; i < 15; i++ {

		e, b := sentinel.Entry(resName, sentinel.WithTrafficType(base.Inbound))

		if b != nil {
			fmt.Println("被限流了")
		} else {
			fmt.Println("检查通过")
			e.Exit()
		}
	}
}

```

## 预热冷启动

WarmUp方式，即预热/冷启动方式。当系统长期处于低水位的情况下，当流量突然增加时，直接把系统拉升到高水位可能瞬间把系统压垮。通过"冷启动"，让通过的流量缓慢增加，在一定时间内逐渐增加到阈值上限，给冷系统一个预热的时间，避免冷系统被压垮。这块设计和Java类似，可以参考[限流 冷启动](https://github.com/alibaba/Sentinel/wiki/限流---冷启动) 通常冷启动的过程系统允许通过的 QPS 曲线如下图所示：



![img](/Users/guozhu/Documents/micro-service/_media/68292392-b5b0aa00-00c6-11ea-86e1-ecacff8aab51.png)



```golang
package main

import (
	"fmt"
	sentinel "github.com/alibaba/sentinel-golang/api"
	"github.com/alibaba/sentinel-golang/core/base"
	"github.com/alibaba/sentinel-golang/core/config"
	"github.com/alibaba/sentinel-golang/core/flow"
	"github.com/alibaba/sentinel-golang/logging"
	"log"
	"math/rand"
	"sync"
	"time"
)

var resName string = "test"

type Total struct {
	value int
	mux   sync.Mutex
}

func (receiver *Total) incr() {
	receiver.mux.Lock()
	receiver.value++
	receiver.mux.Unlock()
}

func (receiver *Total) get() int {
	receiver.mux.Lock()
	defer receiver.mux.Unlock()
	return receiver.value
}

func main() {
	conf := config.NewDefaultConfig()
	// for testing, logging output to console
	conf.Sentinel.Log.Logger = logging.NewConsoleLogger()
	err := sentinel.InitWithConfig(conf)
	if err != nil {
		log.Fatal(err)
	}

	//配置限流规则
	_, err = flow.LoadRules([]*flow.Rule{
		{
			Resource:               resName,
			TokenCalculateStrategy: flow.WarmUp, // 冷启动
			ControlBehavior:        flow.Reject, //直接拒绝
			Threshold:              1000,        //中间要经过30 秒才可以达到 1000
			WarmUpPeriodSec:        30,          //30秒，
		},
	})
	if err != nil {
		log.Fatalf("Unexpected error: %+v", err)
		return
	}

	var globalTotal Total
	var passTotal Total
	var blockTotal Total
	ch := make(chan struct{})
	//会在每一秒统计一次，这一秒能通过多少，总共有多少，block 了多少
	for i := 0; i < 20; i++ {
		go func() {
			for {

				globalTotal.incr()
				e, b := sentinel.Entry(resName, sentinel.WithTrafficType(base.Inbound))
				time.Sleep(time.Duration(rand.Uint64()%10) * time.Millisecond)
				if b != nil {
					blockTotal.incr()
					time.Sleep(time.Duration(rand.Uint64()%10) * time.Millisecond)
				} else {
					passTotal.incr()
					time.Sleep(time.Duration(rand.Uint64()%10) * time.Millisecond)
					e.Exit()
				}
			}
		}()
	}

	go func() {
		var oldTotal int //过去 1s 总共有多少
		var oldPass int  //过去的 1s 总共有 pass 多个少
		var oldBlock int //过去 1s 总共 block 多少个

		for {

			oneSecondTotal := globalTotal.get() - oldTotal
			oldTotal = globalTotal.get()

			oneSecodBlock := passTotal.get() - oldPass
			oldPass = passTotal.get()

			oneSecondBlock := blockTotal.get() - oldBlock
			oldBlock = blockTotal.get()
			time.Sleep(time.Second)
			fmt.Printf("total:%d, pass:%d, block:%d \n", oneSecondTotal, oneSecodBlock, oneSecondBlock)
		}
	}()
	<-ch
}

```

## Throttling

Throttling：表示匀速排队的统计策略。它的中心思想是，以固定的间隔时间让请求通过。当请求到来的时候，如果当前请求距离上个通过的请求通过的时间间隔不小于预设值，则让当前请求通过；否则，计算当前请求的预期通过时间，如果该请求的预期通过时间小于规则预设的 timeout 时间，则该请求会等待直到预设时间到来通过（排队等待处理）；若预期的通过时间超出最大排队时长，则直接拒接这个请求。

字段`ControlBehavior`表示表示流量控制器的控制行为，目前Sentinel支持两种：

1. Reject：表示如果当前统计周期内，统计结构统计的请求数超过了阈值，就直接拒绝。
2. Throttling：表示匀速排队的统计策略。它的中心思想是，以固定的间隔时间让请求通过。当请求到来的时候，如果当前请求距离上个通过的请求通过的时间间隔不小于预设值，则让当前请求通过；否则，计算当前请求的预期通过时间，如果该请求的预期通过时间小于规则预设的 timeout 时间，则该请求会等待直到预设时间到来通过（排队等待处理）；若预期的通过时间超出最大排队时长，则直接拒接这个请求。

匀速排队方式会严格控制请求通过的间隔时间，也即是让请求以均匀的速度通过，对应的是漏桶算法。该方式的作用如下图所示： ![img](/Users/guozhu/Documents/micro-service/_media/68292442-d4af3c00-00c6-11ea-8251-d0977366d9b4.png)

这种方式主要用于处理间隔性突发的流量，例如消息队列。想象一下这样的场景，在某一秒有大量的请求到来，而接下来的几秒则处于空闲状态，我们希望系统能够在接下来的空闲期间逐渐处理这些请求，而不是在第一秒直接拒绝多余的请求。

以下规则代表每 100ms 最多通过一个请求，多余的请求将会排队等待通过，若排队时队列长度大于 500ms 则直接拒绝：

```go
{
	Resource:          "some-test",
        TokenCalculateStrategy: flow.Direct,
	ControlBehavior:   flow.Throttling, // 流控效果为匀速排队
        Threshold:             10, // 请求的间隔控制在 1000/10=100 ms
	MaxQueueingTimeMs: 500, // 最长排队等待时间
}
```

上面Threshold是10，Sentinel默认使用1s作为控制周期，表示1秒内10个请求匀速排队，所以排队时间就是1000ms/10 = 100ms；

`MaxQueueingTimeMs` 设为 0 时代表不允许排队，只控制请求时间间隔，多余的请求将会直接拒绝。



## 接口熔断

限流只是限制了并发流量，熔断是下方服务不可用的时候当到达我设置的比例，无脑的全部给它拒绝。

## 熔断器模型

Sentinel 熔断降级基于熔断器模式 (circuit breaker pattern) 实现。熔断器内部维护了一个熔断器的状态机，状态机的转换关系如下图所示：

![circuit-breaker](/Users/guozhu/Documents/micro-service/_media/82635455-ca075f00-9c32-11ea-9e99-d67518923e0d.png)

熔断器有三种状态：

1. Closed 状态：也是初始状态，该状态下，熔断器会保持闭合，对资源的访问直接通过熔断器的检查。
2. Open 状态：断开状态，熔断器处于开启状态，对资源的访问会被切断。
3. Half-Open 状态：半开状态，该状态下除了探测流量，其余对资源的访问也会被切断。探测流量指熔断器处于半开状态时，会周期性的允许一定数目的探测请求通过，如果探测请求能够正常的返回，代表探测成功，此时熔断器会重置状态到 Closed 状态，结束熔断；如果探测失败，则回滚到 Open 状态。

这三种状态之间的转换关系这里做一个更加清晰的解释：

1. 初始状态下，熔断器处于 Closed 状态。如果基于熔断器的统计数据表明当前资源触发了设定的阈值，那么熔断器会切换状态到 Open 状态；
2. Open 状态即代表熔断状态，所有请求都会直接被拒绝。熔断器规则中会配置一个熔断超时重试的时间，经过熔断超时重试时长后熔断器会将状态置为 Half-Open 状态，从而进行探测机制；
3. 处于 Half-Open 状态的熔断器会周期性去做探测。

Sentinel 提供了监听器去监听熔断器状态机的三种状态的转换，方便用户去自定义扩展：

```go
// StateChangeListener listens on the circuit breaker state change event.
type StateChangeListener interface {
        // 熔断器切换到 Closed 状态时候会调用改函数, prev代表切换前的状态，rule表示当前熔断器对应的规则
	OnTransformToClosed(prev State, rule Rule)
        // 熔断器切换到 Open 状态时候会调用改函数, prev代表切换前的状态，rule表示当前熔断器对应的规则， snapshot表示触发熔断的值
	OnTransformToOpen(prev State, rule Rule, snapshot interface{})
        // 熔断器切换到 HalfOpen 状态时候会调用改函数, prev代表切换前的状态，rule表示当前熔断器对应的规则
	OnTransformToHalfOpen(prev State, rule Rule)
}
```

通过上面的三个 hook 函数，用户可以很容易拿到熔断器每次状态切换的事件，以及熔断器对应的 Rule。

> Note 1: 这里需要注意的是，监听器 hook 里面携带的规则是基于 copy 的，也就是用户在监听器里面更改 Rule 不会影响到熔断器。此外这里基于拷贝是有一定性能开销的，用户要尽可能减少无效的监听器注册。
>
> Note 2: 熔断器监听器的注册和清除是非线程安全的，用户必须要在服务启动时配置 Sentinel 时候就注册对应的监听器，应用运行中禁止更改熔断器状态机的监听器。

## 熔断器的设计

我们衡量下游服务质量时候，场景的指标就是RT(response time)、异常数量以及异常比例等等。Sentinel 的熔断器支持三种熔断策略：慢调用比例熔断、异常比例熔断以及异常数量熔断。

用户通过设置熔断规则(Rule)来给资源添加熔断器。Sentinel会将每一个熔断规则转换成对应的熔断器，熔断器对用户是不可见的。Sentinel 的每个熔断器都会有自己独立的统计结构。

熔断器的整体检查逻辑可以用几点来精简概括：

1. 基于熔断器的状态机来判断对资源是否可以访问；
2. 对不可访问的资源会有探测机制，探测机制保障了对资源访问的弹性恢复；
3. 熔断器会在对资源访问的完成态去更新统计，然后基于熔断规则更新熔断器状态机。

## 熔断策略

Sentinel 熔断器的三种熔断策略都支持**静默期 **(规则中通过MinRequestAmount字段表示)。**静默期是指一个最小的静默请求数，在一个统计周期内，如果对资源的请求数小于设置的静默数，那么熔断器将不会基于其统计值去更改熔断器的状态。**静默期的设计理由也很简单，举个例子，假设在一个统计周期刚刚开始时候，第 1 个请求碰巧是个慢请求，这个时候这个时候的慢调用比例就会是 100%，很明显是不合理，所以存在一定的巧合性。所以静默期提高了熔断器的精准性以及降低误判可能性。

Sentinel 支持以下几种熔断策略：

- **慢调用比例策略 (SlowRequestRatio)**：Sentinel 的熔断器不在静默期，并且慢调用的比例大于设置的阈值，则接下来的熔断周期内对资源的访问会自动地被熔断。该策略下需要设置允许的调用 RT 临界值（即最大的响应时间），对该资源访问的响应时间大于该阈值则统计为慢调用。
- **错误比例策略 (ErrorRatio)**：Sentinel 的熔断器不在静默期，并且在统计周期内资源请求访问异常的比例大于设定的阈值，则接下来的熔断周期内对资源的访问会自动地被熔断。
- **错误计数策略 (ErrorCount)**：Sentinel 的熔断器不在静默期，并且在统计周期内资源请求访问异常数大于设定的阈值，则接下来的熔断周期内对资源的访问会自动地被熔断。

注意：这里的错误比例熔断和错误计数熔断指的业务返回错误的比例或则计数。也就是说，如果规则指定熔断器策略采用错误比例或则错误计数，那么为了统计错误比例或错误计数，需要调用API： `api.TraceError(entry, err)` 埋点每个请求的业务异常。

## 熔断降级规则定义

熔断规则的定义如下： refer: https://github.com/alibaba/sentinel-golang/blob/7ddba92fdf319c410df01e712ac5e89fe46d9c23/core/circuitbreaker/rule.go#L36

```go
// Rule encompasses the fields of circuit breaking rule.
type Rule struct {
	// unique id
	Id string `json:"id,omitempty"`
	// resource name
	Resource string   `json:"resource"`
	Strategy Strategy `json:"strategy"`
	// RetryTimeoutMs represents recovery timeout (in milliseconds) before the circuit breaker opens.
	// During the open period, no requests are permitted until the timeout has elapsed.
	// After that, the circuit breaker will transform to half-open state for trying a few "trial" requests.
	RetryTimeoutMs uint32 `json:"retryTimeoutMs"`
	// MinRequestAmount represents the minimum number of requests (in an active statistic time span)
	// that can trigger circuit breaking.
	MinRequestAmount uint64 `json:"minRequestAmount"`
	// StatIntervalMs represents statistic time interval of the internal circuit breaker (in ms).
	StatIntervalMs uint32 `json:"statIntervalMs"`
	// MaxAllowedRtMs indicates that any invocation whose response time exceeds this value (in ms)
	// will be recorded as a slow request.
	// MaxAllowedRtMs only takes effect for SlowRequestRatio strategy
	MaxAllowedRtMs uint64 `json:"maxAllowedRtMs"`
	// Threshold represents the threshold of circuit breaker.
	// for SlowRequestRatio, it represents the max slow request ratio
	// for ErrorRatio, it represents the max error request ratio
	// for ErrorCount, it represents the max error request count
	Threshold float64 `json:"threshold"`
}
```

- `Id`: 表示 Sentinel 规则的全局唯一ID，可选项。
- `Resource`: 熔断器规则生效的埋点资源的名称；
- `Strategy:` 熔断策略，目前支持`SlowRequestRatio`,`ErrorRatio`,`ErrorCount`,三种；
  - 选择以**慢调用比例** (SlowRequestRatio) 作为阈值，需要设置允许的**最大响应时间**（MaxAllowedRtMs），请求的响应时间大于该值则统计为慢调用。通过 `Threshold` 字段设置触发熔断的慢调用比例，取值范围为 [0.0, 1.0]。规则配置后，在单位统计时长内请求数目大于设置的最小请求数目，并且慢调用的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态，若接下来的一个请求响应时间小于设置的最大 RT 则结束熔断，若大于设置的最大 RT 则会再次被熔断。
  - 选择以**错误比例** (ErrorRatio) 作为阈值，需要设置触发熔断的异常比例（`Threshold`），取值范围为 [0.0, 1.0]。规则配置后，在单位统计时长内请求数目大于设置的最小请求数目，并且异常的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态，若接下来的一个请求没有错误则结束熔断，否则会再次被熔断。代码中可以通过 `api.TraceError(entry, err)` 函数来记录 error。
- `RetryTimeoutMs`: 即熔断触发后持续的时间（单位为 ms）。资源进入熔断状态后，在配置的熔断时长内，请求都会快速失败。熔断结束后进入探测恢复模式（HALF-OPEN）。
- `MinRequestAmount`: 静默数量，如果当前统计周期内对资源的访问数量小于静默数量，那么熔断器就处于静默期。换言之，也就是触发熔断的最小请求数目，若当前统计周期内的请求数小于此值，即使达到熔断条件规则也不会触发。
- `StatIntervalMs`: 统计的时间窗口长度（单位为 ms）。
- `MaxAllowedRtMs`: 仅对`慢调用熔断策略`生效，MaxAllowedRtMs 是判断请求是否是慢调用的临界值，也就是如果请求的response time小于或等于MaxAllowedRtMs，那么就不是慢调用；如果response time大于MaxAllowedRtMs，那么当前请求就属于慢调用。
- `Threshold`: 对于`慢调用熔断策略`, Threshold表示是慢调用比例的阈值(小数表示，比如0.1表示10%)，也就是如果当前资源的慢调用比例如果高于Threshold，那么熔断器就会断开；否则保持闭合状态。 对于`错误比例策略`，Threshold表示的是错误比例的阈值(小数表示，比如0.1表示10%)。对于`错误数策略`，Threshold是错误计数的阈值。

一些补充说明：

- Resource、Strategy、RetryTimeoutMs、MinRequestAmount、StatIntervalMs、Threshold 每个规则都必设的字段，MaxAllowedRtMs是慢调用比例熔断规则必设的字段。
- MaxAllowedRtMs 字段仅仅对**慢调用比例** (SlowRequestRatio) 策略有效，对其余策略均属于无效字段。
- StatIntervalMs 表示熔断器的统计周期，单位是毫秒，这个值我们不建议设置的太大或则太小，一般情况下设置10秒左右都OK，当然也要根据实际情况来适当调整。
- RetryTimeoutMs 的设置需要根据实际情况设置探测周期，一般情况下设置10秒左右都OK，当然也要根据实际情况来适当调整。

这里分别给出三种熔断策略规则配置的一个Sample(不可作为线上配置参考):

```go
// 慢调用比例规则
rule1 := &Rule{
        Resource:         "abc",
        Strategy:         SlowRequestRatio,
	RetryTimeoutMs:   5000,
	MinRequestAmount: 10,
	StatIntervalMs:   10000,
	MaxAllowedRtMs:   20,
	Threshold:        0.1,
},
// 错误比例规则
rule1 := &Rule{
        Resource:         "abc",
        Strategy:         ErrorRatio,
	RetryTimeoutMs:   5000,
	MinRequestAmount: 10,
	StatIntervalMs:   10000,
	Threshold:        0.1,
},
// 错误计数规则
rule1 := &Rule{
        Resource:         "abc",
        Strategy:         ErrorCount,
	RetryTimeoutMs:   5000,
	MinRequestAmount: 10,
	StatIntervalMs:   10000,
	Threshold:        100,
},
```

## 最佳场景实践

熔断器一般用于应用对外部资源访问时的保护措施。这里简单描述一些场景：

- 分布式系统中降级：假设存在应用A需要调用应用B的接口(特别是一些对接外部公司或者业务的接口时候)，那么一般用于A调用B的接口时的防护；
- 数据库慢调用的防护： 假设应用需要读/写数据库，但是该读写SQL存在潜在慢SQL的可能性，那么可以对该读写接口做防护，当接口不稳定时候(存在慢SQL)，那么基于熔断器做降级。
- 也可以是应用中任意弱依赖接口做降级防护（即自动降级后不影响业务核心链路）。

## 基于错误的熔断

```golang
// Copyright 1999-2020 Alibaba Group Holding Ltd.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

package main

import (
	"errors"
	"fmt"
	"log"
	"math/rand"
	"time"

	sentinel "github.com/alibaba/sentinel-golang/api"
	"github.com/alibaba/sentinel-golang/core/circuitbreaker"
	"github.com/alibaba/sentinel-golang/core/config"
	"github.com/alibaba/sentinel-golang/logging"
	"github.com/alibaba/sentinel-golang/util"
)

type stateChangeTestListener struct {
}

//OnTransformToClosed 内部装填转移的过程
func (s *stateChangeTestListener) OnTransformToClosed(prev circuitbreaker.State, rule circuitbreaker.Rule) {
	fmt.Printf("rule.steategy: %+v, From %s to Closed, time: %d\n", rule.Strategy, prev.String(), util.CurrentTimeMillis())
}

func (s *stateChangeTestListener) OnTransformToOpen(prev circuitbreaker.State, rule circuitbreaker.Rule, snapshot interface{}) {
	fmt.Printf("rule.steategy: %+v, From %s to Open, snapshot: %d, time: %d\n", rule.Strategy, prev.String(), snapshot, util.CurrentTimeMillis())
}

func (s *stateChangeTestListener) OnTransformToHalfOpen(prev circuitbreaker.State, rule circuitbreaker.Rule) {
	fmt.Printf("rule.steategy: %+v, From %s to Half-Open, time: %d\n", rule.Strategy, prev.String(), util.CurrentTimeMillis())
}

func main() {
	conf := config.NewDefaultConfig()

	// for testing, logging output to console
	conf.Sentinel.Log.Logger = logging.NewConsoleLogger()

	err := sentinel.InitWithConfig(conf)

	if err != nil {
		log.Fatal(err)
	}

	ch := make(chan struct{})
	// Register a state change listener so that we could observer the state change of the internal circuit breaker.
	//如果装填发生改变了，可以加上一些自己的逻辑，最上面的代码就是为了输出中间的逻辑转换关系
	circuitbreaker.RegisterStateChangeListeners(&stateChangeTestListener{})

	_, err = circuitbreaker.LoadRules([]*circuitbreaker.Rule{
		// Statistic time span=5s, recoveryTimeout=3s, maxErrorCount=50
		{
			Resource:                     "abc",
			Strategy:                     circuitbreaker.ErrorCount,
			RetryTimeoutMs:               3000, //三秒之后尝试恢复
			MinRequestAmount:             10,   //静默数，10 个以内全部通过
			StatIntervalMs:               5000, //5秒钟统计一次
			StatSlidingWindowBucketCount: 10,
			Threshold:                    50, //50 个错误
		},
	})
	if err != nil {
		log.Fatal(err)
	}
	logging.Info("[CircuitBreaker ErrorCount] Sentinel Go circuit breaking demo is running. You may see the pass/block metric in the metric log.")
	go func() {
		for {

			e, b := sentinel.Entry("abc")
			if b != nil {

				fmt.Println("G1 被拒绝")
				// g1 blocked
				time.Sleep(time.Duration(rand.Uint64()%20) * time.Millisecond)
			} else {
				fmt.Println("G1 通过")
				if rand.Uint64()%20 > 9 {
					//随机抛出错误，造成熔断安
					sentinel.TraceError(e, errors.New("biz error"))
				}
				// g1 passed 10~90 毫秒以内
				time.Sleep(time.Duration(rand.Uint64()%80+10) * time.Millisecond)
				e.Exit()
			}
		}
	}()
	go func() {
		for {
			e, b := sentinel.Entry("abc")
			if b != nil {
				fmt.Println("G2 被拒绝")
				// g2 blocked
				time.Sleep(time.Duration(rand.Uint64()%20) * time.Millisecond)
			} else {
				fmt.Println("G2 通过")
				// g2 passed
				time.Sleep(time.Duration(rand.Uint64()%80) * time.Millisecond)
				e.Exit()
			}
		}
	}()
	<-ch
}

```

