---
layout: post
title:  "自行配置你的HPA扩缩容速率"
date:   2020-05-19 18:16:18 +0800
categories: k8s
tags:  ["k8s"]
author: zhaojizhuang
---


## HPA介绍

HPA, Pod 水平自动伸缩（Horizontal Pod Autoscaler）特性， 可以基于CPU利用率自动伸缩 replication controller、deployment和 replica set 中的 pod 数量，（除了 CPU 利用率）也可以 基于其他应程序提供的度量指标custom metrics。 pod 自动缩放不适用于无法缩放的对象，比如 DaemonSets。

Pod 水平自动伸缩特性由 Kubernetes API 资源和控制器实现。资源决定了控制器的行为。 控制器会周期性的获取平均 CPU 利用率，并与目标值相比较后来调整 replication controller 或 deployment 中的副本数量。

> [官网文档](https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/#pod-%e6%b0%b4%e5%b9%b3%e8%87%aa%e5%8a%a8%e4%bc%b8%e7%bc%a9%e5%b7%a5%e4%bd%9c%e6%9c%ba%e5%88%b6) 

**HPA 工作机制**

关于 HPA 的原理可以看下 baxiaoshi的云原生学习笔记 [https://www.yuque.com/baxiaoshi/tyado3/yw9deb](https://www.yuque.com/baxiaoshi/tyado3/yw9deb),本文不做过多介绍，只介绍自行配置hpa特性

`HPA` 是由 `hpacontroller` 来实现的, 通过  `--horizontal-pod-autoscaler-sync-period` 参数 指定周期（默认值为15秒）


一个HPA的例子如下：

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
      current:
        averageUtilization: 0
        averageValue: 0

```


## 背景

前面已经介绍了 `HPA` 的相关信息，相信大家都对 `HPA` 有了简单的了解, 当使用 `Horizontal Pod Autoscaler` 管理一组副本缩放时， **有可能因为指标动态的变化造成副本数量频繁的变化，有时这被称为 抖动**。为了避免这种情况，社区引入了 延迟时间的设置，*该配置是针对整个集群的配置，不能对应用粒度进行配置*

`--horizontal-pod-autoscaler-downscale-stabilization: ` 这个 `kube-controller-manager` 的参数表示缩容冷却时间。 即自从上次缩容执行结束后，多久可以再次执行缩容，默认时间是5分钟(5m0s)。`

注意：这个配置是集群级别的，只能配置整个集群的扩缩容速率，用户不能自行配置自己的应用扩缩容速率：

考虑一下场景的应用：

- 对于大流量的的web应用。需要非常快的扩容速率，缓慢的缩容速率（为了迎接下一个流量高峰）
- 对于处理数据类的应用。需要尽快的扩容（减少处理数据的时间），并尽快缩容（降低成本）
- 对于处理常规流量和数据的应用，按照常规的方式配置就可以了

对于上述3种应用场景，1.17以前的集群是不能支持，这也导致了一些未解决的 `issue`：
- [https://github.com/kubernetes/kubernetes/issues/39090](https://github.com/kubernetes/kubernetes/issues/39090)
- [https://github.com/kubernetes/kubernetes/issues/65097](https://github.com/kubernetes/kubernetes/issues/65097)
- [https://github.com/kubernetes/kubernetes/issues/69428](https://github.com/kubernetes/kubernetes/issues/69428)

为此社区 在`1.18`引入了 能自行配置`HPA`速率的新特性 见 相关 [keps](https://github.com/kubernetes/enhancements/blob/master/keps/sig-autoscaling/20190307-configurable-scale-velocity-for-hpa.md)

## 原理介绍

为了自定义扩缩容行为，需要给单个 `hpa` 对象添加一个 `behavior`字段，如下：

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  ***
  behavior:
      scaleUp:
        policies:
        - type: percent
          value: 900%
      scaleDown:
        policies:
        - type: pods
          value: 1
          periodSeconds: 600 # (i.e., scale down one pod every 10 min)
  metrics:
    ***
```

其中 `behavior` 字段 包含 如下对象：

- `scaleUp` 配置扩容时的规则。
    - `stabilizationWindowSeconds`： -该值表示 `HPA` 控制器应参考的采样时间，以防止副本数量波动。
    - `selectPolicy`：可以是`min`或`max`,用于选择 `policies` 中的最大值还是最小值。默认为 `max`。
    - `policies`： 是一个数组，每个元素有如下字段：
        - `type`： 可为`pods`或者`percent`（以Pod的绝对数量或当前副本的百分比表示）。
        - `periodSeconds` 规则应用的时间范围（以秒为单位），即每多少秒扩缩容一次。
        - `value`: 规则的值，设为0 表示禁止 扩容或者缩容
- `scaleDown` 与 `scaleUp` 类似，指定的是缩容的规则。

用户通过控制 HPA中指定参数，来控制HPA扩缩容的逻辑

`selectPolicy` 字段指示应应用的策略。默认情况下为max，也就是：**将使用尽可能扩大副本的最大数量，而选择尽可能减少副本的最小数量。** 有点绕没关系，待会看下面的例子

## 场景

### 场景1：尽可能快的扩容

> 这种模式适合流量增速比较快的场景

HPA的配置如下：

```yaml
behavior:
  scaleUp:
    policies:
    - type: percent
      value: 900%
```
`900%` 表示可以扩容的数量为当前副本数量的9倍，也就是可以扩大到当前副本数量的10倍，其他值都是默认值

如果应用一个开始的pod数为1，那么扩容过程中 pod数量如下：

`1 -> 10 -> 100 -> 1000`

scaleDown没有配置，表示该应用缩容将按照常规方式进行 可参考对应的[扩缩容算法](https://v1-14.docs.kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/#%E7%AE%97%E6%B3%95%E7%BB%86%E8%8A%82) 

### 场景2：尽可能快的扩容，然后慢慢缩容

> 这种模式适合不想快速缩容的场景

HPA的配置如下：

```yaml
behavior:
  scaleUp:
    policies:
    - type: percent
      value: 900%
  scaleDown:
    policies:
    - type: pods
      value: 1
      periodSeconds: 600 # (i.e., 每隔10分钟缩容一个pod)
```
这种配置扩容场景同 场景1，缩容场景下却是不同的，这里配置的是每隔10分钟缩容一个pod

假如说扩容之后有1000个pod，那么缩容过程中pod数量如下：

`1000 -> 1000 -> 1000 -> … (7 more min) -> 999`

### 场景3：慢慢扩容，按常规缩容

> 这种模式适合不太激进的扩容

HPA的配置如下：

```yaml
behavior:
  scaleUp:
    policies:
    - type: pods
      value: 1
```

如果应用一个开始的pod数为1，那么扩容过程中 pod数量如下：

`1 -> 2 -> 3 -> 4`

同样，scaleDown没有配置，表示该应用缩容将按照常规方式进行 可参考对应的[扩缩容算法](https://v1-14.docs.kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/#%E7%AE%97%E6%B3%95%E7%BB%86%E8%8A%82) 

### 场景4： 常规扩容，禁止缩容

> 这种模式适用于 不允许应用缩容，或者你想单独控制应用的缩容的场景

HPA的配置如下：

```yaml
behavior:
  scaleDown:
    policies:
    - type: pods
      value: 0
```

副本数量将按照常规方式扩容，不会发生缩容

### 场景5：延迟缩容

> 这种模式适用于 用户并不想立即缩容，而是想等待更大的负载时间到来，再计算缩容情况

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 600
    policies:
    - type: pods
      value: 5
```
这种配置情况下 `hpa` 缩容策略行为如下：

- 会采集最近 `600s` 的（默认300s）的缩容建议，类似于滑动窗口
- 选择最大的那个
- 按照不大于每秒5个pod的速率缩容

假如 `CurReplicas = 10` , `HPA controller` 每 `1min` 处理一次:

- 前 9 min,算法只会收集扩缩容建议，而不会发生真正的扩缩容，假设 有如下 扩缩容建议：
    `recommendations = [10, 9, 8, 9, 9, 8, 9, 8, 9]`
- 第 10 min，我们增加一个扩缩容建议，比如说是 `8`
    `recommendations = [10, 9, 8, 9, 9, 8, 9, 8, 9，8]`
    HPA 算法会取其中最大的一个 `10`，因此应用不会发生缩容，`repicas` 的值不变
- 第 11 min，我们增加一个扩缩容建议，比如 `7`，维持 `600s` 的滑动窗口，因此需要把第一个 `10` 去掉，如下：
    `recommendations = [9, 8, 9, 9, 8, 9, 8, 9, 8, 7]`
    HPA 算法会取最大的一个 `9`, 应用副本数量变化： `10 -> 9`
    
### 场景6： 避免错误的扩容

> 这种模式在数据处理流中比较常见，用户想要根据队列中的数据来扩容，当数据较多时快速扩容。当指标有抖动时并不想扩容

HPA的配置如下：

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 300
    policies:
    - type: pods
      value: 20
```

这种配置情况下 `hpa` 扩容策略行为如下：

- 会采集最近 `300s` 的（默认0s）的扩容建议，类似于滑动窗口
- 选择最小的那个
- 按照不大于每秒20个pod的速率扩容

假如 `CurReplicas = 2` , `HPA controller` 每 `1min` 处理一次:

- 前 5 min,算法只会收集扩缩容建议，而不会发生真正的扩缩容，假设 有如下 扩缩容建议：
    `recommendations = [2, 3, 19, 10, 3]`
- 第 6 min，我们增加一个扩缩容建议，比如说是 `4`
    `recommendations = [2, 3, 19, 10, 3, 4]`
    HPA 算法会取其中最小的一个 `2`，因此应用不会发生缩容，`repicas` 的值不变
- 第 11 min，我们增加一个扩缩容建议，比如 `7`，维持 `300s` 的滑动窗口，因此需要把第一个 `2` 去掉，如下：
    `recommendations = [3, 19, 10, 3, 4，7]`
    HPA 算法会取最小的一个 `3`, 应用副本数量变化： `2 -> 3`

## 算法原理

算法的伪代码如下：

```go
//  HPA controller 中的for循环
for {
    desiredReplicas = AnyAlgorithmInHPAController(...)
    
    // 扩容场景
    if desiredReplicas > curReplicas {
      replicas = []int{}
      for _, policy := range behavior.ScaleUp.Policies {
        if policy.type == "pods" {
          replicas = append(replicas, CurReplicas + policy.Value)
        } else if policy.type == "percent" {
          replicas = append(replicas, CurReplicas * (1 + policy.Value/100))
        }
      }
      if behavior.ScaleUp.selectPolicy == "max" {
        scaleUpLimit = max(replicas)
      } else {
        scaleUpLimit = min(replicas)
      }
      // 这里的min可以理解为 尽量维持原状，即如果是扩容，尽量取最小的那个
      limitedReplicas = min(max, desiredReplicas)
    }
    // 缩容场景
    if desiredReplicas < curReplicas {
      for _, policy := range behaviro.scaleDown.Policies {
        replicas = []int{}
        if policy.type == "pods" {
          replicas = append(replicas, CurReplicas - policy.Value)
        } else if policy.type == "percent" {
          replicas = append(replicas, CurReplicas * (1 - policy.Value /100))
        }
        if behavior.ScaleDown.SelectPolicy == "max" {
          scaleDownLimit = min(replicas)
        } else {
          scaleDownLimit = max(replicas)
        }
        // 这里的max可以理解为 尽量维持原状，即如果是缩容，尽量取最大的那个
        limitedReplicas = max(min, desiredReplicas)
      }
    }
    storeRecommend(limitedReplicas, scaleRecommendations)
    // 选择合适的扩缩容副本
    nextReplicas := applyRecommendationIfNeeded(scaleRecommendations)
    // 扩缩容
    setReplicas(nextReplicas)
    sleep(ControllerSleepTime)
  }
```

## 默认值

为了平滑得扩速容，默认值是有必要的
`behavior` 的默认值如下：

- behavior.scaleDown.stabilizationWindowSeconds = 300, 缩容情况下默认等待 5 min中再缩容.
- behavior.scaleUp.stabilizationWindowSeconds = 0, 扩容时立即扩容，不等待
- behavior.scaleUp.policies 默认值如下：
    - 百分比策略
        - policy = `percent`
        - periodSeconds = `60`, 扩容间隔为 1min
        - value = `100` 每次最多扩容翻倍
    - Pod个数策略
        - policy = `pods`
        - periodSeconds = `60`, 扩容间隔为 1min
        - value = `4` 每次最多扩容4个
- behavior.scaleDown.policies 默认值如下：
    - 百分比策略
        - policy = `percent`
        - periodSeconds = `60` 缩容间隔为 1min
        - value = `100` 一次缩容最多可以把所有的示例都干掉