---
title: 简单梳理Node.js创建子进程的方法（上）
category: JavaScript
keywords: Node.js,子进程创建,Node.js源码
description: Node.js 创建子进程的方法常用的有如下几种。child_process, exec、execFile、fork、spawn；cluster, fork。
tags:
  - Node.js
  - JavaScript
date: 2020-04-06T08:09:36.472Z
---

**Node.js 创建子进程的方法常用的有如下几种：**

**child_process**

- [exec](http://nodejs.cn/api/child_process.html#child_process_child_process_exec_command_options_callback): 衍生一个 shell 然后在该 shell 中执行 `command`，并缓冲任何产生的输出，最大缓存 1024\*1024 个字节。
- [execFile](http://nodejs.cn/api/child_process.html#child_process_child_process_execfile_file_args_options_callback): 函数类似`exec`，但默认情况下不会衍生 shell。 相反，指定的可执行文件`file` 会作为新进程直接地衍生，使其比`exec` 稍微更高效。和`exec`一样，它也有最大 1024\*1204 个字节的显示缓存。
- [fork](http://nodejs.cn/api/child_process.html#child_process_child_process_fork_modulepath_args_options): 是 `spawn`的一个特例，专门用于衍生新的 Node.js 进程。 与`spawn`一样返回`ChildProcess`对象。 返回的`ChildProcess`将会内置一个额外的通信通道，允许消息在父进程和子进程之间来回传递。
- [spawn](http://nodejs.cn/api/child_process.html#child_process_child_process_spawn_command_args_options): 上诉的几个方法其实都是通过 spawn 实现的。

**cluster**

- [fork](http://nodejs.cn/api/cluster.html#cluster_cluster_fork_env): 衍生出一个新的工作进程，这只能通过主进程调用。

<!-- more -->

<br/>

### **翻翻源码看看他们怎么实现的**

源码版本和之前的[libuv & Node.js EventLoop （一）](https://miser.github.io/2020/03/01/v8-libuv-timer-event-loop/)一样

```markdown
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

**child_process**源码位置

```javascript
Node 
  - lib 
  	- internal 
  		- child_process.js 
  	- child_process.js;
```

在`lib/child_process.js`文件中，定义了`exec`、`execFile`、`fork`和`spawn`等方法，它们最后都会调用在`lib/internal/child_process.js`文件中的`spawn`方法。

<br/>

**exec**

```javascript
// lib/child_process.js
function exec(command, options, callback) {
  const opts = normalizeExecArgs(command, options, callback);
  return module.exports.execFile(opts.file, opts.options, opts.callback);
}
function normalizeExecArgs(command, options, callback) {
  if (typeof options === "function") {
    callback = options;
    options = undefined;
  }

  // Make a shallow copy so we don't clobber the user's options object.
  options = { ...options };
  options.shell = typeof options.shell === "string" ? options.shell : true;

  return {
    file: command,
    options: options,
    callback: callback,
  };
}
```

从上面发现 exec 其实就是封装了参数，主要是开启`shell`参数，然后调用 execFile 方法。

<br/>

**execFile**

```javascript
// lib/child_process.js
function execFile(file /* , args, options, callback */) {
  let args = [];
  let callback;
  let options;

  // ...

  options = {
    encoding: "utf8",
    timeout: 0,
    maxBuffer: MAX_BUFFER,
    killSignal: "SIGTERM",
    cwd: null,
    env: null,
    shell: false, // execFile 默认是不开启shell
    ...options,
  };

  // 通过spawn方法创建子进程
  const child = spawn(file, args, {
    cwd: options.cwd,
    env: options.env,
    gid: options.gid,
    uid: options.uid,
    shell: options.shell,
    windowsHide: !!options.windowsHide,
    windowsVerbatimArguments: !!options.windowsVerbatimArguments,
  });

  var encoding;
  const _stdout = []; // 输出的内容
  const _stderr = []; // 出错的内容
  // ...
  var stdoutLen = 0; // 输出内容的长度
  var stderrLen = 0; // 出错内容的长度
  var killed = false; // 是否已经被杀死
  var exited = false; // 是否已经退出
  var timeoutId; // 是否有定时器

  var ex = null; // 出错的上下文对象

  var cmd = file; // 命令或文件

  // 退出回调方法
  function exithandler(code, signal) {
    if (exited) return;
    exited = true;

    if (timeoutId) {
      clearTimeout(timeoutId);
      timeoutId = null;
    }

    if (!callback) return;

    // merge chunks
    var stdout;
    var stderr;
    if (encoding || (child.stdout && child.stdout.readableEncoding)) {
      stdout = _stdout.join("");
    } else {
      stdout = Buffer.concat(_stdout); // 如果不传入 encoding 参数，默认是Buffer拼接输出
    }
    if (encoding || (child.stderr && child.stderr.readableEncoding)) {
      stderr = _stderr.join("");
    } else {
      stderr = Buffer.concat(_stderr); // 如果不传入 encoding 参数，默认是Buffer拼接输出
    }

    if (!ex && code === 0 && signal === null) {
      callback(null, stdout, stderr); // 没有错误，执行回调
      return;
    }

    if (args.length !== 0) cmd += ` ${args.join(" ")}`;

    if (!ex) {
      // eslint-disable-next-line no-restricted-syntax
      ex = new Error("Command failed: " + cmd + "\n" + stderr);
      ex.killed = child.killed || killed;
      ex.code = code < 0 ? getSystemErrorName(code) : code;
      ex.signal = signal;
    }

    ex.cmd = cmd;
    callback(ex, stdout, stderr); // 有错误，执行回调
  }

  function errorhandler(e) {
    ex = e;

    // child.stdout 和 child.stderr 都销毁
    if (child.stdout) child.stdout.destroy();

    if (child.stderr) child.stderr.destroy();

    exithandler();
  }

  function kill() {
    if (child.stdout) child.stdout.destroy();

    if (child.stderr) child.stderr.destroy();

    killed = true;
    try {
      child.kill(options.killSignal);
    } catch (e) {
      ex = e;
      exithandler();
    }
  }

  // 如果定义了子进程的超时时间，就定时销毁它
  if (options.timeout > 0) {
    timeoutId = setTimeout(function delayedKill() {
      kill();
      timeoutId = null;
    }, options.timeout);
  }

  if (child.stdout) {
    if (encoding) child.stdout.setEncoding(encoding);

    child.stdout.on("data", function onChildStdout(chunk) {
      const encoding = child.stdout.readableEncoding;
      const length = encoding
        ? Buffer.byteLength(chunk, encoding)
        : chunk.length;
      stdoutLen += length; // 不断累加输出的长度

      if (stdoutLen > options.maxBuffer) {
        // 超过的话就报错
        const truncatedLen = options.maxBuffer - (stdoutLen - length);
        _stdout.push(chunk.slice(0, truncatedLen));

        ex = new ERR_CHILD_PROCESS_STDIO_MAXBUFFER("stdout");
        kill();
      } else {
        _stdout.push(chunk); // 不断累积chunk
      }
    });
  }

  // 整体逻辑和上面的child.stdout基本一致
  if (child.stderr) {
    if (encoding) child.stderr.setEncoding(encoding);

    child.stderr.on("data", function onChildStderr(chunk) {
      const encoding = child.stderr.readableEncoding;
      const length = encoding
        ? Buffer.byteLength(chunk, encoding)
        : chunk.length;
      stderrLen += length;

      if (stderrLen > options.maxBuffer) {
        const truncatedLen = options.maxBuffer - (stderrLen - length);
        _stderr.push(chunk.slice(0, truncatedLen));

        ex = new ERR_CHILD_PROCESS_STDIO_MAXBUFFER("stderr");
        kill();
      } else {
        _stderr.push(chunk);
      }
    });
  }

  child.addListener("close", exithandler);
  child.addListener("error", errorhandler);

  return child;
}
```

`exec`和`execFile`最大的一个区别就是参数`shell`默认是否开启，其它基本都是相同的。另外，它们对输出的内容有大小限制是在`child.stderr.on('data')`和`child.stdout.on('data')`获取数据时候被限制。

如果不做类似的规定，`_stderr`和`_stdout`无限被输出，那么内存会不断的膨胀导致性能问题，甚至程序奔溃。（这是我猜的原因）

另外说到底，**最后还是对`spawn`方法的封装了调用。**

<br/>

**fork**

```javascript
// lib/child_process.js
function fork(modulePath /* , args, options */) {
  // Get options and args arguments.
  var execArgv;
  var options = {};
  var args = [];

  // ...

  // fork方法的stdio参数，必须带有一个ipc参数
  if (typeof options.stdio === "string") {
    options.stdio = stdioStringToArray(options.stdio, "ipc");
  } else if (!Array.isArray(options.stdio)) {
    // Use a separate fd=3 for the IPC channel. Inherit stdin, stdout,
    // and stderr from the parent if silent isn't set.
    options.stdio = stdioStringToArray(
      options.silent ? "pipe" : "inherit",
      "ipc"
    );
  } else if (!options.stdio.includes("ipc")) {
    throw new ERR_CHILD_PROCESS_IPC_REQUIRED("options.stdio");
  }

  options.execPath = options.execPath || process.execPath; // 默认用父进程的Node执行文件
  options.shell = false; // 不开启 shell

  return spawn(options.execPath, args, options);
}
```

从上面代码有个非常引人注意就是 12~22 行，fork 方法的 stdio 参数，必须带有一个 ipc 参数，这个`ipc`的作用将在后续深入挖掘后介绍。最后也是调用`spawn`创建子进程。

<br/>

**spawn**

```javascript
// lib/child_process.js
function spawn(file, args, options) {
  const child = new ChildProcess();

  options = normalizeSpawnArguments(file, args, options);
  debug("spawn", options);
  child.spawn(options);

  return child;
}
```

`lib/child_process.js`里的 spawn 方法就简单的将传入的参数做整理，然后直接调用 ChildProcess 实例对象的 spawn。

<br/>

**ChildProcess**

```javascript
// `lib/internal/child_process.js`
function ChildProcess() {
  EventEmitter.call(this);

  // ...

  this._handle = new Process(); // new 出一个 ProcessWrap 对象 （ process_wrap.cc）
}

ChildProcess.prototype.spawn = function (options) {
  let i = 0;

  // 默认使用 pipe
  let stdio = options.stdio || "pipe";

  // 重置stdio变量，这个是个很重要的方法，后续将继续介绍
  stdio = getValidStdio(stdio, false);

  const ipc = stdio.ipc;
  const ipcFd = stdio.ipcFd;
  stdio = options.stdio = stdio.stdio;

  // ...

  // 调用ProcessWrap对应的spawn去创建新进程
  const err = this._handle.spawn(options);

  // 如果err存在，就会有相关代码处理创建子进程出错的场景
  // ...

  this.pid = this._handle.pid;

  for (i = 0; i < stdio.length; i++) {
    const stream = stdio[i];
    // ...
    if (stream.handle) {
      // When i === 0 - we're dealing with stdin
      // (which is the only one writable pipe).
      // 创建 socket
      stream.socket = createSocket(
        this.pid !== 0 ? stream.handle : null,
        i > 0
      );

      if (i > 0 && this.pid !== 0) {
        this._closesNeeded++;
        stream.socket.on("close", () => {
          maybeClose(this);
        });
      }
    }
  }
  this.stdin =
    stdio.length >= 1 && stdio[0].socket !== undefined ? stdio[0].socket : null;
  this.stdout =
    stdio.length >= 2 && stdio[1].socket !== undefined ? stdio[1].socket : null;
  this.stderr =
    stdio.length >= 3 && stdio[2].socket !== undefined ? stdio[2].socket : null;

  this.stdio = [];

  for (i = 0; i < stdio.length; i++)
    this.stdio.push(stdio[i].socket === undefined ? null : stdio[i].socket);

  // Add .send() method and start listening for IPC data
  // setupChannel 方法很长，主要就是实现了数据的发送和接受
  if (ipc !== undefined) setupChannel(this, ipc);

  return err;
};

function stdioStringToArray(stdio, channel) {
  // 主要将stdio参数格式化为[xxx,xxx,xxx]形式的数组
  // 如果有channel，[xxx,xxx,xxx,channel]
}

function getValidStdio(stdio, sync) {
  var ipc;
  var ipcFd;

  stdio = stdio.reduce((acc, stdio, i) => {
    function cleanup() {
      for (var i = 0; i < acc.length; i++) {
        if ((acc[i].type === "pipe" || acc[i].type === "ipc") && acc[i].handle)
          acc[i].handle.close();
      }
    }

    if (stdio === "ignore") {
      acc.push({ type: "ignore" }); // 子进程的输出不需要在控制台显示
    } else if (stdio === "pipe" || (typeof stdio === "number" && stdio < 0)) {
      var a = {
        type: "pipe",
        readable: i === 0, // 0是stdin，需要读
        writable: i !== 0, // 1是stdout，2是stderr 需要写
      };
      // spawn 调用的时候，sync为false，为stdin、stdout、stderr创建一个SOCKET类型Pipe
      if (!sync) a.handle = new Pipe(PipeConstants.SOCKET);

      acc.push(a);
    } else if (stdio === "ipc") {
      // 创建一个IPC类型Pipe
      ipc = new Pipe(PipeConstants.IPC);
      ipcFd = i; // ipc位置

      acc.push({
        type: "pipe",
        handle: ipc,
        ipc: true,
      });
    } else if (stdio === "inherit") {
      acc.push({
        type: "inherit",
        fd: i,
      });
    }
    // ...还有很多代码，不在讨论范围
    return acc;
  }, []);

  return { stdio, ipc, ipcFd };
}
```

`ChildProcess`并不能直接创建新的进程，需要底层 V8 的帮助，在构造函数里面直接 new ProcessWrap 赋给了 this.\_handle。

`ChildProcess.prototype.spawn`开始主要处理主要的[`stdio`](http://nodejs.cn/api/child_process.html#child_process_options_stdio)参数，明确父子进程通过哪些方式来获取数据信息，官方文档给出了一些示例，如果不清楚可以多做点实验。如果是`pipe`或者是`ipc`都会实例化一个 Pipe 对象，只是参数类型不同。

```c
// pipe_wrap.cc
void PipeWrap::New(const FunctionCallbackInfo<Value>& args) {
  switch (type) {
    case SOCKET:
      provider = PROVIDER_PIPEWRAP;
      ipc = false;
      break;
    case IPC:
      provider = PROVIDER_PIPEWRAP;
      ipc = true;
      break;
  }
  new PipeWrap(env, args.This(), provider, ipc);
}
PipeWrap::PipeWrap(Environment* env,
                   Local<Object> object,
                   ProviderType provider,
                   bool ipc)
    : ConnectionWrap(env, object, provider) {
  int r = uv_pipe_init(env->event_loop(), &handle_, ipc);
}
// deps/uv/src/unix/pipe.c
int uv_pipe_init(uv_loop_t* loop, uv_pipe_t* handle, int ipc) {
  uv__stream_init(loop, (uv_stream_t*)handle, UV_NAMED_PIPE);
  handle->shutdown_req = NULL;
  handle->connect_req = NULL;
  handle->pipe_fname = NULL;
  handle->ipc = ipc;
  return 0;
}
```

它们唯一的区别就是`uv_pipe_t`的 ipc 参数是 true 还是 false，**所有的事件都被老老实实的绑定在 libuv 上**。

回到`ChildProcess.prototype.spawn`中，已经重置了 stdio 参数后，到了真正创建子进程的地方了，`this._handle.spawn(options);`，通过`process_wrap.cc`里的 ProcessWrap.Spawn 去创建，整个创建方法也极长，主要是对传入的参数进行处理，然后再调用`uv_spawn`。

```c
// deps/uv/src/unix/process.c
int uv_spawn(uv_loop_t* loop,
             uv_process_t* process,
             const uv_process_options_t* options) {
#if defined(__APPLE__) && (TARGET_OS_TV || TARGET_OS_WATCH)
  /* fork is marked __WATCHOS_PROHIBITED __TVOS_PROHIBITED. */
  return UV_ENOSYS;
#else
  // ...

  uv_signal_start(&loop->child_watcher, uv__chld, SIGCHLD);

  /* Acquire write lock to prevent opening new fds in worker threads */
  uv_rwlock_wrlock(&loop->cloexec_lock);
  pid = fork();

  if (pid == -1) {
    err = UV__ERR(errno);
    uv_rwlock_wrunlock(&loop->cloexec_lock);
    uv__close(signal_pipe[0]);
    uv__close(signal_pipe[1]);
    goto error;
  }

  if (pid == 0) {
    uv__process_child_init(options, stdio_count, pipes, signal_pipe[1]);
    abort();
  }

  /* Release lock in parent process */
  uv_rwlock_wrunlock(&loop->cloexec_lock);
  uv__close(signal_pipe[1]);

  // ...
}
```

通过系统层面的`fork`函数创建子进程，由于 fork 的特殊性，一次调用返回二次，当返回 0 的时候回执行子进程的逻辑，回去通过`uv__process_child_init`初始化整个子进程的上下文信息。

再回到`ChildProcess.prototype.spawn`中，遍历`stdio`，如果成员有 handle 字段，就通过`createSocket`为其创建一个 socket 对象

```javascript
function createSocket(pipe, readable) {
  return net.Socket({ handle: pipe, readable, writable: !readable });
}
```

无论是参数`stdio`是 pipe 还是 ipc 都会创建 socket，在父子进程通信的时候，父进程通过子进程暴露出来的 stdin、stdout 和 stderr 来展示子进程执行的信息，缺乏之间的数据互通性，这也是导致`exec`和`execFile`可使用的场景有限，而`fork`会带一个 ipc 参数给 stdio 参数（可以回过去翻翻 fork 源码），所以可以执行父子进程的通信操作，比如 send 方法等，具体的实现可以看`setupChannel`。

<br/>

<br/>

**综上，**Node.js 在`child_process`里创建进程的流程大致梳理了下，在 JavaScript 层面并没有什么复杂的，在 libuv 层面注册了很多相关的事件，有空可以研究研究。之后会写一篇关于 cluster.fork 的介绍，其实就是对 child_process.fork 更多的封装。
