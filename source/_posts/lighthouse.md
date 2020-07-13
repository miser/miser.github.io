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
      // 如果有 puppeteerScript 参数，那么会出触发该脚本
      await puppeteer.invokePuppeteerScriptForUrl(url);
      //
      await runOnUrl(url, options, { puppeteer });
    }
  } finally {
    await close();
  }

  process.stdout.write(`Done running Lighthouse!\n`);
}

async function runOnUrl(url, options, context) {
  const runner = getRunner(options); //

  // 默认执行3次
  for (let i = 0; i < options.numberOfRuns; i++) {
    // 每次新开一个Node.js进程 调用 lighthouse-cli
    const lhr = await runner.runUntilSuccess(url, {
      ...options,
      settings,
    });
    saveLHR(lhr); // 保存 lighthouse result 数据
  }
}
```

- [puppeteer](https://github.com/puppeteer/puppeteer)：一个通过 DevTools Protocol 操控 Headless Chrome 的 Node.js 类库
- numberOfRuns 默认是 3，会分别开启 3 个 Node.js 进程各走一次流程，最后在 .lighthouseci 目录下生成 3 对 lhr-xxx.json 和 lhr-xxx.html 文件（前者是分析结果，后者是渲染的网页），及一个 assertion-results.json 文件
- runUntilSuccess 开始执行 lighthouse-cli 里的逻辑

```javascript
// lighthouse-cli 部分代码
async function runLighthouse(url, flags, config) {
  let launchedChrome;

  // 通过 chrome-launcher 启动一个 headless Chrome
  launchedChrome = await getDebuggableChrome(flags);
  flags.port = launchedChrome.port; // 需要知道该Chrome端口号，为了之后的 websocket 通信

  // lighthouse-core/index.js
  const runnerResult = await lighthouse(url, flags, config);

  if (runnerResult) {
    await saveResults(runnerResult, flags);
  }

  return runnerResult;
}
```

在 lighthouse-cli 中主要就是启动一个 headless Chrome（无界面的 Chrome）,后续交由 lighthouse-core 核心模块完成。

```javascript
// lighthouse-core/index.js
async function lighthouse(url, flags = {}, configJSON, connection) {
  // 将自定义配置和默认配置做合并
  // 根据配置信息创建大量继承Gatherer的对象，有3个方法beforePass、pass、afterPass
  const config = generateConfig(configJSON, flags);

  // ChromeProtocol 封装了和 Chrome 通信的 websocket
  const connection =
    connection || new ChromeProtocol(flags.port, flags.hostname);

  // 收集 Gather 和计算 Audit 的核心方法
  return Runner.run(connection, { url, config });
}

async function run(connection, runOpts) {
  // 从浏览器中获取所有的 artifacts
  artifacts = await Runner._gatherArtifactsFromBrowser(
    requestedUrl,
    runOpts,
    connection
  );
  // 根据获取的artifacts，按照需要计算的 audits，计算出结果
  const auditResults = await Runner._runAudits(
    settings,
    runOpts.config.audits,
    artifacts,
    lighthouseRunWarnings
  );

  const lhr = {
    // ...
  };

  // 按照 lhr 格式生成报告
  lhr.i18n.icuMessagePaths = i18n.replaceIcuMessageInstanceIds(
    lhr,
    settings.locale
  );

  const report = generateReport(lhr, settings.output);

  return { lhr, artifacts, report };
}

