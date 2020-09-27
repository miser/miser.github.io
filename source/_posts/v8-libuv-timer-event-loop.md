---
title: libuv & Node.js EventLoop （一）
category: JavaScript
keywords: libuv,EventLoop,V8
description: libuv和Node.js EventLoop关于各个阶段和setTimeout的实现已经做了简单介绍。具体的还是需要看各自版本的代码而定，不能轻易去“相信”网上的介绍，比如最后一个例子就很容易在不同版本出现不同的执行结果。之后，有时间介绍 `nextTick`的回调实现，这个比较复杂。
tags:
  - Node.js
  - JavaScript
date: 2020-03-01T05:39:36.647Z
---

在网络上查询[libuv](https://libuv.org/)和EventLoop相关信息的时候，经常看到不同的文章所表达的意思差距较多，主要原因有二吧：

- 它们的`libuv`和`V8`大版本不同，导致具体的实现略有差异
- 另外它们的代码错综复杂，又是大多数JavaScript工作者不擅长的C/C++，只是从上而下的看，或许一些细节无法完全理解或是认知的分歧

与其受他人影响，不如自己来好好梳理下。

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

没有列出具体版本号的代码分析都是耍流氓，2010年的代码和2020年的代码可能差距甚远，“上古”分析固然在当时是对的，但是在今日也许是错误的。

<!-- more -->

<br/>

<br/>

### libuv & EventLoop文档比较

![_images/loop_iteration.png](/images/v8-libuv-timer-event-loop/loop_iteration.png)

上图出自`libuv`的[The I/O loop](http://docs.libuv.org/en/v1.x/design.html#the-i-o-loop)介绍，具体的信息可以看其文档。

![image-20200301145547994](/images/v8-libuv-timer-event-loop/image-20200301145547994.png)

上图出自[Event Loop Explained](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#event-loop-explained)，两者的各个阶段基本是对应的。哪怕是很多其它自行画的图解中，也基本差不多，但唯独`pending callbacks`阶段，我看到很多文章里面把它标为`I/O callbacks`，不知道是不是历史原因，但是就从二者的目前文档解释来看是不妥的。

> Pending callbacks are called. All I/O callbacks are called right after polling for I/O, for the most part. There are cases, however, in which calling such a callback is deferred for the next loop iteration. If the previous iteration deferred any I/O callback it will be run at this point. —— libuv

**libuv**里的`pending callbacks`：大多数的I/O callbacks应该在polling阶段完成，有部分会被延迟到下一个pending callbacks阶段执行。

<br/>

> This phase executes callbacks for some system operations such as types of TCP errors. For example if a TCP socket receives `ECONNREFUSED` when attempting to connect, some *nix systems want to wait to report the error. This will be queued to execute in the **pending callbacks** phase. —— Node.js

**Node.js**里的`pending callbacks`：这个阶段会执行系统上因一些错误而引起的callbacks，如TCP错误。

我觉得`pending callbacks`比较合理，一方面如libuv所说，它是一部分延迟的I/O回调，在Node.js里面指的是一些系统上的错误（这些错误也是I/O引起的）。而大多数的I/O操作其实是在 `poll`阶段。

<br/>

<br/>

## libuv的大致流程代码

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) { // 默认 mode 是 UV_RUN_DEFAULT
  int timeout;
  int r;
  int ran_pending;

  // 是否还存在alive的事件
  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop); // 存在的话更新当前的时间，可以把这个时间理解为libuv里面统一的时间，方便触发定时任务

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop); // 和libuv图里的 Update loop time对应
    uv__run_timers(loop); // 执行 timers 阶段
    ran_pending = uv__run_pending(loop); // 执行pending 阶段；返回0表示空，1表示有；
    uv__run_idle(loop); // idle 阶段； Node.js里面不太关心
    uv__run_prepare(loop); // prepare 阶段； Node.js里面不太关心

    timeout = 0; // 初始化阻塞 poll 阶段的超时时间
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop); // 算出阻塞 poll 阶段的超时时间
    // 根据timers里最近的超时时间算出一个差值 diff = loop.time - min.timeout
    // 如果 diff >= 0 , timeout = 0
    // 否则 timeout = min(diff, INT_MAX)
    
    uv__io_poll(loop, timeout); // 执行 poll 阶段
    uv__run_check(loop); // 执行 check 阶段
    uv__run_closing_handles(loop); // 执行 close 阶段

    // mode 默认 UV_RUN_DEFAULT 所以不执行下面
    if (mode == UV_RUN_ONCE) {
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    // 查看是否还有alive事件
    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```

上述`uv_run`方法同样表达了之前图的循环流程，接下来我们看看各个阶段的具体执行方法。

<br/>

```c
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node; // timers 里的都是按照最小堆存放的
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min(timer_heap(loop)); // 从堆顶取出一个
    if (heap_node == NULL) // 不存在就退出
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node); 
    if (handle->timeout > loop->time) // 触发时间没达到也退出
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    
    handle->timer_cb(handle); // 执行回调
  }
}
```

timers里存放的事件都是最小堆的数据结构排列的，不断的取出根节点比较当前的`loop->time`就能知道是执行还是退出，**那么这里的`handle->timer_cb`是我们平时JavaScript里的setTimeout回调吗？**后续解答

<br/>

```c
static int uv__run_pending(uv_loop_t* loop) {
  QUEUE* q;
  QUEUE pq;
  uv__io_t* w;

  if (QUEUE_EMPTY(&loop->pending_queue))
    return 0;

  QUEUE_MOVE(&loop->pending_queue, &pq);

  while (!QUEUE_EMPTY(&pq)) { // 双向链表，为空就跳出
    q = QUEUE_HEAD(&pq);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);
    w = QUEUE_DATA(q, uv__io_t, pending_queue);
    w->cb(loop, w, POLLOUT); // 执行回调
  }

  return 1;
}
```

libuv里面事件很多是由双向链表构建而成，等着被一个个执行，双向链表的好处就是插入很容易。

<br/>

```c
#define UV_LOOP_WATCHER_DEFINE(name, type)                                    \
	void uv__run_##name(uv_loop_t* loop) {                                      \
    uv_##name##_t* h;                                                         \
    QUEUE queue;                                                              \
    QUEUE* q;                                                                 \
    QUEUE_MOVE(&loop->name##_handles, &queue);                                \
    while (!QUEUE_EMPTY(&queue)) {                                            \
      q = QUEUE_HEAD(&queue);                                                 \
      h = QUEUE_DATA(q, uv_##name##_t, queue);                                \
      QUEUE_REMOVE(q);                                                        \
      QUEUE_INSERT_TAIL(&loop->name##_handles, q);                            \
      h->name##_cb(h);                                                        \
    }                                                                         \
  }     

UV_LOOP_WATCHER_DEFINE(prepare, PREPARE)
UV_LOOP_WATCHER_DEFINE(check, CHECK)
UV_LOOP_WATCHER_DEFINE(idle, IDLE)
```

`UV_LOOP_WATCHER_DEFINE`宏直接初始化了`prepare`、`check`和`idle`3个阶段，都是双向链表。

<br/>

```c
// uv__io_poll 太长了，不重要的代码已经移除
void uv__io_poll(uv_loop_t* loop, int timeout) {
  // ...
  if (loop->nfds == 0) { // 没有需要执行的 直接退出
    assert(QUEUE_EMPTY(&loop->watcher_queue));
    return;
  }

  // 将loop->watcher_queue列队里的待观察的文件描述符绑定到epoll上
  while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);
    // ...
    if (w->events == 0)
      op = EPOLL_CTL_ADD; // 添加
    else
      op = EPOLL_CTL_MOD; // 修改

    epoll_ctl(loop->backend_fd, op, w->fd, &e) // epoll_ctl 底层的系统函数，将文件描述符关联起来

    w->events = w->pevents;
  }

  sigmask = 0;
  if (loop->flags & UV_LOOP_BLOCK_SIGPROF) {
    sigemptyset(&sigset);
    sigaddset(&sigset, SIGPROF);
    sigmask |= 1 << (SIGPROF - 1);
  }

  base = loop->time;
  real_timeout = timeout;

  for (;;) {
    nfds = epoll_wait(loop->backend_fd,
                        events,
                        ARRAY_SIZE(events),
                        timeout);
    
    if (nfds == 0) { // 超时，没有新的事件准备好
      assert(timeout != -1); // -1 表示不会超时，而nfds为0表示超时，存在矛盾所以抛出异常
      if (timeout == 0) // 退出
        return;
      
      goto update_timeout; // 重新更新时间 准备下次循环 epoll_wait 
    }

    if (nfds == -1) { // 出错
      if (errno == ENOSYS) {
        /* epoll_wait() or epoll_pwait() failed, try the other system call. */
        assert(no_epoll_wait == 0 || no_epoll_pwait == 0);
        continue;
      }

      if (errno != EINTR)
        abort();

      if (timeout == -1)
        continue;

      if (timeout == 0)
        return;

      goto update_timeout;
    }

    have_signals = 0;
    nevents = 0;

    loop->watchers[loop->nwatchers] = (void*) events;
    loop->watchers[loop->nwatchers + 1] = (void*) (uintptr_t) nfds;
    for (i = 0; i < nfds; i++) {
      pe = events + i;
      fd = pe->data.fd;
      w = loop->watchers[fd];
      w->cb(loop, w, pe->events); // 这个for循环很长，这里我简化了，其实就是准备好的IO就执行回调
    }

    if (timeout == 0)
      return;

    if (timeout == -1)
      continue;

update_timeout:
    assert(timeout > 0);

    real_timeout -= (loop->time - base);
    if (real_timeout <= 0)
      return;

    timeout = real_timeout;
  }
}
```

`uv__io_poll`里的timeout参数是根据`timers`里的最近一次定时时间计算出来的。方法内部使用了`epoll`，是IO多路复用的一个概念。一共有三种：`select`、`poll`和`epoll`。

> epoll是[Linux内核](https://baike.baidu.com/item/Linux内核)为处理大批量[文件描述符](https://baike.baidu.com/item/文件描述符/9809582)而作了改进的poll，是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量[并发连接](https://baike.baidu.com/item/并发连接/3763280)中只有少量活跃的情况下的系统[CPU](https://baike.baidu.com/item/CPU/120556)利用率。另一点原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。epoll除了提供select/poll那种IO事件的水平触发（Level Triggered）外，还提供了边缘触发（Edge Triggered），这就使得用户空间程序有可能缓存IO状态，减少epoll_wait/epoll_pwait的调用，提高应用程序效率。 ——百度百科

我觉得libuv使用epoll主要是因为支持其自身的异步特点

- epoll监控的文件描述符远远大于select/poll
- 在大量并发的时候epoll性能远远高于select/poll，而少量异步select/poll相对好点。Node.js利用了libuv的高并发特点。

<br/>

```c
static void uv__run_closing_handles(uv_loop_t* loop) {
  uv_handle_t* p;
  uv_handle_t* q;

  p = loop->closing_handles;
  loop->closing_handles = NULL;

  while (p) {
    q = p->next_closing;
    uv__finish_close(p);
    p = q;
  }
}
```

一个个触发`close`回调

<br/>

上述就是libuv各个阶段的大体流程，`poll`最为复杂，还有很多东西可以留着细品。

<br/>

<br/>



## Node.js setTimeout

上文中提到**那么这里的`handle->timer_cb`是我们平时JavaScript里的setTimeout回调吗？回答：不是的**

看看Node.js里面如何具体实现这个`setTimemout`方法的

```javascript
function setTimeout(callback, after, arg1, arg2, arg3) {
  var args = [arg1, arg2, arg3]; // 简化了代码
  const timeout = new Timeout(callback, after, args, false);
  active(timeout);

  return timeout;
}
function Timeout(callback, after, args, isRepeat) {
  // 省略了部分代码
  after *= 1; 
  if (!(after >= 1 && after <= TIMEOUT_MAX)) {
    after = 1; // 定时验证不过 就为1 
  }
  this._idleTimeout = after;
  this._onTimeout = callback;
  this._timerArgs = args;
}
function active(item) {
  insert(item, true, getLibuvNow());  // getLibuvNow 获取 loop->time时间
}
function insert(item, refed, start) {
  let msecs = item._idleTimeout;
  if (msecs < 0 || msecs === undefined)
    return;

  // Truncate so that accuracy of sub-millisecond timers is not assumed.
  msecs = Math.trunc(msecs);

  item._idleStart = start;

  // Use an existing list if there is one, otherwise we need to make a new one.
  // timerListMap 是一个键值对，key 是定时时间，value 是一个TimersList
  var list = timerListMap[msecs];
  if (list === undefined) {
    const expiry = start + msecs; // 过期时间
    timerListMap[msecs] = list = new TimersList(expiry, msecs); // 双向链表
    timerListQueue.insert(list); // timerListQueue 用数组实现的最小堆

    if (nextExpiry > expiry) {
      scheduleTimer(msecs); // 如果此次定时任务的有效时间小的话，调用 V8 scheduleTimer
      nextExpiry = expiry;
    }
  }
  // ... 这边简略代码
  L.append(list, item); // 将setTimeout创建的Timeout对象添加到list尾部
}
```

通过上述代码，我们也可以看出JavaScript层面也有一个维护timers的最小堆，并没有吧具体的某个setTimeout注册到V8里面，只是将定时时间告诉了V8`scheduleTimer(msecs);`，下面是V8相关代码。

```c
// timers.cc
void ScheduleTimer(const FunctionCallbackInfo<Value>& args) {
  auto env = Environment::GetCurrent(args);
  env->ScheduleTimer(args[0]->IntegerValue(env->context()).FromJust());
}
// env.cc
void Environment::ScheduleTimer(int64_t duration_ms) {
  if (started_cleanup_) return;
  uv_timer_start(timer_handle(), RunTimers, duration_ms, 0);
}
void Environment::RunTimers(uv_timer_t* handle) {
  Environment* env = Environment::from_timer_handle(handle);
  
  // timers_callback_function 从何而来呢？
  Local<Function> cb = env->timers_callback_function();
  do {
    ret = cb->Call(env->context(), process, 1, &arg);
  } while (ret.IsEmpty() && env->can_call_into_js());
}
// uv/timer.c
int uv_timer_start(uv_timer_t* handle,
                   uv_timer_cb cb,
                   uint64_t timeout,
                   uint64_t repeat) {
  // 给uv_timer_t对象初始化相关属性
  // 并没有JavaScript层面的回调方法，具体的回调也只是C++的RunTimers
  handle->timer_cb = cb;
  handle->timeout = clamped_timeout;
  handle->repeat = repeat;
  handle->start_id = handle->loop->timer_counter++;

  // 将uv_timer_t添加到timer_heap上
  heap_insert(timer_heap(handle->loop),
              (struct heap_node*) &handle->heap_node,
              timer_less_than);
  uv__handle_start(handle);

  return 0;
}

