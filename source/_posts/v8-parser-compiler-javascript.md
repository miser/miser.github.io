---
title: V8是如何怎么处理JavaScript的
category: JavaScript
keywords: V8,Node.js,解析编译
description: Parer和Compiler是2个重要的过程和概念，理解它们可以帮助开发者根据业务需求写出对V8或其它JavaScript引擎更为“友善”的代码，毕竟花在这两个过程中的成本是巨大的。
tags:
  - Node.js
  - JavaScript
date: 2020-02-08T09:18:53.485Z
---

> 此文介绍的内容已经算是`旧闻`了，17年的时候就有大量的文章介绍过了，只是2020年伊始的疫情把人困在家里实在无聊，重新翻来几个视频打发下时间，以下文字算是简单梳理，更多的瑰宝需要我们自己翻阅资料研究，**文章中的很多数据应该都过时了吧，仅用来参考吧。**



**Parer**和**Compiler**是2个重要的过程和概念，理解它们可以帮助开发者根据业务需求写出对V8或其它JavaScript引擎更为“友善”的代码，毕竟花在这两个过程中的成本是巨大的。

![国外几大网站花在Parser上的时间大约在15-20%](/images/v8-parser-compiler/v8-pare-compile-cost.png)

<!-- more -->



# **Parser**

![红色部分就是接下来讨论的Parser部分](/images/v8-parser-compiler/parser-phase.png)

**解析速率大约为 `1 MB / s`**

<br/>

#### **V8的Parser分2次解析：**

`Layze（Pre-Parsing）：`

- 跳过还未被使用的代码
- 不会生成`AST`，会产生不带有变量引用和声明的`Scopes`信息
- 解析速度是Eage的2倍
- 根据JavaScript规范抛出一些特定的错误

`Eage（Full-Parsing）：`

- 解析那些被使用的代码
- 生成`AST`
- 构建具体的`Scopes`信息，变量的引用、声明等
- 抛出所有的语法错误

<br/>

#### Q:为什么会有2次解析？

如果都是用Full-Parsing的话，那么整个解析会非常漫长浪费时间。我们通过DevTools的`coverage`工具可以发现页面上大量的代码并没有被使用

<br/>

#### Q:2次解析会有什么负面影响？

如果代码已经被Pre-Parsing解析过了，当被执行的时候还是会被Full-Parser解析一次那么开销是: `0.5 * parse + 1 * parse = 1.5 parse` 从某个角度来说更复杂了、开销更大了。鱼和熊掌不可兼得！

<br/>

#### Q:什么样的代码会被 `Pre-Parsing` 处理，什么样的会被 `Full-Parsing` 处理？

```javascript
let a = 0; // Top-Level 顶层的代码都是 Full-Parsing

// 立即执行函数表达式 IIFE = Immediately Invoked Function Expression
(function eager() {...})(); // 函数体是 Full-Parsing
                   
// 顶层的函数非IIFE
function lazy() {...} // 函数体是 Pre-Parsing

lazy(); // -> Full-Parsing 开始解析和编译！
                 
// 强制触发Full-Parsing解析
!function eager2() {...}, function eager3() {...} // All eager
 
let f1 = function lazy() { ... }; // 函数体是 Pre-Parsing
              
let f2 = function lazy() {...}(); // 先触发了lazy 解析, 然后又eager解析
```

<br/>

#### Q:如何强制Full-Parsing （eager ）？

