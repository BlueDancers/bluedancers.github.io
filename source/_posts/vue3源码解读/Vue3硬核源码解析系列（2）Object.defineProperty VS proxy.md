---
title: （2）Object.defineProperty vs proxy
categories:
  - JavaScript-2023
tags:
  - Vue
  - 源码解读
toc: true
date: 2023-01-30
---



## 专栏前言

本文是**vue3源码解析系列**的第二篇文章，这一章我们主要学习**vue3**源码中涉及到的一些核心**api**。

后续的源码解读是非常复杂的，所以相关基础知识一定要牢固哦~



## 前言

大部分使用过**vue3**的同学都知道，**vue3**的底层的响应式实现由**Object.defineProperty**更换成了**Proxy**。

**为什么vue3要更换呢？proxy相对于前者又有何优势呢？**

接下来让我们通过案例去一探究竟吧！

​	

## 当响应式不存在

我们先看一个例子

```js
let shoes = {
  num: 3,
  price: 10,
}

let total = shoes.num * shoes.price
console.log(total) // 30

shoes.num = 5
console.log(total)  // 30
```

第二次打印依旧是**30**，虽然我们的**num**发生了变化，但是下一次获取**total**的值依旧是之前的值，因为**total**已经被运算过了。

那应该怎么做，才能实时的获取到当前最新的**total**呢？



也很简单，我们每次获取之间，**手动重新计算**一次就好了。

```js
let shoes = {
  num: 3,
  price: 10,
}

let total = 0

function effect() {
  total = shoes.num * shoes.price
}

effect() // 重新计算
console.log(total) // 30

shoes.num = 5
effect() // 重新计算
console.log(total) // 50
```

​	我们增加**effect**方法来手动触发依赖，这样我们实现了需求。

​	但是这样手动触发的方式，在真实业务中过于繁琐，难以维护，本质上依旧是命令式思维。

​	**如何实现值的修改，后续逻辑的自动执行呢？**



## vue2的解决方案

通过**Object.defineProperty**来对字段进行代理，**通过set，get方法，完成逻辑的自动触发**。

```js
let num = 3
let shoes = {
  num: num,
  price: 10,
}
let total = 0
function effect() {
  console.log('开始计算', shoes)
  total = shoes.num * shoes.price
}
// 被代理的值无法不可再get中使用了 因为会触发ett的死循环
// 所以,必须增加一个变量来做被代理的值,所以我们监听shoes.num的get set内部实际修改和读取的都是num
Object.defineProperty(shoes, 'num', {
  set(newVal) {
    num = newVal
    effect()
  },
  get() {
    return num
  },
})
```

​	我们再以上代码，再次修改shoes.num，将触发代理中的set，进而触发effect，实现依赖的自动触发，vue2的底层也正是如此实现的，这样看起来我们的需求已经解决了，那为何vue3有放弃了**Object.defineProperty**呢？

​	接下来我们就要聊聊他的缺陷。



## Object.defineProperty的缺陷

该API确实满足了我们上面提到的案例，但是他在一些场景也存在很多问题。

比如大家一定都遇到过的问题

1. object中新增字段 没有响应性
2. array中指定下标的方式增加字段 没有响应性的

为什么会这样呢？vue的官方解释是

由于 JavaScript 的限制，Vue **不能检测**数组和对象的变化。

尽管如此我们还是有一些办法来回避这些限制并保证它们的响应性。



**那JavaScript到底限制了什么呢？**

​	**object.defineProperty**只能监听到指定对象的**指定属性的get set**，这些工作其实是vue初始化阶段完成，所以指定对象的指定元素发生变化的时候，我们可以监听到变化，vue中也确实是这么表现的；

​	但是如果，我们在指定对象上面新增属性，**object.defineProPerty**是无法监听到的，无法监听则无法处理被新增的字段，自然字段就不具备响应式；

​	在vue2中，如果想解决以上问题，需要使用**Vue.$set**进行手动增加响应式字段，解决无法监听到字段新增的问题。



## vue3的解决方案

**vue3**中改用了**proxy**，为什么响应式核心api做了修改，**proxy**是什么？我们先实现一个类似**vue2**的案例

```js
let shoes = {
  num: 3,
  price: 10,
}

let shoesProxy = new Proxy(shoes, {
  // target 被代理对象 key 本次修改的对象中的键 newValue 修改后的值 receiver 代理对象
  set(target, key, newValue, receiver) {
    console.log('触发了写入事件')
    shoes[key] = newValue
    effect()
    return true
  },
  // target 被代理对象 key 本次读取的值 receiver 代理对象
  get(tartget, key, receiver) {
    console.log('触发了获取事件')
    return shoes[key]
  },
})

let total = 0
function effect() {
  console.log('开始计算', shoes)
  // 如果使用被代理对象本身shoes,这不会触发
  // 如果使用代理对象shoesProxy,则这里会触发proxy的get事件
  total = shoes.num * shoes.price
}
```



通过以上代码，我们可以看到一些差别

**object.defineproperty**

- 代理的并非对象本身，而是对象中的属性

- 只能监听到对象被代理的指定属性，无法监听到对象本身的修改

- 修改对象属性的时候，是对原对象进行修改的，原有属性，则需要第三方的值来充当代理对象

  

