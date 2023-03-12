---
title: （6）vue3 reactivity-watch源码解析
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-02-14
---

## 前言

​	上一章我们说完了computed的实现逻辑，今天就让我们来看看他的好兄弟，也是我们经常用的另一个api，watch是如何实现的；

​	watch即为监听的意思：**监听响应式数据，每当状态发生变化，就会触发回调函数**。

​	如果大家对之前的源码分析有所理解的话，我相信大家可以简单猜到watch的实现原理，一定是初始化的时候依赖收集，依赖项发生变化的时候依赖触发。

​	如果能领悟到这一层，那么对vue3的核心实现你已经有所理解啦。

​	接下来就让我们走进watch的世界，让我们看看，vue3是如何实现他的吧。



## 带着问题看源码

起初我刚开始使用vue3 watch的时候，就出现了让我非常迷糊的情况，举个例子

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

​	以上案例，我相信大部分写vue的同学，都很难在第一时间准确判断有效无效，这样的问题，我们就需要去源码中寻找答案。





## 正文

watch的源码并不在reactivity中，而是在runtime-core中

### watch初始化

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

通过以上代码，我们可以了解watch是存在三个参数的

1. **source** 默认类型为any，这一项就是我们的监听项
2. **cb**，也就是我们的watch的回调函数
3. **options** 关于watch的设置，内部存在2个参数
   1. **immediate** boolean类型 是否首次运行
   2. **deep** boolean类型 是否深度监听

​	这些消息和我们通过Vue文档了解到的信息基本一致，然后我们继续往下看，会发现实际返回了一个doWatch函数，并将watch的三个参数传递了进去。

​	doWatch里面的代码比较复杂，所以我们我们将初始化阶段再分为三个阶段，分别于三个参数对应



## 监听项分析

### ref

```ts
if (isRef(source)) {
  // 如果当前source的值是ref, 则处理为() => source.value 这里注意const num = ref(1) num是ref，num，value不是ref
  getter = () => source.value
  forceTrigger = isShallow(source) // 判断是否是浅层的监听
}
```

### reactive

```ts
if (isReactive(source)) {
  // 如果是reactive则,直接处理成() => source
  getter = () => source
  // 同时将deep赋值为true 因为reactive可能多层嵌套需要深度递归
  deep = true
}
```

### Array

```ts
if (isArray(source)) {
  isMultiSource = true // 标识为多个监听项
  forceTrigger = source.some(s => isReactive(s) || isShallow(s)) // 标识是否是浅层监听
  // getter 函数处理为对source的map循环返回结果
  getter = () => 
    source.map(s => {
      if (isRef(s)) {
        return s.value
      } else if (isReactive(s)) {
        return traverse(s)
      } else if (isFunction(s)) {
        return callWithErrorHandling(s, instance, ErrorCodes.WATCH_GETTER)
      } else {
        __DEV__ && warnInvalidSource(s)
      }
    })
}


export function callWithErrorHandling(fn,instance,type,) {
  return fn()
}
```

### Function

```ts
if (isFunction(source)) {
  // callWithErrorHandling函数实际上就是返回了source函数本身
  getter = () => callWithErrorHandling(source, instance, ErrorCodes.WATCH_GETTER)
}
```

### 未知类型

```ts
export const NOOP = () => {}

getter = NOOP // 如果watch的第一个参数不是以上类型，这起getter函数为空
```



第一个参数的目的是为了完成后续的**依赖收集**，所以getter的返回值必须是proxy对象，不能是基础对象，否则将无法完成期望的依赖收集。

看到这里我们文章开头提出的问题已经有了明确答案，我们回过头来继续看看

```ts
const user = reactive({ name: '卖鱼强' })

watch user
// user是reactive，将会被处理为（）=> user，同时deep参数设置为true，reactive中的所有依赖都将会触发依赖收集，watch有效
watch user.name
// name是reactive内的基础对象，将会被识别为未知类型，自然watch无效
watch () => user
// 函数直接返回，并未访问proxy的属性，无法完成依赖收集，所以watch无效
watch () => user.name
// 函数直接返回 而user.name是proxy下的属性，将会触发get，完成依赖收集，所以watch有效
```



## 构建ReactiveEffect

> 这一小节，我们将了解到watch是如何完成依赖触发

```ts
if (cb && deep) {
  // 如果deep为true,则获取到() => source
  const baseGetter = getter
  getter = () => traverse(baseGetter()) 
  // traverse函数比较复杂，其作用就是深度递归对象，以此触发reactive中每个属性的get行为
}

// 非数组的情况下isMultiSource一定为false 所以oldValue这里一定是空对象
let oldValue = isMultiSource ? [] : INITIAL_WATCHER_VALUE

 // watch的核心实现
const job: SchedulerJob = () => {
	// 内部逻辑非常复杂，我们这里简化处理
  const newValue = effect.run()
  // 如果deep为true 或者新旧值不一致则将新旧值传入到cb中（watch的第二个参数）
  if (deep || hasChange(newValue, oldValue)) {
    cb(newValue, oldValue)
  }
}

let scheduler: EffectScheduler
// flush 回调的刷新时机 
// 默认pre 即组件更新之前调用 queuePreFlushCb
// post 则是组件更新之后调用(可以获取到更新后的dom) queuePostRenderEffect
// 如果是sync则异步调用 job本身
// queuePreFlushCb  queuePostRenderEffect 后续再说
if (flush === 'sync') {
  scheduler = job as any // the scheduler function gets called directly
} else if (flush === 'post') {
  scheduler = () => queuePostRenderEffect(job, instance && instance.suspense)
} else {
  // default: 'pre'
  scheduler = () => queuePreFlushCb(job)
}

// getter为具备返回响应式能力的函数 scheduler是确定了执行实际的cb函数
const effect = new ReactiveEffect(getter, scheduler)

if (cb) {
  // 如果immediate为true,则代表默认watch初始化阶段自动执行一次
  if (immediate) {
    job() // job内部也会运行一次effect.run 只不过同时也执行了cb函数 完成了第一次watch的触发
  } else {
    // 关键步骤 执行一次proxy的get行为,将当前除触发watch的监听数据的依赖收集，并将ReactiveEffect与watch的依赖项完成绑定
    // 同时获取一次oldvalue的值（watch变化之前的值）
    oldValue = effect.run()
  }
}

return () => {
  // 返回了effect的stop函数，则意味着，watch api存在返回值，只需要执行一下返回值 就会结束掉watch的监听
  effect.stop()
}
```





## 依赖触发

### async

### pre

### post



1. watch初始化，生成ReactiveEffect，并将watch的匿名函数（cb）放入ReactiveEffect的scheduler中
2. 触发一次watch的第一个参数，完成依赖收集（proxy与watch建立联系）
3. proxy发生变化，触发watch的ReactiveEffect的scheduler，也就是watch初始化阶段传入的cb
4. cb按照指定flush开始执行，完成watch监听函数的触发。



​	其实通过以上流程我们可以发现watch的原理相对来说竟然是最简单的，watch的复杂并不在于核心逻辑的复杂，而是其内部进行了大量兼容性判断，多种类型的第一参数，多种类型的第三参数，导致整体代码的复杂度极高。



## 特点

1. watch的依赖收集是被动触发的
2. watch的依赖触发，实际上是调度器scheduler，然后通过不同的flush，达到控制执行顺序、规则的目的。

