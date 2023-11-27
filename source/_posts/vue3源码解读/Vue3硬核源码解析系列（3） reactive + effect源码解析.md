---
title: （3）vue3 reactive源码解析
categories:
  - JavaScript-2023
tags:
  - Vue
  - 源码解读
toc: true
date: 2023-02-01
---



## 专栏前言

​	本文是**vue3源码解析系列**的第三篇文档，在前两篇文章中，我们了解了vue3源码的运行、调试，以及阅读前的一些前置知识点，从本节开始，我们就可以正式的开始**vue3**的源码阅读了。

​	我们首先阅读的模块是@vue/reactivity 中的**reactive**以及相关**api**，**effect**的源代码。



在正文开始之前，我先将本节的简化版源码放出来，有兴趣的同学可以clone到本地，一边debug，一边阅读文章，这样效果更佳~

https://github.com/BlueDancers/vue3-mini/tree/reactive



## 前言

**reactive**的含义如其名称，通过**reactive**创建的对象都是具备响应式的。即**reactive**对象的改变会造成**副作用**。

于是我们引出**副作用API（effect）**，如果**effect**内部依赖了**reactive**，**则reactive的改变会重新触发effect**。

现在让我们走进案例与源码，看看究竟是如何实现响应式的。



## 案例

```js
 let { reactive, effect } = Vue
 const obj = reactive({
   name: '卖鱼强',
 })

 effect(() => {
   document.querySelector('#app').innerText = obj.name
 })

setTimeout(() => {
  obj.name = '狂飙强'
}, 2000)
```

以上测试案例，我们涉及到了三个重要的阶段

1. reactive初始化
2. effect初始化
3. reactive发生修改

最后形成了effect的自动触发，我们就从以上三个角度去切入源码实现。



## reactive初始化

> 为了方便阅读与理解，以下仅贴出核心源码

`packages/reactivity/src/reactive.ts`

```js
export function reactive(target) {
  return createReactiveObject(
    target, // reactive里面的值
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap
  )
}

function createReactiveObject(target, isReadonly, baseHandlers, collectionHandlers, proxyMap) {
  // 判断是否已经被代理过了，如果是，则获取缓存中的值，并直接返回
  // 我们这里第一次指定，必然是不存在的，所以跳过这个
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }
  // 对reactive中的变量进行代理，我们这里的target类型是obejct，targetType为common，所以接下来进入baseHandlers逻辑
  // 而baseHandlers从reactive被当做参数传递过来的，实际执行的是mutableHandlers
  const proxy = new Proxy(
    target,
    baseHandlers
  )
  proxyMap.set(target, proxy)
  return proxy
}

// reactive中变量类型为object场景下，proxy的监听逻辑会走到这里
export const mutableHandlers = {
  get, 
  set,
}
```

​	通过源码 我们可以看得出来，使用**reactive**，内部实际执行的是**createReactiveObject**，函数就是新建了**proxy**，并最终返回。

​	不过要注意一点的是，经过**reactive**处理过的对象，都会以**target**为**WeakMap**键，**proxy**为值，进行一次缓存，这样同一个值再次进行**reactive**的时候就会读取缓存中的值。

<img src="https://www.vkcyan.top/Fo-SnMAmNY3ZCNiZ_GONRZTFVefm.png" style="zoom:50%;" />



​	接下来，让我们进入初始化阶段的**mutableHandlers**，也就是**proxy**中核心的**get set**函数，看看内部做了些什么。

### 初始化读取（get）

当触发**obj.name**的读取行为的时候，就会触发代理对象的**get**函数

`packages/reactivity/src/baseHandlers.ts`

```js
const get = createGetter()

function createGetter() {
  return function get(target, key, receiver) {
    const res = Reflect.get(target, key, receiver) // 读取被代理对象
    
		// 核心逻辑(track)：依赖收集，后续单独看

    // 如果当前值是reactive则递归proxy处理
    if (isObject(res)) {
      return reactive(res)
    }
    return res
  }
}
```

get内部的逻辑很简单，通过**Reflect**完成被代理对象的读取操作。

如果被读取对象的属性是**object**则会再次进入**reactive**逻辑中进行**proxy**处理，确保嵌套对象的响应式。

> 也许有的人会说了proxy不是自身就实现了对象的拦截了吗？为什么我们还是要递归处理嵌套obj呢？
>
> 这里我给大家解释一下，proxy确实会拦截到所有操作，但是他也只能拦截当前层级的。
>
> 如果没有递归处理， obj.name.abc = 123的时候，只会触发obj.name的get事件，但是不会触发obj.name.abc的set事件。



### 初始化修改（set）

当触发`obj.name`的修改行为，将会触发代理对象的**set**函数

`packages/reactivity/src/baseHandlers.ts`

```js
const set = createSetter()

function createSetter(shallow = false) {
  return function set(target, key, value, receiver) {
    // 修改被代理数据，完成数据更新
    const res = Reflect.set(target, key, value, receiver)
    
    // 核心逻辑(trigger)：依赖触发，后续单独看
    
    return res // true
  }
}
```

通过**Reflect**完成被代理对象值的更新，最后返回本次Reflect.set的结果，完成逻辑。

