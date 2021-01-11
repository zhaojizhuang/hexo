---
layout: post
title:  "【Knative系列】看完这篇还不懂 Knative Serving，你来打我~"
date:   2020-10-15 14:10:18 +0800
categories: knative
tags:  ["knative","serving"]
author: zhaojizhuang
---

> 本文主要讲解 Knative serving 系统及组件：

## 发展历程

Knative 是谷歌开源的 serverless 架构方案，旨在提供一套简单易用的 serverless 方案，把 serverless 标准化。目前参与的公司主要是 **Google、Pivotal、IBM、Red Hat**，2018年7月24日才刚刚对外发布，当前还处于快速发展的阶段(3.4k star, 3.3k issue)。



Knative 包含 build（已被tekton取代），serving，event三个部分，本文主要介绍serving。


先看下Knative的发展里历程

 ![Generated](/images/k_history.png)

| 项目    | 时间                | 发起公司                                            |
| ------- | ------------------- | --------------------------------------------------- |
| k8s     | 2014年6月           | Google （同年加入 微软、RedHat、IBM、Docker）       |
| istio   | **2017** 年6 月8 日 | Google，IBM 和 Lyft                                 |
| knative | **2018**年7月24日   | Google （目前主要为 Google、Pivotal、IBM、Red Hat） |

## 前言 有了 ”k8s，为什么还要 knative”

通常情况下 **Serverless = Faas + Baas**，Faas 无状态（业务逻辑），Baas 有状态（通用服务：数据库，认证，消息队列）。

既然有了 k8s (paas), 为什么还需要 Knative (Serverless),下面从四个方面来进行解释：**资源利用率，弹性伸缩，按比例灰度发布，用户运维复杂性**

### 1.  资源利用率

 

讲资源利用率之前先看下下面两个应用，左边**应用 A** 这个是典型的中长尾应用，**中长尾应用就是那些每天大部分时间都没有流量或者有很少流量的应用。**

 ![Generated](/images/changwei.png)

想一下，如果用 `paas(k8s)` 来实现的话，对于应用 A，需按照资源占用的资源最高点来申请规格，也就是 **4U10G**, 而且 paas 最低实例数**>=1**, 长此以往, 当中长尾应用足够多时，资源利用率可想而知。有可能会出现 大部分边缘集群资源被预占，但是利用率却很低。


而 Knative，恰恰可以解决应用A的资源占用问题，因为 Knative 可以将实例缩容为0，并根据请求自动扩缩容，缩容到零可以大大增加集群的资源利用率，因为中长尾应用都是按需所取，不会过度空占用资源。

 

比较合理的是对应应用A 用 `Knative(Serverless)`，对于应用 B 用 `k8s（Paas）`

### 2. 弹性伸缩

 

大家可能会想到，k8s 也有 hpa 进行扩缩容，但是 Knative 的 kpa 和 k8s 的 hpa 有很大的不同：

• **Knative** **支持缩容到****0****和从****0****启动，反应更迅速适合流量突发场景；**

• **K8s HPA** **不支持缩容到****0****，反应比较保守**

 

具体比较如下

*

|              | Knative KPA                                                  | k8s HPA                                                      |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 指标类型     | 可以根据 请求量扩速容                                        | 只能根据 cpu memory 等指标扩缩容（或自定义指标）             |
| 01启动       | **可以缩容到0和冷启动**                                      | 只能缩容到1（如果缩容到0，就没有实例了，流量进不来，metrics数据永远为0，此时HPA也无能为力） |
| 指标获取方式 | Knative 指标获取有两种方式，Activator 和queue-proxy, activator的metrics 是通过**websocket** **主动push** 给Autoscaler的，反应更迅速 | k8s 只能是通过prometheus 轮询获取。                          |
| 反应速度     | Knative 默认会 计算 60 秒窗口内的平均并发数， 也会计算 6 秒的恐慌窗口，6s内达到目标并发的 2 倍，则会进入恐慌模式。在恐慌模式下，Autoscaler 在更短、更敏感的紧急窗口上工作 | 而且 HPA 本身设计比较保守，有一个稳定期（默认5min）默认**在5min**内没有重新扩缩容的情况下，才会触发扩缩容。当大流量突发过来时，如果正处在5min内的HPA稳定期，这个时候根据HPA的策略，会导致无法扩容。 |

 

### 3. 按比例灰度发布

设想一下，假如通过 k8s来进行灰度发布怎么做，只能是通过两个Deployment和两个service，如果灰度升级的话只能通过修改两个 Deployment 的rs，一个逐渐增加，一个逐渐减少，如果想要按照百分比灰度，只能在外部负载均衡做文章，所以要想 Kubernetes 原生实现，至少需要**一个按流量分发的网关，两个 service，两个 deployment ，两个 ingress ， hpa，prometheus** 等，实现起来相当复杂。


