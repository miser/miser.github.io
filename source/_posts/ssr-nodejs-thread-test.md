---
layout: minos
title: Node.js 除了 Cluster 还有 Worker Threads
category: JavaScript
description: Cluster 这种复杂的编程方式对轻巧的 Node.js 并不有好，加之容器化技术的不断发展，多个 K8s Pod 完全就可以覆盖 Cluster 功能，Egg.js 从 18 年就开始讨论是否使用单进程模型，目前新版本已近支持。所以，对于 Node.js 而言 Worker Threads 才是今后解决密集计算的方向
tags:
  - SSR
keywords: Node.js,Worker Threads,Cluster,单线程,多线程

date: 2020-09-02
---

#### [Cluster](https://nodejs.org/dist/latest-v14.x/docs/api/cluster.html)

这是一个比较熟悉的模块，早在 Node.js V0.8 版本的时候就已经被加入进来，平时在生产部署 Web 应用的时候，为了充分压榨多核 CPU，总是根据核数开启对应的数量的应用进程，来处理用户请求，这些在 PM2 工具 或者 Egg.js 框架里有相应的介绍，之前自己写的 [简单梳理 Node.js 创建子进程的方法（下）—— cluster](https://mlib.wang/2020/05/03/node-js-net-cluster-fork/?s=node-js-net-cluster-fork) 有其原理介绍。

虽然 Cluster 已经很成熟，但是也有一些问题：

- 进程开销较大
- 很多第三方工具对 Cluster 不是很友好，还是以单进程为主，比如一些监控系统
- 需要有一个管理进程，在某个 Cluster 进程出错退出后将其重启起来；单进程出错奔溃，运维可以通过健康检查未能通过的方式重启整个 Docker 等，相比而言 Cluster 模式复杂的多
- 目前从框架上自然支持 Cluster 的只有 Egg.js，其它都需要辅助工具，比如 PM2，然而它并完全免费，所以真使用该项技术，又面临框架选择的问题

<br />

#### [Worker Threads](https://nodejs.org/dist/latest-v14.x/docs/api/worker_threads.html)

在 Node.js V10.5.0 加入，但需要通过`--experimental-worker`开启，直到 V12 才默认支持。在密集计算任务处理上带来了新的解决方案。

我带领的团队有一个重要的业务需求，就是服务端渲染（SSR)，我们使用 Egg.js 的 Cluster 模式来帮助我们提升吞吐量。虽然业务问题解决了，但是 Egg.js 比起 Express 或者 Koa 这样的框架来说复杂很多，让一个前端同事学习或者招募有经验的新同事还是有点棘手。另外就是和其它非其生态圈的工具搭配使用也遇到不少麻烦。

好在 Worker Threads 同样可以压榨 CPU，提升整体吞吐量。

<!-- more -->

<br />

#### 实验

**实验说明**

对一个 List（100 条）数据进行渲染，分别查看单线程和多线程所花费的时间和吞吐量

**实验环境：**

- MacBook Pro (Retina, 13-inch, Mid 2014)
- CPU 2.6 GHz 双核 Intel Core i5
- 内存 8 GB 1600 MHz DDR3

**实验代码：**

[ssr-thread-test](https://github.com/miser/ssr-thread-test)

```javascript
node one.js // 启动单线程的 web服务
```

```javascript
node multiple.js // 启动多线程的 web服务
```

多线程使用了线程池，基本上是[官方的例子](https://nodejs.org/dist/latest-v14.x/docs/api/async_hooks.html#async_hooks_using_asyncresource_for_a_worker_thread_pool)，不过官方用的是事件监听的机制，在高并发下会发出发出超出最大监听数的警告，我将其改为队列的形式。

分别进行压力测试， 2W 个请求，每次并发 30 个

```shell
ab -n 20000 -c 30 -k "http://localhost:9000/test"
```

**得分计算：**
对单线程和多线程各跑 10 次压测，去掉最高最低算平均分

|                       | 单线程  | 多线程  |
| --------------------- | ------- | ------- |
| Time Taken For Tests  | 47.187  | 45.147  |
| Requests Per Second   | 423.938 | 444.377 |
| 95%的请求小于多少毫秒 | 44.571  | 40      |

从此次实验可以得出，多线程比单线程的性能有一定提升，当然这是在简单的环境下，更为复杂的 SSR 计算相信会有更多的性能提升。

Cluster 这种复杂的编程方式对轻巧的 Node.js 并不有好，加之容器化技术的不断发展，多个 K8s Pod 完全就可以覆盖 Cluster 功能，Egg.js 从 18 年就开始讨论是否使用单进程模型，目前新版本已近支持。所以，对于 Node.js 而言 Worker Threads 才是今后解决密集计算的方向。
