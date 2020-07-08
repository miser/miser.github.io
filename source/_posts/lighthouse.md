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

## 原理分析

架构图

lghi 启动

创建express服务

Lighthouse CI

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

