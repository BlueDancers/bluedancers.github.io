---
title: Vue3硬核源码解析系列（6） 100行代码 实现mini版ref
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-03-22
---

## 专栏前言

​	在上一节，我们完成了**vue3**的**ref**核心源码解读，其实**基础类型的ref的核心逻辑还是非常简单的**，所以在我们的简易版源码环节，我们直接切入基础类型，复杂类型仅做支持，不做讲解。

> 注：单基础类型场景的ref源码，几乎可以说是整个vue3源码中最简单的一部分，所以这一节的学习难度是最小的

[mini版vue3仓库地址](https://github.com/BlueDancers/vue3-mini/tree/ref)，还请大家不要吝啬star，下次不迷路~



**仅保留最核心逻辑，极大减低阅读难度，80行代码实现ref**，让我们直接进入源码实现环节！



## 逻辑图（基础类型）

> 完整版ref逻辑图，请看 Vue3硬核源码解析系列（5） ref源码解析

![](https://www.vkcyan.top/FjE3zqx5l7zpmv0is0_Fusim1mhf.png)





## 具体逻辑

> 如同逻辑图所示，我们简易版源码的具体实现也从 **初始化 依赖收集 依赖触发**三个角度来进行实现

### 初始化

**ref**的初始化非常简单，逻辑流程如下

1. 判断传入对象是否已经是**ref**，如果是，这直接返回，如果不是，则继续运行代码
2. **ref**的本质就是一个**Class RefImpl**
3. 初始化**RefImpl**的时候，将ref的参数保存到`_value`，同时将参数的原始值保存到`_rawValue`
4. 通过**get value**，实现**ref.value**的访问
5. 使用**set value**，实现**ref.value = xx**的更新逻辑

确定实现逻辑的同时，我们也仿照vue3的源码结构开始输出吧~

````ts
/**
 * 入口函数
 */
export function ref(value?: unknown) {
  return createRef(value, false)
}

function createRef(rawValue: unknown, shallow: boolean) {
  // 判断是否已经是ref,如果是直接返回其本身
  if (isRef(rawValue)) {
    return rawValue
  }
  // ref本质上就是RefImpl的实例
  return new RefImpl(rawValue, shallow)
}

class RefImpl<T = any> {
  private _value: T // ref每次读取与返回的属性
  private _rawValue: T // ref中value的原始属性
  public dep: Dep | undefined // 当前ref相关effect
  public readonly __v_isRef: boolean = true // 标记__v_isRef为true,以后将无法在通过isRef()的判断
  constructor(value: T, public readonly __v_isShallow: boolean) {

    this._rawValue = value     // 赋值原始值
    // ref API中 __v_isShallow,一定为false （__v_isShallow 表示是否浅层代理）
    // value是基础类型,则toReactive返回原值，value是复杂类型,则toReactive会将其处理成为reactive(proxy)再返回,这就意味着,此时的value是一个proxy
    this._value = __v_isShallow ? value : toReactive(value)
  }
  // 实现ref.value能力
  get value() {
    // 配合effect阶段保存的activeEffect,将依赖收集到this.dep中（依赖收集）
    trackRefValue(this)
    // 返回最新value
    return this._value
  }
  // 实现ref.value = xx能力
  set value(newVal) {
    // 判断当前set的value是否存在变化, 有变化则进入if
    if (hasChange(newVal, this._rawValue)) {
      // 保存最新的参数原始值，便于下次hasChange判断
      this._rawValue = newVal
      // 如果value是基础类型, 则toReactive返回value本身，否则返回通过toReactive生成的proxy
      this._value = toReactive(newVal)
      // 触发get阶段收集在this.dep中的依赖（依赖触发）
      triggerRefValue(this)
    }
  }
}
````

### 依赖收集

> ref = Class RefImpl

​	经过我们上一章的**ref**源码分析我们可以了解到，**ref**的依赖收集，并不是依赖**WeakMap**进行完成，而是其自行完成依赖收集，收集在自身**class**的**dep**中，逻辑大概是这样的

1. 每次触发**ref**的**get**的时候，都会执行一次**trackRefValue**（trackRefValue的作用是完成依赖收集）
2. 每次执行**effect**的时候，都会将**effect**本身保存到变量**activeEffect**中（具体请看[Vue3硬核源码解析系列（3） reactive + effect源码解析](https://juejin.cn/post/7202132390549553211)）
3. 如果**RefImpl**的**dep**不存在，则说明是第一次进行依赖收集，将通过**createDep**将**RefImpl.dep**赋值为**Set**
4. 将**activeEffect**，也就是当前正在运行的**effect**，**push**到**RefImpl**的**dep**中，**ref**完成依赖收集

明确了逻辑之后，我们依旧结合vue3的源码结构，来完成ref依赖收集的代码输出。

```ts
get value() {
  trackRefValue(this)
  return this._value
}

/**
 * ref 依赖收集
 */
export function trackRefValue(ref: RefImpl) {
  // 判断当前是否存在需要收集的依赖
  if (activeEffect) {
    // 判断RefImpl的实例中的dep是否被初始化过
    if (!ref.dep) {
      // 如果没有, 则赋值为Set
      ref.dep = createDep()
    }
    // 将当前effect收集到当前RefImpl实例的dep中, 完成依赖收集
    trackEffects(ref.dep)
  }
}

/**
 *
 * @param dep
 */
export function trackEffects(dep: Dep) {
  dep.add(activeEffect!)
}
```



### 依赖触发

若干时间后，**ref**的**value**被更新，触发**RefImpl**的**set value**，在更新**value**的同时，也会执行其内部的**triggerRefValue**，开始依赖触发逻辑

1. 获取到当前ref，也就是class **RefImpl**本身的**dep**
2. 循环**dep**中存储的所有**effect**，并执行其**fn**，完成依赖触发。

```ts
/**
 * ref 依赖触发
 */
export function triggerRefValue(ref: RefImpl) {
  // 当前当前RefImpl实例中是否存在收集的依赖
  if (ref.dep) {
    // 触发依赖
    triggerEffects(ref.dep)
  }
}

/**
 * 处理所有待触发依赖
 */
export function triggerEffects(dep: Dep) {
  // const effects = isArray(dep) ? dep : [...dep]
  const effects = [...dep]
  for (const effect of effects) {
    triggerEffect(effect)
  }
}

/**
 * 触发执行依赖
 */
function triggerEffect(effect: ReactiveEffect) {
  effect.run()
}
```

到此为止，我们的**ref**就具备响应式的能力了，是不是很简单~



## 小结

​	这时候肯定有同学要说了，**你这ref不保熟啊**，仅支持基础类型，不支持复杂类型啊，这不是阉割版ref吗？

​	这里必须澄清一下，虽然简易版ref 100行代码不到，但是他是支持复杂类型的响应式的，因为复杂类型的响应式是依赖**reactive**进行完成的，不过**reactive**的源码解读，并不是本文的重点，所以，这里就跳过了，有兴趣的同学，请看这里[Vue3硬核源码解析系列（3） reactive + effect源码解析](https://juejin.cn/post/7202132390549553211)，了解**reactive**的响应式实现，再看[Vue3硬核源码解析系列（5）ref源码解析](https://juejin.cn/post/7212910997778350136)，了解复杂类型场景下的源码执行逻辑吧。

​	最后，建议大家[clone](https://github.com/BlueDancers/vue3-mini/tree/ref)源码到本地实际运行一下，静下心来一步一步调试，将简易版逻辑弄明白，有兴趣的可以在看看正式的vue3源码，然后在简历上留下浓墨重彩的一笔~
