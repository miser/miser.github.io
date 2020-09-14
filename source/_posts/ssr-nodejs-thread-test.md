---
layout: minos
title: Node.js 除了 Cluster 还有 Worker Threads
category: JavaScript
description:
tags:
  - SSR
keywords:

date: 2020-09-02
---

#### [Cluster](https://nodejs.org/dist/latest-v14.x/docs/api/cluster.html)

这是一个比较熟悉的模块，早在 Node.js V0.8 版本的时候就已经被加入进来，平时在生产部署 Web 应用的时候，为了充分压榨多核 CPU，总是根据核数开启对应的数量的应用进程，来处理用户请求，这些在 PM2 工具 或者 Egg.js 框架里有相应的介绍，之前自己写的 [简单梳理 Node.js 创建子进程的方法（下）—— cluster](https://mlib.wang/2020/05/03/node-js-net-cluster-fork/?s=node-js-net-cluster-fork) 有其原理介绍。

虽然 Cluster 已经很成熟，但是也有一些问题：

- 进程开销较大
- 很多第三方工具对 Cluster 不是很友好，还是以单进程为主，比如一些监控系统
- 需要有一个管理进程，在某个 Cluster 进程出错退出后将其重启起来；单进程出错奔溃，运维可以通过健康检查未能通过的方式重启整个 Docker 等，相比而言 Cluster 模式复杂的多
- 目前从框架上自然支持 Cluster 的只有 Egg.js，其它都需要辅助工具，比如 PM2，然而它并完全免费，所以真使用该项技术，又面临框架选择的问题。

#### [Worker threads](https://nodejs.org/dist/latest-v14.x/docs/api/worker_threads.html)

在 Node.js V10.5.0 加入，但需要通过`--experimental-worker`开启，直到 V12 才默认支持。在密集计算任务处理上带来了新的解决方案。

我带领的团队有一个重要的业务需求，就是服务端渲染（SSR)，我们使用 Egg.js 的 Cluster 模式来帮助我们提升吞吐量。虽然业务问题解决了，但是 Egg.js 比起 Express 或者 Koa 这样的框架来说复杂很多，让一个前端同事学习或者招募有经验的新同事还是有点棘手。另外就是和其它非其生态圈的工具搭配使用也遇到不少麻烦。

<!-- ab -n 20000 -c 30 -k "http://localhost:9000/test" -->
<!-- https://github.com/miser/ssr-thread-test -->
<!-- 47.187; 423.938; 44.571;-->
<!-- 45.147; 444.377; 40;-->
