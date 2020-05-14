---
layout: post
title:  "Go 内存管理"
date:   2020-04-05 18:16:18 +0800
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

```shell
main.go:14:9: &smallStruct literal escapes to heap
```

再来看这段代码生成的汇编指令来详细的展示内存分配的过程, 执行下面

 ```shell
go tool compile "-m" main.go 

0x001d 00029 (main.go:14)   LEAQ   type."".smallStruct(SB), AX
0x0024 00036 (main.go:14)  PCDATA $0, $0
0x0024 00036 (main.go:14)  MOVQ   AX, (SP)
0x0028 00040 (main.go:14)  CALL   runtime.newobject(SB)
```

`runtime.newobject` 是 `Go` 内置的申请堆内存的函数，对于堆内存的分配，`Go` 中有两种策略: **大内存的分配和小内存的分配**

### 小内存的分配

对于小于 `32kb`的小内存，`Go` 会尝试在 `P` 的 本地缓存 `mcache` 中分配, `mcache` 保存的是各种大小的Span，并按Span class分类，小对象直接从mcache分配内存，它起到了缓存的作用，并且可以 **无锁访问**

![](/images/neicun2.png)

每个 `M` 绑定一个 `P` 来运行一个`goroutine`, 在分配内存时，当前的 `goroutine` 在当前 `P` 的本地缓存 `mcache` 中查找对应的span， 从 `span list` 中来查找第一个可用的空闲 `span`

span class 分为 `8 bytes ~ 32k bytes` 共70多种类型，分别对应不同的内存大小

![](/images/neicun3.png)

![](/images/nc4.png)

![](/images/nc5.png)

![](/images/nc6.png)

![](/images/nc7.png)

![](/images/nc8.png)

## 总结

go 内存分配的概览

![](/images/nc9.png)


[Go内存分配那些事，就这么简单！]:https://lessisbetter.site/2019/07/06/go-memory-allocation