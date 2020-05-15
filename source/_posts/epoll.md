---
layout: post
title:  " 谈谈 epoll "
date:   2019-05-10 11:40:18 +0800
categories: 
tags:  ["linux", "epoll"]
author: zhaojizhuang
---


## epoll

## IO 多路复用

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



参考文章 
1. https://www.jianshu.com/p/dfd940e7fca2

[2 罗培羽：如果这篇文章说不清epoll的本质，那就过来掐死我吧！(1)](https://zhuanlan.zhihu.com/p/63179839)

[3 罗培羽：如果这篇文章说不清epoll的本质，那就过来掐死我吧！(2)](https://zhuanlan.zhihu.com/p/64138532)

[4 罗培羽：如果这篇文章说不清epoll的本质，那就过来掐死我吧！(3)](https://zhuanlan.zhihu.com/p/64746509)
