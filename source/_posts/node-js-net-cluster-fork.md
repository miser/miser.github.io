---
layout: minos
title: 简单梳理Node.js创建子进程的方法（下）—— cluster
date: 2020-05-03 15:24:53
category: JavaScript
tags:
  - Node.js
  - JavaScript
keywords: child_process,cluster,fork,Node.js,共同监听TCP端口
description: 在默认的RoundRobinHandle模式下，从上可以看出，cluster子进程之所以可以共同监听同个TCP端口，是在net模块里面做了master和child区分，child并没有真正的监听端口，而是child会去master查询该Server是否已经存在，如果没有会在RoundRobinHandle中创建中创建server，一旦有新的TCP连接进入，会转发给free里的worker（cluster.child）处理
---

前文简单梳理了Node.js使用**child_process**模块创建子进程的4种方法，`exec`、`execFile`、`fork`和`spawn`。接下来我们看看**cluster**模块如何创建子进程，后续更多内容会介绍cluster.fork启动Net Server时候为何不会因为共同监听同一个端口而不报错。

**cluster**

- [fork](http://nodejs.cn/api/cluster.html#cluster_cluster_fork_env): 衍生出一个新的工作进程，这只能通过主进程调用。

<!-- more -->

<br/>

**翻翻源码看看他们怎么实现的**

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

**cluster**源码位置

```markdown
lib
  - internal
    - cluster
    - child.js
      - master.js
      - round_robin_handle.js
      - shared_handle.js
      - utils.js
      - worker.js
  - cluster.js
```

<br/>

**官方示例**

```javascript
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  console.log(`主进程 ${process.pid} 正在运行`);

  // 衍生工作进程。
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`工作进程 ${worker.process.pid} 已退出`);
  });
} else {
  // 工作进程可以共享任何 TCP 连接。
  // 在本例子中，共享的是 HTTP 服务器。
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end('你好世界\n');
  }).listen(8000);

  console.log(`工作进程 ${process.pid} 已启动`);
}
```

<br/>

### Q:cluster.isMaster模块如何区分是主进程还是子进程的？

```javascript
// lib/cluster.js
const childOrMaster = 'NODE_UNIQUE_ID' in process.env ? 'child' : 'master';
module.exports = require(`internal/cluster/${childOrMaster}`);
```

`lib/cluster.js`是整个cluster的入口，根据环境变量中是否有`NODE_UNIQUE_ID`来区分主或子进程。

主进程通过`cluster.fork`创建子进程的时候，会将`NODE_UNIQUE_ID`传入子进程的环境变量中，最后通过**child_process.fork**去创建新的子进程。

```javascript
// lib/internal/cluster/master.js
// ...
cluster.isMaster = true;
// ...
// fork的代码下文还会提到
const workerEnv = { ...process.env, ...env, NODE_UNIQUE_ID: `${id}` };
return fork(cluster.settings.exec, cluster.settings.args, {
  // ...
  env: workerEnv,
});

// lib/internal/cluster/child.js
// ...
cluster.isMaster = false;
```

### Q:cluster.fork创建子进程过程中做了些什么？

`cluster.setupMaster`是对整个环境参数的配置；

通过`createWorkerProcess`里的`child_process.fork`去创建子进程；

之后将其和一个`Worker`对象做关联，worker、其子进程和当前cluster(master)都会收到几乎相同的`message`、`exit`和`disconnect`事件，**Worker**这边不多扩展，可以查阅`lib/internal/cluster/worker.js`；

子进程会监听`internalMessage`事件，什么是internalMessage事件呢？看看下面官方的介绍。

> 当发送 **{cmd: 'NODE_foo'}** 消息时有一种特殊情况。 **cmd** 属性中包含 **NODE_** 前缀的消息是预留给 Node.js 内核内部使用的，将不会触发子进程的 **'message'**事件。 相反，这种消息可使用 **'internalMessage'** 事件触发，且会被 Node.js 内部消费。 应用程序应避免使用此类消息或监听**'internalMessage'** 事件，因为它可能会被更改且不会通知。

此处`internalMessage`事件的回调方法是`internal(worker, onmessage)`，**internal**是`lib/internal/cluster/master.js`里的方法，主要作用是判断监听的消息里面是否存在需要执行的回调，如果没有就会执行入参回调，这里指的是**onmessage**；

**onmessage**里面有很多if-else语句，主要是根据cluster.child传送进来的消息类型(`act`)做出不同的处理，这里列出了一个`queryServer`（因为后面会介绍Net Server里多个子进程如何监听同一个端口的）。

```javascript
// lib/internal/cluster/master.js
cluster.fork = function (env) {
  cluster.setupMaster();
  const id = ++ids;
  // 创建子进程
  const workerProcess = createWorkerProcess(id, env);
  //
  const worker = new Worker({
    id: id,
    process: workerProcess, // 新建的子进程
  });
  
  worker.on("message", function (message, handle) {
    cluster.emit("message", this, message, handle);
  });
  worker.process.once("exit", (exitCode, signalCode) => {
    // ...
    worker.state = "dead";
    worker.emit("exit", exitCode, signalCode);
    cluster.emit("exit", worker, exitCode, signalCode);
  });
  worker.process.once("disconnect", () => {
    // ...
    worker.state = "disconnected";
    worker.emit("disconnect");
    cluster.emit("disconnect", worker);
  });

  worker.process.on("internalMessage", internal(worker, onmessage));
  // 触发 fork 事件
  process.nextTick(emitForkNT, worker);
  cluster.workers[worker.id] = worker; 
  return worker;
};

function createWorkerProcess(id, env) {
  const workerEnv = { ...process.env, ...env, NODE_UNIQUE_ID: `${id}` };
  // ... 对一些参数的组合和设定
  // 调用child_process的fork方法创建子进程
  return fork(cluster.settings.exec, cluster.settings.args, {
    cwd: cluster.settings.cwd,
    env: workerEnv,
    silent: cluster.settings.silent,
    windowsHide: cluster.settings.windowsHide,
    execArgv: execArgv,
    stdio: cluster.settings.stdio,
    gid: cluster.settings.gid,
    uid: cluster.settings.uid,
  });
}
function onmessage(message, handle) {
  const worker = this;
  // ... if message.act 类型很多 这里主要讲 queryServer
  if (message.act === "queryServer") queryServer(worker, message);
}
```

```javascript
// lib/internal/cluster/utils.js
// 在cluster的chilid和master里的send都会调用sendHelper
let seq = 0;
function sendHelper(proc, message, handle, cb) {
  if (!proc.connected) return false;
  // NODE_* 开头的命令触发 internalMessage 
  message = { cmd: "NODE_CLUSTER", ...message, seq };

  if (typeof cb === "function") callbacks.set(seq, cb); // 缓存回调

  seq += 1;
  // cluster/child.js handle => null
  // cluster/master.js handle => null
  return proc.send(message, handle);
}
```

以上就是cluster.fork的大致过程，引入一个Worker和internalMessage概念，之后会用得到。cluster的child和master之间传输信息，都是通过`sendHelper`方法。

### Q:cluster.fork创建的子进程如何共同监听TCP端口？

解答这个问题，主要是看`net`模块如何创建Server的，还有就是cluster中`child`和`master`如何通信的。

官方示例虽然用的是`http`创建的服务，但它底层是继承的`net`模块，为了方便梳理，我们从`net.createServer`入手一步步查看源码，主要的逻辑从`listen`开始。

#### Child部分:

cluster.child里创建一个TCP服务，参数`port`是8000，`host`没有传参；

调用`listenInCluster`方法，一看这名字就知道和**cluster**有关；

`listen`是在子进程里触发的，它会通过`cluster._getServer`拼出一个`act`为serverQuery的消息发送给cluster.master；

#### Master部分:

前文提到，`master.onmessage`方法会根据消息的act不同而做出不同的处理，此处正是`serverQuery`；

进入`queryServer`方法，默认使用`RoundRobinHandle`循环分配任务；

在`RoundRobinHandle`构造函数中，会调用`net.createServer`创建一个Server，由于是在cluster.master里调用的，所以会在`listenInCluster`里调用`server._listen2`，会new 出 `TCP`（src/tcp_wrap.cc）作为句柄，并将其赋给`server._handle`，至此cluster.master已经拥有了处理TCP请求的能力，不过master有该能力是不行的，还需要让child拥有才行；

`RoundRobinHandle`中一旦server触发了`listening`事件后，它会接管`server._handle`，用`distribute`重置其`onconnection`方法；

`distribute`的作用就是转发新的请求TCP给cluster.child，从`free`列队中取出一个之前`add`进来的worker（这个worker和cluster.child有关联关系），发送一个`act`为**newconn**的消息让其处理这个TCP；

#### 回到Child部分:

cluster.child的`onconnection`收到`act`为**newconn**的请求后，会找到之前的创建的server，然后调用其`onconnection`（child里的该方法没有被重置）方法，然后封装出一个`Socket`对象，触发`onnection`事件；

到此，后面的就是普通业务逻辑代码了。

*下面贴上了部分相关代码，还是比较多的，细节部分我也加上了注释。*

```javascript
// lib/net.js
function createServer(options, connectionListener) {
  return new Server(options, connectionListener);
}
function Server(options, connectionListener) {
  // ... 大量内置属性和参数的初始化
}
Server.prototype.listen = function (...args) {
  // ...
  // 传了port参数（8000），没有host
  var backlog;
  if (typeof options.port === "number" || typeof options.port === "string") {
    backlog = options.backlog || backlogFromArgs;
    if (options.host) {
      // ...
    } else {
      // 
      listenInCluster(
        this,
        null,
        options.port | 0,
        4,
        backlog,
        undefined,
        options.exclusive
      );
    }
    return this;
  }
	// ...
};


function listenInCluster(
  server,
  address,
  port,
  addressType,
  backlog,
  fd,
  exclusive,
  flags
) {
  exclusive = !!exclusive;

  if (cluster === undefined) cluster = require("cluster");

  if (cluster.isMaster || exclusive) {
    server._listen2(address, port, addressType, backlog, fd, flags);
    return;
  }

  const serverQuery = {
    address: address,
    port: port, 
    addressType: addressType,
    fd: fd,
    flags,
  };
  cluster._getServer(server, serverQuery, listenOnMasterHandle);

  function listenOnMasterHandle(err, handle) {
    // ...
    server._handle = handle;
    server._listen2(address, port, addressType, backlog, fd, flags);
  }
}
```

```javascript
// lib/internal/cluster/child.js
cluster._getServer = function (obj, options, cb) {
  let address = options.address;
	// ...
  // 当前创建的server信息是否之前已经在cluster.child里查询过
  // 有的话就累加计数index值
  const indexesKey = [
    address, 
    options.port,
    options.addressType,
    options.fd,
  ].join(":"); 

  let index = indexes.get(indexesKey);

  if (index === undefined) index = 0;
  else index++;

  indexes.set(indexesKey, index);

  const message = {
    act: "queryServer",
    index,
    data: null,
    ...options,
  };
  
  // ...

  send(message, (reply, handle) => {
    // ...
    if (handle) shared(reply, handle, indexesKey, cb);
    // Shared listen socket.
    else rr(reply, indexesKey, cb); // Round-robin 返回的 handle 是null
  });
  // ...
};
function rr(message, indexesKey, cb) {
  if (message.errno) return cb(message.errno, null);

  var key = message.key;

  function listen(backlog) {
    return 0;
  }

  function close() {
    if (key === undefined) return;
    send({ act: "close", key });
    handles.delete(key);
    indexes.delete(indexesKey);
    key = undefined;
  }

  function getsockname(out) {
    if (key) Object.assign(out, message.sockname);
    return 0;
  }

  const handle = { close, listen, ref: noop, unref: noop };

  if (message.sockname) {
    handle.getsockname = getsockname; // TCP handles only.
  }

  assert(handles.has(key) === false);
  handles.set(key, handle);
  cb(0, handle); // 将封装好的handle，作为listenInCluster的回调handle，赋给server._handle
}
// Round-robin connection.
function onconnection(message, handle) {
  const key = message.key; // maseter里的key
  const server = handles.get(key); // 是client创建的server？
  const accepted = server !== undefined;

  send({ ack: message.seq, accepted }); // 答复 master

  // 虽然cluster.child rr里没有为server绑定，onconnection
  // 但是在cb回到net.js里，后面的逻辑绑定了onconnection方法
  if (accepted) server.onconnection(0, handle);
}
```

```javascript
// lib/internal/cluster/master.js
function queryServer(worker, message) {
  // worker是cluster.child worker
  const key =
    `${message.address}:${message.port}:${message.addressType}:` +
    `${message.fd}:${message.index}`;
  var handle = handles.get(key);

  if (handle === undefined) {
  
    // 默认是RoundRobin，Shared模式暂不讨论有兴趣可以看源码
    var constructor = RoundRobinHandle;
    // ...
    handle = new constructor(
      key,
      address,
      message.port,
      message.addressType,
      message.fd,
      message.flags
    );
    handles.set(key, handle);
  }
  // ...
  // 将cluster.child的worker添加到handle中
  handle.add(worker, (errno, reply, handle) => {
    const { data } = handles.get(key);
    send(
      worker,
      {
        errno,
        key,
        ack: message.seq,
        data,
        ...reply,
      },
      handle // round_robin_handle 里返回的是一个null
    );
  });
}
```

```javascript
// lib/internal/cluster/round_robin_handle.js
function RoundRobinHandle(key, address, port, addressType, fd, flags) {
  this.key = key;
  this.all = new Map();
  this.free = [];
  this.handles = [];
  this.handle = null;
  this.server = net.createServer(assert.fail);

  if (fd >= 0) this.server.listen({ fd });
  else if (port >= 0) {
    this.server.listen({
      port,
      host: address,
      // Currently, net module only supports `ipv6Only` option in `flags`.
      ipv6Only: Boolean(flags & constants.UV_TCP_IPV6ONLY),
    });
  } else this.server.listen(address); // UNIX socket path.

  this.server.once("listening", () => {
    this.handle = this.server._handle;
    // 重置onconnection方法
    // distribute 做任务派发
    this.handle.onconnection = (err, handle) => this.distribute(err, handle);
    this.server._handle = null;
    this.server = null;
  });
}

RoundRobinHandle.prototype.add = function (worker, send) {
  assert(this.all.has(worker.id) === false);
  this.all.set(worker.id, worker);

  const done = () => {
    if (this.handle.getsockname) {
      // tcp\udp 会有getsockname
      const out = {};
      this.handle.getsockname(out);
      // TODO(bnoordhuis) Check err.
      send(null, { sockname: out }, null);
    } else {
      send(null, null, null); // UNIX socket.
    }

    this.handoff(worker); // In case there are connections pending.
  };

  if (this.server === null) return done();

  // Still busy binding.
  this.server.once("listening", done);
  this.server.once("error", (err) => {
    send(err.errno, null);
  });
};

RoundRobinHandle.prototype.distribute = function (err, handle) {
  this.handles.push(handle);
  const worker = this.free.shift();

  if (worker) this.handoff(worker);
};

RoundRobinHandle.prototype.handoff = function (worker) {
  // worker如果不存在那就跳出
  if (this.all.has(worker.id) === false) {
    return; // Worker is closing (or has closed) the server.
  }

  const handle = this.handles.shift(); // 取出第一个待处理任务

  if (handle === undefined) {
    this.free.push(worker);  // 没有的话就会将worker归还到free里
    return;
  }

  const message = { act: "newconn", key: this.key };

  sendHelper(worker.process, message, handle, (reply) => {
    if (reply.accepted) handle.close();
    else this.distribute(0, handle); // Worker is shutting down. Send to another.

    this.handoff(worker);
  });
};
```

### 总结：

`cluster`利用child_process的fork方法创建子进程，并传入新的环境变量`NODE_UNIQUE_ID`用于区分主子进程从而在require('cluster')时候可以加载到对应的`master.js`和`child.js`文件。另外在默认的`RoundRobinHandle`模式下，`cluster`子进程之所以可以共同监听同个TCP端口，是在`net`模块里面做了master和child区分，child并没有真正的监听端口，而是child会去master查询该Server是否已经存在，如果没有会在`RoundRobinHandle`中创建中创建server，一旦有新的TCP连接进入，会转发给`free`里的worker（cluster.child）处理。