**proxy**

- proxy针对对象本身进行代理
- 代理对象属性的变化都可以被代理到
- 修改对象属性的时候，我们针对代理对象进行修改



无论是逻辑的可读性，还是API能力上，**proxy**都比**object.defineProPerty**要强很多，这也是vue3选择proxy的原因。



## proxy的好兄弟Reflect

​	在**vue3**的源码中的**@vue/reactivity**中，**我们会经常看到在proxy的set、get中存在Reflect的身影**，但是从我们上面对**proxy**的使用来看，赋值 读取都实现了，为什么**vue3**中使用了**Reflect**呢？

首先我们了解一下**Reflect**是干嘛的

官方解释：**Reflect** 是一个内置的对象，它提供拦截 JavaScript 操作的方法。

似乎比较难理解，我们举个例子吧

```js
let obj = { num:10 }
obj.num // 10
Reflect.get(obj,'num') // 10
```

这么来看，似乎这个api很普通啊，反而把简单的读取值写复杂了。



这时候我们就要提一下Reflect.get 的第三个参数了

```js
Reflect.get(target, propertyKey, receiver]) // receiver 如果target对象中指定了propertyKey，receiver则为getter调用时的this值。
```

这次我们知道了，第三个参数receiver具有强制修改this指向的能力，接下来我们来看一个场景

```js
let data = {
  name: '张三',
  age: '12岁',
  get useinfo() {
    return this.name + this.age
  },
}

let dataProxy = new Proxy(data, {
  get(target, key, receiver) {
    console.log('属性被读取')
    return target[key]
  },
})
console.log(dataProxy.useinfo)
```

打印情况如下

```
属性被读取
张三12岁
```

​	**dataProxy.useinfo**的get输出的值是正常的，但是get只被触发了一次，这是不正常的；

​	因为useinfo里面还读取了被代理对象**data**的**name**、**age**，理想情况应当是**get**被触发三次。

​	为什么会出现这样的情况呢，这是因为调用**userinfo**的时候，**this指向了data，实际执行的是data.userinfo，此时的this指向data，而不是dataProxy**，此时get自然是监听不到name、age的get了。



​	这时候我们就用到了Reflect的第三个参数，**来重置get set的this指向**。

```js
let dataProxy = new Proxy(data, {
  get(target, key, receiver) {
    console.log('属性被读取')
    return Reflect.get(target, key, receiver) // this强制指向了receiver
    // return target[key]
  },
})
```

打印情况如下

```
属性被读取
属性被读取
属性被读取
张三12岁
```

现在打印就正常了，**get**被执行的3次，此时的**this**指向了**dataProxy**，**Reflect**很好的解决了以上的this指向问题。



​	通过以上案例，我们可以看到使用**target[key]**有些情况下是不符预期的，比如案例中的被代理对象this指向问题，而使用**Reflect**则可以更加稳定的解决这些问题，在vue3源码中也确实是这么用的。



## 补充章节（WeakMap）

​	通过以上文章，我们了解到了**object.defineproperty**相较于**proxy**的劣势，以及搭配**proxy**同时出现的**Reflect**的原因，这是**vue3**最核心的**api**。

​	但是仅仅知道理解**proxy+reflect**，还不太够，为了尽量轻松的阅读**Vue3**源码，我们还要学习一个**原生API**，那就是**WeakMap**。

[WeakMap MDN中文文档地址](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)

​	**weakMap**和**map**一样都是**key value**格式，但是他们还是存在一些差别。

- **weakMap**的**key**必须是对象，并且是**弱引用**关系
- **Map**的**key**可以是任何值（基础类型+对象），但是key所引用的对象是**强引用**关系

​	通过查阅MDN我们可以发现，**weakMap**可以实现的功能，**Map**也是可以实现的，那为什么**Vue3**内部使用了**WeakMap**呢，问题就在**引用关系**上



**强引用：不会因为引用被清除而失效**

**弱引用：会因为引用被清除而自动被垃圾回收**

概念似乎还无法体现其实际作用，我们通过以下案例即可明白

```js
// Map
let obj = { name: '张三' }
let map = new Map()
map.set(obj, 'name')
obj = null // obj的引用类型被垃圾回收
console.log(map) // map中key obj依旧存在

// WeakMap
let obj = { name: '张三' }
let map = new WeakMap()
map.set(obj, 'name')
obj = null // obj的引用类型被垃圾回收
console.log(map) // weakMap中key为obj的键值对已经不存在
```

通过以上案例我们可以了解到

- 弱引用在**对象与key共存**场景存在优势，**作为key的对象被销毁的同时，WeakMap中的key value也自动销毁了**。
- 弱引用也解释了为什么**weakMap**的**key**不能是基础类型，因为基础类型存在栈内存中，不存在弱引用关系；

在vue3的依赖收集阶段，源码中用到了WeakMap，具体什么作用？我们下一节进行解答。



## 结语

​	通过本篇文章，我们认识到了**object.defineproperty**相较于**proxy**的劣势，以及搭配**proxy**同时出现的**Reflect**的原因，还有一个**Map**的原生的**API**，**WeakMap**的作用。

​	接下来我们就可以正式走进**vue3**源码的世界~





























