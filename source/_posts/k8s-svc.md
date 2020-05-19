---
layout: post
title:  "kubernetes&容器网络（1）之seivice"
date:   2018-11-28 13:19:10 +0800
categories: k8s
tags:  ["k8s", "iptables"]
author: zhaojizhuang

---


## 1. Service介绍

`Kubernetes` 中有很多概念，例如 `ReplicationController、Service、Pod`等。我认为 `Service` 是 `Kubernetes` 中最重要的概念，没有之一。

为什么 `Service` 如此重要？因为它解耦了前端用户和后端真正提供服务的 `Pods` 。在进一步理解 `Service` 之前，我们先简单地了解下 `Pod` ， `Pod` 也是 `Kubernetes` 中很重要的概念之一。

在 `Kubernetes` 中，`Pod` 是能够创建、调度、和管理的最小部署单元，而不是单独的应用容器。`Pod` 是容器组，一个`Pod`中容器运行在一个共享的应用上下文中。这里的共享上下文是为多个 `Linux Namespace` 的联合。例如：

- `PID`命名空间（在同一个 `Pod` 中的应用可以看到其它应用的进程）
- `Network`名字空间（在同一个 `Pod` 中的应用可以访问同样的IP和端口空间）
- `IPC`命名空间（在同一个 `Pod` 中的应用可以使用`SystemV IPC`或者`POSIX`消息队列进行通信）
- `UTS`命名空间（在同一个 `Pod` 中的应用可以共享一个主机名称）
-  `Pod` 是一个和应用相关的“逻辑主机”， `Pod` 中的容器共享一个网络名字空间。 `Pod` 为它的组件之间的数据共享、通信和管理提供了便利。

我们可以看出， `Pod` 在资源层面抽象了容器。

因此，在`Kubernetes`中，让我们暂时先忘记容器，记住 `Pod` 。

`Kubernetes`对`Service`的定义是：` Service  is an abstraction which defines a logical set of  Pods  and a policy by which to access them。`我们下面理解下这句话。

刚开始的时候，生活其实是很简单的。一个提供特定服务的进程，运行在一个容器中，监听容器的`IP`地址和端口号。客户端通过`<Container IP>:<ContainerPort>`，或者通过使用Docker的端口映射`<Host IP>:<Host Port>`就可以访问到这个服务了。`The simplest, the best`，简单的生活很美好。

![](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/A1HKVXsfHNmswyx38Qh8WVkLPHpr9pex3e7DRk9H0AicQGXP2r1xP8ibkx5QqSnsMc6pQf3wbHPAKcxibFVicz1ChA/640)

但是美好的日子总是短暂的。小伙伴们太热情了，单独由一个容器提供服务不够用了，怎么办？很简单啊，由多个容器提供服务不就可以了吗。问题似乎得到了解决。

可是那么多的容器，客户端到底访问哪个容器中提供的服务呢？访问容器的请求不均衡怎么办？假如容器所在的主机故障了，容器在另外一台主机上拉起了，这个时候容器的IP地址变了，客户端怎么维护这个容器列表呢？所以，由多个容器提供服务的情况下，一般有两种做法：

客户端自己维护提供服务的容器列表，自己做负载均衡，某个容器故障时自己做故障转移；
提供一个负载均衡器，解耦用户和后端提供服务的容器。负载均衡器负责向后端容器转发流量，并对后端容器进行健康检查。客户端只要访问负载均衡器的IP和端口号就行了。
我们在前面说Service解耦了前端用户和后端真正提供服务的 `Pod` s。从这个意义上讲，`Service`就是`Kubernetes`中 `Pod` 的负载均衡器。

从生命周期来说， `Pod` 是短暂的而不是长久的应用。 `Pod` 被调度到节点，保持在这个节点上直到被销毁。当节点死亡时，分配到这个节点的 `Pod` 将会被删掉。

但`Service`在其生命周期内，`IP`地址是稳定的。对于`Kubernetes`原生的应用，`Kubernetes`提供了一个`Endpoints`的对象，这个`Endpoints`的名字和`Service`的名字相同，它是一个<Pod IP>:<targetPort>的列表，负责维护`Service`后端的`Pods`的变化。

总结一下，`Service`解耦了前端用户和后端真正提供服务的`Pods`，`Pod`在资源层面抽象了容器。由于它们的存在，使得这个简单的世界变得复杂了。