```

**V8部分注册在timers阶段的是一个C++的回调方法**，其内部是执行`timers_callback_function`方法，它从何来来？

```c++
// timers.cc
void SetupTimers(const FunctionCallbackInfo<Value>& args) {
  CHECK(args[0]->IsFunction());
  CHECK(args[1]->IsFunction());
  auto env = Environment::GetCurrent(args);

  env->set_immediate_callback_function(args[0].As<Function>()); // 注册了immediate回调
  env->set_timers_callback_function(args[1].As<Function>()); // 注册了timers回调
}
```

```javascript
// node.js
const {
  setupTaskQueue,
  queueMicrotask
} = require('internal/process/task_queues');
const { nextTick, runNextTicks } = setupTaskQueue();
process.nextTick = nextTick;
process._tickCallback = runNextTicks;

const { getTimerCallbacks } = require('internal/timers');
const { setupTimers } = internalBinding('timers'); // c++ SetupTimers 方法传到了JavaScript层
const { processImmediate, processTimers } = getTimerCallbacks(runNextTicks);
setupTimers(processImmediate, processTimers);
```

我们通过上述代码看到在`node.js`里面会将`processImmediate`和`processTimers`这两个JavaScript方法注册到对应的C++回调里，之后执行`timers_callback_function`其实就是执行`processTimers`方法。

另外，需要注意`微任务`相关的代码会被带到`getTimerCallbacks`。

```javascript
function getTimerCallbacks(runNextTicks) {
  function processImmediate() {
    // 之后文章在做介绍
  }


  function processTimers(now) {
    nextExpiry = Infinity;

    let list;
    let ranAtLeastOneList = false;
    while (list = timerListQueue.peek()) {
      if (list.expiry > now) { // 当前列表的过期时间大于now(libuv loop->time), 还没有到过期或到触发时间
        nextExpiry = list.expiry; // nextExpiry 设置为 当前列表的过期时间
        return refCount > 0 ? nextExpiry : -nextExpiry;
      }
      if (ranAtLeastOneList)
        runNextTicks(); // 微任务会被触发
      else
        ranAtLeastOneList = true;
      listOnTimeout(list, now);
    }
    return 0;
  }

  function listOnTimeout(list, now) {
    const msecs = list.msecs;

    debug('timeout callback %d', msecs);

    var diff, timer;
    let ranAtLeastOneTimer = false;
    while (timer = L.peek(list)) { // 如果不是一个空的list，持续执行
      // _idleStart 是插入libuv (loop->time)的时间
      // now 是当前libuv 执行此次callback的时间
      diff = now - timer._idleStart; 

      // Check if this loop iteration is too early for the next timer.
      // This happens if there are more timers scheduled for later in the list.
      if (diff < msecs) {
        list.expiry = Math.max(timer._idleStart + msecs, now + 1);
        list.id = timerListId++;
        timerListQueue.percolateDown(1);
        debug('%d list wait because diff is %d', msecs, diff);
        return;
      }

      if (ranAtLeastOneTimer)
        runNextTicks(); // 微任务会被触发
      else
        ranAtLeastOneTimer = true;

      // The actual logic for when a timeout happens.
      L.remove(timer);

      const asyncId = timer[async_id_symbol];
      if (!timer._onTimeout) {
        continue;
      }
      try {
        const args = timer._timerArgs;
        if (args === undefined)
          timer._onTimeout(); // _onTimeout setTimeout 回调
        else
          timer._onTimeout(...args);
      } finally {
        // ...
      }
    }
    if (list === timerListMap[msecs]) {
      delete timerListMap[msecs];
      timerListQueue.shift();
    }
  }

  return {
    processImmediate,
    processTimers
  };
}
```

上述代码很长，但是很容易理解，就是根据`libuv`的时间，将到期的定时任务一一执行了，格外需要注意的是微任务的执行`runNextTicks`，**每次setTimeout的callback之后都会将`微任务`清空，而不是网上很多文章说的timer阶段之后将微任务清空，这个改动在Node.js 11版本**

> Timers
> Interval timers will be rescheduled even if previous interval threw an error. #20002
> **nextTick queue will be run after each immediate and timer. [#22842](https://github.com/nodejs/node/pull/22842)**

具体的原因是希望和浏览器里的行为更为一直，下面的代码在11之前和之后的执行结果略有不同。

```javascript
setTimeout(() => {
  console.log('time1');
  Promise.resolve().then(() => {
    console.log('promise1');
  });
});

setTimeout(() => {
  console.log('time2');
  Promise.resolve().then(() => {
    console.log('promise2');
  });
});

/* 11之后
time1
promise1
time2
promise2
*/

/* 11之前
time1
time2
promise1
promise2
*/

```



<br/>

<br/>

## 总结

至此，libuv和Node.js EventLoop关于`各个阶段`和`setTimeout`的实现已经做了简单介绍。具体的还是需要看各自版本的代码而定，不能轻易去“相信”网上的介绍，比如最后一个例子就很容易在不同版本出现不同的执行结果。之后，有时间介绍 `nextTick`的回调实现，这个比较复杂。