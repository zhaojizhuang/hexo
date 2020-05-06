---
title:  "Go 深入分析"
date:   2020-04-02 14:19:10 +0800
categories: Go
tags:  ["Go","epoll","linux"]
author: zhaojizhuang
mathjax: true
---


# Go深入分析


## go程序是如何运行的

[参考链接1](https://juejin.im/post/5d1c087af265da1bb5651356)

## defer 源码分析

[参考链接](https://eddycjy.com/posts/go/defer/2019-05-27-defer/)

defer、return、返回值三者的执行逻辑应该是：return最先执行，return负责将结果写入返回值中；接着defer开始执行一些收尾工作；最后函数携带当前返回值退出

## 逃逸分析 堆栈分配

[参考链接](https://eddycjy.com/posts/go/talk/2019-05-20-stack-heap/)

`go build -gcflags '-m -l' xxx.go` 就可以看到逃逸分析的过程和结果

## go性能大杀器 pprof

[参考链接1](https://zhuanlan.zhihu.com/p/71529062)
[参考链接2](https://github.com/eddycjy/blog/blob/master/content/posts/go/tools/2018-09-15-go-tool-pprof.md)

### 

- pprof中自带 web 火焰图，需要安装graphviz
`go tool pprof -http=:8181 xxx,pprof`

- 下面的语句 可以**结合代码查看哪个函数**用时最多
`go tool pprof main.go xxxx.prof  进入pprof后执行 list  <函数名> `

### 对于web开放的pprof （在http的go程序中 添加 `_ "net/http/pprof"`的import,会增加 debug/pprof 的endpoint),结束后将默认进入 pprof 的交互式命令模式

    ```shell
    go tool pprof http://localhost:6060/debug/pprof/profile?seconds=60
    go tool pprof http://localhost:6060/debug/pprof/heap
    ```
    
## go性能大杀器 trace

同pprof

- 对于web开放的pprof （在http的go程序中 添加 `_ "net/http/pprof"`的import

```shell    
curl http://127.0.0.1:6060/debug/pprof/trace\?seconds\=20 > trace.out
go tool trace trace.out # 此处和pprof不同，不用加 -http=:8181 这里他会自动选择端口
```

- 对于后台应用,后台程序main启动时添加 trace.Start(os.Stderr)直接运行下面的命令即可 

`go run main.go 2> trace.out`

它能够跟踪捕获各种执行中的事件，例如 Goroutine 的创建/阻塞/解除阻塞，Syscall 的进入/退出/阻止，GC 事件，Heap 的大小改变，Processor 启动/停止等等

## map原理分析

map 结构体

```go
type hmap struct {
    count     int    // 元素的个数
    flags     uint8  // 状态标志
    B         uint8  // 可以最多容纳 6.5 * 2 ^ B 个元素，6.5为装载因子
    noverflow uint16 // 溢出的个数
    hash0     uint32 // 哈希种子

    buckets    unsafe.Pointer // 桶的地址
    oldbuckets unsafe.Pointer // 旧桶的地址，用于扩容
    nevacuate  uintptr        // 搬迁进度，小于nevacuate的已经搬迁
    overflow *[2]*[]*bmap 
}
// A bucket for a Go map.
type bmap struct {
    // 每个元素hash值的高8位，如果tophash[0] < minTopHash，表示这个桶的搬迁状态
    tophash [bucketCnt]uint8
    // bucketCnt是常量8,接下来是8个key、8个value，但是我们不能直接看到；为了优化对齐，go采用了key放在一起，value放在一起的存储方式，
8个k，8个v得内存地址
    // 再接下来是hash冲突发生时，下一个溢出桶的地址
}
```
bmap不只tophash还有两个方法 overflow 和setoverflow

```go
func (b *bmap) overflow(t *maptype) *bmap {
	return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize))
}

func (b *bmap) setoverflow(t *maptype, ovf *bmap) {
	*(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize)) = ovf
}
```

hmap中的buckets中的原色bucket就是bmap，即 buckets[0],bucket[1],... bucket[2^B-1]如下图

![](https://raw.githubusercontent.com/zhaojizhuang/zhaojizhuang.github.io/master/_posts/images/hmap.png)

**bucket就是bmap**

bmap 是存放 k-v 的地方，我们把视角拉近，仔细看 bmap 的内部组成。

![](https://raw.githubusercontent.com/zhaojizhuang/zhaojizhuang.github.io/master/_posts/images/bmap.png)

key 经过哈希计算后得到哈希值，共 64 个 bit 位（64位机，32位机就不讨论了，现在主流都是64位机），计算它到底要落在哪个桶时，只会用到**最后 B 个 bit 位**。还记得前面提到过的 B 吗？如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32

例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：

 `10010111 | 000011110110110010001111001010100010010110010101010 │ 01010`
 
用最后的 5 个 bit 位，也就是 `01010`，值为 10，也就是 10 号桶。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的位操作代替。

再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。

buckets 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。冲突的解决手段是用链表法：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到对应的桶，再去遍历 bucket 中的 key

**map的两种表示方式**：

- 开放寻址法（hash冲突时，在当前index往后查找第一个空的位置即可）
- 拉链法

map在写入过程会发生扩容，`runtime.mapassign` **函数会在以下两种情况发生时触发哈希的扩容**：

- 装载因子已经超过 6.5；装载因子=总数量/桶的数量
- 哈希使用了太多溢出桶；溢出捅的数量 超过正常桶的数量 即 noverflow 大于 1<<B buckets

**每次都会将桶的数量翻倍**


扩容机制：
- **翻倍扩容**：哈希在存储元素过多时状态会触发扩容操作，每次都会将桶的数量翻倍，整个扩容过程并不是原子的，而是通过 runtime.growWork 增量触发的，在扩容期间访问哈希表时会使用旧桶，向哈希表**写入数据时**会触发旧桶元素的分流；
- **等量扩容**，**为了解决大量写入、删除造成的内存泄漏问题**，哈希引入了 sameSizeGrow这一机制，在出现较多溢出桶时会对哈希进行『内存整理』减少对空间的占用。

参考链接 
- [https://www.jianshu.com/p/aa0d4808cbb8](https://www.jianshu.com/p/aa0d4808cbb8)

- [https://segmentfault.com/a/1190000018387055](https://segmentfault.com/a/1190000018387055)

- draveness [https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)

## interface 

1. 不含有任何方法的 `interface`
```go
type eface struct { // 16 bytes
	_type *_type
	data  unsafe.Pointer
}
```
2. 含有 方法的 `interface`

```go
type iface struct { // 16 bytes
	tab  *itab
	data unsafe.Pointer
}
```
		


|变量类型|结构体实现接口|结构体指针实现接口|
|----|----|----|
|结构体初始化变量|	通过|	**不通过**|
|结构体指针初始化变量|	通过|	通过|

不通过的如下

```go
type Duck interface {
	Quack()
}

type Cat struct{}

func (c *Cat) Quack() {
	fmt.Println("meow")
}

func main() {
	var c Duck = Cat{}  // 将结构体变量传到指针类型接受的函数是不行的，反过来可行
	c.Quack()
}

$ go build interface.go
./interface.go:20:6: cannot use Cat literal (type Cat) as type Duck in assignment:
	Cat does not implement Duck (Quack method has pointer receiver)
```

Go中函数调用都是值拷贝，使用 c.Quack() 调用方法时都会发生**值拷贝**：

- 对于 &Cat{} 来说，这意味着拷贝一个新的 &Cat{} 指针，这个指针与原来的指针指向一个相同并且唯一的结构体，所以编译器可以隐式的对变量解引用（dereference）获取指针指向的结构体；
- 对于 Cat{} 来说，这意味着 Quack 方法会接受一个全新的 Cat{}，因为方法的参数是*Cat，编译器不会无中生有创建一个新的指针；即使编译器可以创建新指针，这个指针指向的也不是最初调用该方法的结构体；

## panic

panic只会调用当前Goroutine的defer（）

```go
func main() {
	defer println("in main")
	go func() {
		defer println("in goroutine")
		panic("")
	}()

	time.Sleep(1 * time.Second)
}

$ go run main.go
in goroutine
panic:
```

![](https://img.draveness.me/2020-01-19-15794253176199-golang-panic-and-defers.png)

## make  new

new 返回的是指针，指向一个type类型内存空间的指针
new等价于   

```go
var  a  typeA  // tpyeA的零值
b:=&a

```
但是 new不能对 chanel map  slice进行初始化 ，这几个必须经过make进行结构体的初始化才能用


## epoll

参考文章 
1. https://www.jianshu.com/p/dfd940e7fca2

[2 罗培羽：如果这篇文章说不清epoll的本质，那就过来掐死我吧！(1)](https://zhuanlan.zhihu.com/p/63179839)

[3 罗培羽：如果这篇文章说不清epoll的本质，那就过来掐死我吧！(2)](https://zhuanlan.zhihu.com/p/64138532)

[4 罗培羽：如果这篇文章说不清epoll的本质，那就过来掐死我吧！(3)](https://zhuanlan.zhihu.com/p/64746509)

## IO 多路复用 & epoll

目前支持I/O多路复用的系统调用有 `select，pselect，poll，epoll` ，I/O多路复用就是通过一种机制，`一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作`

### select

调用后select函数会阻塞，直到有描述符就绪（有数据可读、可写），或者超时，函数返回。当select函数返回后，**可以通过遍历fdset，来找到就绪的描述符**。

select的流程

假如程序同时监视如下图的sock1、sock2和sock3三个socket，那么在调用select之后，**操作系统把进程A分别加入这三个socket的等待队列中**

![](https://pic4.zhimg.com/80/v2-0cccb4976f8f2c2f8107f2b3a5bc46b3_720w.jpg)

当任何一个socket收到数据后，中断程序将唤起进程,将进程从所有**fd（socket）的等待队列中**移除，再将进程加入到工作队列里面

进程A被唤醒后，它知道至少有一个socket接收了数据。**程序需遍历一遍socket列表，可以得到就绪的socket**

缺点：

- 其一，每次调用select都需要将进程加入到所有监视socket的等待队列，每次唤醒都需要从每个队列中移除。这里涉及了两次遍历，而且每次都要将整个fds列表传递给内核，有一定的开销。正是因为遍历操作开销大，出于效率的考量，才会规定select的最大监视数量，**默认只能监视1024个socket**。

- 其二，进程被唤醒后，程序并不知道哪些socket收到数据，还需要遍历一次。

### poll与select一样，只是去掉了 1024的限制

### epoll

`epoll` 事先通过 `epoll_ctl()` 来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似 `callback` 的回调机制，迅速激活这个文件描述符，当进程调用 `epoll_wait()` 时便得到通知。(**此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在**。)

epoll使**用一个文件描述符(`eventpoll`)管理多个描述符**，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次

```c
int s = socket(AF_INET, SOCK_STREAM, 0);   
bind(s, ...)
listen(s, ...)

int epfd = epoll_create(...);
epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中

while(1){
    int n = epoll_wait(...)
    for(接收到数据的socket){
        //处理
    }
}
```
流程：
首先创建 epoll对象
创建epoll对象后，可以用epoll_ctl添加或删除所要监听的socket

假设计算机中正在运行进程A和进程B，在某时刻进程A运行到了epoll_wait语句。如下图所示，内核会将进程A放入eventpoll的等待队列中，阻塞进程

![](https://pic1.zhimg.com/80/v2-90632d0dc3ded7f91379b848ab53974c_720w.jpg)

当socket接收到数据，**中断程序一方面修改rdlist**，另一方面唤醒eventpoll等待队列中的进程，进程A再次进入运行状态（如下图）。也因为rdlist的存在，**进程A可以知道哪些socket发生了变化**。

![](https://pic4.zhimg.com/80/v2-40bd5825e27cf49b7fd9a59dfcbe4d6f_720w.jpg)