对了，`Service`怎么知道是哪些后端的Pods在真正提供自己定义的服务呢？在创建`Pods`的时候，会定义一些label；在创建`Service`的时候，会定义L`abel Selector，Kubernetes`就是通过`Label Selector`来匹配后端真正服务于`Service`的后端`Pods`的。

## 2. 定义一个`Service`
接下来就有点没意思了，我要开始翻译上面说的很重要的`Services in Kubernetes`了。当然，我会加入自己的理解。

`Service`也是`Kubernetes`中的一个`REST`对象。可以通过向`apiserver`发送`POST`请求进行创建。当然，`Kubernetes`为我们提供了一个强大和好用的客户端`kubectl`，代替我们向`apiserver`发送请求。但是`kubectl`不仅仅是一个简单的`apiserver`客户端，它为我们提供了很多额外的功能，例如`rolling update`等。

我们可以创建一个`yaml`或者`json`格式的`Service`规范文件，然后通过`kubectl create -f <service spec file>`创建之。一个`Service`的例子如下：

```json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "my-service"
    },
    "spec": {
        "selector": {
            "app": "MyApp"
        },
        "ports": [
            {
                "protocol": "TCP",
                "port": 80,
                "targetPort": 9376
            }
        ]
    }
}
```


以上规范创建了一个名字为`my-service`的`Service`对象，它指向任何有`app=MyApp`标签的、监听`TCP`端口`9376`的任何`Pods`。

`Kubernetes`会自动地创建一个和`Service`名字相同的`Endpoints`对象。`Service`的`selector`会被持续地评估哪些`Pods`属于这个`Service`，结果会被更新到相应的`Endpoints`对象。

当然，你也可以在定义`Service`的时候为其指定一个`IP`地址（ClusterIP，必须在kube-apiserver的`--service-cluster-ip-range`参数定义内，且不能冲突）。

`Service`会把到<ClusterIP>:<Port>的流量转发到`targetPort`。缺省情况下`targetPort`等于`port`的值。一个有意思的情况是，这里你可以定义`targetPort`为一个字符串（我们可以看到`targetPort`的类型为`IntOrString`），这意味着在后端每个`Pods`中实际监听的端口值可能不一样，这极大地提高了部署`Service`的灵活性。

`Kubernetes`的`Service`支持`TCP`和`UDP`协议，缺省是`TCP`。

## 3. `Service`发布服务的方式
`Service`有三种类型：`ClusterIP，NodePort和LoadBalancer`。

### 3.1 ClusterIP

关于`ClusterIP`：

通过`Service`的`spec.type: ClusterIP`指定；
使用`cluster-internal ip`，即`kube-apiserver`的`--service-cluster-ip-range`参数定义的IP范围中的IP地址；
缺省方式；
只能从集群内访问，访问方式：`<ClusterIP>:<Port>`；
`kube-proxy`会为每个`Service`，打开一个本地随机端口，通过`iptables`规则把到`Service`的流量trap到这个随机端口，由`kube-proxy`或者`iptables`接收，进而转发到后端`Pods`。

### 3.2 NodePort

关于`NodePort`：

通过`Service`的`spec.type: NodePort`指定；
包含`ClusterIP`功能；
在集群的每个节点上为`Service`开放一个端口（默认是`30000-32767`，由`kube-apiserver`的`--service-node-port-range`参数定义的节点端口范围）；
可从集群外部通过<`NodeIP>:<NodePort>`访问；
集群中的每个节点上，都会监听`NodePort`端口，因此，可以通过访问集群中的任意一个节点访问到服务。

### 3.3 LoadBalancer

关于`LoadBalancer`：

通过`Service`的`spec.type: LoadBalancer`指定；
包含`NodePort`功能；
通过`Cloud Provider`（例如`GCE`）提供的外部`LoadBalancer`访问服务，即`<LoadBalancerIP>:<Port>`；
`Service`通过集群的每个节点上的`<NodeIP>:<NodePort>`向外暴露；
有的`cloudprovider`支持直接从`LoadBalancer`转发流量到后端`Pods`（例如`GCE`），更多的是转发流量到集群节点（例如`AWS`，还有`HWS`）；
你可以在`Service`定义中指定`loadBalancerIP`，但这需要`cloudprovider`的支持，如果不支持则忽略。真正的`IP`在`status.loadBalancer.ingress.ip`中。
一个例子如下：

