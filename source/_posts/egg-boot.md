---
title: Egg Boot
category: javascript
---

### **#Preface**

This article will introduce a web framework called Eggjs for Node.js.

It bases on Koa and customizes your requirement through a large of plugins and middleware, even an own framework. It is very important that you can create a cluster, an agent process, and some workers process when it is run. The cluster makes it stronger. Next, we can understand it by reading the origin code.

Eggjs has a few major libs, egg-core、egg、egg-cluster、egg-bin、egg-scripts and so on.

**egg-core**: it extends Koa and is as a parent object of every agent and worker.

**egg**: it defines some actions for agent and worker,  you can almost use these actions to create an app of a single process.

**egg-cluster**: it creates a cluster and manages them.

**egg-scripts and Egg-bin**: their job is run the whole app in a different environment.

<br/>

_**Tips: We will discuss Eggjs with basing 2.x.x version.**_

<br/>

<!-- more -->

## **#Scan Libs Code**

_* The directory and the code segment are not whole content after this post, these major are only for a better explanation._

### **egg-core**

```
— lib
  — / loader (dir)
  — mixin (dir)
  — utils (dir)
  egg.js
  lifecycle.js
```

The above is the directory structure of egg-core.The _**egg.js**_ and folder of the _**loader**_ are important to point for this lib.

See egg.js

```javascript
const KoaApplication = require('koa')
const Lifecycle = require('./lifecycle')

class EggCore extends KoaApplication {
  constructor(options = {}) {
    // ...
    this.lifecycle = new Lifecycle({
      baseDir: options.baseDir,
      app: this,
      logger: this.console
    })

    const Loader = this[EGG_LOADER]
    assert(Loader, "Symbol.for('egg#loader') is required")
    this.loader = new Loader({
      baseDir: options.baseDir,
      app: this,
      plugins: options.plugins,
      logger: this.console,
      serverScope: options.serverScope,
      env: options.env
    })
    // ...
  }
  beforeStart(scope) {
    this.lifecycle.registerBeforeStart(scope)
  }
  ready(flagOrFunction) {
    return this.lifecycle.ready(flagOrFunction)
  }
  get [EGG_LOADER]() {
    return require('./loader/egg_loader')
  }
}
```

First, we can know why it is called that bases on Koa because EggCore extends KoaApplication.

Second, it defines a few new fields in the construction function, such as _lifecycle_ and _loader_. The _loader_ field helps app for creating important feature include config、plugin、controller、extend、router、middleware、service and so on, it will load some js file in the special directory when the app is starting.

Another side, both _beforeStart_ and _ready_ often are called when we need to write some plugins.

### **egg**

```
— / app
— / config
— / lib
  - / core
    - / messenger
  - / jsdoc
  - / loader
  - agent.js
  - application.js
  - egg.js
  - start.js
- index.js
```

There is an egg.js file that is the same name in the egg-core, but it bases on _EggCore Class_ and extends the lifecycle field, creates new messenger field and cluster field and dumps app config info.

See egg.js

```javascript
const EggCore = require('egg-core').EggCore
const cluster = require('cluster-client')
const Messenger = require('./core/messenger')

class EggApplication extends EggCore {
  constructor(options = {}) {
    this.loader.loadConfig()
    this.messenger = Messenger.create(this)

    // trigger serverDidReady hook when all app workers
    // and agent worker is ready
    this.messenger.once('egg-ready', () => {
      this.lifecycle.triggerServerDidReady()
    })

    this.ready(() =>
      process.nextTick(() => {
        const dumpStartTime = Date.now()
        this.dumpConfig()
        this.dumpTiming()
        this.coreLogger.info(
          '[egg:core] dump config after ready, %s',
          ms(Date.now() - dumpStartTime)
        )
      })
    )

    this.cluster = (clientClass, options) => {
      options = Object.assign({}, this.config.clusterClient, options, {
        singleMode: this.options.mode === 'single',
        // cluster need a port that can't conflict on the environment
        port: this.options.clusterPort,
        // agent worker is leader, app workers are follower
        isLeader: this.type === 'agent',
        logger: this.coreLogger
      })
      const client = cluster(clientClass, options)
      this._patchClusterClient(client)
      return client
    }
  }
}
```

