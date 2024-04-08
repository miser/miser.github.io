---
title: 前端错误捕获提交错误日志
category: JavaScript
date: 2018-10-23T10:29:29.089Z
---

#### 为什么需要捕获？

前端代码运行在客户端的浏览器里，当客户端（浏览器）出现任何问题，在没有错误日志的情况下，我们都是不知道问题发生在哪，我们只能依靠猜测或者自己不断尝试才知道，或者永远不知道问题。

#### 客户端怎么捕获？

1.通过 window.onerror，可惜只能获得基础的 js 错误，Promise、async/await 里的错误无法捕获，它收到同源决策的影响

2.Promise 通过**catch**方法

3.async/await 通过 **try - catch**

4.Vue 可以通过全局 Vue.config.errorHandler 去获得非 Promise、async/await 里的错误，可以理解为 Vue 里的 window.onerror

<!-- more -->

#### 不同的捕获错误用法（测试环境 chrome & https://jsbin.com）

##### [window.onerror](https://developer.mozilla.org/zh-CN/docs/Web/API/GlobalEventHandlers/onerror)

```javascript
window.onerror = function (message, source, lineno, colno, error) {
  /*
    message：错误信息（字符串）。可用于HTML onerror=""处理程序中的event。
    source：发生错误的脚本URL（字符串）
    lineno：发生错误的行号（数字）
    colno：发生错误的列号（数字）
    error：Error对象（对象）
    */
};
```

```javascript
window.onerror = function () {
  console.log(arguments);
};

let data;
let info = data.info;

/* console 输出
[object Arguments] {
  0: "Uncaught TypeError: Cannot read property 'info' of undefined",
  1: "yiveral.js",
  2: 6,
  3: 17,
  4: [object Error] { ... }
}
*/
```

虽然**onerror**无法捕获 Promise 里的错误，但是如果 Promise 里面是被 setTimeout 包裹的 js 还是能捕获的

```javascript
window.onerror = function () {
  console.log(arguments);
};

function timer() {
  setTimeout(function () {
    let data;
    let info = data.info;
  }, 100);
}

function p() {
  return new Promise(function (resolve, reject) {
    timer();
  }).catch(function (error) {
    console.log(error);
    console.log("inner error");
  });
}

p()
  .then(function () {
    console.log("running then");
  })
  .catch(function (error) {
    console.log(error);
    console.log("outer error");
  });

/* console 输出
[object Arguments] {
  0: "Uncaught TypeError: Cannot read property 'info' of undefined",
  1: "yiveral.js",
  2: 8,
  3: 22,
  4: [object Error] { ... }
}
*/
```

##### [Promise catch](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch)

##### Q：如果没有 catch 方法，是否能捕获 Promise 里的错误？

```javascript
window.onerror = function () {
  console.log(arguments);
  console.log("onerror");
};

function errorFn() {
  let data;
  let info = data.info;
}

function p() {
  return new Promise(function (resolve, reject) {
    errorFn();
  });
}
try {
  p().then(function (res) {
    console.log("running then");
  });
} catch (e) {
  console.log(e);
  console.log("try - catch");
}

/* console 没有任何输出
 */
```

我们通过上面的代码发现，Promise 里的错误无论在**try - catch**还是**onerror**里都无法被捕获

```javascript
function errorFn() {
  let data;
  let info = data.info;
}

function p() {
  return new Promise(function (resolve, reject) {
    errorFn();
  }).catch(function (error) {
    console.log(error);
    console.log("inner error");
    return "return inner error";
  });
}
try {
  p()
    .then(function (res) {
      console.log(res);
      console.log("running then");
    })
    .catch(function (error) {
      console.log(error);
      console.log("outer error");
    });
} catch (e) {
  console.log(e);
  console.log("try - catch");
}

/* console 输出
[object Error] { ... }
"inner error"
"return inner error"
"running then"
*/
```

通过上面代码发现，已经被捕获的错误代码，在外层不会再被捕获而是继续执行 then 里的方法，可见在一条 Promise 链上的错误，会被之后最近的**catch**捕获。

##### [async/await](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/async_function) 通过 **try - catch**

```javascript
window.onerror = function () {
  console.log(arguments);
};

function errorFn() {
  let data;
  let info = data.info;
}

function p() {
  return new Promise(function (resolve, reject) {
    errorFn();
  });
}
(async function () {
  let res = await p();
  console.log(res);
})();

/* console 没有任何输出
 */
```