async function _gatherArtifactsFromBrowser(
  requestedUrl,
  runnerOpts,
  connection
) {
  const driver = runnerOpts.driverMock || new Driver(connection);
  const gatherOpts = {
    driver,
    requestedUrl,
    settings: runnerOpts.config.settings,
  };
  // 收集所有 Gather，生成结果集合 artifacts
  const artifacts = await GatherRunner.run(
    runnerOpts.config.passes,
    gatherOpts
  );
  return artifacts;
}
```

上述代码的逻辑非常清晰，就是根据现有数据（artifacts），计算出结果数据（audits），生成报告。

#### 如何得到 artifacts 这些信息的呢？

```javascript
// gather-runner.js
async function run(passConfigs, options) {
  const artifacts = {};
  const driver = options.driver;

  // 和 Chrome 进行连接
  await driver.connect();
  // 加载个 about:blank 空白页面
  await GatherRunner.loadBlank(driver);

  // 创建 artifacts
  // 一些基础信息，比如userAgent、移动还是桌面emulate
  const baseArtifacts = await GatherRunner.initializeBaseArtifacts(options);
  // 通过一定数量的字符串拼接计算出待测试驱动的性能
  baseArtifacts.BenchmarkIndex = await options.driver.getBenchmarkIndex();

  // 将环境配置好
  await GatherRunner.setupDriver(driver, options);

  // passConfigs 就是 需要收集的Gather集合
  // runPass是一个很长的周期，真正开始做数据分析的方法
  for (const passConfig of passConfigs) {
    const passContext = {
      driver,
      url: options.requestedUrl,
      settings: options.settings,
      passConfig,
      baseArtifacts,
      LighthouseRunWarnings: baseArtifacts.LighthouseRunWarnings,
    };
    // loadBlank => setupPassNetwork => cleanBrowserCaches
    // => beforePass // gather 对象对外暴露的接口
    // => beginRecording
    // => loadPage
    // => pass // gather 对象对外暴露的接口
    // => endRecording
    // => afterPass // gather 对象对外暴露的接口
    // => collectArtifacts
    const passResults = await GatherRunner.runPass(passContext);
    // 将所有结果挂在 artifacts 对象上
    Object.assign(artifacts, passResults.artifacts);
  }
  return { ...baseArtifacts, ...artifacts }; // Cast to drop Partial<>.
}
// Gatherer 基类
class Gatherer {
  get name() {
    return this.constructor.name;
  }
  beforePass(passContext) {}

  pass(passContext) {}

  afterPass(passContext, loadData) {}
}
```

runPass 是一个重要的生命周期，如果要给 Lighthouse 写自定义的扩展，必须要了解它，我们需要通过它将信息收集起来，并挂载到 artifacts 上。

#### 如何收集浏览器的信息呢？

以收集图片信息的 `ImageElements` 为例，它对外暴露了 afterPass 接口，是一个经典应用

```javascript
class ImageElements extends Gatherer {
  async afterPass(passContext, loadData) {
    const driver = passContext.driver;
    const indexedNetworkRecords = loadData.networkRecords.reduce(
      (map, record) => {
        if (
          /^image/.test(record.mimeType) &&
          record.finished &&
          record.statusCode === 200
        ) {
          map[record.url] = record;
        }
        return map;
      }
    );

    const expression = `(function() {
      ${pageFunctions.getElementsInDocumentString}; // define function on page
      ${getClientRect.toString()};
      ${getHTMLImages.toString()};
      ${getCSSImages.toString()};
      ${collectImageElementInfo.toString()};

      return collectImageElementInfo();
    })()`;

    const elements = await driver.evaluateAsync(expression);
    const top50Images = Object.values(indexedNetworkRecords)
      .sort((a, b) => b.resourceSize - a.resourceSize)
      .slice(0, 50);

    const imageUsage = [];
    for (let element of elements) {
      // ...
      // 在Chrome注入并执行 determineNaturalSize 方法，获取图片的原始宽和高
    }
    return imageUsage;
  }
}

function determineNaturalSize(url) {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.addEventListener("error", (_) =>
      reject(new Error("determineNaturalSize failed img load"))
    );
    img.addEventListener("load", () => {
      resolve({
        naturalWidth: img.naturalWidth,
        naturalHeight: img.naturalHeight,
      });
    });

    img.src = url;
  });
}
```

所有 afterPass 方法是在页面加载完后执行的，所以会比 beforePass 多一个`loadData`参数，记录了网络加载的数据，比如，图片。

在 ImageElements 中

- 先找出所有正常加载的图片
- expression，定义一个闭包，将相关需要用到的方法通过字符串的形式拼接起来，再用 driver.evaluateAsync 将它们注入到 Chrome, 并执行
- 按照尺寸大小排序，获取最大的前 50 个图片信息
- elements 和 top50Images，进行相关的逻辑处理获取图片的原始尺寸，最后返回结果集 imageUsage

**从上述代码我们可以知道，将 JavaScript 方法通过 `driver.evaluateAsync` 注入到 Chrome 里并执行，收集浏览器的信息。**

#### 有了信息，该如何计算呢？

```javascript
const auditResults = await Runner._runAudits(
  settings,
  runOpts.config.audits,
  artifacts, // 之前收集的信息
  lighthouseRunWarnings
);

async function _runAudits(settings, audits, artifacts, runWarnings) {
  for (const auditDefn of audits) {
    const auditResult = await Runner._runAudit(
      auditDefn,
      artifacts,
      sharedAuditContext,
      runWarnings
    );
    auditResults.push(auditResult);
  }

  log.timeEnd(status);
  return auditResults;
}
```
