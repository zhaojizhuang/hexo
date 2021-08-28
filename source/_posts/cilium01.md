---
layout: post
title:  "【理解 Cilium 系列文章】(一) 初识 Cilium"
date:   2021-08-17 20:16:18 +0800
categories: cilium
tags:  ["cilium"]
author: zhaojizhuang
---




> `Cilium` 作为近两年最火的云原生网络方案，可谓是风头无两。作为第一个通过 ebpf 实现了 `kube-proxy` 所有功能的网络插件，它的神秘面纱究竟是怎样的呢？本系列文章将带大家一起来慢慢揭晓

> 作为`《理解 Cilium 系列文章》`的第一篇，本文主要介绍 `Cilium` 的发展，相关功能以及使用，深入理解及底层原理将在后续文章中继续介绍

### 背景

随着云原生的普及率越来越高，各大厂商基本上或多或少都实现了业务的 k8s 容器化，头部云计算厂商更是不用说。 

而且随着 k8s 的 普及，当前集群逐渐呈现出以下两个特点：

1. **容器数量越来越多**，比如：k8s 官方单集群就已经支持 `15000 pod`
2. **`Pod` 生命周期越来越短**，`Serverless` 场景下甚至短至几分钟，几秒钟

随着容器密度的增大，以及生命周期的变短，对原生容器网络带来的挑战也越来越大

### 当前 k8s Service 负载均衡的实现现状

在 `Cilium` 出现之前， `Service` 由 `kube-proxy` 来实现，实现方式有 `userspace`，`iptables`，`ipvs` 三种模式。

#### Userspace

当前模式下，`kube-proxy` 作为反向代理,监听随机端口，通过 `iptables` 规则将流量重定向到代理端口，再由 `kube-proxy` 将流量转发到 后端 `pod`。`Service` 的请求会先从用户空间进入内核 `iptables`，然后再回到用户空间，代价较大，性能较差。

#### Iptables

存在的问题：

1.可扩展性差。随着 `service` 数据达到数千个，其控制面和数据面的性能都会急剧下降。原因在于 iptables 控制面的接口设计中，每添加一条规则，需要遍历和修改所有的规则，其控制面性能是`O(n²)`。在数据面，规则是用链表组织的，其性能是`O(n)`

2.LB 调度算法仅支持随机转发

#### Ipvs 模式

`IPVS` 是专门为 `LB` 设计的。它用 `hash table` 管理 `service`，对`service` 的增删查找都是`O(1)`的时间复杂度。不过 `IPVS` 内核模块没有 `SNAT` 功能，因此借用了 `iptables` 的 `SNAT` 功能。

