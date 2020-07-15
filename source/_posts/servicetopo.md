---
layout: post
title:  "服务亲和性路由在华为云 k8s 中的实践【Kubernetes中的服务拓扑】"
date:   2018-07-07 23:40:18 +0800
categories: k8s
tags:  ["k8s"]
author: zhaojizhuang
mathjax: true
---

> 本文根据华为工程师DJ在LC3上的演讲整理：


## 1. 拓扑概念

首先，我们讲一下`kubernetes`中的拓扑。根据`kubernetes`现在的设计，我觉得拓扑可以是任意的。用户可以指定任何拓扑关系，比如az(available zone可用区)、region、机架、主机、交换机，甚至发电机。`Kubernetes`中拓扑的概念已经在调度器中被广泛使用。

## 2. `Kubernetes`调度器中的拓扑

在`kubernetes`中，`pod`是工作的基本单元，所以调度器的工作可以简化为“在哪里运行这些`pod`”，当然我们知道`pod`是运行在节点里，但是怎么选择节点，这是调度器要解决的问题。

在`kubernetes`中，我们通过`label`选择节点，从而确定`pod`应该放在哪个节点中。下面是`kubernetes`原生提供的一些`label`：

- k8s.io/hostname

此外，`kubernetes`允许集群管理员和云提供商自定义label，比如机架、磁盘类型等。

## 3.` Kubernetes`中基于拓扑的调度

比如，如果我们要将`PostgreSQL`服务运行在不同的`zone`中，假设`zone`的名字分别是`1a`和`1b`，那么我们可以在`podSpec`中定义节点亲和，如下图所示。主服务器需要运行在`zone a`的节点中，备用服务器需要运行在属于`zone b`的节点中。

![](/images/svctop1.jpeg)

在k`ubernetes`中，`pod`与`node`的亲和或反亲和是刚性的要求。那么我们之前的问题“在哪里运行这些`pod`”就可以简化成“可以在这个节点运行`pod`吗”，答案取决于节点的一些情况，比如节点的名字，节点所属的`region`，节点是否有`SSD`盘等。

另外，调度器还会考虑另外一个问题，“可以把`pod`和其他`pod`放在同一区域吗”，比如，`PostgreSQL`服务肯定不能和`MySQL`服务运行在同一个节点中。`Kubernetes`通过下面三个步骤解决这个问题：

- 1. 定义“其他`pod`”。这个过程与选择`node`的过程类似，我们通过`label`选择`pod`。比如，我们可以选择带有“`app`=`web-frontend`”`label`的`pod`

- 2. 定义“同一区域”。如图中的两个`pod`，是在同一区域吗?不是，因为它们`podSpec`中的t`opologyKey`字段，它是`node label中`的`key`，用于指定拓扑区域，比如`zone`、`rack`、`主机`等。比如如果我们使用“`k8s.io/hostname`”作为`topologyKey`，那么同一区域就表示在同一个主机上。

![](/images/svctop2.jpeg)

我们可以使用之前提到的任何`nodelabel`作为“同一区域”的标识，比如在同一个`zone`。或者使用自定义的`label`，下图给出了通过自定义拓扑`label`创建`node`组。

![](/images/svctop3.jpeg)

- 3. 最后是决定是否可以。这取决于亲和和反亲和（`affinity/anti-affinity`）。

4. `Kubernetes`中其他依赖拓扑关系的特性

上面讲到的部署`pod`时的拓扑感知。除了调度之外，还有许多特性会依赖拓扑关系：

- 工作负载：在缩容或滚动升级的时候，控制器决定先杀掉哪些pod

- 卷存储：卷存储会有拓扑限制，来决定可以挂在卷的node集。比如GCE的持久卷只能挂在在同一个zone的节点上，本地卷被所在节点访问。

## 依赖拓扑的服务亲和性路由

### 1. `Service`和`endpoint`

首先我们来了解一下`kubernetes`中的`service`和`endpoint`的 概念。

`Kubernetes`中的`service`是一个抽象的概念，它通过`label`选择一个`pod`的集合，并且定义了这些`pod`的访问策略。简言之，`service`指定一个虚拟`IP`，作为这些`pod`的访问入口，在4层协议上工作。

`Kubernetes`中的`endpoint`是`service`后端的`pod`的地址列表。作为使用者，我们不需要感知它们，`service`创建的时候`endpoint`会自动创建，并且会根据后端的`pod`自动配置好。

![](/images/svctop4.jpeg)

### 2. `Service`的工作原理

