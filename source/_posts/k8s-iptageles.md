---
layout: post
title:  "k8s iptables"
date:   2020-03-28 13:19:10 +0800
categories: k8s
tags:  ["k8s", "iptables"]
author: zhaojizhuang

---


# k8s svc之 iptables规则


## iptables规则

> 参考 [http://www.zsythink.net/archives/tag/iptables/](http://www.zsythink.net/archives/tag/iptables/)

![](http://www.zsythink.net/wp-content/uploads/2017/02/021217_0051_2.png)

**用户空间，例如从pod中流出的流量就是从ouput链流出**

上图表示的`iptables`的链，链 和表的关系如下，以`PREROUTING`链为例

![](http://www.zsythink.net/wp-content/uploads/2017/02/021217_0051_4.png)

这幅图是什么意思呢？它的意思是说，prerouting"链"只拥有nat表、raw表和mangle表所对应的功能，所以，prerouting中的规则只能存放于nat表、raw表和mangle表中。

## NAT 

**NAT的三种类型:**

- SNAT  
```shell
iptables -t nat -A POSTROUTING -s 10.8.0.0/255.255.255.0 -o eth0 -j SNAT --to-source192.168.5.3
# 目标流向eth0，源地址是xxx的，做SNAT，源地址改为xxx
```
- DNAT
```shell
iptables-t nat -A POSTROUTING -s 10.8.0.0/255.255.255.0 -o eth0 -j SNAT --to-source192.168.5.3-192.168.5.5
```
- MASQUERADE 是SNAT的一种，可以自动获取网卡的ip来做SNAT，如果是ADSL这种动态ip的，如果用SNAT需要经常更改iptables规则 
```shell
iptables-t nat -A POSTROUTING -s 10.8.0.0/255.255.255.0 -o eth0 -j MASQUERADE
# 源地址是xxx，流向eth0的，流向做自动化SNAT
```
*masquerade* 应为英文伪装

## iptabels 常用命令

```shell
iptables [-t 表名] 管理选项 [链名] [匹配条件] [-j 控制类型]
# 控制类型包括 ACCETP REJECT DROP LOG 还有自定义的链（k8s的链）等
iptabels -t nat（表名） -nvL POSTROUTING(链的名字)
```
https://www.jianshu.com/p/ee4ee15d3658

## 分析k8s下的iptables规则

以如下 `service` 为例 

```yaml
Name:                     testapi-smzdm-com
Namespace:                zhongce-v2-0
Labels:                   <none>
Selector:                 zdm-app-owner=testapi-smzdm-com
Type:                     LoadBalancer
IP:                       172.17.185.22
LoadBalancer Ingress:     10.42.162.216
Port:                     <unset>  809/TCP
TargetPort:               809/TCP
NodePort:                 <unset>  39746/TCP
Endpoints:                10.42.147.255:809,10.42.38.222:809
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

即 `cluster ip` 为 `172.17.185.22` 后端 `podip` 为 `10.42.147.25510.42.38.222`
此外还有1个 `loadbalancer ip` `10.42.162.216`

### svc的访问路径

    - 集群内部，通过 `clusterip` 到访问到后端 `pod
    - 集群外部，通过直接访问`nodeport`；或者通过 `elb` 负载均衡到 `node` 上再通过 `nodeport` 访问

### cluster ip 的基本原理

如果是集群内的应用访问 cluster ip，那就是从**用户空间**访问**内核空间网络协议栈**,走的是 `OUTPUT` 链

1. 从`OUTPUT` 链开始

```shell
[root@10-42-8-102 ~]# iptables -t nat -nvL OUTPUT
Chain OUTPUT (policy ACCEPT 4 packets, 240 bytes)
 pkts bytes target     prot opt in     out     source               destination         
3424K  209M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

```
2. `OUTPUT`下的规则 直接把流量交给 `KUBE-SERVICES` 链

```shell
[root@10-42-8-102 ~]# iptables -t nat -nvL KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination     
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      0.0.0.0/0        172.17.185.22        /* zhongce-v2-0/testapi-smzdm-com: cluster IP */ tcp dpt:809
    0     0 KUBE-SVC-G3OM5DSD2HHDMN6U  tcp  --  *      *       0.0.0.0/0            172.17.185.22        /* zhongce-v2-0/testapi-smzdm-com: cluster IP */ tcp dpt:809
   10   520 KUBE-FW-G3OM5DSD2HHDMN6U  tcp  --  *      *       0.0.0.0/0            10.42.162.216        /* zhongce-v2-0/testapi-smzdm-com: loadbalancer IP */ tcp dpt:809
     0     0 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTY
```
上述3条规则是顺序执行的：

- 第1条规则匹配发往 `Cluster IP` `172.17.185.22` 的流量，跳转到了 `KUBE-MARK-MASQ` 链进一步处理，其作用就是打了一个 `MARK` ，稍后展开说明。
- 第2条规则匹配发往 `Cluster IP` `172.17.185.22` 的流量，跳转到了 `KUBE-SVC-G3OM5DSD2HHDMN6U` 链进一步处理，稍后展开说明。
- 第3条规则匹配发往集群外 `LB IP` 的 `10.42.162.216` 的流量，跳转到了
`KUBE-FW-G3OM5DSD2HHDMN6U` 链进一步处理，稍后展开说明。
- 第4条  KUBE-NODEPORTS的规则在末尾，只要dst ip是node 本机ip的话 （（–dst-type LOCAL），就跳转到KUBE-NODEPORTS做进一步判定：）
    **第2条规则要做dnat转发到后端具体的后端pod上**

```shell
[root@10-42-8-102 ~]# iptables -t nat -nvL KUBE-SVC-G3OM5DSD2HHDMN6U
Chain KUBE-SVC-G3OM5DSD2HHDMN6U (3 references)
 pkts bytes target     prot opt in     out     source               destination         
   18   936 KUBE-SEP-JT2KW6YUTVPLLGV6  all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.50000000000
   21  1092 KUBE-SEP-VETLC6CJY2HOK3EL  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

**两条 对应 后端pod的链**

```shell
[root@10-42-8-102 ~]# iptables -t nat -nvL KUBE-SEP-JT2KW6YUTVPLLGV6
Chain KUBE-SEP-JT2KW6YUTVPLLGV6 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.42.147.255        0.0.0.0/0           
   26  1352 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:10.42.147.255:809

[root@10-42-8-102 ~]# iptables -t nat -nvL KUBE-SEP-VETLC6CJY2HOK3EL
Chain KUBE-SEP-VETLC6CJY2HOK3EL (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.42.38.222         0.0.0.0/0           
    2   104 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:10.42.38.222:809
```

流量经过路由表从eth0出去，在流量流出本机之前会经过POSTROUTING 链

### 在流量离开本机的时候会经过 `POSTROUTING` 链

```shell
[root@10-42-8-102 ~]# iptables -t nat -nvL POSTROUTING
Chain POSTROUTING (policy ACCEPT 274 packets, 17340 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 632M   36G KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */

[root@10-42-8-102 ~]# iptables -t nat -nvL KUBE-POSTROUTING
Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination         
  526 27352 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ mark match 0x4000/0x4000
```

其实直接就跳转到了 `KUBE-POSTROUTING`，然后匹配打过`0x4000 MARK` 的流量，将其做 `SNAT` 转换，而这个 `MARK` 其实就是之前没说的 `KUBE-MARK-MASQ` 做的事情

```shell
[root@10-42-8-102 ~]# iptables -t nat -nvL KUBE-MARK-MASQ
Chain KUBE-MARK-MASQ (183 references)
 pkts bytes target     prot opt in     out     source               destination         
  492 25604 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000
```
当流量离开本机时，src IP会被修改为node的IP，而不是发出流量的POD IP了

### 通过loadbalance ip进行访问

最后还有一个KUBE-FW-G3OM5DSD2HHDMN6U链没有讲，从本机发往LB IP的流量要做啥事情呢？

**其实也是让流量直接发往具体某个Endpoints，就别真的发往LB了，这样才能获得最佳的延迟**：
```shell
[root@10-42-8-102 ~]# iptables -t nat -nvL KUBE-FW-G3OM5DSD2HHDMN6U
Chain KUBE-FW-G3OM5DSD2HHDMN6U (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    2   104 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* zhongce-v2-0/testapi-smzdm-com: loadbalancer IP */
    2   104 KUBE-SVC-G3OM5DSD2HHDMN6U  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* zhongce-v2-0/testapi-smzdm-com: loadbalancer IP */
    0     0 KUBE-MARK-DROP  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* zhongce-v2-0/testapi-smzdm-com: loadbalancer IP */
```

### 通过nodeport 来访问

回顾一下 KUBE_SERVICES规则

```shell
[root@10-42-8-102 ~]# iptables -t nat -nvL KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination     
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !172.17.0.0/16        172.17.185.22        /* zhongce-v2-0/testapi-smzdm-com: cluster IP */ tcp dpt:809
    0     0 KUBE-SVC-G3OM5DSD2HHDMN6U  tcp  --  *      *       0.0.0.0/0            172.17.185.22        /* zhongce-v2-0/testapi-smzdm-com: cluster IP */ tcp dpt:809
   10   520 KUBE-FW-G3OM5DSD2HHDMN6U  tcp  --  *      *       0.0.0.0/0            10.42.162.216        /* zhongce-v2-0/testapi-smzdm-com: loadbalancer IP */ tcp dpt:809
    0     0 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```
**`KUBE-NODEPORTS` 是最后一条规则**

```shell
[root@10-42-8-102 ~]# iptables -t nat -nvL KUBE-NODEPORTS
Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination        
    0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* zhongce-v2-0/testapi-smzdm-com: */ tcp dpt:39746
    0     0 KUBE-SVC-G3OM5DSD2HHDMN6U  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* zhongce-v2-0/testapi-smzdm-com: */ tcp dpt:39746
```
第1条匹配dst port如果是39746，那么就打mark。
第2条匹配dst port如果是39746，那么就跳到负载均衡链做DNAT改写。


### 总结

- `KUBE-SERVICES` 链的规则存在于 `OUTPUT POSTROUTING PREROUTING` 三个链上
- 对于 `KUBE-SERVICES KUBE-NDOEPORTS-xxx KUBE-SEP-xxx` 下都会对符合条件（匹配条件）的规则打上MARK 可以重复打MARK
- 在流量出node的时候做SNAT


## 从集群内出去的流量怎么回来

出node的流量在做SNAT的时候，netfilter有个连接跟踪机制，保存在 conntrack记录中

这就是Netfilter的连接跟踪（conntrack）功能了。对于TCP协议来讲，肯定是上来先建立一个连接，可以用`源/目的IP+源/目的端口`  （四元组），唯一标识一条连接，这个连接会放在conntrack表里面。
当时是这台机器去请求163网站的，虽然源地址已经Snat成公网IP地址了，但是 `conntrack` 表里面还是有这个连接的记录的。当163网站返回数据的时候，会找到记录，从而找到正确的私网IP地
址。

## 参考文档
[k8s 的iptales规则详解](https://yuerblog.cc/2019/12/09/k8s-%E6%89%8B%E6%8A%8A%E6%89%8B%E5%88%86%E6%9E%90service%E7%94%9F%E6%88%90%E7%9A%84iptables%E8%A7%84%E5%88%99/)

