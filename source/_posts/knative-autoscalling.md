---
layout: post
title:  "【Knative系列】理解 Knative 扩缩容系统的设计原理"
date:   2020-09-30 18:16:18 +0800
categories: knative
tags:  ["knative","autoscalling"]
author: zhaojizhuang
---

> 本文主要讲解 Knative 扩缩容系统的设计原理及实现细节，主要从以下三个方面进行讲解：
> * Knative Serving 扩缩容系统的组件；
> * 涉及的API；
> * 扩缩容和冷启动时的 控制流和数据流的一些细节；

## 组件

Knative Serving 是 Knative 系统的核心，而理解 Knative Serving 系统内的组件能更容易了理解 Knative Serving 系统的实现：
了解其中的控制流和数据流的走向，了解其在扩缩容过程中的作用。因篇幅有限，这里只对组件进行简要描述，后续会针对每个组件进行详细的单独讲解。

### 1. queue-proxy

`queue-proxy` 是 一个伴随着用户容器运行的 Sidecar 容器，跟用户容器运行在同一个 Pod 中。每个请求到达业务容器之前都会经过 `queue-proxy` 容器，
这也是它问什么叫 `proxy` 的原因。

`queue-proxy` 的主要作用是统计和限制到达业务容器的请求并发量，当对一个 Revision 设置了并发量之后（比如设置了5），`queue-proxy` 会确保不会同时有超过5个请求打到业务容器。当有超过5个请求到来时，`queue-proxy`会先把请求暂存在自己的队列 `queue` 里，（这也是为什么名字里有个 queue的缘故）。`queue-proxy` 同时会统计进来的请求量，同时会通过指定端口提供平均并发量和 rps（每秒请求量）的查询。

### 2. Autoscaler
 
`Autoscaler` 是 Knative Serving 系统中一个重要的 pod，它由三部分组成：

* PodAutoscaler reconciler
* Collector
* Decider

`PodAutoscaler reconciler` 会监测 `PodAutoscaler`（KPA）的变更，然后交由 `Collector` 和 `Decider` 处理

`Collector` 主要负责从应用的 `queue-proxy` 那里收集指标， `Collector` 会收集每个实例的指标，然后汇总得到整个系统的指标。为了实现扩缩容，会搜集所有应用实例的样本，并将收集到的样本反映到整个集群。

`Decider` 得到指标之后，来决定多少个Pod 被扩容出来。简单的计算公式如下：

```shell 
want = concurrencyInSystem/targetConcurrencyPerInstance
```

另外，扩缩容的量也会受到 `Revision` 中最大最小实例数的限制。同时 `Autoscaler` 还会计算当前系统中剩余多少突发请求容量（可扩缩容多少实例）进来决定  请求是否走 `Activator` 转发。

### 3. Activator 

`Activator` 是整个系统中所用应用共享的一个组件，是可以扩缩容的，主要目的是缓存请求并给 `Autoscaler`主动上报请求指标

`Activator` 主要作用在从零启动和缩容到零的过程，能根据请求量来对请求进行负载均衡。当 `revision` 缩容到零之后，请求先经过 `Activator` 而不是直接到 `revision`。 当请求到达时，`Activator` 会缓存这这些请求，同时携带请求指标（请求并发数）去触发 `Autoscaler`扩容实例，当实例 ready后，`Activator` 才会将请求从缓存中取出来转发出去。同时为了避免后端的实例过载，`Activator` 还会充当一个负载均衡器的作用，根据请求量决定转发到哪个实例（通过将请求分发到后端所有的Pod上，而不是他们超过设置的负载并发量）。 Knative Serving 会根据不同的情况来决定是否让请求经过 `Activator`，当一个应用系统中有足够多的pod实例时，`Activator` 将不再担任代理转发角色，请求会直接打到 `revision` 来降低网络性能开销。

跟 `queue-proxy` 不同，`Activator` 是通过 websocket 主动上报指标给 `Autoscaler`，这种设计当然是为了应用实例尽可能快的冷启动。`queue-proxy` 是被动的拉取：`Autoscaler`去 `queue-proxy`指定端口拉取指标。

## API

### PodAutoscaler (PA,KPA)

**API:** `podautoscalers.autoscaling.internal.knative.dev`

`PodAutoscaler` 是对扩缩容的一个抽象，简写是 KPA 或 PA ，每个 `revision`
 会对应生成一个 `PodAutoscaler`。 
 可通过下面的指令查看
 