总体就是对proxy的简单利用，还是很简单的嘛



### 小结

​	以上代码是去除所有边界判断，以及响应式逻辑后，reactive的核心代码；我们可以发现，其实就是**proxy + Reflect**的基础使用。

​	目前数据已经具备响应式，但是数据变化后，引用数据的**effect**如何实现自动执行呢？接下来我们就去看看effect初始化的时候究竟做了什么。



## effect初始化

### 读取 - 依赖收集（track）

​	我们回到测试demo中，根据我们使用**vue3**的预期，在初始化完成后，**effect**会触发一次，若干时间后，**setTimeout**内**set**触发，依赖`obj.name`的 **effect**的函数还会被触发一次，这又是如何实现的呢？

​	这里我要提到Vue3中第一个非常非常非常重要的概念，**依赖收集（track）**，整个reactivity都利用到了这个概念。

​	接下来，我们就要通过源码去了解，**effect**的初始化的时候，到底发生了什么，Vue3在此阶段是如何完成**依赖收集**的。

`packages/reactivity/src/effect.ts`

```js
/**
 * 当前被执行的effect
 */
export let activeEffect: ReactiveEffect | undefined

export function effect(fn) {
  const _effect = new ReactiveEffect(fn) // 首先执行new ReactiveEffect，所以我们跳转到ReactiveEffect中
  _effect.run() // 并立刻执行了run方法，run方法内实际执行的就是effect内部函数
}

export class ReactiveEffect {
  parent: ReactiveEffect | undefined = undefined
  
  constructor(
    public fn: () => T, // 这里的fn就是effect内部的匿名函数
  ) {}

  run() {
    try {
      activeEffect = this // 将effect对象，也就是new ReactiveEffect的结果，保存到activeEffect
      shouldTrack = true // 表示开始依赖收集
      return this.fn() // 这里的fn，实际上就是effect内部的匿名函数 
    }
  }
}
```

> vue3的依赖收集几乎都是通过ReactiveEffect进行完成的，简单来说就是ReactiveEffect.run一旦运行后，就会将当前正在运行的匿名函数保存到内存中，以便于proxy get事件触发的时候，收集保存在内存中的匿名函数，进而完成依赖收集。

​	effect方法内部，首先**new ReactiveEffect** 最终执行了一次**fn**，但是在执行之前，将activeEffect赋值为this，**将自身保存到了公共变量activeEffect之中**。

让我们来看看此时运行的fn是什么

```js
() => {
	document.querySelector('#app').innerText = obj.name
}
```

​	匿名函数的内部读取了**obj.name**，**触发了被代理对象obj的get方法**.

​	所以接下来我们回到get方法中，查看之前忽略的依赖收集逻辑。

`packages/reactivity/src/baseHandlers.ts`

````js
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    const res = Reflect.get(target, key, receiver) // 读取被代理对象

    if (!isReadonly) { // obj为可读代码 所以isReadony一定为false 进入if中
      track(target, TrackOpTypes.GET, key) 
    }
    return res
  }
}
// 依赖收集
function track(target, type, key) {
  if (shouldTrack && activeEffect) { // 在effect中执行run方法的时候，我们确保了shouldTrack为true activeEffect 存在值，所以进入判断
    let depsMap = targetMap.get(target) // targetMap是一个全局变量，实际上是一个new WeakMap 首次depsMap肯定是不存在的
    if (!depsMap) {
      // 这里的target为被代理对象，{name: '张三'}，该值做为key，Map作为value
      targetMap.set(target, (depsMap = new Map()))
    }
    let dep = depsMap.get(key) // 当前key为name 首次也是不存在的
    if (!dep) {
      // depsMap是一个Map结构，key是name value是createDep()的返回值，我们进入createDep
      depsMap.set(key, (dep = createDep()))
    }
    // 将dep作为参数传递到trackEffects中，此时的dep为Set
    trackEffects(dep, undefined)
  }
}

export const createDep = (effects?) => {
  const dep = new Set(effects) // 实际上就是生成了Set结构（Set我们简单理解为元素不可重复的数组）
  dep.w = 0
  dep.n = 0
  return dep
}

export function trackEffects(
  dep: Dep,
) {
  // 一系列边界判断，合法的情况下shouldTrack为true
  if (shouldTrack) {
    dep.add(activeEffect!) 
    // 将全局变量activeEffect（包含effect的匿名函数）加入到dep（Set）中
    // 到这里 我们将响应式数据与effect函数建立起了联系 标志着我们完成了依赖收集

  }
}
````

​	**effect**内部的**fn**被触发，**fn**执行中触发了**obj**的**get**，**get**内部触发了**依赖收集（track）**，**track**内部通过构建**targetMap**，来维护**变量**与**effect**之间的关系，进而实现所谓的**依赖收集**。

​	我们来梳理一下他的数据结构

