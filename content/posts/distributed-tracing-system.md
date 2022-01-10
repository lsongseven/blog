---
title: "Distributed Tracing System"
date: 2021-09-03T17:47:55+08:00
draft: false
---

# Distributed Tracing System

## Preface

- Trace有什么用

从最早期的巨石单体（Monolithic）到分布式（Distributed），再到微服务（Microservices）、容器化（Containerization）、容器编排（Container Orchestration），最后到服务网格（[Service Mesh](https://www.servicemesher.com/istio-handbook/GLOSSARY.html#service-mesh)）、无服务器（Serverless）,复杂度不断提升，问题定位越来越复杂

![Generated](/image/jaeger/topology.png)

- Trace可以快速定位问题 + 理清网络拓扑

## OpenTracing

OpenTracing是业务、框架等代码与分布式追踪实现厂商（如jaeger、zipkin、skywalking）之间的一层标准层。OpenTracing通过提供平台无关、厂商无关的API，使得开发人员能够方便的添加（或更换）追踪系统的实现。

```text
   +-------------+  +---------+  +----------+  +------------+
   | Application |  | Library |  |   OSS    |  |  RPC/IPC   |
   |    Code     |  |  Code   |  | Services |  | Frameworks |
   +-------------+  +---------+  +----------+  +------------+
          |              |             |             |
          |              |             |             |
          v              v             v             v
     +-----------------------------------------------------+
     | · · · · · · · · · · OpenTracing · · · · · · · · · · |
     +-----------------------------------------------------+
       |               |                |               |
       |               |                |               |
       v               v                v               v
 +-----------+  +-------------+  +-------------+  +-----------+
 |  Tracing  |  |   Logging   |  |   Metrics   |  |  Tracing  |
 | System A  |  | Framework B |  | Framework C |  | System D  |
 +-----------+  +-------------+  +-------------+  +-----------+
 ```

- Terminology: Trace, Span

一个trace代表一个潜在的，分布式的，存在并行数据或并行执行轨迹（潜在的分布式、并行）的系统。Trace由Span组成，可以看作是一个由Span组成的有向无环图(Directed Acyclic Graph)， Span之间的边被称为Refrence。

一个Span代表系统中具有开始时间和执行时长的逻辑运行单元。Span之间通过嵌套或者顺序排列建立逻辑因果关系。

- Span之间的关系
- ChildOf : 一个span可能是一个父级span的孩子，即"ChildOf"关系。在"ChildOf"引用关系下，父级span某种程度上取决于子span。
- FollowsFrom: 一些父级节点不以任何方式依然他们子节点的执行结果，这种情况下，我们说这些子span和父span之间是"FollowsFrom"的因果关系。

```text
Causal relationships between Spans in a single Trace


        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `ChildOf` Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G `FollowsFrom` Span F)
```

```text
Temporal relationships between Spans in a single Trace


––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
```

- OpenTracing中Span的定义包括
  - 操作的名称
  - 开始时间戳
  - 结束时间戳
  - Span Tags
  - Span Logs
  - SpanContext
    - 每个span必须提供方法访问SpanContext，包含<trace\_id, span\_id, sampled>元组
    - 跨越进程依赖于OpenTracing的Inject/Extract
    - 跨越进程边界，封装Baggage Items
  - Baggage Item
    - Baggage是存储在SpanContext中的一个键值对(SpanContext)集合。它会在一条追踪链路上的所有Span内*全局*传输，包含这些Span对应的SpanContexts
  - 与此Span相关联的其他Span的引用

传递上下文的一个案例

```go
import (
    "net/http"

    opentracing "github.com/opentracing/opentracing-go"
    "github.com/opentracing/opentracing-go/ext"
)

...

tracer := opentracing.GlobalTracer()

clientSpan := tracer.StartSpan("client")
defer clientSpan.Finish()

url := "http://localhost:8082/publish"
req, _ := http.NewRequest("GET", url, nil)

// Set some tags on the clientSpan to annotate that it's the client span. The additional HTTP tags are useful for debugging purposes.
ext.SpanKindRPCClient.Set(clientSpan)
ext.HTTPUrl.Set(clientSpan, url)
ext.HTTPMethod.Set(clientSpan, "GET")

// Inject the client span context into the headers
tracer.Inject(clientSpan.Context(), opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(req.Header))
resp, _ := http.DefaultClient.Do(req)
```

```go
import (
    "log"
    "net/http"

    opentracing "github.com/opentracing/opentracing-go"
    "github.com/opentracing/opentracing-go/ext"
)

func main() {

    // Tracer initialization, etc.

    ...

    http.HandleFunc("/publish", func(w http.ResponseWriter, r *http.Request) {
        // Extract the context from the headers
        spanCtx, _ := tracer.Extract(opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(r.Header))
        serverSpan := tracer.StartSpan("server", ext.RPCServerOption(spanCtx))
        defer serverSpan.Finish()
    })

    log.Fatal(http.ListenAndServe(":8082", nil))
}
```

## Jaeger vs. Skywalking

### Architecture

![Generated](/image/jaeger/jaeger-arch.png)

![Generated](/image/jaeger/jaeger-arch-with-kafka.png)

- Client: 实现了OpenTracing API的客户端，用于增强业务代码。
- Agent: 监听UDP发来的Span数据，可部署为sidecar或者DaemonSet，屏蔽client与collector之间的路由和服务发现细节
- Collector 将采集到的 trace 数据写入中间缓冲区 Kafka 中，Ingester 读取 Kafka 中的数据并持久化到 DB 里。同时，Flink jobs 持续读取 Kafka 中的数据并将计算出的拓扑关系写入 DB 中。
- Sampling: Constant, Probabilistic, Rate Limiting, Remote

![Generated](/image/jaeger/skywalking-arch.png)

- Observability Analysis Platform (OAP)

[\[*\] why doesn't skywalking involve mq in its architecture](https://skywalking.apache.org/docs/main/latest/en/faq/why_mq_not_involved/#why-doesnt-skywalking-involve-mq-in-its-architecture)

### Enhancement

- skywalking
- 依靠字节码增强技术，可依赖一个agent类来实现特定的增强。这个agent类需要实现void premain(String agentArgs, Instrumentation instrumentation)这样一个方法，具体的增强细节涵盖在其中。（Java中广泛使用的技术，如rpc框架dubbo、诊断工具arthas等）

```java
// java -javaagent:xxx.jar -jar yy.jar

/**
 * The main entrance of sky-walking agent, based on javaagent mechanism.
 */
public class SkyWalkingAgent {
    /**
     * Main entrance. Use byte-buddy transform to enhance all classes, which define in plugins.
     */
    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
       // instrument
       ...
       
    }
    
}
```

- SDK

## Cloud Native

## Refrence

- <https://opentracing.io/docs/>

- <https://wu-sheng.gitbooks.io/opentracing-io/content/>

- <https://eng.uber.com/distributed-tracing/>

- <https://medium.com/opentracing/towards-turnkey-distributed-tracing-5f4297d1736>