~~We have more discussion about the above code later. I think the cluster field is complex and maybe need to have a new post about it, but other points must know them.~~

See agent.js and application.js

```javascript
// agent.js
const EggApplication = require('./egg')
const AgentWorkerLoader = require('./loader').AgentWorkerLoader
const EGG_LOADER = Symbol.for('egg#loader')

class Agent extends EggApplication {
  constructor(options = {}) {
    options.type = 'agent'
    super(options)
    this.loader.load()
  }
  get [EGG_LOADER]() {
    return AgentWorkerLoader
  }
}

// application.js
const EggApplication = require('./egg')
const AgentWorkerLoader = require('./loader').AppWorkerLoader
const EGG_LOADER = Symbol.for('egg#loader')

class Application extends EggApplication {
  constructor(options = {}) {
    options.type = 'application'
    super(options)
    this.loader.load()
  }
  get [EGG_LOADER]() {
    return AppWorkerLoader
  }
}
```

We can see that they override the _EGG_LOADER_ property, which uses to create a new _loader_ field in the construction of EggCore that is their parent class. Finally, they call the _load_ function.

We have the base application code. rNext, look a few of libs for starting web app.

### **egg-scripts and egg-bin**

Both these libs start the web app in a different environment. Egg-scripts is easier and more clear, uses the production environment. Egg-bin has much code for helping debug, dev, test and so on, we usually use it in the dev, debug or test environment. We almost don't know them at most of time.

```javascript
// package.json
{
  "scripts": {
    "start": "env egg-scripts start",
    "dev": "env egg-bin dev",
    "stop": "egg-scripts stop",
    "debug": "egg-bin debug",
    "test": "npm run lint -- --fix && npm run test-local",
    "test-local": "env egg-bin test",
    "cov": "egg-bin cov"
}
```

The _scripts_ command of this eggjs app conforms to the above description.

### **egg-cluster**

The cluster is an important feature for Eggjs. This lib is a bridge for connecting one single agent process and many workers processes. It like a manager.

```
- / lib
  - / utils
  - agent_worker.js
  - app_worker.js
  - master.js
- index.js
```

I had written an [article](/2019/01/18/egg-cluster) about egg-cluster in Chinese. I always think this is the core of Eggjs, so I will explain the whole Eggjs framework around it.

## **#From start to getting the first request**

**What happens when we input `npm run dev` or `npm start` in the terminal?**

- **egg-scripts(prod):** it can require _framework_ by `child_process.spawn` and calls _startCluster_ function.
- **egg-bin(dev):** it can require _framework_ by `child_process.fork` and calls _startCluster_ function.
- `[parent process]`: The command of running is in one process called the parent process. The system will create a new process called the master process when the parent process requires a framework.As usual, the `framework` is the file path of Eggjs.If you want to use a custom framework, you can add a param in the command, such as `--framework { your path }`

**Pseudo Code**

```javascript
// egg-scripts
// parent process
const spawn = require('child_process').spawn // create new process
spawn('node', 'require({{ framework path }}).startCluster(...)', options))

// egg-bin
// parent process
const cp = require('child_process')
cp.fork('require({{ framework path }}).startCluster(...)', args, options) // create new process
```

Egg-scripts uses `spawn` function to require framework while egg-scripts calls `fork` function.The latter is a special case of the former. Egg-bin has more code than egg-scripts, these features help us for developing or debug the app.

We have two processes. In general, this newly created process is called the master process.

**Why is the master process called birdge?**

```javascript
// egg index.js
exports.startCluster = require('egg-cluster').startCluster
```

Actually, we exec `egg-cluster`'s startCluster function when we require the framework. We open `index.js` file in the egg-cluster lib. 

__First, it is real enter point for whole web app.__

