---
title: Vue3硬核源码解析系列（7）watch源码解析.md
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-05-18
---

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07d7fdd1b1a44560ae1fd052db6e37e0~tplv-k3u1fbpfcp-zoom-crop-mark:1512:1512:1512:851.awebp?)

## 前言

​	原本打算本章讲讲**computed**，但是**computed**的源码相当复杂，使用文章的形式说清楚，难度真的很大，所以暂时跳过**computed**，先说说**watch**。

​	**watch**即为监听的意思：**监听响应式数据，每当状态发生变化，就会触发回调函数**。



​	如果大家对之前的源码分析有所理解的话，我相信大家可以猜到watch实现原理，**一定是初始化的时候进行依赖收集，依赖项发生变化的时候依赖触发**。

​	如果能领悟到这一层，那么对vue3的核心实现你已经有所理解啦。

​	接下来就让我们走进watch的世界，让我们看看，vue3是如何实现他的吧。



首先还是放出watch的逻辑图，watch的逻辑相对简单，因为对于watch而言，响应式是其一部分逻辑。

![](https://www.vkcyan.top/FmR0g8twA5yVrokQ-uXkSNKqD5Cn.png)



## 带着问题看源码

在我刚刚使用Vue3 watch的时候，经常出现以下让我无法解释的情况。

```ts
// reactive的案例
const user = reactive({ name: '卖鱼强' })
watch(user, (value) => console.log('第一', value)) // 有效
watch(user.name, (value) => console.log('第二', value)) // 无效
watch(() => user, (value) => console.log('第三', value))// 无效
watch(() => user.name, (value) => console.log('第四', value))// 有效

user.name = '狂飙强' // 修改reactive 期望触发watch

// ref案例
const user = ref('卖鱼强')
watch(user, value => console.log('第一个watch', value)) // 有效
watch(user.value, value => console.log('第二个watch', value)) // 无效
watch(() => user, value => console.log('第三次watch', value)) // 无效
watch(() => user.value, value => console.log('第四次watch', value)) // 有效

user.value = '狂飙强' // 修改reactive 期望触发watch
```

以上案例，我相信大部分写vue的同学，都很难在第一时间准确判断其watch是否有效无效，接下来就让我们一起从源码中寻找答案。





## 正文

**watch**的源码并不在**reactivity**中，而是在**runtime-core**中

> 关于这一点我会谈谈我的想法，讨论一下为什么不在**reactivity**中，而在**runtime-core**中。

### watch初始化

当我们使用**watch**的时候，其执行的具体源码位置为`packages/runtime-core/src/apiWatch.ts` **line131**

```ts
export function watch<T = any, Immediate extends Readonly<boolean> = false>(
  source: T | WatchSource<T>,
  cb: any,
  options?: WatchOptions
): WatchStopHandle {
  return doWatch(source as any, cb, options)
}

export interface WatchOptions {
  immediate?: boolean
  deep?: boolean
}
```

通过以上代码，我们可以了解到，**watch**是存在三个参数的

1. **source** ：监听项
2. **cb**：watch的回调函数
3. **options**： 关于watch的设置，内部存在2个参数
   1. **immediate**  首次是否运行
   2. **deep**  是否深度监听

这些消息和我们通过Vue文档了解到的信息完全一致，最终我们会发现，其实际返回了一个**doWatch**函数，并将**watch**的三个参数传递了进去。

**doWatch内部的逻辑就是watch实现的核心逻辑了**，我们从三个阶段分析**doWatch**的代码。



**第一阶段：处理source，监听项分析**

**第二阶段：构建响应式模块，完成依赖收集**

**第三阶段：明确依赖触发方式**



## 第一阶段：处理source，监听项分析

我们在使用**watch**的时候，第一个参数，也就是被监听项，是可以传入很多类型的，**ref reactive function array**，在**doWatch**函数中，我们可以看到，针对不同类型与属性的**source**，都做了个性化的依赖处理。

接下来就让我们看看，**doWatch**都是如何处理这些变量的吧。

### ref

> 后续getter函数一旦执行，将会访问ref，**触发 ref本身的依赖收集**

```ts
if (isRef(source)) {
  // 如果当前source的值是ref, 则处理为() => source.value
  // 这里注意const num = ref(1) num是ref，num.value并不是ref而是基础类型
  getter = () => source.value
}
```

### reactive

> 后续getter函数一旦执行，将会访问reactive，**触发 ReactiveEffect 完成依赖收集**

```ts
if (isReactive(source)) {
  // 如果是reactive则,直接处理成() => source
  getter = () => source
  // 同时将deep赋值为true 因为reactive为object，一般为多层嵌套，需要深度递归
  deep = true
}
```

### Function

>  后续getter函数一旦执行，将会运行fn，访问函数返回值，如果fn返回的是ref 或者reactive 就会**触发相应的依赖收集**

```ts
if (isFunction(source)) {
  // callWithErrorHandling函数比较复杂，这里就不做展示了
  // 函数效果为：返回 () => fn()
  // 后续getter
  getter = () => callWithErrorHandling(source, instance, ErrorCodes.WATCH_GETTER)
}
```

### Array

>  后续getter函数一旦执行，将会访问getter中的所有的访问值，如果fn返回的是ref 或者reactive 就会**触发相应的依赖收集**

```ts
if (isArray(source)) {
  isMultiSource = true // 标识为多个监听项
  // array类型的source，可能包含ref reactive Function 所以都需要进行处理
  // 其中reactive比较复杂，需要通过traverse函数，递归触发所有依赖项，也可以说array类型的source，默认deep参数就是true
  getter = () => 
    source.map(s => {
      if (isRef(s)) {
        return s.value // 
      } else if (isReactive(s)) {
        return traverse(s)
      } else if (isFunction(s)) {
        return callWithErrorHandling(s, instance, ErrorCodes.WATCH_GETTER)
      }
    })
}


export function callWithErrorHandling(fn,instance,type,) {
  return fn()
}
```

### 未知类型

```ts
export const NOOP = () => {}

getter = NOOP // 如果watch的第一个参数不是以上类型，这起getter函数为空
```

以上就是**watch**针对所有类型的**source**的处理。

我们可以发现其实就做了一件事，就是将其包装为**getter**函数，**getter**函数一旦运行，便可以触发相关**依赖收集**。



完成第一阶段的分析，其实我们文章开头提出的问题已经有了明确答案，我们回过头来继续看看

```ts
const user = reactive({ name: '卖鱼强' })

watch(user, () => {})
// user是reactive，将会被处理为（）=> user，同时deep参数默认设置为true，reactive中的所有依赖都将会触发依赖收集，watch有效
watch(user.name, () => {})
// name是reactive内的基础对象，将会被识别为未知类型，所以watch无效
watch(() => user, () => {})
// 函数返回，并未访问proxy的属性，无法完成依赖收集，所以watch无效
watch(() => user.name, () => {})
// 函数返回 而user.name是proxy下的属性，将会触发依赖收集，所以watch有效
```

以上就是**reactive + watch**不同使用方式的效果解读。

有兴趣的小伙伴可以试试解读一下**ref + watch**的结果。

如果真的记不住，我们就记住下面的这句话：**watch 监听对象本身，使用对象的形式；watch监听对象内部属性，使用函数形式。**



## 第二阶段：构建响应式模块，完成依赖收集

> 这上小节，我们完成**getter**函数的构建，这一步我们需要进行依赖触发，与依赖收集，使**watch**的监听功能正式生效。

```ts
if (cb && deep) {
  // 如果deep为true, 则将getter函数再通过traverse进一步处理，使其可以被深度监听
  // traverse的作用前面说过，目的就是递归触发对象所有属性的get。
  const baseGetter = getter
  getter = () => traverse(baseGetter()) 
}

// 初始化oldValue，如果source是数组isMultiSource为true，否则为false
let oldValue = isMultiSource ? [] : {}

// watch的核心实现，注意一下，此刻我们还没有执行
// 内部逻辑非常复杂，我们这里简化处理
// 简单来说就是每次watch的属性或者字段发生变化，都会触发该方法，可以触发的原因是我们getter函数完成了依赖收集的必要逻辑
const job: SchedulerJob = () => {
  const newValue = effect.run() // 获取被监听项的最新值
  // 如果deep为true 或者新旧值不一致, 则会执行watch的cb，也就是我们需要触发的函数
  if (deep || hasChange(newValue, oldValue)) {
    cb(newValue, oldValue)
  }
}

let scheduler: EffectScheduler
// flush：回调的刷新时机 
// queuePreFlushCb  queuePostRenderEffect 后续再说，我们先假设flush就是async
if (flush === 'sync') {
  scheduler = job
} else if (flush === 'post') {
  scheduler = () => queuePostRenderEffect(job, instance && instance.suspense)
} else {
  // default: 'pre'
  scheduler = () => queuePreFlushCb(job) // queuePreFlushCb 暂时先忽略
}

// getter是第一步处理的，可以访问到响应式字段的函数
// scheduler是watch监听字段发生变化，实际需要执行的回调函数，我们可以理解为scheduler = job = getter
const effect = new ReactiveEffect(getter, scheduler)

if (cb) {
  // 如果immediate为true,则代表默认watch初始化阶段自动执行一次
  if (immediate) {
    job()
    // job中的effect.run运行，完成依赖收集，建立其了变量与cb函数之间的联系。
    // 同时也执行了cb函数，首次watchcb被执行
  } else {
    // 如果immediate为false，则直接运行effect.run()，完成依赖收集，建立其了变量与cb函数之间的联系。
    oldValue = effect.run()
  }
}

return () => {
  // 返回了effect的stop函数，则意味着，watch api存在返回值，只需要执行一下返回值 就会结束掉watch的监听
  effect.stop()
}
```

​	到此为止，我们可以明确了解到，在**Vue**的初始化阶段，**watch**其内部通过**ReactiveEffect**，以及**effect.run()**的触发，完成了**watch**需要监听的变量与触发函数的绑定，**ReactiveEffect**逻辑在[Vue3硬核源码解析系列（3） reactive + effect源码解析](https://juejin.cn/post/7202132390549553211)可以了解其具体实现。

​	**也就是相当于说，watch内部通过手动访问source，触发source的get事件**，**source**依赖一旦触发，就会开始依赖收集，就会**收集到watch的第二个参数cb**，经进而完成**watch**的依赖收集；**只要source发生改变，一定会触发cb函数。**

​	其实到这里**watch**的核心源码就已经结束了，依赖已经完成收集；

​	当被监听变量或者属性发生变化的时候，**cb**函数一定会执行，但是**watch**的执行时机是非常有讲究的；

​	**所以接下来就要讲讲watch第三个参数的flush，该字段就是控制cb函数的执行时机。**



## 第三阶段：依赖触发

当我们**watch**监听的字段发生变化的时候，**watch**的第二个参数，**cb**会被触发，但是并不是监听字段发生变化的下一步就立刻触发。

这里我们回顾一下**watch**源码中变量**scheduler**的相关逻辑

```js
if (flush === 'sync') {
  scheduler = job
} else if (flush === 'post') {
  scheduler = () => queuePostRenderEffect(job)
} else {
  // default: 'pre'
  scheduler = () => queuePreFlushCb(job)
}

 // 为了便于理解，暂时认为__FEATURE_SUSPENSE__为false，此处一定等于queuePostFlushCb
export const queuePostRenderEffect = __FEATURE_SUSPENSE__
  ? queueEffectWithSuspense
  : queuePostFlushCb
```

我们可以看到，**flush**参数不同的时候**scheduler**的值也是不同的

如果我们指定了**flush**是**sync**，则**source**发生变化下一个同步任务就是执行**watch**的**cb**函数，

如果我们不进行指定，默认将是**pre**，则会触发**queuePreFlushCb(job)**

如果指定为**post**，则会触发**queuePostFlushCb(job)**

根据文档我们可以了解到当**flush**为**pre**的时候，**watch**第二个参数**cb**，将会在**Vue**组件更新之前被调用，**post**则会让**cb**函数在**Vue**组件更新之后被调用

接下来就让我们看看**queuePreFlushCb**与**queuePostFlushCb**内部是如何实现的吧！



### queuePreFlushCb与queuePostFlushCb

```js
const resolvedPromise =  Promise.resolve()
let currentFlushPromise = null
let isFlushPending = false

const pendingPreFlushCbs: SchedulerJob[] = []
let activePreFlushCbs: SchedulerJob[] | null = null
let preFlushIndex = 0

// cb传入到queueCb中
export function queuePreFlushCb(cb: SchedulerJob) {
  // 执行方法附带一些关于pre队列的全局变量
  queueCb(cb, activePreFlushCbs, pendingPreFlushCbs, preFlushIndex)
}

// cb传入到queueCb中
export function queuePostFlushCb(cb: SchedulerJobs) {
  // 执行方法附带一些关于post队列的全局变量
  queueCb(cb, activePostFlushCbs, pendingPostFlushCbs, postFlushIndex)
}

// 将cb加入到全局变量pendingPreFlushCbs或者pendingPostFlushCbs中，我们可以理解为缓存了cb函数，并执行了queueFlush
function queueCb(
  cb: SchedulerJobs,
  activeQueue: SchedulerJob[] | null,
  pendingQueue: SchedulerJob[],
  index: number
) {
  pendingQueue.push(cb)
  queueFlush()
}

// 该函数最终将缓存的cb函数访问到Promise的.then中，resolvedPromise已经是resolve状态，则意味着，将会在下一次微任务的时候触发flushJobs
function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}


// 若干时间后watch被触发，然后一轮事件循环结束，开始触发flushJobs
function flushJobs() {
  isFlushPending = false
  flushPreFlushCbs()
  try {
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job && job.active !== false) {
        callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
      }
    }
  } finally {
    flushPostFlushCbs(seen)
    isFlushing = false
  }
}

// 依次触发之前存储的所有cb函数
export function flushPreFlushCbs() {
  if (pendingPreFlushCbs.length) {
    let activePreFlushCbs = [...new Set(pendingPreFlushCbs)]
    
    pendingPreFlushCbs.length = 0
    for (let i = 0; i < activePreFlushCbs.length; i++) {
      activePreFlushCbs[i]()
    }
  }
}

// 依次触发之前存储的所有cb函数
export function flushPostFlushCbs(seen?: CountMap) {
  flushPreFlushCbs()
  if (pendingPostFlushCbs.length) {
    const deduped = [...new Set(pendingPostFlushCbs)]
    pendingPostFlushCbs.length = 0
    activePostFlushCbs = deduped

    for (postFlushIndex = 0; postFlushIndex < activePostFlushCbs.length; postFlushIndex++) {
      activePostFlushCbs[postFlushIndex]()
    }
    activePostFlushCbs = null
    postFlushIndex = 0
  }
}
```

以上代码看起来似乎比较复杂，但是执行的逻辑其实非常简单，**Vue3的更新队列存在三种分别是pre，queue，post，这三个队列按照顺序执行相应代码**

1. 执行**pre**队列中的代码
2. 执行**queue**队列中的代码，（**queue为组件update的相关逻辑**）
3. 执行**post**队列中的代码

这里对照**vue3**文档，我们可以发现，我们的分析是符合文档描述的。

> 因为涉及到**vue3**的更新队列，这并非**watch**关联的知识，为了方便源码阅读，可以假设**watch**的**flush**的参数为**async**，这样是最好理解的。



到此为止，我们的**watch**核心源码分析就全部完毕了。



## 关于ref的一些问题

### watch的源码为什么在runtime-core中？

关于这一点我是这么理解的，watch不仅仅是一个响应式组件，他涉及到了组件的生命周期，更新渲染等等逻辑，放在runtime中更好与组件系统进行集成，





## 总结

​	通过以上源码分析我们可以发现，watch的响应式原理相对来说是比较简单的，完全依赖我们的之前说过的ReactiveEffect，所以如果小伙伴了解reactive的源码，相信看watch的源码的响应式部分是非常轻松的

​	相对于其他api，watch的响应式实现具备一下2个特点

1. watch的依赖收集是**被动触发**的
2. watch的依赖触发，实际上是调度器scheduler，然后通过不同的flush，达到控制执行顺序、规则的目的。

​	watch的源码分析就到这里，我们下期再见吧~👋🏻
