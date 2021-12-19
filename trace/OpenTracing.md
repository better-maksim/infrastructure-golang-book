### OpenTraing 解析

## 快速认知 OpenTraing

- OpenTraing 通过提供平台无关，厂商无关的 API，使得开发人员能够方便的添加（或更换）追踪系统的实现
- 可以很自由的在不同的分布式追踪系统中切换
- 不负责具体实现





Tracing 是在上世纪 90 年代就已出现的技术，但真正让该领域流行起来的还是源于 Google 的一篇 Dapper 论文。分布式追踪系统发展很快，种类繁多，但无论哪种组件，其核心步骤一般有 3 步：代码埋点、数据存储和查询展示，如下图所示为链路追踪组件的组成。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9QZDFvbzNUQkt5ZnRsTWppYndNSjB5VzdxdHVBaGYyWENQdmp2MGdiSWlhVUJiSmh5SGpNTUhVRUhKMjd1eDRqRG1qZjhBS3pvS01uWDFuYzV5Z24zbnp3LzY0MA?x-oss-process=image/format,png)
目前流行的链路追踪组件有 Jaeger、Zipkin、Skywalking 和 Pinpoint 等。在数据采集过程中，**对用户代码的入侵和不同系统 API 的兼容性，导致切换链路追踪系统需要巨大的成本**。为了解决不同的分布式追踪系统 API 不兼容的问题，诞生了 OpenTracing 规范。

OpenTracing 于 2016 年 10 月加入 CNCF 基金会，是继 Kubernetes 和 Prometheus 之后，第三个加入 CNCF 的开源项目。它是一个中立的（厂商无关、平台无关）分布式追踪的 API 规范，提供统一接口，可方便开发者在自己的服务中集成一种或多种分布式追踪的实现。

OpenTracing 是一个轻量级的标准化层，它位于应用程序/类库和追踪或日志分析程序之间。OpenTracing 提供了 6 种语言的中立工具：Go、JavaScript、Java、Python、Objective-C 和 C ++。如下图所示为 OpenTracing 的架构。它支持 Zipkin、LightStep 和 Appdash 等追踪组件，并且可以轻松集成到开源的框架中，例如 gRPC、Flask、Django 和 Go-kit 等。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9QZDFvbzNUQkt5ZnRsTWppYndNSjB5VzdxdHVBaGYyWENrc3FvZGlhRlQyUjI0S0RzTlZDWUdxTWJZeWR6R3pBaERuSmlhWGljM3Q1S09HaFJOZFJCUVpOV2cvNjQw?x-oss-process=image/format,png)



```golang
package opentracing

import "time"

type Tracer interface {

	StartSpan(operationName string, opts ...StartSpanOption) Span

	Inject(sm SpanContext, format interface{}, carrier interface{}) error

	Extract(format interface{}, carrier interface{}) (SpanContext, error)
}


```

OpenTracing 已经定义好了接口，第三方类库只需要实现其接口即可，这样就可以做到使用少量代码就可以切换到其他平台上。

OpenTraing 主要组成

- Trace：一个 Trace 代表了一个事务或者流程在（分布式）系统中执行过程。
- Span：记录 trace 在执行过程中的信息，我们把其中某一块业务的调用信息保存起来，比如说什么时候发起的，什么时候结束的，运行时长是多少。
- 无限极分类：服务与服务之间使用无限极分类的方式，通过 Http 头或者请求地址传输到最底层，从而把整个调用链穿起来。



![image-20211211125216244](/Users/guozhu/Library/Application Support/typora-user-images/image-20211211125216244.png)



一个请求进来，就会有一个 span，span-s 代表一个服务端，span-c代表客户端，每一个请求都会有一个 trace_id，请求链路中 trace_id 都是一样的，parent_id 就是为了保证父子关系，客户端的 id 为 249714，到了 rpcb 中我们可以看到他的 parent_id 就是 249714.

这样就可以通过无限极分类，将整个请求链路整合起来，变成一个 tracer。



对应到代码中就是：Tracer、Span、SpanContext

- Tracer：现在我们有一个接口，以我们业务中进行举例子，我们在客户端 FCS 中，展示班级的课堂监控视频，中间要经过 FCS 用户模块 -> 班级模块 -> 业务系统获取真正的线下课课表 -> 回到 FCS 获取课堂视频 -> 展示，这里面每一步都可能想要去监控，在 opentracing 中就需要把这些模块的访问时间串联起来，最后成为一条路径 tracer，所以想要监听某一个接口，一定要先穿件一个 tracer。

- Span：

  - An operation name，操作名称
  - A start timestamp，起始时间
  - A finish timestamp，结束时间
  - **Span Tag**，一组键值对构成的Span标签集合。键值对中，键必须为string，值可以是字符串，布尔，或者数字类型。
  - **Span Log**，一组span的日志集合。 每次log操作包含一个键值对，以及一个时间戳。 键值对中，键必须为string，值可以是任意类型。 但是需要注意，不是所有的支持OpenTracing的Tracer,都需要支持所有的值类型。
  - **SpanContext**，Span上下文对象 (下面会详细说明)
  - **References**(Span间关系)，相关的零个或者多个Span（**Span**间通过**SpanContext**建立这种关系）

  每一个**SpanContext**包含以下状态：

  - 任何一个OpenTracing的实现，都需要将当前调用链的状态（例如：trace和span的id），依赖一个独特的Span去跨进程边界传输
  - **Baggage Items**，Trace的随行数据，是一个键值对集合，它存在于trace中，也需要跨进程边界传输

- SpanContext：Span 之间是通过 SPanContext 建立关系的。

  - 任何一个OpenTracing的实现，都需要将当前调用链的状态（例如：trace和span的id），依赖一个独特的Span去跨进程边界传输
  - **Baggage Items**，Trace的随行数据，是一个键值对集合，它存在于trace中，也需要跨进程边界传输

一个Span可以与一个或者多个**SpanContexts**存在因果关系。OpenTracing目前定义了两种关系：`ChildOf`（父子） 和 `FollowsFrom`（跟随，主要在异步的调用当中，比如发起了一个 MQ 的消息）。**这两种关系明确的给出了两个父子关系的Span的因果模型。** 将来，OpenTracing可能提供非因果关系的span间关系。（例如：span被批量处理，span被阻塞在同一个队列中，等等）。



