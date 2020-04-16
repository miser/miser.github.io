---
title: CSS、JS文件对网页的影响
category: JavaScript
---

我们常说浏览器是单线程的，那么我们在加载资源的时候页面是在等待加载完成呢？还是继续执行后续的操作？加载不同资源对浏览器的操作会有相同响应吗？我们可以通过一个一个简单的实验测试来了解。

<!-- more -->

## 环境

*所有资源均在localhost，浏览器chrome 69，不同浏览器或版本会有少许不同*

-- css
  - css1.css 2秒后返回 body { background-color: #444 }
  - css2.css 立即返回 body { font-size: 50px;font-weight: bold; }

-- js
  - js1.js 1秒后返回 console.log("js1.js loaded") 
  - js2.js 立即返回 console.log("js2.js loaded")
  - js3.js 3秒后返回 console.log("js2.js loaded")

-- image
  - img1.png 3秒后 返回一张黄色js图片
  - img2.png 立即返回一张 nodejs图片


服务器端代码：

```js
const Koa = require('koa')
const Router = require('koa-router')
const views = require('koa-views')
const fs = require('fs')

const app = new Koa()
const router = new Router()

app.use(views(__dirname + '/views'))

router.get('/', async (ctx, next) => {
  await ctx.render('index.html')
})

router.get('/css1.css', async ctx => new Promise(resolve => {
  setTimeout(function () {
    ctx.set('Content-Type', 'text/css')
    ctx.body = 'body { background-color: #444 }'
    resolve()
  }, 1000 * 2)
}))

router.get('/css2.css', async ctx => {
  ctx.set('Content-Type', 'text/css')
  ctx.body = 'body { font-size: 50px;font-weight: bold; }'
})

router.get('/js1.js', async ctx => new Promise(resolve => {
  setTimeout(function () {
    ctx.body = 'console.log("js1.js loaded")'
    resolve()
  }, 1000)
}))

router.get('/js2.js', async ctx => {
  ctx.body = 'console.log("js2.js loaded")'
})

router.get('/js3.js', async ctx => new Promise(resolve => {
  setTimeout(function () {
    ctx.body = 'console.log("js3.js loaded")'
    resolve()
  }, 1000 * 3)
}))

router.get('/img1.png', async ctx => new Promise(resolve => {
  setTimeout(function () {
    ctx.set('Content-Type', 'image/png; charset=UTF-8')
    ctx.body = fs.createReadStream('1.png')
    resolve()
  }, 1000 * 3)
}))

router.get('/img2.png', async ctx => {
  ctx.set('Content-Type', 'image/png; charset=UTF-8')
  ctx.body = fs.createReadStream('2.png')
})

app
  .use(router.routes())
  .use(router.allowedMethods())

app.listen(3000)
```

## 实验一： CSS加载是否影响DOM解析

index.html

```html
<!DOCTYPE html>
<html>
<head>
  <title></title>
  <script type="text/javascript">
    console.time('test')
  </script>
  <script type="text/javascript">
    setTimeout(function() {
      const el = document.getElementById('app')
      console.log(el)
    }, 0)
  </script>
  <link rel="stylesheet" type="text/css" href="/css1.css" />
  <link rel="stylesheet" type="text/css" href="/css2.css" />
</head>
<body>
  <div id="app">hello world</div>
  <img style="width: 100px" src="/img1.png" />
  <img style="width: 100px" src="/img2.png" />
</body>
<script type="text/javascript">
  console.timeEnd('test')
</script>
</html>

<!-- console 输出
app元素对象
test: 1947.620849609375ms
-->
```

从控制台的输出可以看到 app 元素会被正确输出，2秒左右再输出 test 的时间，可见__CSS加载并不会阻碍DOM解析__。

## 实验二： CSS加载是否影响JS执行和DOM渲染

index.html

```html
<!DOCTYPE html>
<html>
<head>
  <title></title>
  <script type="text/javascript">
    console.time('test')
  </script>
  <link rel="stylesheet" type="text/css" href="/css1.css" />
  <link rel="stylesheet" type="text/css" href="/css2.css" />
  <script type="text/javascript" src="/js1.js"></script>
  <script type="text/javascript" src="/js2.js"></script>
</head>
<body>
  <div id="app">hello world</div>
  <img style="width: 100px" src="/img1.png" />
  <img style="width: 100px" src="/img2.png" />
</body>
<script type="text/javascript">
  console.timeEnd('test')
</script>
</html>

<!-- console 输出
js1.js loaded
js2.js loaded
(index):17 test: 1921.02099609375ms
-->
```

当打开index.html页面后，会发现所有资源均同时发起了请求，页面会先处于白屏加载状态（DOM无法渲染），当2秒（左右）后页面除img1.png未渲染外，其它样式和图片均渲染。

从控制台的输出可以看出，虽然js比css快1秒左右加载完毕，但是此刻是处于阻塞状态并没有执行，当css加载完成后，才从上至下的执行（虽然js2.js比js1.js早加载好，但是执行的时候还是从上至下的），当css1.css加载完成后，页面立即渲染，图片img1.png晚1秒左右显示。可见__CSS加载会阻碍JS执行和DOM的渲染__。

## 实验三： JS加载是否影响DOM解析和渲染

index.html

```html
<!DOCTYPE html>
<html>
<head>
  <title></title>
  <script type="text/javascript">
    console.time('test')
  </script>
  <script type="text/javascript">
    setTimeout(function() {
      const el = document.getElementById('app')
      console.log(el)
    }, 0)
  </script>
  <script type="text/javascript" src="/js3.js"></script>
</head>
<body>
  <div id="app">hello world</div>
  <img style="width: 100px" src="/img1.png" />
  <img style="width: 100px" src="/img2.png" />
</body>
<script type="text/javascript">
  console.timeEnd('test')
</script>
</html>

<!-- console 输出
null
js3.js loaded
test: 2943.9951171875ms
-->
```

我们仅引入一个js3.js文件，设置它的返回时间为3秒，从控制台的输出可以看到 app 元素没有被正确输出（输出null），3秒左右再输出 test 的时间，可见__JS加载会阻碍DOM解析，既然解析都被影响自然必定影响渲染了__。

## 实验四： DOM的DOMContentLoaded和onLoad事件

index.html

```html
<!DOCTYPE html>
<html>
<head>
	<title></title>
  <script type="text/javascript">
    console.time('test')
    console.time('testDOMContentLoaded')
    console.time('testonload')
  </script>
  <script type="text/javascript">
    document.addEventListener('DOMContentLoaded', function () {
      console.timeEnd('testDOMContentLoaded')
    }, false)
    window.onload = function () {
      console.timeEnd('testonload')
    }
  </script>
  <link rel="stylesheet" type="text/css" href="/css1.css" />
  <link rel="stylesheet" type="text/css" href="/css2.css" />
  <script type="text/javascript" src="/js1.js"></script>
	<script type="text/javascript" src="/js2.js"></script>
</head>
<body>
  <div id="app">hello world</div>
  <img style="width: 100px" src="/img1.png" />
  <img style="width: 100px" src="/img2.png" />
</body>
<script type="text/javascript">
  console.timeEnd('test')
</script>
</html>

<!-- console 输出
js1.js loaded
js2.js loaded
test: 1955.489990234375ms
testDOMContentLoaded: 1956.044189453125ms
testonload: 2964.078857421875ms
-->
```

我们通过控制台会发现 testDOMContentLoaded 会在2秒左右打印出来，testonload会在3秒左右打印出来，由此可知__DOMContentLoaded是js和css文件的加载后触发，onload是整个页面所有资源加载完后触发（比如图片等）__。

## 实验五： script async 属性

index.html

```html
<!DOCTYPE html>
<html>
<head>
	<title></title>
  <script type="text/javascript">
    console.time('testDOMContentLoaded')
  </script>
  <script type="text/javascript">
    document.addEventListener('DOMContentLoaded', function () {
      console.timeEnd('testDOMContentLoaded')
    }, false)
  </script>
  <script type="text/javascript">
    setTimeout(function() {
      const el = document.getElementById('app')
      console.log(el)
    }, 0)
  </script>
  <script async type="text/javascript" src="/js1.js"></script>
  <script async type="text/javascript" src="/js2.js"></script>
  <script async type="text/javascript" src="/js3.js"></script>
</head>
<body>
	<div id="app">hello world</div>
</body>
</html>
```

3个js文件都加上"async",会发现输出顺序是
- testDOMContentLoaded 会立即打印出来
- app元素对象
- js2.js loaded
- js1.js loaded
- js3.js loaded

之后我们将js1.js上的"async"移除，会发现输出顺序是

- el 为 null
- js1.js loaded
- testDOMContentLoaded 1秒左右时间
- js2.js loaded
- js3.js loaded

之后我们将js3.js上的"async"移除，会发现输出顺序是

- el 为 null
- js1.js loaded
- js2.js loaded
- js3.js loaded
- testDOMContentLoaded 3秒左右时间

可见__async会打乱js的执行顺序，有async的js文件哪个先加载完哪个先执行，DOMContentLoaded的触发时间不在和async有关系，不会影响页面的渲染和解析__

## 实验六： script defer 属性

index.html

```html
<!DOCTYPE html>
<html>
<head>
	<title></title>
  <script type="text/javascript">
    console.time('testDOMContentLoaded')
  </script>
  <script type="text/javascript">
    document.addEventListener('DOMContentLoaded', function () {
      console.timeEnd('testDOMContentLoaded')
    }, false)
  </script>
  <script type="text/javascript">
    setTimeout(function() {
      const el = document.getElementById('app')
      console.log(el)
    }, 0)
  </script>
  <script defer type="text/javascript" src="/js1.js"></script>
  <script defer type="text/javascript" src="/js2.js"></script>
  <script defer type="text/javascript" src="/js3.js"></script>
</head>
<body>
	<div id="app">hello world</div>
</body>
</html>

```

3个js文件都加上"defer",会发现输出顺序是

- app元素对象
- js1.js loaded
- js2.js loaded
- js3.js loaded
- testDOMContentLoaded 3秒左右时间

这和没有加defer和async时一样

之后我们将js1.js上的"defer"移除，会发现输出顺序是

- el 为 null
- js1.js loaded
- js2.js loaded
- js3.js loaded
- testDOMContentLoaded 3秒左右时间

这和没有加defer和async时一样

之后我们将js3.js上的"defer"移除，会发现输出顺序是
- js1.js loaded
- js3.js loaded
- js2.js loaded
- testDOMContentLoaded 3秒左右时间

可见__defer会打乱js的执行顺序，有defer的js文件会晚于没有的，但是它们（含有defer）依旧保持从上而下依次执行，DOMContentLoaded的触发时间晚于defer，不会影响页面的渲染和解析__

## 实验七： script defer & async 都加上 属性

index.html

```html
<!DOCTYPE html>
<html>
<head>
  <title></title>
  <script type="text/javascript">
    console.time('testDOMContentLoaded')
  </script>
  <script type="text/javascript">
    document.addEventListener('DOMContentLoaded', function () {
      console.timeEnd('testDOMContentLoaded')
    }, false)
  </script>
  <script async defer type="text/javascript" src="/js1.js"></script>
  <script async defer type="text/javascript" src="/js2.js"></script>
  <script async defer type="text/javascript" src="/js3.js"></script>
</head>
<body>
  <div id="app">hello world</div>
</body>
</html>
```

3个js文件都加上"async",会发现输出顺序是
- testDOMContentLoaded 会立即打印出来
- js2.js loaded
- js1.js loaded
- js3.js loaded

async优先级比defer高

## 实验八： 动态创建script

index.html

```html
<!DOCTYPE html>
<html>
<head>
  <title></title>
  <script type="text/javascript">
    console.time('testDOMContentLoaded')
  </script>
  <script type="text/javascript">
    document.addEventListener('DOMContentLoaded', function () {
      console.timeEnd('testDOMContentLoaded')
    }, false)

    setTimeout(function () {
       var appEl = document.getElementById('app')
       console.log(appEl)
    }, 0)
  </script>
  <script type="text/javascript">
    var head = document.getElementsByTagName('head')[0]
    for (var i = 0; i < 3; i++) {
      var script = document.createElement('script')
      script.type = 'text/javascript'
      script.src = 'js' + (i + 1) + '.js'
      head.appendChild(script)
    }
  </script>
</head>
<body>
  <div id="app">hello world</div>
</body>
</html>
```
- testDOMContentLoaded 会立即打印出来
- appEl 对象
- js2.js loaded
- js1.js loaded
- js3.js loaded

如果我们动态创建js1.js和js2.js，将js3.js依旧按照常规写法写在页面中的话

index.html
```html
<!DOCTYPE html>
<html>
<head>
  <title></title>
  <script type="text/javascript">
    console.time('testDOMContentLoaded')
  </script>
  <script type="text/javascript">
    document.addEventListener('DOMContentLoaded', function () {
      console.timeEnd('testDOMContentLoaded')
    }, false)

    setTimeout(function () {
       var appEl = document.getElementById('app')
       console.log(appEl)
    }, 0)
  </script>
  <script type="text/javascript" src="js3.js"></script>
  <script type="text/javascript">
    var head = document.getElementsByTagName('head')[0]
    for (var i = 0; i < 2; i++) {
      var script = document.createElement('script')
      script.type = 'text/javascript'
      script.src = 'js' + (i + 1) + '.js'
      head.appendChild(script)
    }
  </script>
</head>
<body>
  <div id="app">hello world</div>
</body>
</html>
```
- appEl 为 null 立即打印出来
- js3.js loaded 
- testDOMContentLoaded 3秒左右会立即打印出来
- js2.js loaded
- js1.js loaded

可见__动态创建script基本是同时加载，哪个先加载完哪个先执行，但是他们都晚于DOMContentLoaded事件__


## 总结
CSS加载会阻塞DOM渲染和JS执行，但是不影响页DOM解析；JS加载会阻塞DOM解析和渲染，给script标签加上defer & async属性将不再影响DOM解析和渲染，async 是哪个先返回先执行按个，defer会晚于常规标签同时按照含有defer属性的script加载的顺序执行