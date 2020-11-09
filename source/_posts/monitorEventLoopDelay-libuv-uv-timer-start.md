---
title: monitorEventLoopDelay 实现原理
date: 2020-11-06 16:16:40
category: JavaScript
description: monitorEventLoopDelay原理介绍，通过向libuv注册uv_timer_t句柄，定时触发回调事件
tags:
  - 原理
  - V8
  - Node.js
keywords: Node.js,monitorEventLoopDelay
---

## monitorEventLoopDelay 是什么

perf_hooks.monitorEventLoopDelay([options])

- options: Object
  -- `resolution`: The sampling rate in milliseconds. Must be greater than zero. Default: 10.
- Returns: Histogram
  > Creates a Histogram object that samples and reports the event loop delay over time. The delays will be reported in nanoseconds.
  > Using a timer to detect approximate event loop delay works because the execution of timers is tied specifically to the lifecycle of the libuv event loop. That is, a delay in the loop will cause a delay in the execution of the timer, and those delays are specifically what this API is intended to detect.

监控 EventLoop 运行情况是判断系统是否健康的重要指标之一，如果有大量的延迟，说明系统存在密集计算，降低了系统的吞吐。Node.js 在 v11 版本引入了`monitorEventLoopDelay`，而之前需要自己去实现。

<!-- monitorEventLoopDelay 的功能介绍 -->

<!-- more -->

**版本**

```c++
// libuv
#define UV_VERSION_MAJOR 1
#define UV_VERSION_MINOR 33
#define UV_VERSION_PATCH 1

// V8
#define V8_MAJOR_VERSION 7
#define V8_MINOR_VERSION 8
#define V8_BUILD_NUMBER 279
#define V8_PATCH_LEVEL 17

// Node.js
#define NODE_MAJOR_VERSION 14
#define NODE_MINOR_VERSION 0
#define NODE_PATCH_VERSION 0
```

## 实现原理

_代码中隐去了当前介绍过程中不重要的代码，有兴趣可以查看源码_

```javascript
// perf_hooks.js
function monitorEventLoopDelay(options = {}) {
  // _ELDHistogram 是 C++ 暴露出来的对象
  return new ELDHistogram(new _ELDHistogram(resolution));
}

// ELDHistogram 基本是对 handle 入参对象的代理
class ELDHistogram {
  constructor(handle) {
    this[kHandle] = handle;
  }
  enable() {
    // 开启监控
    return this[kHandle].enable();
  }
}
```

```c++
// node_perf.h
class ELDHistogram : public HandleWrap, public Histogram {
 public:
  ELDHistogram(Environment* env,
               Local<Object> wrap,
               int32_t resolution);
  bool Enable();
 private:
  // Enable 开启后，注册到 EventLoop里的回调
  static void DelayIntervalCallback(uv_timer_t* req);

  bool enabled_ = false;
  // 注册在EventLoop上的 Handle
  uv_timer_t timer_;
};

// node_perf.cc
bool ELDHistogram::Enable() {
  if (enabled_ || IsHandleClosing()) return false;
  enabled_ = true;
  // resolution_ 可以通过 js 传入的参数
  // 多少时间记录一次数据
  uv_timer_start(&timer_,
                 DelayIntervalCallback,
                 resolution_,
                 resolution_);
  //  设置成 unref
  uv_unref(reinterpret_cast<uv_handle_t*>(&timer_));
  return true;
}

// 每次定时触发回调，记录相关的值
void ELDHistogram::DelayIntervalCallback(uv_timer_t* req) {
  ELDHistogram* histogram = ContainerOf(&ELDHistogram::timer_, req);
  histogram->RecordDelta();
  TRACE_COUNTER1(TRACING_CATEGORY_NODE2(perf, event_loop),
                 "min", histogram->Min());
  TRACE_COUNTER1(TRACING_CATEGORY_NODE2(perf, event_loop),
                 "max", histogram->Max());
  TRACE_COUNTER1(TRACING_CATEGORY_NODE2(perf, event_loop),
                 "mean", histogram->Mean());
  TRACE_COUNTER1(TRACING_CATEGORY_NODE2(perf, event_loop),
                 "stddev", histogram->Stddev());
}
```

从上述代码我们可以和很容易的知道，js 调用一个`ELDHistogram`对象，该对象代理底层 C++的方法，一旦开启监控，就会在 EventLoop 上注册一个回调，每次时间一到就记录相关的值放在[HdrHistogram_c](https://github.com/HdrHistogram/HdrHistogram_c)中，它是 C 对[HdrHistogram](https://github.com/HdrHistogram/HdrHistogram)的实现，可以获取从记录开始到当前时间点的最大、最小、平均值、95 线等数据。

直白点讲就是开启一个定时器，通过向 libuv 注册 uv_timer_t 句柄，执行回调，获取数据存在一个特定的容器中，我们从该容器拿被处理好的数据。