- lazy 预编译由前2位首字母决定；所以如果我们想跳过 lazy 触发 eager 编译，我们应该在前面加位操作符，例如'!|~'。
- 使用 [optimize-js](https://github.com/nolanlawson/optimize-js) 重新编译代码，具体的性改变可以参考它Github里的测试数据

<br/>

#### Q:什么是连续重新解析

```javascript
function lazy_outer() { // lazy parse this
  function inner() {
    function inner2() {
      // ...
    }
  }
}

lazy_outer(); // lazy parsing inner & inner2
inner(); // lazy parsing inner & inner2 (3rd time!)
```

从上可知，大量的深度内嵌的代码对解析有着性能影响，每一层的深度调用都会引发新一轮的`Pre-Parsing`。

<br/>

#### Q:既然Parser阶段会性能消耗很大，我们该怎么优化代码？

1. **尽量减少代码**，可以通过DevTools的`coverage`工具查看当前页面代码的使用率。
2. V8会缓存Parser阶段的结果并保存72小时，如果bundle中间有部分代码被修改了，那么整个bundle的Parser缓存都会失效，所以把经常变动的打包在一起，非经常变动的在一起，比如公共类库和业务代码分离。

<br/>

#### Q:如何评估当前网站代码的Parser时间呢？（包括后面将的Compiler时间）

1.使用 `chrome://tracing/`工具，我们以[爱眠物](http://aimianwu.com)为例

2.打开tracing界面，点击左上角的`Record`按钮

3.在弹出的界面中选择`Web developer`和在Edit categories里面选择`v8.runtime_stats`

4.在新的Tab中输入 http://aimianwu.com ，回到tracing Tab

5.等待数据采集完成，然后"停止"记录

6.选择对应的Process和数据

![](/images/v8-parser-compiler/parse-time.png)

![](/images/v8-parser-compiler/parse-time-1.png)

7.查看V8里的Parse和Compile数据

![](/images/v8-parser-compiler/parse-time-2.png)

<br/>

<br/>

# **Compiler Pipeline**

随着V8的迭代，整个Compiler Pipeline也在发生翻天覆地的变化。最近的一次大更新是在V8 5.9版本，用 Ignition + TurboFan 代替了从2010一直服务的Full-codegen + Crankshaft组合，当然整个过程也不是一蹴而就的，中间夹杂着特殊的版本， Ignition + TurboFan 、Full-codegen + Crankshaft它们以特殊的方式共存了一段时间，因为一开始TurboFan的性能无法满足需求，可以看[这个PPT](https://docs.google.com/presentation/d/1chhN90uB8yPaIhx_h2M3lPyxPgdPmkADqSNAoXYQiVE/edit#slide=id.g18d89eb289_1_389)。

<br/>

#### Q:为什么做了替换？[官方介绍](https://v8.dev/blog/launching-ignition-and-turbofan)
因为老的版本比较激进，直接将JavaScript翻译成了机器码，在执行性能上确实很快，但是带来了几个大问题

* 直接将JavaScript编译成机器码既费时间又费内存，几乎占用了V8约1/3的堆内存，导致实际可被使用的内存减少；另外由于复杂的设计导致Crankshaft重复编译代码，拖累性能。
* Crankshaft没有友善处理 try、catch、finally 等关键词 ；维护成本高，需要为多个芯片架构提供优化代码，但性能提升不够明显；对ES新的语法特性支持不够好、也无法支持WebAssembly；
在PC端老的组合感受还好，但是在移动端随着网页的不断复杂化，该组合的启动时间和性能慢慢有些力不从心了。

<br/>

#### Q:什么样的代码对 V8 Compiler 友好？

虽然JavaScript是动态语言，如橡皮泥一样随意被开发者“随意”塑造、快速开发出一个又一个应用，但是没有规矩就会带来混乱，增加编译器的优化负担，目前在V8中优化工作由TurboFan完成。Ignition会收集大量信息交给TurboFan去优化，多方面条件都满足的情况下会被优化成机器码，这个过程成为`Optimize`，当判断无法优化时就触发去优化——Deoptimize，这些代码逻辑又重新回到Ignition中成为字节码。

主要有以下2点[视频](https://www.youtube.com/watch?v=p-iiEDtpy6I)

* 自然是经常被调用的代码部分
* 不要总是在改变对象类型（虽然JavaScript是动态的）

![](/images/v8-parser-compiler/not-change-types-1.png)

![](/images/v8-parser-compiler/not-change-types-2.png)

![](/images/v8-parser-compiler/not-change-types-3.png)

> 如果你总是在改变Objects，V8无法对它做优化。即使做了优化也会被De-optimisation，这意味着会有性能损失。

```javascript
function load(obj) {
  return obj.x
}

var obj = {
  x: 1,
  y: 4
} 
```



对编译器而言 `obj = {}` 是一种类型， `obj = { x: "Number" }` 是另一种类型，`obj = { y: "Number" }` 又是一种类型等等，也就说数据类型和字段名必须一致，如果用过静态语言比如C++、Java就很容易理解。

像下面这样的代码

```javascript
load({ x: 4, a: 7 });
load({ x: 4, b: 9 });
load({ x: 4, c: 3 });
load({ x: 4, d: 1 });
```

没办法被优化，只有将参数的入参格式一致才行，比如

```javascript
load({ x: 4, a: 7, b: undefined, c: undefined, d: undefined });
load({ x: 4, a: undefined, b: 9, c: undefined, d: undefined });
load({ x: 4, a: undefined, b: undefined, c: 3, d: undefined });
load({ x: 4, a: undefined, b: undefined, c: undefined, d: 1 });

```

是不是觉得学习TypeScript很重要了呢？



# 总结

`V8`在不断的迭代和进步，还有[Script Streaming](https://blog.chromium.org/2015/03/new-javascript-techniques-for-rapid.html)等等技术没有介绍。随便翻阅 https://v8.dev/ 就会发现数不尽的干货在里面躺着，等我们动手发掘和尝试。作为一个JavaScript工程师而言，无论是学习V8还是其它的编译引擎都能使得我们更好的写出高性能代码、优化代码、甚至在做架构的时候提供帮助。

另外，随着WebAssembly进入浏览器和Node.js，越来越多的C++等技术会被更方便的加入到JavaScript阵营当中。最近买了树莓派和Arduino，在它们身上使用JavaScript做一些功能多多少少离不开C/C++，感觉到了需要系统化学习C/C++的时候了。





### 主要参考：

[Parsing JavaScript - better lazy than eager](https://www.youtube.com/watch?v=Fg7niTmNNLg)

[JavaScript engines - how do they even? ](https://www.youtube.com/watch?v=p-iiEDtpy6I)

[JavaScript Start-up Performance](https://medium.com/reloading/javascript-start-up-performance-69200f43b201)

[V8: Hooking up the Ignition to the Turbofan](https://docs.google.com/presentation/d/1chhN90uB8yPaIhx_h2M3lPyxPgdPmkADqSNAoXYQiVE/edit#slide=id.g1ba5e472dd_0_103)