```javascript
// index.js
const Master = require('./lib/master')
exports.startCluster = function(options, callback) {
  new Master(options).ready(callback)
}

// ./lib/master.js
const ready = require('get-ready')
const detectPort = require('detect-port')
const Manager = require('./utils/manager')
const Messenger = require('./utils/messenger')

class Master extends EventEmitter {
  constructor(options) {
    super()
    this.options = parseOptions(options)
    this.workerManager = new Manager()
    this.messenger = new Messenger(this)

    ready.mixin(this)

    this.ready(() => {
      this.isStarted = true
      const action = 'egg-ready'
      this.messenger.send({
        action,
        to: 'parent',
        data: { port: this[REALPORT], address: this[APP_ADDRESS] }
      })
      this.messenger.send({ action, to: 'app', data: this.options })
      this.messenger.send({ action, to: 'agent', data: this.options })

      // start check agent and worker status
      if (this.isProduction) {
        this.workerManager.startCheck()
      }
    })

    // fork app workers after agent started
    this.once('agent-start', this.forkAppWorkers.bind(this))

    detectPort((err, port) => {
      /* istanbul ignore if */
      if (err) {
        err.name = 'ClusterPortConflictError'
        err.message = '[master] try get free port error, ' + err.message
        this.logger.error(err)
        process.exit(1)
      }
      this.options.clusterPort = port
      this.forkAgentWorker()
    })
  }
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


// the callback of ready function will be trigger after all major boots had been loaded

// forkAgentWorker
const agent = new Agent(options)
agent.ready(err => {
  if (err) return
  process.send({ action: 'agent-start', to: 'master' })
})

// fork a single worker
const app = new Application(options);
function startServer(err) {  
  let server;
  if (options.https) {
    const httpsOptions = Object.assign({}, options.https, {
      key: fs.readFileSync(options.https.key),
      cert: fs.readFileSync(options.https.cert),
    });
    server = require('https').createServer(httpsOptions, app.callback());
  } else {
    server = require('http').createServer(app.callback());
  }

  // emit `server` event in app
  app.emit('server', server);
  
  server.listen(...args);
}
app.ready(startServer);
```

On the other hand, this web will be constructed during creating a `Master` object.

