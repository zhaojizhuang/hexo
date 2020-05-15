---
layout: post
title:  "Go 内存管理"
date:   2019-11-12 18:16:18 +0800
categories: Go
tags: ["Go","内存管理"]
author: zhaojizhuang
---



 Go这门语言抛弃了C/C++中的开发者管理内存的方式：主动申请与主动释放，增加了**逃逸分析和GC**，将开发者从内存管理中释放出来，让开发者有更多的精力去关注软件设计，而不是底层的内存问题。这是Go语言成为高生产力语言的原因之一  引自【 [Go内存分配那些事，就这么简单！]】

## 堆内存的分配


先看下面这段代码，思考下 `smallStruct` 会被分配在堆上还是栈上:

```go
package main

type smallStruct struct {
   a, b int64
   c, d float64
}

func main() {
   smallAllocation()
}

//go:noinline
func smallAllocation() *smallStruct {
   return &smallStruct{}
}
```

通过 `annotation //go:noinline` 禁用内联函数，不然这里不会产生堆内存的分配 **【逃逸分析】**

**Inline 内联**: 是在编译期间发生的，将函数调用调用处替换为被调用函数主体的一种编译器优化手段。

将文件保存为 `main.go`, 并执行 `go tool compile "-m" main.go` ,查看Go 堆内存的分配 **【逃逸分析】**的过程

如果不加 `annotation //go:noinline` 可以用 
`go build -gcflags '-m -l' main.go` `-l`可以禁止内联函数，效果是一样的，下面是逃逸分析的结果：

```shell
main.go:14:9: &smallStruct literal escapes to heap
```

再来看这段代码生成的汇编指令来详细的展示内存分配的过程, 执行下面

 ```shell
go tool compile -S  main.go 

0x001d 00029 (main.go:14)   LEAQ   type."".smallStruct(SB), AX
0x0024 00036 (main.go:14)  PCDATA $0, $0
0x0024 00036 (main.go:14)  MOVQ   AX, (SP)
0x0028 00040 (main.go:14)  CALL   runtime.newobject(SB)
```

`runtime.newobject` 是 `Go` 内置的申请堆内存的函数，对于堆内存的分配，`Go` 中有两种策略: **大内存的分配和小内存的分配**

### 小内存的分配

#### 从 P 的 `mcache` 中分配

对于小于 `32kb`的小内存，`Go` 会尝试在 `P` 的 本地缓存 `mcache` 中分配, `mcache` 保存的是各种大小的Span，并按Span class分类，小对象直接从mcache分配内存，它起到了缓存的作用，并且可以 **无锁访问**

![](/images/neicun2.png)

每个 `M` 绑定一个 `P` 来运行一个`goroutine`, 在分配内存时，当前的 `goroutine` 在当前 `P` 的本地缓存 `mcache` 中查找对应的span， 从 `span list` 中来查找第一个可用的空闲 `span`

`span class` 分为 `8 bytes ~ 32k bytes` 共 66 种类型（还有个大小为0的 size class0，并未用到，用于大对象的堆内存分配），分别对应不同的内存大小,`mspan`里保存对应大小的object，

1个 `size class` 对应2个 `span class`，2个 `span class` 的 `span` 大小相同，只是功能不同，1个用来存放包含指针的对象，一个用来存放不包含指针的对象，不包含指针对象的 `Span` 就无需 `GC` 扫描了。

前面的例子里，struct的大小为 `(64bit/8)*4=8bit*4=32b` 所以会在 `span class` 大小为 `32bytes` 的 `mspan` 里分配

![](/images/neicun3.png)

#### 从全局的缓存  `mcentral` 中分配

当 `mcache`中没有空闲的 `span` 时怎么办呢，Go 还维护了一个全局的缓存  `mcentral`,

`mcentral` 和 `mcache` 一样，都134个 `span class` 级别(67个 `size class` )，但每个级别都保存了2个span list，即2个span链表：

- nonempty：这个链表里的span，所有span都**至少有1个空闲的对象空间**。这些span是mcache释放span时加入到该链表的。
- empty：这个链表里的span，所有的span都不确定里面是否有空闲的对象空间。当一个span交给mcache的时候，就会加入到empty链表。

**mcache从mcentral获取和归还mspan的流程**：引自【 [图解Go语言内存分配|码农桃花源]】

- 获取：加锁；从 `nonempty` 链表找到一个可用的 `mspan` ；并将其从 `nonempty` 链表删除；将取出的 `mspan` 加入到 `empty` 链表；将 `mspan` 返回给工作线程；解锁。
- 归还：加锁；将 `mspan` 从 `empty` 链表删除；将 `mspan` 加入到 `nonempty` 链表；解锁。
- 
另外，GC 扫描的时候会把部分 `mspan` 标记为未使用，并将对应的 `mspan` 加入到 `nonempty list` 中

![](/images/nc4.png)

`mcache` 从 `mcentral`获取 过程如下：

![](/images/nc5.png)

#### `mcentral ` 从 `heap` 中分配

当 `mcentral` 中的 `nonempty list` 没有可分配的对象的时候，Go会从 `mheap` 中分配对象，并链接到 `nonempty list` 上,  `mheap` 必要时会向系统申请内存

![](/images/nc6.png)

`mheap` 中还有 `arenas` ,主要是为了大块内存需要，`arena` 也是用 `mspan` 组织的

![](/images/nc7.png)

### 大内存的分配

大内存的分配就比较简单了，大于 `32kb` 的内存都会在 `mheap` 中直接分配

![](/images/nc8.png)


## 总结

go 内存分配的概览

![](/images/nc9.png)


----

## 参考文章

- Go内存分配那些事，就这么简单！:[https://lessisbetter.site/2019/07/06/go-memory-allocation](https://lessisbetter.site/2019/07/06/go-memory-allocation)
- 图解Go语言内存分配|码农桃花源:[https://qcrao.com/2019/03/13/graphic-go-memory-allocation](https://qcrao.com/2019/03/13/graphic-go-memory-allocation)
- 聊一聊goroutine stack：[https://zhuanlan.zhihu.com/p/28409657](https://zhuanlan.zhihu.com/p/28409657)


[Go内存分配那些事，就这么简单！]:https://lessisbetter.site/2019/07/06/go-memory-allocation

[图解Go语言内存分配|码农桃花源]:https://qcrao.com/2019/03/13/graphic-go-memory-allocation



