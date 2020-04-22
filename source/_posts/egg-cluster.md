---
title: Egg Cluster 简单介绍
category: JavaScript
keywords: 进程管理,Eggjs启动,Egg Cluster
description: Egg.js的进程管理和通信自然不会像文章里说的那么简单，但大体如此。弄清楚它们的工作原理对开发程序、插件、中间件有很大的帮助，个人认为这个才是这个框架的精髓之处。
tags:
  - Node.js
  - JavaScript
date: 2019-01-18T13:48:01.464Z
---

如果不清楚什么是Egg.js，希望能移步到它的[官网](https://eggjs.org)简单看下。另外说它是__约定大于配置__的话，我只能说你真的不了解它，或者说不了解框架，哪个框架没有约定？毕竟没有规矩不成方圆，何况是逻辑性的程序呢？官方列出的特性如下：

> 1.提供基于 Egg [定制上层框架](https://eggjs.org/zh-cn/advanced/framework.html) 的能力
> 2.高度可扩展的[插件机制](https://eggjs.org/zh-cn/basics/plugin.html)
> 3.内置[多进程管理](https://eggjs.org/zh-cn/advanced/cluster-client.html)
> 4.基于 [Koa](http://koajs.com/) 开发，性能优异
> 5.框架稳定，测试覆盖率高
> 6.[渐进式开发](https://eggjs.org/zh-cn/tutorials/progressive.html)

第1条，它有那么Koa也有啊。第2条，它有，难道Koa、Express等就没有嘛？第4条，更好的补充了Koa不是更好吗？第5条，难道别的框架就不稳定了？第6条，前端鼓吹渐进式、后端也鼓吹，那究竟什么是渐进式呢？

在我看来最吸引我的是第3条，__内置多进程管理__，这个在其它主流nodejs框架中是稀缺的特性，此文就简单聊聊它。

<!-- more -->



### __#从源码慢慢了解__

- __egg-core__: 定义了一个__EggCore__类，它继承KoaApplication，也就是特性中提到的第4条 *基于Koa开发，性能优异*
- __egg__: 定义了一个继承于EggCore的__EggApplication__类，并且Application和Agent分别继承于EggApplication
- __egg-cluster__: 这个类库主要就是做__多进程管理__的工作

*egg-cluster让Egg.js变得与众不同*，看看它做了什么。

### egg-cluster

```js
// egg-cluster index.js 唯一对外暴露的接口 startCluster
exports.startCluster = function (options, callback) {
    new Master(options).ready(callback)
}
```

egg-cluster就是靠Master在管理egg里面的angent和workers（application）,另外它也是它们之间通信的中转站，看下官网给出的图解：
![Master-Agent-Works 模型](/images/egg-cluster/1.jpg)

### Agent-Works怎么启动的？
```js
class Master extends EventEmitter {
    constructor(options) {
        this.workerManager = new Manager();
    	this.messenger = new Messenger(this);
        // ...
		this.once('agent-start', this.forkAppWorkers.bind(this));
        // ...
        detectPort((err, port) => {
            // 试着找个可以用的port
            this.options.clusterPort = port;
            // 启动 agent
            this.forkAgentWorker();
        });
        // ...
    }
    forkAgentWorker() {
        // ... childprocess.fork egg-cluster/lib/agent_worker.js
        const agentWorker = childprocess.fork(this.getAgentWorkerFile(), args, opt);
        // ...
    }
    forkAppWorkers() {
        // 将需要数量的 worker 一个个创建出来
        // cluster.fork egg-cluster/lib/agent_worker.js 它们将监听同一个服务端口
        // 创建 http或https服务
    }
}
```

agent_worker.js主要逻辑就是创建egg类库里的Agent类，完成后发"agent-start"给父进程，触发Master的订阅创建Workers

```js
// egg-cluster/lib/agent_worker.js
// ...
process.send({ action: 'agent-start', to: 'master' }); 
// ...
```

### Agent-Works怎么通信呢? ([IPC](https://eggjs.org/zh-cn/core/cluster-and-ipc.html))

- 在Master中维护着一个Messenger（egg-cluster/lib/utils/messenger.js）实例
- EggApplication中维护了另一个Messenger（egg/lib/core/messenger.js）实例

由于Agent和Worker(Application)都继承EggApplication，它们调用Messenger的时候会send到创建它们的Master里，然后Master再根据传过来的参数send给不同的Agent或Worker，Master里的转发逻辑如下。

```js
// egg-cluster master.js
class Master extends EventEmitter {
    forkAppWorkers() {
        // ...
        cluster.on('fork', worker => {
            // ...
            worker.on('message', msg => {
            	if (typeof msg === 'string') msg = { action: msg, data: msg };
	            msg.from = 'app';
    	        this.messenger.send(msg);
            });
            // ...
        })
    }
    forkAgentWorker() {
        // ...
        agentWorker.on('message', msg => {
          if (typeof msg === 'string') msg = { action: msg, data: msg };
          msg.from = 'agent';
          this.messenger.send(msg);
        });
        // ...
    }
}
```

```js
// egg messenger.js
class Messenger extends EventEmitter {
    send(action, data, to) {
        sendmessage(process, {
          action,
          data,
          to,
        });
        return this;
     }
}
```

一条信息必定有__from__...__to__...信息

![Master-Agent-Works 通信模型](/images/egg-cluster/2.jpg)

### 官网里提到的“[多进程研发模式增强](https://eggjs.org/zh-cn/advanced/cluster-client.html)”

![一个程序运行n个worker连m个远程服务](/images/egg-cluster/3.jpg)

- n * m 个连接导致大量连接资源“浪费”
- 减少Master转发带来的额外性能消耗
- 另外，egg的作者们担心不当的IPC通信把Master搞挂，从而整个服务异常

所以还有一种socket通信方式（使用了另一个库[cluster-client](https://github.com/node-modules/cluster-client)）：

- 将Agent作为Leader，从服务端获取数据，并做缓存
- 将Worker作为Follower，订阅Agent获取的数据

典型的场景有，Leader（Agent）获取disconf里的配置、获取euerka里的服务等，Follower（Worker）使用这些配置和服务。

![socket通信](/images/egg-cluster/4.jpg)

### cluster-client 源码一瞥

```js
class ClusterClient extends Base {
    // ...
    async [init]() {
        const name = this.options.name;
        const port = this.options.port;
        let server;
        if (this.options.isLeader === true) {
          server = await ClusterServer.create(name, port);
          if (!server) {
            throw new Error(`create "${name}" leader failed, the port:${port} is occupied by other`);
          }
        } else if (this.options.isLeader === false) {
          // wait for leader active
          await ClusterServer.waitFor(port, this.options.maxWaitTime);
        } else {
          debug('[ClusterClient:%s] init cluster client, try to seize the leader on port:%d', name, port);
          server = await ClusterServer.create(name, port);
        }

        if (server) {
          this[innerClient] = new Leader(Object.assign({ server }, this.options));
          debug('[ClusterClient:%s] has seized port %d, and serves as leader client.', name, port);
        } else {
          this[innerClient] = new Follower(this.options);
          debug('[ClusterClient:%s] gives up seizing port %d, and serves as follower client.', name, port);
        }
        // ...
    }
}
```

- port数值就是上文开始处通过detectPort获取的clusterPort数值
- 然后net.create 创建TCP服务，之后所有的Leader和Follower都会走它提供的服务进行socket通信
- Leader获取数据触发publish，传给订阅的Follower中

cluster-client源码是很复杂的，中间还涉及到专递数据的格式，进行数据包的解析等等，这边就不扩展介绍了，有兴趣可以自己撸源码。



### __#总结__

Egg.js的进程管理和通信自然不会像文章里说的那么简单，但大体如此。弄清楚它们的工作原理对开发程序、插件、中间件有很大的帮助，个人认为这个才是这个框架的精髓之处。