`Endpoints`控制器会`watch`到创建好的`service`和`pod`，然后创建相应的endpoint。`Kube-proxy`会`watch` `service`和`endpoint`，并创建相应的`proxy`规则。在`kubernetes1.8`之前`proxy`是通过底层的`iptables`实现，但是`iptables`只支持随机的负载均衡策略，并且可扩展性很差。

在1.8之后，我们实现并在社区持续推动了基于`ipvs`的`proxy`，这种`proxy`模式相对原来的`iptables`模式有很多优势，比如支持很多负载均衡算法，并且在大规模场景下接近无限扩展性等。好消息是，现在`kubernetes`社区基于`ipvs`的`proxy`已经`svc`，大家可以在生产环境使用。那么问题来了，既然我们已经有`IPVS`加持了，为什么还需要服务亲和性路由呢？

![](/images/svctop5.jpeg)

### 3. 服务亲和性路由

先看一下用户的使用场景，当我们将`pod`正确的放到了用户指定的区域之后，就会有下面的问题。

- 1. **单节点通信**（访问`serviceIP`的时候只能访问到本节点的应用）

    - 我们使用`daemonset`部署`fluent`的时候，应用只需要与当前节点的`fluent`通信。

    - 一些用户出于安全考虑，希望确保他们只能与本地节点的服务通信，因为跨节点的流量可能会携带来自其他节点的敏感信息。

- 2. **限制跨zone的流量**

    -  因为跨`zone`的流量会收费，而同一个`zone`的流量则不会。而有些云提供商，比如阿里云甚至不允许跨`zone`的流量。

    -  性能优势：显然，到达本区域（节点/`zone`等）的流量肯定比跨区访问的流量有更低的延时和更高的带宽。

### 4. 亲和性路由的实现需要解决的问题

正如我们前边讲到的，本地意味着一定的拓扑等级，我们需要有一个可以根据拓扑选择`endpoint`子集的机制。

这样我们就面临着如下问题：

- 是软亲和还是硬亲和。硬亲和意味着只需要本地的后端，而软亲和意味着首先尝试本地的，如果本地没有则尝试更广范围的。

- 如果是软亲和，那么判定标准是什么呢？可能给每个拓扑区域增加权重是一个解决方案。

- 如果多个后端满足条件，那么选择的依据又是什么呢？随机选择还是引入概率？

### 5. 我们的方案

我们提供了一个解决方案，引用一种新的资源“`ServicePolicy`”。集群管理员可以通过`ServicePolicy`配置“`local`”的选择标准，以及各种拓扑的权重。`ServicePolicy`是一种可选的`namespace`范围内的资源，有三种模式：`Required/Perferred/Ignored`，分别代表硬亲和/软亲和/忽略。

![](/images/svctop6.jpeg)

上图是我们引入和`ServicePolicy`资源和`endpoint`引入的字段示例。在我们的示例中，`ServicePolicy`会选择`namespace foo`中带有`label app=bar`的`service`。由于我们将`hostname`设置为`ServicePolicy`的拓扑依据，那么对这些`service`的访问会只路由到与`kube-proxy`有在同一个`host`的后端。

需要说明的是，`service`和`ServicePolicy`是多对多的关系，一个`ServicePolicy`可以通过label选择多个`service`，一个`service`也可以被多个`ServicePolicy`选中，有多个亲和性要求。

另外我们还希望`endpoint`携带节点的拓扑信息，因此我们为`endpoint`添加一个新的字段`Topology`，用于识别`pod`属于的拓扑区域，比如在哪个`host/rack/zone/region`等。

这会改变现有的逻辑。如下图所示，`Endpoint`控制器需要watch两种新的资源，`node`和`ServicePolicy`，它需要维护`node`与`endpoint`的对应关系，并根据`node`的拓扑信息更新`endpoint`的`Topology`字段。另外，`kube-proxy`也会相应地作一些改动。它需要过滤掉与自身不在同一个拓扑区域的`endpoint`，这意味着`kube-proxy`会在不同的节点上创建不同的规则。

![](/images/svctop7.jpeg)

下图表示从`ServicePolicy`到`proxy`规则的数据流。首先`ServicePolicy`通过`label`选择一组`service`，我们可以根据这些`service`找到它们的`pod`。`Endpoint`控制器会将`pod`所在节点的拓扑`label`放到对应的`endpoint`中。`Kube-proxy`负责仅为处在同一拓扑区域的`endpoint`创建`proxy`规则，并且当多个`endpoint`满足要求时提供路由策略。

![](/images/svctop8.jpeg)

[总 结]

目前我们已经在华为云的CCE服务上实现了服务亲和性路由，效果很好，欢迎大家体验。我们很乐意把这个特性开源出来，并且正在做这件事，相信它会像`IPVS`一样，成为`kubernetes`下一个版本的一个重要特性。