我们通过上面的代码发现，Promise 构造函数里的错误并没有被**onerror**捕获

```javascript
window.onerror = function () {
  console.log(arguments);
};

function errorFn() {
  let data;
  let info = data.info;
}

function p() {
  return new Promise(function (resolve, reject) {
    resolve("resolve");
  });
}
(async function () {
  let res = await p();
  console.log("get res");
  errorFn();
})();

/* console 输出
get res
*/
```

虽然 Promise 正常执行，但是当后续的代码出错**onerror**依旧没有被捕获

```javascript
function errorFn() {
  let data;
  let info = data.info;
}

function p() {
  return new Promise(function (resolve, reject) {
    errorFn();
  });
}
(async function () {
  try {
    let res = await p();
    console.log(res);
  } catch (e) {
    console.log(e);
    console.log("try - catch");
  }
})();

/* console 输出
[object Error] { ... }
"try - catch"
*/
```

**try - catch**捕获了

```javascript
function errorFn() {
  let data;
  let info = data.info;
}

function p() {
  return new Promise(function (resolve, reject) {
    errorFn();
  }).catch(function (error) {
    console.log(error);
    console.log("inner error");
    return "return inner error";
  });
}
(async function () {
  try {
    let res = await p();
    console.log(res);
  } catch (e) {
    console.log(e);
    console.log("try - catch");
  }
})();

/* console 输出
[object Error] { ... }
"inner error"
"return inner error"
*/
```

从上面代码我们知道，如果 Promise 构造函数里的错误被它自己 catch 的话，那么 async/await 后续的 **try - catch**将不再对它捕获

##### [Vue.config.errorHandler](https://cn.vuejs.org/v2/api/index.html#errorHandler)

```javascript
Vue.config.errorHandler = function (err, vm, info) {
  // handle error
  // `info` 是 Vue 特定的错误信息，比如错误所在的生命周期钩子
  // 只在 2.2.0+ 可用
};
```

> 指定组件的渲染和观察期间未捕获错误的处理函数。这个处理函数被调用时，可获取错误信息和 Vue 实例。

我们该如何去理解官方对 errorHandler 的解释呢？通过 vue-cli 构建工具，创建一个非常基础的 vue 项目，做一些实验。

测试代码库：https://github.com/miser/vue-capture-error

在 main.js

```javascript
Vue.config.errorHandler = function (err, vm, info) {
  console.log(arguments);
  console.log("vue errorHandler");
};
```

在 App.vue

```javascript
{
  // ...
  created () {
    this.normal()
  },
  methods: {
    normal () {
      let data
      let info = data.info
    }
  }
  // ...
}

/* 刷新页面 console 输出
0: TypeError: Cannot read property 'info' of undefined at VueComponent.normal …
1: VueComponent {_uid: 1, _isVue: true, $options: {…}, _renderProxy: Proxy, _self: VueComponent, …}
2: "created hook
*/
```

从上面代码可以看出，errorHandler 确实可以满足我们的需求，在一个统一的地方捕获代码的错误，但是真的如此吗？上文也提到 errorHandler 和 window.onerror 类似，那么当我们使用 Promse 或者 async/await 时会不会得愿以偿。