- workerManager: it will hold on all worker porcesses.
- messenger: we make `master` a transit station that helps process for communicating (IPC). If you read more code, you can get it. I have said above that it is a bridge. I have write an [article(Chinese)]([http://localhost:4000/2019/01/18/egg-cluster/#Agent-Works%E6%80%8E%E4%B9%88%E9%80%9A%E4%BF%A1%E5%91%A2-IPC](http://localhost:4000/2019/01/18/egg-cluster/#Agent-Works怎么通信呢-IPC)) about it.

> - The Master maintains a Messenger instance (egg-cluster/lib/utils/messenger.js)
> - EggApplication maintain the other Messenge instance （egg/lib/core/messenger.js）
> - Both the agent and worker process base on EggApplication, them can send info to the master process creating them when calling Messenger. The master process is according to entering params to transmit to the agent or worker, you read forkAppWorkers and forkAgentWorker function in the master

- ready.mixin & this.ready: 'get-ready' often is used by the official when an object need trigger a few callbacks after it whole initializes. Here it will broadcast an event to the parent, workers and then agent process, telling them that I am ok.

- detectPort: apply for an available port, default is 7001.If Successly, it will `fork` an agent process and register a callback message for new created an agent object.

- `[agent process]`: create new Agent and run loadPlugin, loadConfig, loadAgentExtend, loadContextExtend, and loadCustomAgent.

  - loadPlugin: find all plugin, record their dir paths => this.dirs

  - loadConfig: merge all config, the config content of the app level is more priority than the framework and the latter is more priority than the plugin.
  - loadAgentExtend: load and merge all of the extending of agent object  (*app > plugin > core*)
  - loadContextExtend: load and merge all of the extending of context object  (*app > plugin > core*)
  - loadCustomAgent: it is important that the lifecycle of app boot will be serially triggered. This lifecycle field is defined in EggCore constructor, we can look at the whole process in the yellow background under the image.【[Application Startup Configuration(official)](https://eggjs.org/en/basics/app-start.html)】. It help you for better writing a few plugins.
  - Last, all major boots had been loaded, the agent process will trigger function registered in the callback array, such as sending `'agent-start'` to the master process.

![egg boot process](/images/egg-boot/egg-boot.jpg)

- `[master process]`: start to fork a few worker processes in accordance with specifying or using the default being CPU kernel count after the master process get `'agent-start'` message from the agent process. It will open a new relatable load, we take a single worker process example.

- `[a single worker process]`: create new Agent and run loadPlugin, loadConfig, loadApplicationExtend, loadRequestExtend, loadResponseExtend, loadContextExtend, loadHelperExtend, loadCustomApp, loadService, loadMiddleware, loadController, loadRouter, and loadCustomLoader.

  - loadPlugin: same as the agent

  - loadConfig: same as the agent

  - loadApplicationExtend: same as the agent  (*app > plugin > core*)

  - loadRequestExtend: load and merge all of the extending of request object (*app > plugin > core*)

  - loadResponseExtend: load and merge all of the extending of response object (*app > plugin > core*)

  - loadContextExtend: same as the agent

  - loadHelperExtend: load and merge all of the extending of helper object (*app > plugin > core*)

  - loadCustomApp: same as the agent （*app > plugin*）

  - loadService: load and merge all of the extending of helper object （*app > plugin*）

  - loadMiddleware: load middlewares and iterate them => mw, if it conforms the middleware standard, it will be used with app.use(mw) （*app > plugin > core*）

  - loadController: iterate all controllers' functions => key, wrap a function, make it be a middleware function and is bound to controller.xxx.xxx (only *app*). This middleware will be triggered when getting a new request, the Controller is defined in the `app/controller ` directory and the key is it's function.

    ```javascript
    function methodToMiddleware(Controller, key) {
      return function classControllerMiddleware(...args) {
        const controller = new Controller(this);
        if (!this.app.config.controller || !this.app.config.controller.supportParams) {
          args = [ this ];
        }
        return utils.callFn(controller[key], args, controller);
      };
    }
    ```

    

  - loadRouter: it makes request’s path associated with the controllers' function that has been become to middleware (only *app*)

  - loadCustomLoader: load ourselves function to create some built-in objects for app object or other.[Customloader](https://eggjs.org/en/advanced/loader.html#customloader)

- `[master process]`: it listens to all worker processes and triggers the master's ready once they have finished booting.
- send `egg-ready` to the parent process, the agent process, the app worker processes
- Last, traverse BOOTS and run serverDidReady function of each item.

It is the whole boot for Eggjs framework but doesn't include, such as restarting a worker process when it exits or disconnects,  shuting down...

__Second, every process is alone if we have not the master. It like a bridge organizing all island, from creation to IPC.__



**What happens when web app get an Http request?**

*We only discuss Eggjs code in the applaction layer. :)*

The `agent` can't deal with any request because it doesn't listen port. All request always hand over to `workers`. We look at creating worker code, it will run a app.callback function and listen port when it has booted.

This callback is the members of Appliaction in Koa. If server get new request, it will create a new context and handle it. In Eggjs, this handleRequest function has been overwrited. Last, the framework will iterate over all middleware under the current route including the converted controller's function.

```javascript
// koa/lib/application
class Application extends Emitter {
  callback() {
    const fn = compose(this.middleware);

    if (!this.listenerCount('error')) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }
  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
}

// egg/lib/application
class Application extends EggApplication {
	handleRequest(ctx, fnMiddleware) {
    this.emit('request', ctx)
    super.handleRequest(ctx, fnMiddleware)
    onFinished(ctx.res, () => this.emit('response', ctx))
  }
}
```



__In summary, we have know how to boot web app and deal with request in Eggjs.__ When we know it, we can easily write some plugins and middlewares to finish the business requirements.

