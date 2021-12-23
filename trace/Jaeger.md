# 链路追踪

### jaeger 的安装

```shell
docker run --rm --name jager -p6831:6831/udp -p16686:16686 jaegertracing/all-in-one:latest
```

安装后访问：http://localhost:16686

![image-20211206220826865](/Users/guozhu/Library/Application Support/typora-user-images/image-20211206220826865.png)

这么启动 jaeger 的话是没有持久化的，一旦关闭或者退出 docoker 数据就会丢失。

### jaeger 架构

下图是Jaeger官方绘制的Jaeger架构图：

![img](http://images4.10qianwan.com/10qianwan/20190916/b_0_201909161757558027.jpg)

我们可以看虚线的部分，jaeger-clinet 就是我们的客户端的，当我们的应用想要发消息给 jaeger 的时候，是通过 udp 的方式 发送给 agent ，agent 发送给 jeager，我们不用关心 agent 是如何启动的，我们启动 client 就可以了。

- Jaeger Client - 为不同语言实现了符合 OpenTracing 标准的 SDK。应用程序通过 API 写入数据，client library 把 trace 信息按照应用程序指定的采样策略传递给 jaeger-agent。
- Agent - 它是一个监听在 UDP 端口上接收 span 数据的网络守护进程，它会将数据批量发送给 collector。它被设计成一个基础组件，部署到所有的宿主机上。Agent 将 client library 和 collector 解耦，为 client library 屏蔽了路由和发现 collector 的细节。
- Collector - 接收 jaeger-agent 发送来的数据，然后将数据写入后端存储。Collector 被设计成无状态的组件，因此您可以同时运行任意数量的 jaeger-collector。
- DB - 后端存储被设计成一个可插拔的组件，支持将数据写入 cassandra、elastic search。
- Query - 接收查询请求，然后从后端存储系统中检索 trace 并通过 UI 进行展示。Query 是无状态的，您可以启动多个实例，把它们部署在 nginx 这样的负载均衡器后面。



这里我们只需要理解如何使用就可以了，分布式追踪发展很快，种类繁多，但核心步骤一般有三个：**代码埋点，数据存储，查询展示**。

## Jaeger 的实现

- 提取
- 注入
- 提交

### 提取

**为什么要提取？**

提取的主要作用就是为了找到父亲。

**从哪里提取？**

进程内，不同进程之间各自约定

**提取什么？**

Traceid:spanid:parentid 是否提取

uber-trace-id:

github.com/uber/jaeger-client-go/propagation.go

### 注入

为什么要注入？

主要是为了让孩子能够找到爸爸

注入到哪里？

和提取相对

注入了什么？

## 提交 异步 report

- Span.finish



- 把 Span 放入队列
- 从队列中取出，shengchengthrift 翻入 spanBuffer
- Flush 到远程

 

低消耗

Jaeger-client 作用于应用层，提取，注入，生成 span，序列化成 thrift，发送到远程等，一系列操作这些都会带来性能上的消耗



选择合适的采集策略

- const，全量采集，采样率设置0,1 分别对应打开和关闭
- probabilistic ，概率采集，默认万份之一，0~1之间取值，
- rateLimiting ，采样器使用漏斗速率限制器来确保以一定的恒定速率对轨迹进行采样。
- remote ，采样器会向Jaeger代理咨询有关在当前服务中使用的适当采样策略。

应用透明

约定第一个参数为 ctx，把 parentSpan 放入 ctx

## jaeger 使用

### Go 发送单个 span

 ```go
 package main
 
 import (
 	"github.com/uber/jaeger-client-go"
 	jaegercfg "github.com/uber/jaeger-client-go/config"
 	"time"
 )
 
 func main() {
 	//并不是所有的请求都需要放到 jaeger 中，因为会有开销
 	cfg := jaegercfg.Configuration{Sampler: &jaegercfg.SamplerConfig{
 		Type:  jaeger.SamplerTypeConst, //设置const 1 就是进行采样
 		Param: 1,
 	}, Reporter: &jaegercfg.ReporterConfig{
 		LogSpans:           true,             //是否要打印日志
 		LocalAgentHostPort: "127.0.0.1:6831", //服务器
 	}, ServiceName: "jaeger_test"}
 
 	tracer, i, err := cfg.NewTracer(jaegercfg.Logger(jaeger.StdLogger)) //输出到屏幕
 
 	if err != nil {
 		panic(err)
 	}
 
 	defer i.Close()
 
 	span := tracer.StartSpan("go-grpc-web")
 	//模拟一下
 	time.Sleep(10 * time.Second)
 	defer span.Finish()
 }
 
 ```

```shell
2021/12/11 13:28:25 debug logging disabled
2021/12/11 13:28:25 Initializing logging reporter
2021/12/11 13:28:25 debug logging disabled
2021/12/11 13:28:35 Reporting span 5a528285d6a72f51:5a528285d6a72f51:0000000000000000:1

```

运行完成：

![image-20211211132904470](/Users/guozhu/Library/Application Support/typora-user-images/image-20211211132904470.png)

### Go 发送多层嵌套 Span



```go
package main

import (
	"github.com/opentracing/opentracing-go"
	"github.com/uber/jaeger-client-go"
	jaegercfg "github.com/uber/jaeger-client-go/config"
	"time"
)

func main() {
	//并不是所有的请求都需要放到 jaeger 中，因为会有开销
	cfg := jaegercfg.Configuration{Sampler: &jaegercfg.SamplerConfig{
		Type:  jaeger.SamplerTypeConst, //设置const 1 就是进行采样
		Param: 1,
	}, Reporter: &jaegercfg.ReporterConfig{
		LogSpans:           true,             //是否要打印日志
		LocalAgentHostPort: "127.0.0.1:6831", //服务器
	}, ServiceName: "jaeger_test"}

	tracer, i, err := cfg.NewTracer(jaegercfg.Logger(jaeger.StdLogger)) //输出到屏幕

	if err != nil {
		panic(err)
	}

	defer i.Close()

	parentSpan := tracer.StartSpan("main")

	span := tracer.StartSpan("Func A", opentracing.ChildOf(parentSpan.Context()))
	//模拟一下
	time.Sleep(1 * time.Second)
	span.Finish()

	// 创建多层嵌套 SPan //他们两个的 trace id 不一样
	span2 := tracer.StartSpan("Func B", opentracing.ChildOf(parentSpan.Context()))
	//模拟一下
	time.Sleep(2 * time.Second)
	span2.Finish()
	parentSpan.Finish()

}

```

```
2021/12/11 17:57:18 debug logging disabled
2021/12/11 17:57:18 Initializing logging reporter
2021/12/11 17:57:18 debug logging disabled
2021/12/11 17:57:19 Reporting span 7f5d710b856778fc:58ef3ecbf7817f40:7f5d710b856778fc:1
2021/12/11 17:57:21 Reporting span 7f5d710b856778fc:5cc5282c3184370e:7f5d710b856778fc:1
2021/12/11 17:57:21 Reporting span 7f5d710b856778fc:7f5d710b856778fc:0000000000000000:1
```



![image-20211211175807333](/Users/guozhu/Library/Application Support/typora-user-images/image-20211211175807333.png)

![image-20211211175744805](/Users/guozhu/Library/Application Support/typora-user-images/image-20211211175744805.png)

在 GRPC 中使用 Jaeger

```
go get github.com/grpc-ecosystem/grpc-opentracing/go/otgrpc
```



# 参考资料

[阿里云 链路追踪 Tracing Analysis 快速入门](https://help.aliyun.com/document_detail/196631.html)

[#29 opentracing jaeger 集成及源码分析 【 Go 夜读 】_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1ut411b7Qu?from=search&seid=11448079709854986189&spm_id_from=333.337.0.0)

[jaeger-动态配置采样率 - zzhi.wang - 博客园 (cnblogs.com)](https://www.cnblogs.com/zhangzhi19861216/p/15518690.html)