js 中的异步很大一部分来自网络请求，那么在这我们用 [axios](https://github.com/axios/axios) （它做了一层 ajax 与 Promise 之间的封装）。

main.js 里添加

```javascript
const request = axios.create();
request.interceptors.response.use((response) => {
  return response;
});

Vue.request = (args) => {
  return new Promise((resolve, reject) => {
    request(args)
      .then((res) => {
        resolve(res);
      })
      .catch((err) => {
        reject(err);
      });
  });
};
```

在 App.vue

```javascript
{
  // ...
  created () {
    this.fetch1()
  },
  methods: {
    fetch1 () {
        Vue.request('https://api1.github.com/')
	      .then(response => {
            console.log(response)
          })
    }
  }
  // ...
}
```

api.github.com 会返回 github 的 api 列表，当我们拼错域名，比如上面代码中的 api1.github.com 时，那肯定是无法获得我们想要的，可是 errorHandler 并没有获得该错误，不过幸好，我们可以在全局统一的 Vue.request 里的 catch 方法去统一捕获网络层面的错误。那如果是非网络层面的呢？比如数据请求回来了，但是绑定数据的时候，后端因为业务的修改等原因并没有返回我们需要的字段，造成 Promise.then 方法的业务处理错误。

在 App.vue

```javascript
{
  // ...
  created () {
    this.fetch2()
  },
  methods: {
    fetch2 () {
      Vue.request('https://api.github.com/')
      .then(response => {
        let data = response.data
        let info = data.api.info
      })
    }
  }
  // ...
}
```

上诉代码运行后，errorHandler 同样未能捕获错误，从 vue 的 issue 里面去查询关于捕获 Promise 或者 async/await 时，会得到作者的答复:

> https://github.com/vuejs/vue/issues/6551
>
> Vue cannot capture errors that are thrown asynchronously, similar to how try... catch won't catch async errors. It's your responsibility to handle async errors properly, e.g. using Promise.catch — @yyx990803

那么该怎么办，不可能每个地方都加 Promise.catch 方法吧！

> https://github.com/vuejs/vue/issues/7653
>
> @Doeke 在这个地方给出一个解决方案，通过全局 mixin，给那些 Promise 方法外面包一层 Promise，在这个外层 Promise 链上 catch 里面的错误，不过这样需要做代码的约定，就是原来的方法需要返回一个 Promise 对象。

main.js 里添加@Doeke 的思路

```javascript
Vue.mixin({
  beforeCreate: function () {
    const methods = this.$options.methods || {};
    Object.entries(methods).forEach(([key, method]) => {
      if (method._asyncWrapped) return;
      const wrappedMethod = function (...args) {
        const result = method.apply(this, args);
        const resultIsPromise = result && typeof result.then === "function";
        if (!resultIsPromise) return result;

        return new Promise(async (resolve, reject) => {
          try {
            resolve(await result);
          } catch (error) {
            if (!error._handled) {
              const errorHandler = Vue.config.errorHandler;
              errorHandler(error);
              error._handled = true;
            }
            reject(error);
          }
        });
      };
      wrappedMethod._asyncWrapped = true;
      methods[key] = wrappedMethod;
    });
  },
});
```

在 App.vue

```javascript
{
  // ...
  created () {
    this.fetch2()
    this.fetch3()
  },
  methods: {
    fetch2 () {
      Vue.request('https://api.github.com/')
        .then(response => {
          let data = response.data
          let info = data.api.fetch2
        })
    },
    fetch3 () {
      return Vue.request('https://api.github.com/')
        .then(response => {
          let data = response.data
          let info = data.api.fetch3
        })
    }
  }
  // ...
}
```

通过运行并观察 console 打印可以看出，fetch3 的错误被 errorHandler 捕获，而 fetch2 的错误并没有。

那么 Promise 里的错误统一捕获的问题差不多应该解决了，那么 async/await 的呢？

在 App.vue

```javascript
{
  // ...
  created () {
    this.fetch4()
    this.fetch5()
  },
  methods: {
    async fetch4 () {
      let response = await Vue.request('https://api.github.com/')
      let data = response.data
      let info = data.api.fetch4
    },
    async fetch5 () {
      let response = await Vue.request('https://api.github.com/')
      let data = response.data
      let info = data.api.fetch5
      return response
    }
  }
  // ...
}
```

fetch4 并没有返回 Promise，fetch5 返回的也不是 Promise 对象，但是当运行的时候我们会发现 fetch4 和 fetch5 的错误信息都被捕获了，这是为什么呢？因为 async/await 本身就是 Promise 的语法糖，在 [babeljs](https://babeljs.io) 官网的 “Try it out" 尝试用 async/await，你会发现最后编译后的代码就是在外包了一层 Promise。

#### 在哪里捕获更为优雅？（尽量以更少的代码覆盖大部分或者全部代码）

**网络层**：可以在 axios.create 创建的实例中

**逻辑层**：非 Promise 本身就会被 errorHandler 捕获；Promise 相关的可以通过全局 mixin 给返回 Promise 对象的方法做一个外层包装，统一 catch 并调用 errorHandler 处理（**_这个方法的是否有副作用还需要研究!_**）

#### 捕获的错误存放在哪？

**# 自己简易服务 ？**

感觉成本很大（人力和工时）

**# 官方推荐的 [Sentry](https://sentry.io/) **

注册后安装官方的 JS SDK

```
npm install raven-js --save
```

修改 main.js

```javascript
// ...
import Raven from "raven-js";
import RavenVue from "raven-js/plugins/vue";

Raven.config("https://1dfc5e63808b41058675b4b3aed4cfb6@sentry.io/1298044") // sentry token
  .addPlugin(RavenVue, Vue)
  .install();

Vue.config.errorHandler = function (err, vm, info) {
  Raven.captureException(err);
};
Vue.request = (args) => {
  return new Promise((resolve, reject) => {
    request(args)
      .then((res) => {
        resolve(res);
      })
      .catch((err) => {
        Raven.captureException(err);
        reject(err);
      });
  });
};
// ...
```

修改 App.vue （我们从最普通的 js 测试起）

```javascript
// ...
created () {
    this.normal()
    // this.fetch1()
    // this.fetch2()
    // this.fetch3()
    // this.fetch4()
    // this.fetch5()
}
// ...
```

打开 sentry 页面查看
![错误报告列表](/images/js-capture-error/1.png)
![错误报告详情（模糊了IP部分）](/images/js-capture-error/2.jpg)
我们通过上面 2 张图片可以看出，sentry 自带一个简单的 issue 管理功能，此外详情页面的错误栈已经方便我们知道问题出在哪里了。

测试 fetch1 的 ajax 请求错误
![成功截获api1.github.com这个错误域名](/images/js-capture-error/3.jpg)

除了 fetch2 无法被捕获外（之前提过，它没有返回 Promise 对象），其它的都能被捕获。不过 Promise 和 async/await 的错误栈比较少。尤其是 Promise.then 里的错误，如下 2 张图的对比：

![Promise.then里的错误](/images/js-capture-error/4.jpg)
![async/await里的错误](/images/js-capture-error/5.jpg)

除了默认的数据的收集外，还能收集一些其他数据，比如用户信息

```javascript
Raven.setUser({
  name: "miser name",
  id: "miser id",
});
```

![用户信息顺收集](/images/js-capture-error/6.jpg)

**我们测试了代码未被压缩的情况，如果代码压缩了呢？**

![通过npm run build 压缩代码打开首页，一脸懵逼](/images/js-capture-error/7.jpg)

显然我们不能直观的获得错误定位，不过 sentry 提供[SourceMaps](https://github.com/google/closure-compiler/wiki/Source-Maps)存储服务，它能方便的 debug 被压缩的代码。

我们可以通过[webpack-sentry-plugin](https://www.npmjs.com/package/webpack-sentry-plugin)工具将整个上传过程写进 webpack 里，我们创建一个 vue.config.js 文件

```javascript
const SentryPlugin = require("webpack-sentry-plugin");

module.exports = {
  configureWebpack: {
    plugins: [
      new SentryPlugin({
        organization: "fe-org", // 组织名称 类似公司名吧（一个用户下可以有多个组织）
        project: "popcorn-vue", // 项目名称 （一个组织下可以有多个项目）
        apiKey:
          "17c7d61a800f495c803196e2c02cadeb1b41454247db4f06a5c54193510da150",
        release: "1.2.4-beta", // 发布后的代码和这个对应，可以找到这个sourcemaps
      }),
    ],
  },
};
```

修改 main.js

```javascript
Raven.config("https://1dfc5e63808b41058675b4b3aed4cfb6@sentry.io/1298044", {
  release: "1.2.4-beta", // 新增
})
  .addPlugin(RavenVue, Vue)
  .install();
```

```
npm run build
```

查看 sentry 里 popcorn-vue 项目中的**版本**

![1.2.4-beta是目前的，1.2.3-beta是以前版本](/images/js-capture-error/8.jpg)
![点击 1.2.4-beta 进去，很容易找到刚刚上传的js和js.map文件](/images/js-capture-error/9.jpg)

我们打开 build 完的 index.html，虽然错误成功捕获但依旧和上图的一样，无法被 SourceMaps 解析，大概的原因是 js 和 js.map 的目录结构问题。

这个 issue https://github.com/getsentry/sentry-electron/issues/54 是一个很经典的例子，它犯了 2 个错误

-- 仅仅传了 js.map 而没有传被压缩的 js 文件，它们应该一一对应的上传到服务器上
-- js 和 js.map 目录路径不匹配

这 2 个原因都会导致无法正常解析被压缩的文件。

那么不直接通过浏览器打开 index.html（file:///**\*\***/vue-capture-error/dist/index.html），通过 nginx 去模拟正式环境。

```
brew install nginx
nginx
```

将 build 出的代码 dist 拷贝到 nginx 默认目录下 /usr/local/var/www/，打开浏览器 http://localhost:8080

回到 sentry 中查看新的错误记录

![已经很详细的记录了出错的方法](/images/js-capture-error/10.jpg)
