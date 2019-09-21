---
layout: post
title: [转]Design patterns for container-based distributed systems
tags: kubernetes
---

***

***

[TOC]

## 1. 概述

基于容器化组件构建的微服务架构

容器的一大独特优势在于：良好的边界——恰好适合应用开发的隔离性。

## 2. 基于容器的分布式系统的设计模式

### 2.1 Single-container management patterns

- 容器的传统接口有run(), pause(), stop()
- 可以有更丰富的接口
  - “向上看”的视角：metrics, config, logs等，通常用HTTP+JSON来暴露 
  - “向下看”的视角：lifecycle（提供生命周期回调钩子）, priority（为了空出资源给高优先级任务，甚至能提前关掉低优先级任务），"replicate yourself"（迅速创建一组相同的服务容器来应对突发流量）

### 2.2 Single-node, multi-container application patterns

#### 2.2.1 Sidecar模式

![1568859437137](D:\git\YunWang.github.io\_posts\kubernetes\assets\1568859437137.png)

- 例如：web server + log collector
- 前提是容器间能共享“存储卷”之类的资源

##### 为什么要用多容器，而非容器内多进程？

- **资源审计和分配**：这点很重要，虽然多进程也勉强能做，但多容器做得更成熟
- 职责划分、**解耦**
- **重用**：如果一个容器包含多种进程，就笨重而难以重用了，小巧的单用途容器更适合重用
- **故障隔离**：重启一个容器，比修复容器内的故障进程要容易多了
- **交付隔离**：每个容器能独立地升级/降级

#### 2.2.2 Ambassador模式

![1568859596305](D:\git\YunWang.github.io\_posts\kubernetes\assets\1568859596305.png)

- 类似于OOP的proxy模式
- 例如：memcache + twemproxy，考虑到本分类为单结点多容器模式，因此twemproxy要和其中1个memcache部署到同一结点上
- 前提是容器间能共享localhost网络接口

#### 2.2.3 Adapter模式

![1568859670450](D:\git\YunWang.github.io\_posts\kubernetes\assets\1568859670450.png)

- 例如：统一的metrics接口（JMX，statsd等统一收集到HTTP端点）
- 可以免于修改已有的容器（因为已经以容器作为软件开发的单位了）

### 2.3 Multi-node application patterns

#### 2.3.1 Leader election模式

- 领导者选举这件事已经有很多库了，但往往对编程语言有所限定
- 容器则对编程语言中立
- 容器只需构建一次就能重用，高度贯彻了抽象和封装的原则

#### 2.3.2 Work queue模式

![1568859819202](D:\git\YunWang.github.io\_posts\kubernetes\assets\1568859819202.png)

- 传统的工作队列框架对编程语言有所限定
- 实现了run(), mount()接口的容器，可作为“工作”的抽象
- 基于此可打造一种通用的工作队列框架（可能是Kubernetes的未来方向）
- 虽然没给例子，但有些类似Mesos的可插拔调度器架构

#### 2.3.3 Scatter/gather模式

![1568859931084](D:\git\YunWang.github.io\_posts\kubernetes\assets\1568859931084.png)

- 这样传递请求：client->root->server
- root结点把请求分发给很多servers，再把它们的响应汇总到一起，交给client
- 例如：搜索引擎，分布式查询引擎
- 多个leaf容器+1个merge容器，就能实现通用的scatter/gather框架（可能也是Kubernetes的未来方向）

## 结语

总之，论文将容器视为软件开发的一等公民，像OOP的对象一样重要，提倡单用途可组合可重用的容器。
这似乎是对UNIX编程艺术的重申。

## 参考

[Design patterns for container-based distributed systems](https://www.usenix.org/system/files/conference/hotcloud16/hotcloud16_burns.pdf)

[论文解读：Design patterns for container-based distributed systems](https://segmentfault.com/a/1190000018563570)