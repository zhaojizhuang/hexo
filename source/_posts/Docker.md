---
layout: post
title:  "Docker 介绍及其应用"
date:   2017-06-11 23:40:18 +0800
categories: docker
tags:  docker
author: zhaojizhuang
mathjax: true
---



> [重要链接！！！知乎上的深入浅出](https://www.zhihu.com/question/22969309)

## 1.Docker 介绍
**便于入题，首先用 Docker 的logo解释下：**
![Docker logo](https://raw.githubusercontent.com/zhaojizhuang/zhaojizhuang.github.io/master/_posts/images/what.jpg)

那个大鲸鱼（或者是货轮）就是操作系统

把要交付的应用程序看成是各种货物，原本要将各种各样形状、尺寸不同的货物放到大鲸鱼上，你得为每件货物考虑怎么安放（就是应用程序配套的环境），还得考虑货物和货物是否能叠起来（应用程序依赖的环境是否会冲突）。

现在使用了集装箱（容器）把每件货物都放到集装箱里，这样大鲸鱼可以用同样地方式安放、堆叠集装了，省事省力。[参考知乎](https://www.zhihu.com/question/28300645/answer/50922662)

**步入正题：**

Docker 是 PaaS 提供商 dotCloud 开源的一个基于 [LXC](#lxc) 的高级容器引擎，源代码托管在 Github 上, 基于go语言并遵从Apache2.0协议开源。

Docker可以轻松的为任何应用创建一个轻量级的、可移植的、自给自足的容器。开发者在笔记本上编译测试通过的容器可以批量地在生产环境中部署，包括VMs（虚拟机）、bare metal、OpenStack 集群和其他的基础应用平台。

![Docker的c/s架构](https://raw.githubusercontent.com/zhaojizhuang/zhaojizhuang.github.io/master/_posts/images/docker_cs.png)

如图所示，Docker 使用客户端-服务器 (C/S) 架构模式。

- Docker 客户端会与 Docker 守护进程进行通信。
- Docker 守护进程会处理复杂繁重的任务，例如建立、运行、发布你的 Docker 容器。
- Docker 客户端和守护进程 Daemon 可以运行在同一个系统上，当然你也可以使用 Docker 客户端去连接一个远程的 Docker 守护进程。Docker 客户端和守护进程之间通过 socket 或者 RESTful API 进行通信，就像下图。

![docker架构图](https://raw.githubusercontent.com/zhaojizhuang/zhaojizhuang.github.io/master/_posts/images/docker_frame.jpg)

### 1.1 Docker 守护进程

如上图所示，Docker 守护进程运行在一台主机上。用户并不直接和守护进程进行交互，而是通过 Docker 客户端间接和其通信。

### 1.2 Docker 客户端
Docker 客户端，实际上是 docker 的二进制程序，是主要的用户与 Docker 交互方式。它接收用户指令并且与背后的 Docker 守护进程通信，如此来回往复。

### 1.3 Docker 内部
要理解 Docker 内部构建，需要理解以下三种部件：

**_Docker 镜像_** - Docker images  
_**Docker 仓库**_ - Docker registeries  
**_Docker 容器_** - Docker containers

1.  **Docker 镜像**:Docker 镜像是 Docker 容器运行时的只读模板，每一个镜像由一系列的层 (layers) 组成。Docker 使用 UnionFS 来将这些层联合到单独的镜像中。UnionFS 允许独立文件系统中的文件和文件夹(称之为分支)被透明覆盖，形成一个单独连贯的文件系统。正因为有了这些层的存在，Docker 是如此的轻量。当你改变了一个 Docker 镜像，比如升级到某个程序到新的版本，一个新的层会被创建。因此，不用替换整个原先的镜像或者重新建立(在使用虚拟机的时候你可能会这么做)，只是一个新 的层被添加或升级了。现在你不用重新发布整个镜像，只需要升级，层使得分发 Docker 镜像变得简单和快速。

2.  **Docker 仓库**:Docker 仓库用来保存镜像，可以理解为代码控制中的代码仓库。同样的，Docker 仓库也有公有和私有的概念。公有的 Docker 仓库名字是 Docker Hub。Docker Hub 提供了庞大的镜像集合供使用。这些镜像可以是自己创建，或者在别人的镜像基础上创建。Docker 仓库是 Docker 的分发部分。

3.  **Docker 容器**:Docker 容器和文件夹很类似，一个Docker容器包含了所有的某个应用运行所需要的环境。每一个 Docker 容器都是从 Docker 镜像创建的。Docker 容器可以运行、开始、停止、移动和删除。每一个 Docker 容器都是独立和安全的应用平台，Docker 容器是 Docker 的运行部分。

## 2. Docker 8个的应用场景
> 本小节介绍了常用的8个Docker的真实使用场景，分别是简化配置、代码流水线管理、提高开发效率、隔离应用、整合服务器、调试能力、多租户环境、快速部署

![Docker 的8个应用场景](https://raw.githubusercontent.com/zhaojizhuang/zhaojizhuang.github.io/master/_posts/images/dockeruse.png)

一些Docker的使用场景，它为你展示了如何借助Docker的优势，在低开销的情况下，打造一个一致性的环境。

### 1.简化配置

这是Docker公司宣传的Docker的主要使用场景。虚拟机的最大好处是能在你的硬件设施上运行各种配置不一样的平台（软件、系统），Docker在降低额外开销的情况下提供了同样的功能。它能让你将运行环境和配置放在代码中然后部署，同一个Docker的配置可以在不同的环境中使用，这样就降低了硬件要求和应用环境之间耦合度。

### 2. 代码流水线（Code Pipeline）管理

前一个场景对于管理代码的流水线起到了很大的帮助。代码从开发者的机器到最终在生产环境上的部署，需要经过很多的中间环境。而每一个中间环境都有自己微小的差别，Docker给应用提供了一个从开发到上线均一致的环境，让代码的流水线变得简单不少。

### 3. 提高开发效率

不同的开发环境中，我们都想把两件事做好。一是我们想让开发环境尽量贴近生产环境，二是我们想快速搭建开发环境。

理想状态中，要达到第一个目标，我们需要将每一个服务都跑在独立的虚拟机中以便监控生产环境中服务的运行状态。然而，我们却不想每次都需要网络连接，每次重新编译的时候远程连接上去特别麻烦。这就是Docker做的特别好的地方，开发环境的机器通常内存比较小，之前使用虚拟的时候，我们经常需要为开发环境的机器加内存，而现在Docker可以轻易的让几十个服务在Docker中跑起来。

### 4. 隔离应用

有很多种原因会让你选择在一个机器上运行不同的应用，比如之前提到的提高开发效率的场景等。

我们经常需要考虑两点，一是因为要降低成本而进行服务器整合，二是将一个整体式的应用拆分成松耦合的单个服务（译者注：微服务架构）。

### 5. 整合服务器

正如通过虚拟机来整合多个应用，Docker隔离应用的能力使得Docker可以整合多个服务器以降低成本。由于没有多个操作系统的内存占用，以及能在多个实例之间共享没有使用的内存，Docker可以比虚拟机提供更好的服务器整合解决方案。

### 6. 调试能力

Docker提供了很多的工具，这些工具不一定只是针对容器，但是却适用于容器。它们提供了很多的功能，包括可以为容器设置检查点、设置版本和查看两个容器之间的差别，这些特性可以帮助调试Bug。

### 7. 多租户环境

另外一个Docker有意思的使用场景是在多租户的应用中，它可以避免关键应用的重写。我们一个特别的关于这个场景的例子是为IoT（物联网）的应用开发一个快速、易用的多租户环境。这种多租户的基本代码非常复杂，很难处理，重新规划这样一个应用不但消耗时间，也浪费金钱。

使用Docker，可以为每一个租户的应用层的多个实例创建隔离的环境，这不仅简单而且成本低廉，当然这一切得益于Docker环境的启动速度和其高效的diff命令。

### 8. 快速部署

在虚拟机之前，引入新的硬件资源需要消耗几天的时间。虚拟化技术（Virtualization）将这个时间缩短到了分钟级别。而Docker通过为进程仅仅创建一个容器而无需启动一个操作系统，再次将这个过程缩短到了秒级。这正是Google和Facebook都看重的特性。

你可以在数据中心创建销毁资源而无需担心重新启动带来的开销。通常数据中心的资源利用率只有30%，通过使用Docker并进行有效的资源分配可以提高资源的利用率。

#### 本文的几个概念

- <span id="lxc">**LXC**</span>: LXC为Linux Container的简写。可以提供轻量级的虚拟化，以便隔离进程和资源，而且不需要提供指令解释机制以及全虚拟化的其他复杂性。属于操作系统层次之上的虚拟化
