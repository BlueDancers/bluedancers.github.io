---
title: 三个案例带你理解Event Loop流程
categories:
  - JavaScript-2022
tags:
  - JavaScript
toc: true
date: 2022-3-15
---

## 为什么 JavaScript 需要 Event Loop

 JavaScript 在创建之初就确认了一点，JavaScript 是一门单线程语言，因为 JavaScript 主要提供用户互动以及操作 DOM，假定存在两个线程，同时对一个 Dom 进行操作，那究竟以谁为准呢，为了避免这种复杂性，JavaScript 确认了单线程这个核心概念

 在单线程中，除了 JavaScript 本身的逻辑之外，还存在一些 IO 操作，例如从服务端获取数据，在单线程中就需要等待结果的返回才能继续下面的逻辑，这时候机器是挂起状态，为了避免这个低效问题，这里就引入了同步任务与异步任务的概念，依旧是是单线程，但是有些任务不等待其运行结果

 于是同步任务都在函数执行栈（Stack）上执行，所有的异步任务，在有了运行结果之后，就会将其放入任务队列（callback queue），如果 Stack 中任务执行完毕，就会再去检查任务队列是否存在待执行的回调任务，将其任务放入 Stack，再执行，这里就会不断循环此操作

 Stack 执行 - Stack 执行完毕 - 检查任务队列 - 将任务加入 Stack - Stack 执行，这样重复的过程就需要 Event Loop 来持续不断的循环检查任务队列，确保异步任务被准时加入到 Stack

![](http://www.vkcyan.top/Fr09Z8JDsWNH_CThK4l2piOyaagp.png)

## 相关名词介绍

**函数执行栈（Stask）**：遵循后进先出原则，同步函数执行栈

**webAPIs**：异步任务的发起者，事件是首先进入 stack，例如 click，change，再将 callback 加入任务队列

**回调队列（callback queue）**：也可以被称为任务队列，回调函数到达了执行时机就会进入任务队列，他们将会被 Event Loop 持续打入函数执行栈

## 事件循环解析

我们用 Event Loop 来解释一个简单的 demo

```js
consoleo.log("1");

setTimeout(() => {
  console.log("2");
}, 5000);

console.log("3");
```

我相信大部分人都知道，这里打印的顺序为 1 3 2，下面我们用事件循环的流程来说明为什么是这个结果

**注： cb 为 callback**

1. `console.log('hi')`，进栈 执行 出栈，**打印 1**

2. `setTimeout`进栈 执行 cb 加入异步队列 自身出栈

3. `console.log('end')`，进栈 执行 出栈 **打印 3**
4. 5s 后，cb 加入任务队列，event Loop 检查当前执行栈是否存在函数，检查发现不存在，将 cb 加入函数执行栈
5. `cb`进栈 执行 出栈 **打印 2**



 我们通过 event Loop 的角度进行解析，就能很轻易的解释为什么代码执行顺序是 1 3 2，这是非常简单的场景，接下来我们会分析一些更加有难度的代码

## 关于宏任务与微任务

 通过上面的介绍，我相信大部分人都事件循环有一个基础的认知的，但是与以上我们通过简单的示例，演示了异步代码在事件循环中的运行流程，并没有涉及宏任务与微任务，这两货是什么呢，为什么要在这里出现？

**首先，宏任务（macrotask），微任务（microtask）都是异步任务**

宏任务：`setTimeout` `setInterval` `setImmediate` `I/O 键盘事件` `网络事件` `UI rendering`

微任务：`pormise` `MutationObserver` `process.nextTick`

他们在执行层面上存在一定差异

### 例子 1

```javascript
console.log("script start");

setTimeout(function () {
  console.log("setTimeout");
}, 0);

Promise.resolve()
  .then(function () {
    console.log("promise1");
  })
  .then(function () {
    console.log("promise2");
  });

console.log("script end");
```

结果为 `script start` `script end` `promise1` `promise2` `setTimeout`

pormise 与 setTimeout 虽然都是异步任务，但是上图的代码你会发现，promise 仿佛插队了，这便是宏任务与微任务在任务队列最大的不同之处

 **event Loop 在事件循环中,首先解析 script，将宏任务加入宏任务队列，将微任务加入微任务队列，栈空了之后，执行当前微任务，第一轮事件循环结束**

**在第二轮事件循环中，首先执行宏任务 callback 中的第一个，执行完毕，栈空了之后，再执行当前微任务，后面同理**

<img src="http://www.vkcyan.top/FqUHDZ_5cScznXGrkeEfGhaXCWUw.png" style="zoom: 50%;" />

### 例子 2

```js
console.log("a");

setTimeout(() => {
  console.log("b");
  setTimeout(() => {
    console.log("g");
  }, 0);
  new Promise((resolve) => {
    resolve();
  }).then(() => {
    console.log("h");
  });
}, 0);
new Promise((resolve) => {
  console.log("c");
  resolve();
})
  .then(function () {
    console.log("d");
  })
  .then(function () {
    console.log("e");
  });

console.log("f");
```

大家可以先别看答案，自己先尝试将答案推算出来

#### 第一轮事件循环

`console.log("a");`进入栈 执行 出栈 **打印 a**

`setTimeout`进入栈 **callback 加入宏任务队列** 本身出栈

`new Promise` 进入栈 执行

`console.log("c");`进入栈 执行 出栈 **打印 c**

`new Promise.then` 进入微任务队列

`console.log("f");`进入栈 执行 出栈 **打印 f**

`new Promise` 出栈

**---宏任务完毕，开始执行微任务---**

`console.log("d")`进入栈 执行 出栈 **打印 d**

`console.log("e")`进入栈 执行 出栈 **打印 e**

**--微任务执行完毕 第一轮事件循环完毕--**

结果为 a c f d e

#### 第二轮事件循环

第一轮循环中 setTimeout 在宏任务中，开始执行

`console.log("b")` 进入栈 执行 出栈 **打印 b**

`setTimeout`进入栈 **callback 加入宏任务队列** 本身出栈

`new Promise` 进入栈 执行

`new Promise.then` 进入微任务队列

`new Promise` 出栈

**---宏任务完毕，开始执行微任务---**

`console.log("h")`进入栈 执行 出栈 **打印 h**

**--微任务执行完毕 第二轮事件循环完毕--**

结果为 b h

#### 第三轮事件循环

第二轮循环中 setTimeout 在宏任务中，开始执行

`console.log("g");`进入栈 执行 出栈 **打印 g**

**---宏任务完毕，开始执行微任务---**

**--微任务队列为空 第三轮事件循环完毕--**

结果为 g

#### 结果

**a c f d e b h g**

### 宏/微任务的结论

通过上面三个例子，我们可以得出以下结论

- 函数执行栈中如果还存在函数，则等待其结束，才会继续事件循环

- Event Loop 先执行同步任务，再微任务，下一轮循环，宏任务加入队列，执行，所以说先微任务，再宏任务是合理的

- 单次 Event Loop 中，只会执行一次宏任务，但是微任务可以一次执行多个

## 推荐文章

[做一些动图，学习一下 EventLoop](https://juejin.cn/post/6969028296893792286) 通过动图看 Event Loop 更加便于理解

## 总结

 JavaScript 单线程语言的特性注定其需要异步队列，让网页交互体验上更加友好，对于开发来说，需要尽量了解其特性

 首先我们需要了解事件循环机制，搞懂代码执行栈 异步队列的概念，再后面搞懂宏任务与微任务，读懂异步队列的运行机制，这样基本上就可以解决大部分 Event Loop 问题，了解宏任务 微任务 将会对代码执行顺序有更加底层的理解，这样就可以解决
