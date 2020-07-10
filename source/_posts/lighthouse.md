---
title: Lighthouse性能分析
date: 2020-07-07 21:56:46
tags:
---

## 背景

随着公司业务的不断扩展 ，系统也变得越来越臃肿，需要被不断的拆分，引进诸如微前端这样的框架，开发人员也不断的扩充，甚至有不同办公地点的同事协作开发。除了基本的开发规范外，也需要有一套完善的监控来测试和记录每次代码提交是否比之前版本存在性能不足等问题，在 CI 阶段发现问题，提早解决避免上线后带来性能损失而流失用户。团队成员也能在工作中不断的成长、驱动、交付出优质的应用。

### 前端经常要关注的几个指标：

**First Contentful Paint：**浏览器首次绘制文本、图片（包含背景图）、非白色的 canvas 或 SVG 的时间节点。_反映了网络的可用性和页面资源是否庞大导致传输时间过长。_

**First Meaningful Paint：**页面的“主要内容”开始出现在屏幕上的时间点，测量用户加载体验的主要指标。_反映了是否太多非重要资源加载或执行的优先级高于主要的呈现资源。_

**First CPU Idle：**页面主线程首次可以触发 input 操作，通常叫做最小可交互时间。

**Time to Interactive：**页面完全达到可交互状态的时间点。

<br/>

## Lighthouse 介绍

> 是一个开源的自动化工具，用于改进网络应用的质量。 您可以将其作为一个 Chrome 扩展程序运行，或从命令行运行。 您为 Lighthouse 提供一个您要审查的网址，它将针对此页面运行一连串的测试，然后生成一个有关页面性能的报告。

![Lighthouse 报告](/images/lighthouse/report.png)

<br/>

## 原理分析

![Lighthouse 架构图](/images/lighthouse/architecture.png)

**Driver：** 和 Chrome 交互的对象

**Gatherers：** 收集网页的一些基础信息，用于后续的 Audit

**Artifacts：** 一系列 Gatherers 的信息集合

**Audit：** 测试单个功能/优化/指标。用 Artifacts 作为输出，计算出该项指标的得分，生成*Lighthouse 标准的数据对象*

**Report：** 根据 Lighthouse 标准的数据对象生成可视化页面

**Lighthouse 通过 Driver 模块用 DevTools Protocol 与 Chrome 通信，按照需求对其进行操作，在 Gatherers 模块收集一些信息（artifacts）,在 Audits 模块中进行计算，得到最终打分结果，生成 LHR 根式的报告，渲染成 HTML 文件。**

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

- **npm run build：** 将静态资源打包，一般打包后的资源会存放在 dist 目录里

- **lhci autorun：** 常用命令，如其字面意思，自动完成整个测试流程，在它里面包含了`healthcheck`、`collect`、`assert`、`upload`命令

### collect 命令源码分析

> _@lhci/cli 版本 0.4.x_

```javascript
// autorun
async function runCommand(options) {
  // 执行 healthcheck
  const healthcheckStatus = runChildCommand("healthcheck", [
    ...defaultFlags,
    "--fatal",
  ]).status;
  // 执行 collect，重点命令
  const collectStatus = runChildCommand("collect", [
    ...defaultFlags,
    ...collectArgs,
  ]).status;

  // 默认不执行 assert 命令， 需要 assert 参数 并且 没有 upload 参数
  if (
    ciConfiguration.assert ||
    (!ciConfiguration.assert && !ciConfiguration.upload)
  ) {
    const assertStatus = runChildCommand("assert", [
      ...defaultFlags,
      ...assertArgs,
    ]).status;
  }

  // 默认不执行 assert 命令， 例子中有 upload 参数 会执行它
  if (ciConfiguration.upload) {
    const uploadStatus = runChildCommand("upload", defaultFlags).status;
    if (options.failOnUploadFailure && uploadStatus !== 0)
      process.exit(uploadStatus);
    if (uploadStatus !== 0)
      process.stderr.write(`WARNING: upload command failed.\n`);
  }
}
```

从上述代码中可以很直观的看到 `autorun` 里面包含的命令

- healthcheck 一些检查，比如是否安装了 Chrome、客户端版本和服务端版本是否一致等等
- collect 重要命令，它的整个流程基本涵盖了架构图，从信息的采集到生成报告
- assert 性能分析是否通过，会有相关的提示
- upload 上传报告到指定的服务器，上面例子的上传目标是 `temporary-public-storage`，一个 google 提供的临时公共服务器

```javascript
async function runCommand(options) {
  if (!options.additive) clearSavedReportsAndLHRs();

  checkIgnoredChromeFlagsOption(options);

  const puppeteer = new PuppeteerManager(options);

  // 检查是否有 staticDistDir 参数（默认是dist、build或public，也可以通过--static-dist-dir指定）
  // staticDistDir是打包后的静态资源目录
  // 有 => 创建 express server， 将静态资源加载进服务中（一般都是有的，除非手动关闭）
  // 没 => 执行 startServerCommand，开发者自己创建一个服务
  const { urls, close } = await startServerAndDetermineUrls(options);
  try {
    for (const url of urls) {
      await puppeteer.invokePuppeteerScriptForUrl(url);
      await runOnUrl(url, options, { puppeteer });
    }
  } finally {
    await close();
  }

  process.stdout.write(`Done running Lighthouse!\n`);
}
```

[puppeteer](https://github.com/puppeteer/puppeteer)：一个通过 DevTools Protocol 操控 Headless Chrome 的 Node.js 类库

<!-- 'dist',
  // likely a built version of the site
  // default name for create-react-app and preact-cli
  'build',
  // riskier, sometimes is a built version of the site but also can be just a dir of static assets
  // default name for gatsby
  'public', -->
