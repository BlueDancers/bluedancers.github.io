---
title: （3）vue3 reactivity-reactive源码解析
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-02-01
---



## 前言

​	从本篇文章开始，我们正式进入源码解析环节，**vue3**源码相对清晰，每个模块相互独立，我们首先分析**@vue/reactivity** 中的**reactive**。



## 正文

### 测试代码

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

​	以上测试案例，我们涉及到了reactive的**初始化、读取、修改、响应**，接下来走入源码的世界，看看功能是如何实现的。



### reactive初始化

> 为了方便阅读与理解，文中仅贴出最核心的代码

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
  deleteProperty, // 非核心逻辑 忽略（删除被代理对象中字段 以及删除相关依赖）
  has, // 非核心逻辑 忽略 （判断字段是否存在与对象中）
  ownKeys // 非核心逻辑 忽略（返回当前对象的所有字段）
}
```

​	通过源码 我们可以看得出来，使用**reactive**，内部实际执行的是**createReactiveObject**，函数就是新建了**proxy**，并最终返回。

​	不过要注意一点的是，经过**reactive**处理过的对象，都会以**target**为**WeakMap**键，**proxy**为值，进行一次缓存，这样同一个值再次进行**reactive**的时候就会读取缓存中的值。

​	接下来，让我们看看**proxy**中核心的**get set**函数吧，看看内部做了些什么。

### 读取（get）

当触发`obj.name`的读取行为的时候，就会触发代理对象的**get**函数

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

​	get内部的逻辑很简单，通过**Reflect**完成被代理对象的读取操作。

​	如果被读取对象的属性是**object**则会再次进入**reactive**逻辑中进行**proxy**处理，确保嵌套对象的响应式。



### 修改（set）

当触发`obj.name`的修改行为，将会触发代理对象的**set**函数

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

​	通过**Reflect**完成被代理对象值的更新，最后返回本次Reflect.set的结果，完成逻辑。



### 小结

​	以上代码是去除所有边界判断，以及响应式逻辑后，reactive中主流程的代码，我能可以发现，其实就是对**proxy + Reflect**的基础使用。

​	目前数据已经具备响应式，但是数据变化后，引用数据的**effect**自动执行，也就是**响应**，目前还未实现。接下来我们就去看看，他是如何实现的。



### 依赖收集（track）

​	我们回到测试demo中，根据我们使用**vue3**的预期，在初始化完成后，**effect**会触发一次，若干时间后，**setTimeout**内**set**触发，依赖`obj.name`的 **effect**的函数还会被触发一次，这又是如何实现的呢？

​	我们通过源码去了解，effect的初始化的时候到底发生了什么，Vue是如何完成**依赖收集**与**依赖触发**的

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

effect方法内部最终执行了一次**fn**，但是在执行之前，将activeEffect赋值为this，将自身保存到了公共变量之中。

而匿名函数的内部读取了`obj.name`，**触发了被代理对象obj的get方法**，所以接下来我们回到get方法中，查看之前忽略的依赖收集逻辑。

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

### 小结

​	**effect**内部的**fn**被触发，也是我们看到的初始化阶段的第一次执行，**fn**执行中触发了**obj**的**get**，**get**内部触发了**依赖收集（track）**，**track**内部构建了一个复杂对象**targetMap**，来维护变量与**effect**之间的关系。

​	我们来梳理一下他的数据结构

- **WeakMap**
  - **key：被代理对象（{name:'张三'}）**
  - **value：Map对象**
    - **key：响应式对象的指定属性（name）**
    - **value：指定对象的指定属性的使用函数（effect的匿名函数）**

<img src="https://www.vkcyan.top/Fl5yIZxuITKTmxJaKmaQBiThPhA-.png" style="zoom:33%;" />

​	在WeakMap中，我们不仅仅收集了所有的fn函数，还将fn函数与fn函数被读取的变量建立起了联系，在未来的依赖触发逻辑中，weakMap将会发挥巨大作用。

到此为止，effect内的匿名函数执行完毕，同时我们也完成了重要的依赖收集



### 响应（trigger）

​	继续回到demo中，2s后，**obj.name**赋值为李四，此时的现象是effect中的函数自动执行了，这又是如何实现的呢？

​	此处首先一定是触发了代理对象的**set**，所以我们由此处开始分析。

````js
function createSetter(shallow = false) {
  return function set(
    target,
    key,
    value,
    receiver
  ): boolean {
    let oldValue = target[key] // 保存之前的值 张三

    // 当前target是数组,并且本次修改的的是执行下标
    // 如果是 这判断是否是现有下标,是 true 否 false
    // 如果不是 这判断是否为自身属性 是 true 否 false
   	// 当前场景下为true
    const hadKey =
      isArray(target) && isIntegerKey(key) // isIntegerKey 判断是不是数值型字符串
        ? Number(key) < target.length
        : hasOwn(target, key)

    const result = Reflect.set(target, key, value, receiver) // 完成被代理对象的赋值操作
    if (target === toRaw(receiver)) { // 判断是否还是一个对象，如果不是则说明被代理对象改变，则不进行后续逻辑 当前场景一定为true
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        // 当前hadKey为true并且新旧值不一致，所以触发以下trigger方法
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
 
 export function trigger(
  target, type, key?, newValue?, oldValue?, oldTarget?
) {
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

​	经过以上代码，我们可以看到，**obj.name**的改变在触发了**proxy**的**set**方法的同时，也触发了**依赖触发（trigger）**，**trigger**中，通过依赖收集阶段生成的**targetMap**找到了对应的**effect**的**fn**，并完成执行。

​	到此为止完成了整个响应式过程。



### reactive源码总结

我们简单总结一下，reactive中**依赖收集**与**依赖触发**的过程

1. 通过proxy处理reactive包裹的对象，被返回proxy代理对象
2. effect初始化，生成了类**ReactiveEffect**，并执行了其run方法
3. run方法执行后，当前**ReactiveEffect**被保存到了**activeEffect(公共变量)**，随后执行了**effect的fn**
4. **effect的fn**触发，函数内使用到了`obj.name`，触发了代理对象的**get**
5. **get**方法内部触发了**依赖收集（track）**，配合保存到全局的**ReactiveEffect**，最终通过**WeakMap**，建立了**effect的fn**与当前**get的属性**的联系，完成了依赖收集。
6. 若干时间后，代理对象的**set**触发，同时触发了**依赖触发（trigger）**
7. **trigger**内部通过**当前代理对象**以及**具体修改的属性**，在依赖收集阶段保存的**WeakMap**中，找到所有需要触发的effect的fn。
8. 触发effect的fn，完成响应式



最后反映在我们眼前，就是`obj.name`改变的同时，所有使用到**obj.name**的**effet**都被自动触发了**fn**，完成响应式。



### 补充知识（WeakMap）

​	虽然我们完成了源码阅读，但是我们还需要补充一个重点API，**WeakMap**，**weakMap**和**map**一样都是**key value**格式的对象，但是他们还是存在一些差别。

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



反映到Vue3源码中，一旦被代理对象被置为null，**weakMap**中该**key**将会被垃圾回收，达到性能最大化的目的。



### reactive的局限性

1. 解构之后会失去响应性
   - 如果解构对象依旧是Object，这依旧具备响应式，但是，如果解构对象是基础类型，则无法被proxy代理，无法具备响应式
2. reactive只能代理object，其他类型无法被代理

因为以上不足，所有vue3提供了可以解决以上问题的方法的API**ref**，所以下一站，我们将前往ref。