使用 Knative 就可以很简单的实现，只需一个 ksvc 即可

 

### 4. 用户运维复杂性

 

**使用** **Knative** **免运维，低成本：**用户只关心业务逻辑，由工具和云去管理资源，复杂性由平台去做：容器镜像构建，Pod 的管控，服务的发布，相关的运维等。


k8s 本质上还是基础设施的抽象，对应pod的管控，服务的发布，镜像的构建等等需要上层的包装。

 

### 1. 相关概念介绍

**资源介绍**： knative 资源

 **1. *Service(ksvc)***



**2. *Configuration***

 

**3. *Revision***





**4. *Route***



**5. *Ingress(kingress)***



**6. *PodAutoScaler(kpa) ***



**7. *ServerlessService(sks)***



**8. *k8s Service  (public)***







\2. **Knative** **网关**

Knative 从设计之初就考虑到了其扩展性，通过抽象出来 Knative Ingress （kingress）资源来对接不同的网络扩展：**Ambassador****、****Contour****、****Gloo****、****Istio****、****Kong****、****Kourier**


这些网络插件都是基于 **Envoy** 这个新生的云原生服务代理，关键特性是可以**基于流量百分比进行分流**。

感兴趣的可以研究下 https://www.servicemesher.com/envoy/intro/what_is_envoy.html


 ![Generated](file:////Users/zhaojizhuang/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image008.jpg)

 

\3. **Knative** **组件**

\1. **Queue-proxy**

 ![Generated](file:////Users/zhaojizhuang/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image010.jpg)

Queue-proxy 是每个业务pod中都存在的 sidecar，每个发到业务pod的请求都会先经过queue-proxy


queue-proxy的主要作用是 **收集和限制** 业务应用的并发量，比如当一个revision 设定了并发量为 5 ，那么queue-proxy会保证每次到达业务容器的请求数不会大于5. 如果多于5个请求到达，queue-proxy会将请求暂存在本地队列中。


几个端口表示如下：

• 8012， queue-proxy 代理的http端口，流量的入口都会到 8012 

• 8013， http2 端口，用于grpc流量的转发

• 8022,  queue-proxy 管理端口，如健康检查

• 9090， queue-proxy的监控端口，暴露指标供 autoscaler 采集，用于kpa扩缩容

• 9091， prometheus 应用监控指标（请求数，响应时长等）

• USER_PORT， 是用户配置的容器端口，即业务实际暴露的服务端口，ksvc container port 配置的

\2. **Autoscaller**

 ![Generated](file:////Users/zhaojizhuang/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image012.jpg)


AutoScaller 主要是 Knative 的扩缩容实现，通过request指标来决定是否扩缩容实例，指标来源有两个：

• 通过获取每个pod queue-proxy 中的指标

• Activator 通过websocket 主动上报

**扩缩容算法如下：**

autoscaler是基于每个Pod（并发）的运行中请求的平均数量。系统的默认目标并发性为100，但是我们为服务使用了10。我们为服务加载了50个并发请求，因此自动缩放器创建了5个容器（50个并发请求/目标10 = 5个容器）。

 

算法中有两种模式，分别是panic和stable模式，一个是短时间，一个是长时间，为了解决短时间内请求突增的场景，需要快速扩容。

 

**Stable Mode****（稳定模式）**

 

在稳定模式下，Autoscaler 根据每个pod期望的并发来调整Deployment的副本个数。根据每个pod在60秒窗口内的平均并发来计算，而不是根据现有副本个数计算，因为pod的数量增加和pod变为可服务和提供指标数据有一定时间间隔。

 

**Panic Mode** **（恐慌模式）**

 

KPA会在**60****秒**的窗口内计算平均并发性，因此系统需要一分钟时间才能稳定在所需的并发性级别。但是，自动缩放器还会计算一个**6****秒**的紧急窗口，如果该窗口达到目标并发性的2倍，它将进入紧急模式。在紧急模式下，自动缩放器在较短，更敏感的紧急窗口上运行。一旦在60秒内不再满足紧急情况，autoscaler将返回到最初的60秒稳定窗口。

 

\1.                             |

\2.                  Panic Target---> +--| 20

\3.                           | |

\4.                           | <------Panic Window

\5.                           | |

\6.     Stable Target---> +-------------------------|--| 10  CONCURRENCY

\7.              |             | |

\8.              |           <-----------Stable Window

\9.              |             | |

\10. --------------------------+-------------------------+--+ 0

\11. 120            60              0

\12.            TIME


\3. **Activator

**

 ![Generated](file:////Users/zhaojizhuang/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image012.jpg)


Activator 的作用：**流量的负载和缓存，是****Knative** **能缩容到** **0** **的关键**

实例为0 时（冷启动），流量会先转发到 Activator，由 Activator 通过websocket 主动触发 Autoscaler扩缩容。

 

Activator 本身的扩缩容通过 hpa实现

\1. root@admin03.gyct:~# kubectl get hpa -n knative-serving

\2. NAME    REFERENCE       TARGETS  MINPODS  MAXPODS  REPLICAS  AGE

\3. activator  Deployment/activator  8%/100%  1     20    1     73d

可以看到 Activator 默认最大可以扩缩容到 20

 

Activator 只在 冷启动阶段是proxy模式，当当实例足够时，autoscaler 会更新 public service 的endpoints 指向 revision对应的pod，将请求导向真正的后端，这时候处理请求过程中 activator 不在起作用 


**详细步骤如下：**

 ![Generated](file:////Users/zhaojizhuang/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image014.jpg)


冷启动时 activator的角色


\1. Kingress 收到请求（确切的是 网关收到请求后根据kingress生成的配置做转发，如istio是virtualservice）后，将请求导至activator

\2. activator 将请求保存在缓存中

\3. Activator 触发autoscaller， 触发过程中其中做了两件事：

a. Activator 携带了第2步缓存的请求信息到autoscaler；

b. 触发信号促使autoscaller 立即做出扩容决定，而不是等下个扩容周期；

\4. 考虑到有请求需要处理，但是目前实例是0，autoscaller 决定扩容出一个实例，于是设置一个新的scale 目标

\5. activator在等待 autoscaler 和 Serving准备工作的时候，activator 会去轮询Serving 查看实例时候准备完毕

\6. Serving 调用k8s 生成k8s 实例（deploy，pod）

\7. 当实例可用时，Activator 将请求从缓存中proxy到实例

\8. Activator 中的proxy 模块 将请求代理到实例

\9. Activator 中的proxy模块 同样将response 返回到 kingress




\4. **扩缩容原理****

**

\1. **正常扩缩容场景（非****0****实例）**

 ![Generated](file:////Users/zhaojizhuang/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image016.jpg)


**稳定状态下的工作流程如下：**

\1. 请求通过 ingress 路由到 public service ，此时 public service 对应的 endpoints 是 revision 对应的 pod

\2. Autoscaler 会定期通过 queue-proxy 获取 revision 活跃实例的指标，并不断调整 revision 实例。

\3. 请求打到系统时， Autoscaler 会根据当前最新的请求指标确定扩缩容比例。

\4. SKS 模式是 serve, 它会监控 private service 的状态，保持 public service 的 endpoints 与 private service 一致 。

\2. **缩容到****0****的场景**

 ![Generated](file:////Users/zhaojizhuang/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image018.jpg)

 

**缩容到零过程的工作流程如下：**


\1. AutoScaler 通过 queue-proxy 获取 revision 实例的请求指标

\2. 一旦系统中某个 revision 不再接收到请求（此时 Activator 和 queue-proxy 收到的请求数都为 0）

\3. AutoScaler 会通过 Decider 确定出当前所需的实例数为 0，通过 PodAutoscaler 修改 revision 对应 Deployment 的 实例数

\4. 在系统删掉 revision 最后一个 Pod 之前，会先将 Activator 加到 数据流路径中（请求先到 Activator）。Autoscaler 触发 SKS 变为 proxy 模式，此时 SKS 的 public service 后端的endpoints 变为 Activator 的IP，所有的流量都直接导到 Activator

\5. 此时，如果在冷却窗口时间内依然没有流量进来，那么最后一个 Pod 才会真正缩容到零。

 

\3. **从****0** **启动的场景**

 

 ![Generated](file:////Users/zhaojizhuang/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image020.jpg)

**冷启动过程的工作流程如下：**


当 revision 缩容到零之后，此时如果有请求进来，则系统需要扩容。因为 SKS 在 proxy 模式，流量会直接请求到 Activator 。Activator 会统计请求量并将 指标主动上报到 Autoscaler， 同时 Activator 会缓存请求，并 watch SKS 的 private service， 直到 private service 对应的endpoints产生。


Autoscaler 收到 Activator 发送的指标后，会立即启动扩容的逻辑。这个过程的得出的结论是至少一个Pod要被创造出来，AutoScaler 会修改 revision 对应 Deployment 的副本数为为N（N>0）,AutoScaler 同时会将 SKS 的状态置为 serve 模式，流量会直接到导到 revision 对应的 pod上。


Activator 最终会监测到 private service 对应的endpoints的产生，并对 endpoints 进行健康检查。健康检查通过后，Activator 会将之前缓存的请求转发到

健康的实例上。


最终 revison 完成了冷启动（从零扩容）。

 

  