```json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "my-service"
    },
    "spec": {
        "selector": {
            "app": "MyApp"
        },
        "ports": [
            {
                "protocol": "TCP",
                "port": 80,
                "targetPort": 9376,
                "nodePort": 30061
            }
        ],
        "clusterIP": "10.0.171.239",
        "loadBalancerIP": "78.11.24.19",
        "type": "LoadBalancer"
    },
    "status": {
        "loadBalancer": {
            "ingress": [
                {
                    "ip": "146.148.47.155"
                }
            ]
        }
    }
}
```

目前对接`ELB`的实现几大厂商，比如 `HW` 是`ELB`转发流量到集群节点，后面再由`kube-proxy`或者`iptables`转发到后端的`Pods`。


### 3.4 External IPs

`External IPs` 不是一种`Service`类型，它不由`Kubernetes`管理，但是我们也可以通过它暴露服务。数据包通过`<External IP>:<Port>`到达集群，然后被路由到`Service`的`Endpoints`。一个例子如下：

```json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "my-service"
    },
    "spec": {
        "selector": {
            "app": "MyApp"
        },
        "ports": [
            {
                "name": "http",
                "protocol": "TCP",
                "port": 80,
                "targetPort": 9376
            }
        ],
        "externalIPs" : [
            "80.11.12.10"
        ]
    }
}
```

## 4. 几种特殊的`Service`

### 4.1 没有`selector`的`Service`
上面说`Kubernetes`的`Service`抽象了到`Kubernetes`的`Pods`的访问。但是它也能抽象到其它类型的后端的访问。举几个场景：

你想接入一个外部的数据库服务；
你想把一个服务指向另外一个`Namespace`或者集群的服务；
你把部分负载迁移到`Kubernetes`，而另外一部分后端服务运行在`Kubernetes`之外。
在这几种情况下，你可以定义一个没有`selector`的服务。如下：

```json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "my-service"
    },
    "spec": {
        "ports": [
            {
                "protocol": "TCP",
                "port": 80,
                "targetPort": 9376
            }
        ]
    }
}
```

因为没有`selector`，`Kubernetes`不会自己创建`Endpoints`对象，你需要自己手动创建一个`Endpoints`对象，把Service映射到后端指定的`Endpoints`上。

```json
{
    "kind": "Endpoints",
    "apiVersion": "v1",
    "metadata": {
        "name": "my-service"
    },
    "subsets": [
        {
            "addresses": [
                { "IP": "1.2.3.4" }
            ],
            "ports": [
                { "port": 9376 }
            ]
        }
    ]
}
```

注意：`Endpoint IP`不能是`loopback（127.0.0.1）`地址、`link-local（169.254.0.0/16）`和`link-local multicast（224.0.0.0/24）`地址。

看到这里，我们似乎明白了，这不就是`Kubernetes`提供的外部服务接入的方式吗？和`CloudFoundry`的`ServiceBroker`的功能类似。

### 4.2 多端口`（multi-port）`的`Service`
`Kubernetes`的`Service`还支持多端口，比如同时暴露`80`和`443`端口。在这种情况下，你必须为每个端口定义一个名字以示区分。一个例子如下：

```json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "my-service"
    },
    "spec": {
        "selector": {
            "app": "MyApp"
        },
        "ports": [
            {
                "name": "http",
                "protocol": "TCP",
                "port": 80,
                "targetPort": 9376
            },
            {
                "name": "https",
                "protocol": "TCP",
                "port": 443,
                "targetPort": 9377
            }
        ]
    }
}
```

注意：

多端口必须指定`ports.name`以示区分，端口名称不能一样；
如果是`spec.type: NodePort`，则每个端口的`NodePort`必须不一样，否则`Kubernetes`不知道一个`NodePort`对应的是后端哪个`targetPort`；
协议`protocol`和`port`可以一样。
## 4.3 `Headless services`
有时候你不想或者不需要`Kubernetes`为你的服务做负载均衡，以及一个`Service`的`IP`地址。在这种情况下，你可以创建一个`headless`的`Service`，通过指定`spec.clusterIP: None`。

对这类`Service`，不会分配`ClusterIP`。对这类`Service`的`DNS`查询会返回一堆`A`记录，即后端`Pods`的`IP`地址。另外，`kube-proxy`不会处理这类`Service`，`Kubernetes`不会对这类`Service`做负载均衡或者代理。但是`Endpoints Controller`还是会为此类`Service`创建`Endpoints`对象。

这允许开发者减少和k`ubernetes`的耦合性，允许他们自己做服务发现等。我在最后讨论的一些基于`Kubernetes`的容器服务，除了彻底不使用`Service`的概念外，也可以创建这类`headless`的`Service`，自己直接通过`LoadBalancer`把流量转发（负载均衡和代理）到后端`Pods`。

