---
layout: post
title:  "容器网络"
date:   2018-10-25 17:29:18 +0800
categories: k8s
tags:  ["k8s", "容器网络"]
author: zhaojizhuang

---

# 容器网络


##  vxlan 

vxlan原理: VXLAN通过MAC-in-UDP的报文封装，实现了二层报文在三层网络上的透传,属于overlay网络

## Flannel

首先，flannel利用**`Kubernetes-API(这里就是取node.spec.podCIDR)或者etcd`**用于存储整个集群的网络配置，其中最主要的内容为设置集群的网络地址空间。例如，设定整个集群内所有容器的IP都取自网段“10.1.0.0/16”。

接着，flannel在每个主机中运行flanneld作为agent，它会为所在主机从集群的网络地址空间中，获取一个小的网段subnet，本主机内所有容器的IP地址都将从中分配。

`flannel` 的 `UDP` 模式和 `Vxlan` 模式 `host-gw` 模式

- `UDP` 模式是 三层 `overlay`,即，将原始数据包的三层包（IP包）装在 `UDP` 包里,通过 ip+端口 传到目的地，ip为目标node ip 端口为目标节点上flanneld进程监听的8285端口，解析后传入flannel0设备进入内核网络协议栈，
UDP模式下 封包解包是在 flanneld里进行的也就是用户态下

![](https://static001.geekbang.org/resource/image/84/8d/84caa6dc3f9dcdf8b88b56bd2e22138d.png)

![](https://static001.geekbang.org/resource/image/e6/f0/e6827cecb75641d3c8838f2213543cf0.png)

**重要！！！ 《深入解析kubernetes》** 33章  https://time.geekbang.org/column/article/65287

- VxLan 模式 是二层 `overlay`,即将原始Ethernet包（MAC包）封装起来，通过vtep设备发到目的vtep，vxlan是内核模块，vtep是flannneld创建的，vxlan封包解封完全是在内核态完成的
- 

![](https://static001.geekbang.org/resource/image/43/41/43f5ebb001145ecd896fd10fb27c5c41.png)

 - 注意点 
  - inner mac 为 目的vtep的mac
  - outer ip为目的node的ip **这一点和UDP有区别**
下一跳ip对应的mac地址是ARP表里记录的，inner mac对应的arp记录是 flanneld维护的，outer mac arp表是node自学习的

![](https://static001.geekbang.org/resource/image/ce/38/cefe6b99422fba768c53f0093947cd38.png)

- `host-gw` 模式的工作原理,是在 节点上加路由表，其实就是将每个 Flannel 子网（Flannel Subnet，比如：10.244.1.0/24）的“下一跳”，设置成了该子网对应的宿主机的 IP 地址。
这台“主机”（Host）会充当这条容器通信路径里的“网关”（Gateway）。这也正是“host-gw”的含义。

```shell
$ ip route
...
<目的容器IP地址段> via <网关的IP地址> dev eth0# 网关的 IP 地址，正是目的容器所在宿主机的 IP 地址
```

**Flannel host-gw 模式必须要求集群宿主机之间是二层连通的。如果分布在不同的子网里是不行的，只是三层可达**

### POD IP的分配
#### 使用CNI后，即配置了 `kubelet` 的 `--network-plugin=cni`，容器的IP分配：
kubelet 先创建pause容器生成network namespace
调用 网络driver CNI driver
CNI driver 根据配置调用具体的cni 插件
cni 插件给pause 容器配置网络
pod 中其他的容器都使用 pause 容器的网络

#### CNM模式
Pod IP是docker engine分配的，Pod也是以docker0为网关，通过veth连接network namespace
#### flannel的两种方式 CNI CNM总结
CNI中，docker0的ip与Pod无关，Pod总是生成的时候才去动态的申请自己的IP，而CNM模式下，Pod的网段在docker engine启动时就已经决定。
CNI只是一个网络接口规范，各种功能都由插件实现，flannel只是插件的一种，而且docker也只是容器载体的一种选择，Kubernetes还可以使用其他的，

## cluster IP的分配
    是在kube-apiserver中 `pkg/registry/core/service/ipallocator`中分配的
    
    
## network policy 

```yaml
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
    - podSelector:
        matchLabels:
          role: client
```

像上面这样定义的 namespaceSelector 和 podSelector，是“或”（OR）的关系，表示的是yaml数组里的两个元素

```yaml
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
      podSelector:
        matchLabels:
          role: client
```

像上面这样定义的 namespaceSelector 和 podSelector，是“与”（AND）的关系，yaml里表示的是一个数组元素的两个字段

Kubernetes 网络插件对 Pod 进行隔离，其实是靠在宿主机上生成 NetworkPolicy 对应的 iptable 规则来实现的。


## 通过NodePort来访问service的话，client的源ip会被做SNAT

```yaml
          client
             \ ^
              \ \
               v \
   node 1 <--- node 2
    | ^   SNAT
    | |   --->
    v |
 endpoint
```

流程：

- 客户端发送数据包到 node2:nodePort
- node2 使用它自己的 IP 地址替换数据包的源 IP 地址（SNAT）
- node2 使用 pod IP 地址替换数据包的目的 IP 地址
- 数据包被路由到 node 1，然后交给 endpoint
- Pod 的回复被路由回 node2
- Pod 的回复被发送回给客户端

可以将 `service.spec.externalTrafficPolicy` 的值为 Local，请求就只会被代理到本地 endpoints 而不会被转发到其它节点。这样就保留了最初的源 IP 地址 **不会对访问NodePort的client ip做 SNAT了**。如果没有本地 endpoints，发送到这个节点的数据包将会被丢弃。