```shell
kubectl get kpa -n xxx
```

### ServerlessServices (SKS)
**API:** `serverlessservices.networking.internal.knative.dev`
 
 `ServerlessServices` 是 `KPA` 产生的，一个 `KPA` 生成一个 `SKS`，`SKS` 是对 k8s service 之上的一个抽象，
 主要是用来控制数据流是直接流向服务 `revision`（实例数不为零） 还是经过 `Activator`（实例数为0）。
 
 对于每个 `revision`，会对应生成两个k8s service ，一个`public service`，一个 `private service`.
 
 `private service` 是标准的 k8s service，通过label selector 来筛选对应的deploy 产生的pod，即 svc 对应的 endpoints 由 k8s 自动管控。
 
 `public service` 是不受 k8s 管控的，它没有 label selector，不会像 `private service` 一样 自动生成 endpoints。`public service` 对应的 endpoints 
 由 Knative `SKS reconciler` 来控制。 
 
 `SKS` 有两种模式：`proxy` 和 `serve`
 
 * `serve` 模式下 `public service` 后端 endpoints 跟 `private service `一样， 所有流量都会直接指向  `revision` 对应的 pod。
 * `proxy` 模式下 `public service` 后端 endpoints 指向的是 系统中  `Activator` 对应的 pod，所有流量都会流经 `Activator`。 
 
 ## 数据流
 
 下面看几种情况下的数据流向，加深对Knative 扩缩容系统机制的理解。
 
 ### 1. 稳定状态下的扩缩容
 
 ![scale-up-down.png](/images/scale-up-down.png)

 **稳定状态下的工作流程如下：**

 1. 请求通过 `ingress` 路由到 `public service` ，此时 `public service` 对应的 endpoints 是 revision 对应的 pod 
 2. `Autoscaler` 会定期通过 `queue-proxy` 获取 `revision` 活跃实例的指标，并不断调整 revision 实例。
 请求打到系统时，  `Autoscaler` 会根据当前最新的请求指标确定扩缩容比例。
 3.  `SKS` 模式是 `serve`, 它会监控  `private service` 的状态，保持 `public service` 的 endpoints 与 `private service` 一致 。
 
 ### 2. 缩容到零
 
 ![scale-to-0.png](/images/scale-to-0.png)
 
 **缩容到零过程的工作流程如下：**

1. `AutoScaler` 通过  `queue-proxy` 获取 `revision` 实例的请求指标
2. 一旦系统中某个 `revision` 不再接收到请求（此时 `Activator` 和 `queue-proxy` 收到的请求数都为 0）
3. `AutoScaler` 会通过 `Decider` 确定出当前所需的实例数为 0，通过 `PodAutoscaler` 修改 revision 对应 Deployment 的 实例数
4.  在系统删掉 `revision` 最后一个 Pod 之前，会先将 `Activator` 加到 数据流路径中（请求先到 `Activator`）。`Autoscaler` 触发 `SKS` 变为 `proxy` 模式，此时 `SKS` 的 `public service` 后端的endpoints 变为 `Activator` 的IP，所有的流量都直接导到 `Activator` 
5. 此时，如果在冷却窗口时间内依然没有流量进来，那么最后一个 Pod 才会真正缩容到零。
 
 ### 3. 冷启动（从零开始扩容）
 
 ![from0](/images/scale-from-0.png)
 
  **冷启动过程的工作流程如下：**

 当 `revision` 缩容到零之后，此时如果有请求进来，则系统需要扩容。因为 `SKS` 在 `proxy` 模式，流量会直接请求到 `Activator` 。`Activator` 会统计请求量并将 指标主动上报到 `Autoscaler`， 同时 `Activator` 会缓存请求，并 watch `SKS` 的 `private service`， 直到 `private service` 对应的endpoints产生。
 
 `Autoscaler` 收到 `Activator` 发送的指标后，会立即启动扩容的逻辑。这个过程的得出的结论是至少一个Pod要被创造出来，`AutoScaler` 会修改 `revision` 对应 `Deployment` 的副本数为为N（N>0）,`AutoScaler` 同时会将 `SKS` 的状态置为 `serve` 模式，流量会直接到导到 `revision` 对应的 pod上。
 
 `Activator` 最终会监测到 `private service` 对应的endpoints的产生，并对 endpoints 进行健康检查。健康检查通过后，`Activator` 会将之前缓存的请求转发到
 健康的实例上。
 
 最终 `revison` 完成了冷启动（从零扩容）。
 
 
 