- **WeakMap**
  - **key：被代理对象（{name:'张三'}）**
  - **value：Map对象**
    - **key：响应式对象的指定属性（name）**
    - **value：指定对象的指定属性的使用函数（effect的匿名函数）**
  
  ![](https://www.vkcyan.top/Fq7ZRGhuv98fGixwhk7L8UMWEiRr.png)

在**WeakMap**中，我们不仅仅收集了**effect**的匿名函数，还将**effect**与**effect中具体读取的变量建立起了联系**。

在未来的依赖触发逻辑中，weakMap将会发挥巨大作用。

到此为止，**effect**内的匿名函数执行完毕，同时我们也完成了重要的**依赖收集**。



## 修改 - 依赖触发（trigger）

继续回到demo中，2s后，**obj.name**赋值为**狂飙强**，此时的现象是**effect**中的函数自动执行了，这又是如何实现的呢？

此处首先一定是触发了代理对象**obj.name**的**set**，所以我们由此处开始分析。

`packages/reactivity/src/baseHandlers.ts`

````js
function createSetter(shallow = false) {
  return function set(target, key, value, receiver): boolean {
    const result = Reflect.set(target, key, value, receiver) // 完成被代理对象的赋值操作
		trigger(target, TriggerOpTypes.SET, key, value, oldValue)
    return result
  }
 
export function trigger(target, type, key?, newValue?, oldValue?, oldTarget?) {
  // 通过全局变量targetMap（weakMap）获取value
  // 在依赖收集阶段我们收集到了当前target，所以这时候 depsMap存在值 值为Map Map的key为name 值为Set Set内部是effect的fn
  const depsMap = targetMap.get(target)
  
 	triggerEffects(depsMap.get(key))
}
  
export function triggerEffects(dep, debuggerEventExtraInfo?) {
  const effects = isArray(dep) ? dep : [...dep] // 将set处理为数组
  for (const effect of effects) {
    triggerEffect(effect, debuggerEventExtraInfo)
  }
}
  
function triggerEffect(
  effect: ReactiveEffect, // 每一个effect都是ReactiveEffect，内部的fn都是effect的fn
) {
  // 此时的activeEffect为undefined，一定进入if中
  if (effect !== activeEffect || effect.allowRecurse) {
 		effect.run() // effect的run方法就是effect的fn，完成执行
  }
}
````

经过以上代码，我们可以了解到，**obj.name**的改变在触发了**proxy**的**set**方法的同时，也触发了**依赖触发（trigger）**。

**trigger**中，我们首先通过**{name: '狂飙强'}**，找到了**Map**，再通过**name**找到**Set**，最终找到对应的**effect**的**fn**，并进行匿名函数的执行，于是我们便看到了effect函数自动触发。

到此为止完成了整个响应式过程。



## reactive源码总结

我们简单总结一下，reactive中**依赖收集**与**依赖触发**的过程

1. 通过**proxy**处理**reactive**包裹的对象，被返回**proxy**代理对象
2. **effect**初始化，生成了类**ReactiveEffect**，并执行了其**run**方法
3. **run**方法执行后，当前**effect的fn函数本身**被保存到了**activeEffect(公共变量)**，随后执行了**effect的fn**
4. **effect的fn**触发，函数内使用到了**obj.name**，触发了代理对象的**get**
5. **get**方法内部触发了**依赖收集（track）**，配合保存到局部的**activeEffect**，最终通过**WeakMap**，建立了**effect的fn**与当前**get的属性**的联系，完成了依赖收集。
6. 若干时间后，**obj.name = '狂飙强'**，触发**proxy**的**set**，同时触发了**依赖触发（trigger）**
7. **trigger**内部通过**当前代理对象**以及**具体修改的属性**，在依赖收集阶段保存的**WeakMap**中，找到所有需要触发的**effect的fn**。
8. 触发**effect的fn函数**，完成响应式。

最后反映在我们眼前，就是**obj.name**改变的同时，所有使用到**obj.name**的**effet**都被自动触发其匿名函数，完成响应式。



## 关于vue3 reactive的面试题

#### 为什么Vue3的响应式使用WeakMap实现？

​	还记得我们前一篇文章谈到的**WeakMap**吗，一旦被代理对象被置为null，**weakMap**中该**key**将会被垃圾回收，达到性能最大化的目的



#### 简述Vue3的响应式的核心实现逻辑？

​	通过proxy递归代理对象，然后在get中完成依赖收集，在set中完成依赖触发



#### Vue3的reactive为什么不能代理简单类型？

​	reactive底层依赖proxy，但是proxy只能代理对象，无法代理基础类型。



#### 为什么reactive解构会失去响应式？

​		这里要明确一点，只有解构出来的变量是基础类型的时候，才会失去响应式，失去响应式的主要原因是基础类型无法被proxy代理。







## 总结

​	到此为止，我们的vue3中的响应式模块的第一个API，reactive源码解读就完成了；

​	总的来说逻辑还是比较复杂的，尽管我已经很努力的去反复修改与简化，但是还是能可以感觉到，有些东西很难用文字讲清楚。

​	也不知道是否可以帮助到正在阅读文章的你，如果你觉得还不错的话，还麻烦你动动小手点个赞，关注专栏，这是我输出优质文章最大的动力。

​	如果有小伙伴存在视频教程诉求的话，请评论区告诉我，我会评估出几期视频的必要性~

​	下一站，我们将前往ref。

​	