`IPVS` 针对报文做 `DNAT` 后，将连接信息保存在 `nf_conntrack` 中，`iptables` 据此接力做 `SNAT`。该模式是目前 `Kubernetes` 网络性能最好的选择。但是由于 `nf_conntrack` 的复杂性，带来了很大的性能损耗。腾讯针对该问题做过相应的优化[【绕过conntrack，使用eBPF增强 IPVS优化K8s网络性能】](https://mp.weixin.qq.com/s/yLt7NZQeVwQEF6xJruOsDg)

### Cilium 的发展

`Cilium` 是基于 `eBpf` 的一种开源网络实现，通过在 `Linux` 内核动态插入强大的安全性、可见性和网络控制逻辑，提供网络互通，服务负载均衡，安全和可观测性等解决方案。简单来说可以理解为 **Kube-proxy + CNI 网络实现。**

`Cilium` 位于容器编排系统和 `Linux Kernel` 之间，向上可以通过编排平台为容器进行网络以及相应的安全配置，向下可以通过在 `Linux` 内核挂载 `eBPF` 程序，来控制容器网络的转发行为以及安全策略执行

![](https://gblobscdn.gitbook.com/assets%2F-MRFeg2_Fce5b_HxlfVo%2F-MhI1PKBbT6AggztF2Io%2F-MhI5yJjJ58bKXUgL-e9%2Fimage.png?alt=media&token=997ca893-7f8d-49e1-aeda-4653b5e02cbc)

简单了解下 `Cilium` 的发展历程：

1.  2016 Thomas Graf 创立了 Cilium, 现为 Isovalent \(Cilium 背后的商业公司\)的 CTO
2.  2017 年 DockerCon 上 Cilium 第一次发布
3.  2018 年 发布 Cilium 1.0 
4.  2019 年 发布 Cilium 1.6 版本，100% 替代 kube-proxy
5.  2019 年 Google 全面参与 Cilium 
6.  2021 年 微软、谷歌、FaceBook、Netflix、Isovalent 在内的多家企业宣布成立 eBPF 基金会\(Linux 基金会下\)

### 功能介绍

![](https://gblobscdn.gitbook.com/assets%2F-MRFeg2_Fce5b_HxlfVo%2F-MhI1PKBbT6AggztF2Io%2F-MhI6Yc-RTsTKcR17oj9%2Fimage.png?alt=media&token=a018a444-5db6-45cb-bc07-44efeac6cb76)

查看官网，可以看到 `Cilium` 的功能主要包含 三个方面，如上图

一、网络

1. 高度可扩展的 `kubernetes` `CNI` 插件，支持大规模，高动态的 `k8s` 集群环境。支持多种租网模式: 
   1. `Overlay` 模式，支持 `Vxlan` 及 `Geneve`
   2. `Unerlay` 模式，通过 `Direct Routing` （直接路由）的方式，通过 `Linux` 宿主机的路由表进行转发
2. `kube-proxy` 替代品，实现了 四层负载均衡功能。`LB` 基于 `eBPF` 实现，使用高效的、可无限扩容的哈希表来存储信息。对于南北向负载均衡，`Cilium` 作了最大化性能的优化。支持 `XDP、DSR（Direct Server Return，LB` 仅仅修改转发封包的目标 `MAC` 地址）
3. 多集群的连通性，[Cilium Cluster Mesh](https://cilium.io/blog/2019/03/12/clustermesh/) 支持多集群间的负载，可观测性以及安全管控


二、可观测性



![](https://gblobscdn.gitbook.com/assets%2F-MRFeg2_Fce5b_HxlfVo%2F-MhJXGud22jg_jkCC-7v%2F-MhJZwB6pLY9MtaTuyR4%2Fimage.png?alt=media&token=4bc57ed3-4e63-4c3c-b526-c8198bca0ceb)

1. 提供生产可用的可观测性工具 `hubble`, 通过 `pod` 及 `dns` 标识来识别连接信息
2. 提供 `L3/L4/L7` 级别的监控指标，以及 `Networkpolicy` 的 行为信息指标
3. API 层面的可观测性 (http,https)
4. `Hubble` 除了自身的监控工具，还可以对接像 `Prometheus、Grafana` 等主流的云原生监控体系，实现可扩展的监控策略

三、安全

1. 不仅支持 k8s `Network Policy`，还支持 `DNS` 级别、`API` 级别、以及跨集群级别的 `Network Policy`
2. 支持 `ip` 端口 的 安全审计日志
3. 传输加密

总结，**Cilium 不仅包括了 kube-proxy + CNI 网络实现，还包含了众多可观测性和安全方面的特性。**

### 安装部署

> linux 内核要求 `4.19` 及以上

可以采用 `helm` 或者 `cilium cli`，此处笔者使用的是 `cilium cli`（版本为 `1.10.3`）

1. 下载 `cilium cli`

```text
wget https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
```

2. 安装 `cilium`

```bash
cilium install --kube-proxy-replacement=strict # 此处选择的是完全替换，默认情况下是 probe，(该选项下 pod hostport 特性不支持)
```

3. 可视化组件 hubble(选装)

```bash
cilium hubble enable --ui
```

4. 等待 pod ready 后，查看 状态如下：

```bash
~# cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         OK
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/

DaemonSet         cilium             Desired: 1, Ready: 1/1, Available: 1/1
Deployment        cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
Deployment        hubble-relay       Desired: 1, Ready: 1/1, Available: 1/1
Containers:       hubble-relay       Running: 1
                  cilium             Running: 1
                  cilium-operator    Running: 1
Image versions    cilium             quay.io/cilium/cilium:v1.10.3: 1
                  cilium-operator    quay.io/cilium/operator-generic:v1.10.3: 1
                  hubble-relay       quay.io/cilium/hubble-relay:v1.10.3: 1
```

5. `cilium cli` 还支持 集群可用性检查（可选）

```bash
[root@~]# cilium connectivity test
ℹ️  Single-node environment detected, enabling single-node connectivity test
ℹ️  Monitor aggregation detected, will skip some flow validation steps
✨ [kubernetes] Creating namespace for connectivity check...
✨ [kubernetes] Deploying echo-same-node service...
✨ [kubernetes] Deploying same-node deployment...
✨ [kubernetes] Deploying client deployment...
✨ [kubernetes] Deploying client2 deployment...
⌛ [kubernetes] Waiting for deployments [client client2 echo-same-node] to become ready...
⌛ [kubernetes] Waiting for deployments [] to become ready...
⌛ [kubernetes] Waiting for CiliumEndpoint for pod cilium-test/client-6488dcf5d4-rx8kh to appear...
⌛ [kubernetes] Waiting for CiliumEndpoint for pod cilium-test/client2-65f446d77c-97vjs to appear...
⌛ [kubernetes] Waiting for CiliumEndpoint for pod cilium-test/echo-same-node-745bd5c77-gr2p6 to appear...
⌛ [kubernetes] Waiting for Service cilium-test/echo-same-node to become ready...
⌛ [kubernetes] Waiting for NodePort 10.251.247.131:31032 (cilium-test/echo-same-node) to become ready...
⌛ [kubernetes] Waiting for Cilium pod kube-system/cilium-vsk8j to have all the pod IPs in eBPF ipcache...
⌛ [kubernetes] Waiting for pod cilium-test/client-6488dcf5d4-rx8kh to reach default/kubernetes service...
⌛ [kubernetes] Waiting for pod cilium-test/client2-65f446d77c-97vjs to reach default/kubernetes service...
ð­ Enabling Hubble telescope...
⚠️  Unable to contact Hubble Relay, disabling Hubble telescope and flow validation: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial tcp [::1]:4245: connect: connection refused"
ℹ️  Expose Relay locally with:
   cilium hubble enable
   cilium status --wait
   cilium hubble port-forward&
ð Running tests...
```

等 `hubble` 安装完成后，`hubble-ui` `service` 修改为 `NodePort`类型, 即可通过 `NodeIP+NodePort` 来登录 `Hubble` 界面 查看相关信息

![](https://gblobscdn.gitbook.com/assets%2F-MRFeg2_Fce5b_HxlfVo%2F-MgnDBD5NFpGEKRXAU0A%2F-MgoOiBbBc3ptDsFPJuI%2Fimage.png?alt=media&token=2a08b60c-a353-4f3e-bee5-d5cd84015981)


`Cilium` 部署完后，有以下几个组件 `operator`、`hubble(ui, relay)`,`Cilium agent`（`Daemonset` 形式，每个节点一个），其中关键组件为 `cilium agent`。

![](https://gblobscdn.gitbook.com/assets%2F-MRFeg2_Fce5b_HxlfVo%2F-MhIBfaeG36XhjlMq_Bh%2F-MhIDvXCmmHTeemwAY42%2Fimage.png?alt=media&token=20546b16-d69d-4fbb-a3e1-a71b35a2a088)

Cilium Agent作为整个架构中最核心的组件，通过DaemonSet的方式，以特权容器的模式，运行在集群的每个主机上。Cilium Agent作为用户空间守护程序，通过插件与容器运行时和容器编排系统进行交互，进而为本机上的容器进行网络以及安全的相关配置。同时提供了开放的API，供其他组件进行调用。

Cilium Agent在进行网络和安全的相关配置时，采用eBPF程序进行实现。Cilium Agent结合容器标识和相关的策略，生成 eBPF 程序，并将 eBPF 程序编译为字节码，将它们传递到 Linux 内核。



### 相关命令介绍

`Cilium agent` 中内置了一些调试用的命令，下面介绍，`agent`  中的 `cilium` 不同与上述介绍的 `cilium cli` ( 虽然同为 `cilium`)

1. `cilium status` 

主要展示  `cilium` 的一些简单配置信息及状态，如下

```bash
[root@~]# kubectl exec -n kube-system cilium-s62h5 -- cilium status
Defaulted container "cilium-agent" out of: cilium-agent, ebpf-mount (init), clean-cilium-state (init)
KVStore:                Ok   Disabled
Kubernetes:             Ok   1.21 (v1.21.2) [linux/amd64]
Kubernetes APIs:        ["cilium/v2::CiliumClusterwideNetworkPolicy", "cilium/v2::CiliumEndpoint", "cilium/v2::CiliumNetworkPolicy", "cilium/v2::CiliumNode", "core/v1::Namespace", "core/v1::Node", "core/v1::Pods", "core/v1::Service", "discovery/v1::EndpointSlice", "networking.k8s.io/v1::NetworkPolicy"]
KubeProxyReplacement:   Strict   [eth0 10.251.247.131 (Direct Routing)]
Cilium:                 Ok   1.10.3 (v1.10.3-4145278)
NodeMonitor:            Listening for events on 8 CPUs with 64x4096 of shared memory
Cilium health daemon:   Ok
IPAM:                   IPv4: 68/254 allocated from 10.0.0.0/24,
BandwidthManager:       Disabled
Host Routing:           Legacy
Masquerading:           BPF   [eth0]   10.0.0.0/24 [IPv4: Enabled, IPv6: Disabled]
Controller Status:      346/346 healthy
Proxy Status:           OK, ip 10.0.0.167, 0 redirects active on ports 10000-20000
Hubble:                 Ok   Current/Max Flows: 4095/4095 (100.00%), Flows/s: 257.25   Metrics: Disabled
Encryption:             Disabled
Cluster health:         1/1 reachable   (2021-08-11T09:33:31Z)
```

2. `cilium service list`  

展示 `service` 的实现，使用时可通过 `ClusterIP` 来过滤，其中，`FrontEnd` 为 `ClusterIP`，`Backend` 为 `PodIP`

```bash
[root@~]# kubectl exec -it -n kube-system cilium-vsk8j -- cilium service list
Defaulted container "cilium-agent" out of: cilium-agent, ebpf-mount (init), clean-cilium-state (init)
ID    Frontend                 Service Type   Backend
1     10.111.192.31:80         ClusterIP      1 => 10.0.0.212:8888
2     10.101.111.124:8080      ClusterIP      1 => 10.0.0.81:8080
3     10.101.229.121:443       ClusterIP      1 => 10.0.0.24:8443
4     10.111.165.162:8080      ClusterIP      1 => 10.0.0.213:8080
5     10.96.43.229:4222        ClusterIP      1 => 10.0.0.210:4222
6     10.100.45.225:9180       ClusterIP      1 => 10.0.0.48:9180
# 避免过多，此处不一一展示
```

3. `cilium service get`  

通过 `cilium service get < ID> -o json` 来展示详情

```bash
[root@~]# kubectl exec -it -n kube-system cilium-vsk8j -- cilium service get 132 -o json
Defaulted container "cilium-agent" out of: cilium-agent, ebpf-mount (init), clean-cilium-state (init)
{
  "spec": {
    "backend-addresses": [
      {
        "ip": "10.0.0.213",
        "nodeName": "n251-247-131",
        "port": 8080
      }
    ],
    "flags": {
      "name": "autoscaler",
      "namespace": "knative-serving",
      "trafficPolicy": "Cluster",
      "type": "ClusterIP"
    },
    "frontend-address": {
      "ip": "10.98.24.168",
      "port": 8080,
      "scope": "external"
    },
    "id": 132
  },
  "status": {
    "realized": {
      "backend-addresses": [
        {
          "ip": "10.0.0.213",
          "nodeName": "n251-247-131",
          "port": 8080
        }
      ],
      "flags": {
        "name": "autoscaler",
        "namespace": "knative-serving",
        "trafficPolicy": "Cluster",
        "type": "ClusterIP"
      },
      "frontend-address": {
        "ip": "10.98.24.168",
        "port": 8080,
        "scope": "external"
      },
      "id": 132
    }
  }
}
```

还有很多有用的命令，限于篇幅，此处不一一展示，留给读者自己去探索（`cilium status --help`）

`《理解 Cilium 系列文章》`的第一篇到此就讲完了，后续文章将按照 `Cilium` 涉及的基础知识及相关原理进行介绍，敬请期待

