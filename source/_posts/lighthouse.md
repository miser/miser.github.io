---
title: Lighthouse性能分析
date: 2020-07-07 21:56:46
tags:
---

## 背景

随着公司业务的不断扩展 ，系统也变得越来越臃肿，需要被不断的拆分，引进诸如微前端这样的框架，开发人员也不断的扩充，甚至有不同办公地点的同事协作开发。除了基本的开发规范外，也需要有一套完善的监控来测试和记录每次代码提交是否比之前版本存在性能不足等问题，在CI阶段发现问题，提早解决避免上线后带来性能损失而流失用户。团队成员也能在工作中不断的成长、驱动、交付出优质的应用。

### 前端经常要关注的几个指标：

**First Contentful Paint：**浏览器首次绘制文本、图片（包含背景图）、非白色的canvas或SVG的时间节点。*反映了网络的可用性和页面资源是否庞大导致传输时间过长。*

**First Meaningful Paint：**页面的“主要内容”开始出现在屏幕上的时间点，测量用户加载体验的主要指标。*反映了是否太多非重要资源加载或执行的优先级高于主要的呈现资源。*

**First CPU Idle：**页面主线程首次可以触发input操作，通常叫做最小可交互时间。

**Time to Interactive：**页面完全达到可交互状态的时间点。

<br/>

## Lighthouse 介绍

> 是一个开源的自动化工具，用于改进网络应用的质量。 您可以将其作为一个 Chrome 扩展程序运行，或从命令行运行。 您为 Lighthouse 提供一个您要审查的网址，它将针对此页面运行一连串的测试，然后生成一个有关页面性能的报告。

![Lighthouse 报告](/images/lighthouse/report.png)

<br/>

## 原理分析

![Lighthouse 架构图](/images/lighthouse/architecture.png)

**Driver：** 和Chrome交互的对象

**Gatherers：** 收集网页的一些基础信息，用于后续的Audit

**Artifacts：** 一系列Gatherers的信息集合

**Audit：** 测试单个功能/优化/指标。用Artifacts作为输出，计算出该项指标的得分，生成*Lighthouse标准的数据对象*

**Report：** 根据Lighthouse标准的数据对象生成可视化页面

**Lighthouse 通过 Driver 模块用 DevTools Protocol 与 Chrome 通信，按照需求对其进行操作，在Gatherers模块收集一些信息（artifacts）,在Audits模块中进行计算，得到最终打分结果，生成LHR根式的报告，渲染成HTML文件。**

### Lighthouse CI

官方示例

```yaml
name: CI
on: [push]
jobs:
  lighthouseci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v1
      - run: npm install && npm install -g @lhci/cli@0.4.x
      - run: npm run build
      - run: lhci autorun --upload.target=temporary-public-storage
```

- **npm run build：** 将静态资源打包，一般打包后的资源会存放在dist目录里

- **lhci autorun：** 是很常用的命令，如其字面意思自动完成整个，在它里面包含了`healthcheck`、`collect`、`assert`、`upload`命令
  - healthcheck 一些检查，比如是否安装了Chrome、客户端版本和服务端版本是否一致等等
  - collect 重要命令，整个流程基本包含了架构图里的信息，从信息的采集到生成报告
  - assert 性能分析是否通过，会有相关的提示
  - upload 上传报告到指定的服务器，上面例子的上传目标是 `temporary-public-storage`，一个google提供的临时公共服务器