## 5. `Service`的流量转发模式

### 5.1 `Proxy-mode: userspace`

`userspace`的代理模式是指由用户态的`kube-proxy`转发流量到后端`Pods`。如下图所示。

![](https://d33wubrfki0l68.cloudfront.net/e351b830334b8622a700a8da6568cb081c464a9b/13020/images/docs/services-userspace-overview.svg)

关于`userspace`：

`Kube-proxy`通过`apiserver`监控`（watch）Service`和E`ndpoints`的变化；
`Kube-proxy`安装`iptables`规则；
`Kube-proxy`把访问`Service`的流量转发到后端真正的`Pods`上`（Round-Robin）`；
`Kubernetes v1.0`只支持这种转发方式；
通过设置`service.spec.sessionAffinity`: `ClientIP`支持基于`ClientIP`的会话亲和性。

### 5.2 `proxy-mode: iptables`

`iptables`的代理模式是指由内核态的`iptables`转发流量到后端`Pods`。如下图所示。

![](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg)

关于`iptables`：

- `Kube-proxy`通过`apiserver`监控`（watch）Service`和`Endpoints`的变化；
- `Kube-proxy`安装`iptables`规则；
- `iptables`把访问`Service`的流量转发到后端真正的`Pods上（Random）`；
- `Kubernetes v1.1`已支持，但不是默认方式，`v1.2`中将会是默认方式；
- 通过设置`service.spec.sessionAffinity`: `ClientIP`支持基于`ClientIP`的会话亲和性；
- 需要`iptables`和内核版本的支持。`iptables > 1.4.11`，内核支持`route_localnet`参数`(kernel >= 3.6)`；

相比`userspace`的优点：
- 1，数据包不需要拷贝到用户态的`kube-proxy`再做转发，因此效率更高、更可靠。
- 2，不修改`Client IP`。

### 5.3 `proxy-mode: iptables`

> kubernetes v1.8 引入， 1.11正式可用

在 `ipvs` 模式下，`kube-proxy`监视`Kubernetes`服务和端点，调用 `netlink `接口相应地创建 `IPVS` 规则， 并定期将 `IPVS` 规则与 `Kubernetes` 服务和端点同步。 该控制循环可确保　`IPVS`　状态与所需状态匹配。 访问服务时，`IPVS`　将流量定向到后端Pod之一。

![](https://d33wubrfki0l68.cloudfront.net/2d3d2b521cf7f9ff83238218dac1c019c270b1ed/9ac5c/images/docs/services-ipvs-overview.svg)

`IPVS`代理模式基于类似于 `iptables `模式的 `netfilter` 挂钩函数，但是使用哈希表作为基础数据结构，并且在内核空间中工作。 这意味着，与 `iptables` 模式下的 `kube-proxy` 相比`，IPVS` 模式下的 `kube-proxy` 重定向通信的延迟要短，并且在同步代理规则时具有更好的性能。与其他代理模式相比，`IPVS` 模式还支持更高的网络流量吞吐量。

`IPVS`提供了更多选项来平衡后端`Pod`的流量。 这些是：

- rr: round-robin
- lc: least connection (最小连接数)
- dh: destination hashing（目的地址has）
- sh: source hashing（源地址has）
- 等等 

### 5.3 `userspace`和`iptables`转发方式的主要不同点
`userspace`和`iptables`转发方式的主要不同点如下：

| 比较项 | `userspace`	| `iptables` |
|--|--|--|
 谁转发流量到`Pods `|	`kube-proxy`把访问`Service`的流量转发到后端真正的`Pods`上 |`iptables`把访问`Service`的流量转发到后端真正的`Pods`上 
|转发算法	|轮询`Round-Robin`|	随机`Random`
|用户态和内核态	|数据包需要拷贝到用户态的`kube-proxy`再做转发，因此效率低、不可靠	|数据包直接在内核态转发，因此效率更高、更可靠
|是否修改`Client IP`	|因为`kube-proxy`在中间做代理，会修改数据包的`Client IP`|	不修改数据包的`Client IP`
`iptables`版本和内核支持|	不依赖|	`iptables > 1.4.11`，内核支持`route_localnet`参数(`kernel >= 3.6`)

通过设置`kube-proxy`的启动参数`--proxy-mode`设定使用`userspace`还是`iptables`代理模式。


## 6. Service发现方式
现在服务创建了，得让别人来使用了。别人要使用首先得知道这些服务呀，服务治理很基本的一个功能就是提供服务发现。`Kubernetes`为我们提供了两种基本的服务发现方式：环境变量和`DNS`。

### 6.1 环境变量
当一个`Pod`在节点Node上运行时，`kubelet`会为每个活动的服务设置一系列的环境变量。它支持`Docker links compatible`变量，以及更简单的`{SVCNAME}_SERVICE_HOST`和`{SVCNAME}_SERVICE_PORT`变量。后者是把服务名字大写，然后把中划线（-）转换为下划线（_）。

以服务`redis-master`为例，它暴露TCP协议的6379端口，被分配了集群IP地址10.0.0.11，则会创建如下环境变量：

```shell
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```

这里有一个注意点是，如果一个`Pod`要访问一个`Service`，则必须在该Service之前创建，否则这些环境变量不会被注入此Pod。DNS方式的服务发现就没有此限制。

## 6.2 DNS
虽然`DNS`是一个`cluster add-on`特性，但是我们还是强烈推荐使用`DNS`作为服务发现的方式。`DNS`服务器通过`KubernetesAPI`监控新的`Service`的生成，然后为每个`Service`设置一堆`DNS`记录。如果集群设置了`DNS`，则该集群中所有的`Pods`都能够使用`DNS`解析`Sercice`。

例如，如果在`Kubernertes`中的`my-ns`名字空间中有一个服务叫做`my-service`，则会创建一个`my-service.my-ns`的`DNS`记录。在同一个名字空间`my-ns`的`Pods`能直接通过服务名`my-service`查找到该服务。如果是其它的`Namespace`中的`Pods`，则需加上名字空间，例如`my-service.my-ns`。返回的结果是服务的`ClusterIP`。当然，对于我们上面讲的`headless`的`Service`，返回的则是该`Service`对应的一堆后端`Pods`的`IP`地址。

对于知名服务端口，`Kubernetes`还支持`DNS SRV`记录。例如`my-service.my-ns`的服务支持`TCP`协议的`http`端口，则你可以通过一个`DNS SRV`查询`_http._tcp.my-service.my-ns`来发现http的端口。

对于每个`Service`的`DNS`记录，`Kubernetes`还会加上一个集群域名的后缀，作为完全域名（FQDN）。这个集群域名通过svc+安装集群`DNS`的`DNS_DOMAIN`参数指定，默认是`svc.cluster.local`。如果不是一个标准的`Kubernetes`支持的安装，则启动`kubelet`的时候指定参数`--cluster-domain`，你还需要指定`--cluster-dns`告诉`kubelet`集群`DNS`的地址。


## 6.3 如何发现和使用服务？
一般在创建`Pod`的时候，指定一个环境变量`GET_HOSTS_FROM`，值可以设为`env`或者`dns`。在`Pod`中的应用先获取这个环境变量，得到获取服务的方式。如果是`env`，则通过`getenv`获取相应的服务的环境变量，例如`REDIS_SLAVE_SERVICE_HOST`；如果是`dns`，则可以在`Pod`内通过标准的`gethostbyname`获取服务主机名。有个例外是P`od`的定义中，不能设置h`ostNetwork: true`。

获取到服务的地址，就可以通过正常方式使用服务了。

如下是`Kubernetes`自带的`guestbook.php`中的一段相关代码，供参考：

```shell
$host = 'redis-slave';
if (getenv('GET_HOSTS_FROM') == 'env') {
  $host = getenv('REDIS_SLAVE_SERVICE_HOST');
}
$client = new Predis\Client([
  'scheme' => 'tcp',
  'host'   => $host,
  'port'   => 6379,
]);
```

## 7. 一些容器服务中的`Service`
虽然`Service`在`Kubernetes`中如此重要，但是对一些基于`Kubernetes`的容器服务，并没有使用`Service`，或者用的是上面讨论的`headless`类型的`Service`。这种方式基本上是把容器当做`VM`使用的典型，`LoadBalancer`和`Pods`网络互通，通过`LoadBalancer`直接把流量转发到`Pods`上，省却了中间由`kube-proxy`或者`iptables`的转发方式，从而提高了流量转发效率，但是也由`LoadBalancer`自己提供对后端P`ods`的维护，一般需要`LoadBalancer`提供动态路由的功能（即后端`Pods`可以动态地从`LoadBalancer`上注册/注销）。



