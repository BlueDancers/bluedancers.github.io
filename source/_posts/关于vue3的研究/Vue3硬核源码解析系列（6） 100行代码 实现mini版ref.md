---
title: Vue3硬核源码解析系列（6） 100行代码 实现mini版ref
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-03-21
---

## 专栏前言

​	在上一节，我们完成了**vue3**的**ref**核心源码解读，其实基础类型的ref的核心逻辑还是非常简单的，但是增加复杂类型的支持后，其逻辑就有点复杂了，所以在我们的简易版源码环节，我们直接切入基础类型，复杂类型仅做支持，不做过多讲解。

​	**仅保留最核心逻辑，极大减低阅读难度，100行代码实现ref**，让我们直接源码实现环节。

​	单看基础类型的场景的ref源码，几乎可以说是整个vue3源码中最简单的一部分了，所以这一节的学习难度也可以说是最小的。



[简易版vue3仓库地址](https://github.com/BlueDancers/vue3-mini/tree/ref)，**还请大家不要吝啬star，留个标记，下次迷路~**



## 逻辑图（基础类型）

> 完整版ref逻辑图，请看 Vue3硬核源码解析系列（5） ref源码解析

![](https://www.vkcyan.top/FjE3zqx5l7zpmv0is0_Fusim1mhf.png)





## 具体逻辑

### 初始化

ref的初始化非常简单，逻辑是这样的

1. 判断传入对象是否已经是ref，如果是，这直接返回，如果不是，继续代码流程
2. ref的本质就是Class **RefImpl**
3. 初始化RefImpl的时候将值保存到`_value`，同时将初始化保存到`_rawValue`
4. 通过get value实现ref.value的访问
5. 使用set value实现ref.value = xx的更新逻辑

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
    // ref API中__v_isShallow,一定为false
    // 如果value是基础类型,则toReactive返回原值
    // 如果value是复杂类型,则toReactive会将其处理成为reactive(proxy)再返回,这就意味着,此时的value是一个proxy
    this._value = __v_isShallow ? value : toReactive(value) // __v_isShallow 表示是否浅层代理
  }
  // 访问ref.value的时候触发
  get value() {
    // 配合effect阶段保存的activeEffect,将依赖收集到this.dep中
    trackRefValue(this)
    // 返回最新value
    return this._value
  }
  set value(newVal) {
    // 判断当前set的value是否存在变化, 有变化则进入if
    if (hasChange(newVal, this._rawValue)) {
      // 赋值最新原始值
      this._rawValue = newVal
      // 如果value是基础类型, 则toReactive返回原始值
      // 如果value是引用类型, 则通过toReactive将新的value其处理为proxy
      // 新的引用类型的value由于重新赋值, 与原本的effect在WeakMap中断了联系
      // 但是马上触发的ref本身的dep依赖中,将会再次将新的value与effect通过WeakMao完成依赖收集
      this._value = toReactive(newVal)
      // 触发get阶段收集在this.dep中的依赖
      triggerRefValue(this)
    }
  }
}


````

### 依赖收集

这个的参数ref即使上就是RefImpl本身，而RefImpl中确实存在dep，所以依赖收集的逻辑是这样的

1. 每个触发get事件的时候，都会访问trackRefValue
2. 假如是effect中被访问，这effect本身被保存到activeEffect中
3. 假如RefImpl的dep是不存在的，这说明是第一次进行依赖收集，则通过createDep创建new Set()
4. 将activeEffect，也就是当前正在运行的effect，添加到RefImpl的dep中，当前ref完成依赖收集

```ts
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

若干时间后，ref的value被更新，则会触发RefImpl的set value方法，在更新value的同时也会执行其内部的triggerRefValue，开始依赖触发逻辑

1. 获取到当前RefImpl中的所有dep（依赖收集阶段收集到的所有依赖）
2. 循环所有effect，并执行其fn，完成依赖触发。

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
  effect.fn()
}
```



## 如果传入的是对象？

如果传入的是对象的话，其内部的对象将会被reactive处理为proxy，相关逻辑在Vue3硬核源码解析系列（5） ref源码解析有详细解释，有兴趣的小伙伴请去改文章查看吧。
