---
title: （3.1）vue3 手摸手实现mini版reactive.md
categories:
  - JavaScript-2023
tags:
  - Vue
  - 源码解读
toc: true
date: 2023-03-13
---

## 专栏前言

​	在上一节，我们完成了**vue3**的**reactive**的核心源码解读，总的来说还是非常复杂，文章的表现能力有限，我想可能有很多同学无法完全理解其精髓，所以在本节，我将带领大家完成mini版本源码的输出。

​	**仅保留最核心逻辑，极大减低阅读难度，200行代码实现reactive + effect**，话不多说，我们直接开始！



[简易版vue3仓库地址](https://github.com/BlueDancers/vue3-mini/tree/reactive)，**还请大家不要吝啬star，留个标记，下次迷路~**



## 逻辑图

![](https://www.vkcyan.top/FmPJt0I04Cs834SauJHg3WgJgy5O.png)



## 逻辑流程

### reative初始化

​	将**reactive**处理为**proxy**，同时预先声明**set** **get**方法，赋值、取值均通过**Reflect**完成，**get**中存在**track**（依赖收集），**set**中存在**trigger**（依赖触发），完成**reactive**的初始化。



### effect初始化（依赖收集）

> cb  = callback = 回调函数 effect(() => {})   // () => {} 就是cb

​	初始化**effect**函数，通过一个类**ReactiveEffect**运行其**cb**，同时将当前**cb**存储到公共变量，**cb**中读取了**reactive**的属性，进而触发**proxy**的**get**，同时完成**track**（依赖收集），让**reative**收集到存储在公共变量中的**effect**的**cb**，至此完成依赖收集。

```
重点：reactive - key - effect // 依赖收集完成后，将会形成这样的从上到下的可追溯关系
```



### reactive改变（依赖触发）

​	若干时间后，**reactive**属性发生变化，触发**reactive**属性的赋值操作，进而触发**proxy**的**set**事件，同时完成trigger（依赖触发），根据指定的**reative + key**，找到特定**effect**运行，完成依赖触发，形成响应式。

```
重点：reactive + key 找到指定effect，进而完成触发
```



## 具体逻辑

### proxy处理

​	经过真实的源码分析之后，我们都知道**reactive**实际上就是**proxy**，我们仿照源码的格式，将**reactive**经过**proxy**处理后返回就好了。

```ts
// 缓存proxy
const reactiveMap = new WeakMap<object, any>()

// 入口函数
export function reactive(target: object) {
  return createReactiveObject(target, mutableHandlers, reactiveMap)
}

// 处理被代理对象
function createReactiveObject(
  target: object,
  baseHandlers: ProxyHandler<object>,
  proxyMap: WeakMap<object, any>
) {
  // 如果已经被代理过,这直接返回结果
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }
  const proxy = new Proxy(target, baseHandlers)
  proxyMap.set(target, proxy)
  return proxy
}
```



### get set函数编写

​	以上代码我们完成了变量的**proxy**处理，为了完成后续的响应式，我们需要预先声明好**get set**函数，我们依旧仿照源码格式，并只保留核心逻辑，**get**阶段返回结果，并触发**（依赖收集）track**，**set**阶段通过**Reflect**完成赋值，并触发**（依赖触发）trigger**

```ts
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
}

const get = createGetter()
const set = createSetter()

function createGetter() {
  return function get(target: object, key: string, receiver: object) {
    const res = Reflect.get(target, key, receiver)
    // 核心逻辑: 依赖收集
    track(target, key)

    if (isObject(res)) {
      return reactive(res)
    }
    return res
  }
}

function createSetter() {
  return function set(target: object, key: string, newValue: unknown, receiver: object) {
    const res = Reflect.set(target, key, newValue, receiver)
    // 核心逻辑: 依赖触发
    trigger(target, key, newValue)
    return res
  }
}
```



### effect实现

**effect**的核心的实现，就是在运行**effect**的时候**保存当前的this**，以便于后续流程中的**依赖收集**，所以其核心代码非常简单，保证一下2点即可。

- 运行effect本身
- 保存effect的fn到activeEffect即可

```js
export function effect<T = any>(fn: () => T) {
  const _effect = new ReactiveEffect(fn)
  _effect.run()
}

export let activeEffect: ReactiveEffect | undefined

export class ReactiveEffect<T = any> {
  constructor(public fn: () => T) {}
  run() {
    try {
      activeEffect = this
      return this.fn()
    } finally {
      activeEffect = undefined
    }
  }
}
```



### 依赖收集（track）

按照时序，**effect**函数初始化阶段会执行，**effect**函数本身也会被保存到**activeEffect**中，同时触发**effect**中的**reactive**中的**get**事件，进而触发**track**，我们在**track**中完成 **reactive- key - effect之间关系的构建**，确保以后可以在**set**阶段找到**指定的effet的fn**即可。

```ts
export function track(target: object, key: string) {
  if (!activeEffect) {
    return
  }
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    depsMap = new Map()
    targetMap.set(target, depsMap)
  }
  let dep = depsMap.get(key)
  if (!dep) {
    dep = createDep()
    depsMap.set(key, dep)
  }
  trackEffects(dep)
}

function trackEffects(dep: Dep) {
  dep.add(activeEffect!)
}
```



### 依赖触发（trigger）

若干时间后，**reative**中的某个属性发生了变化，也就会发生**set**事件，这时候其实就很简单了，我们只需要通过**reactive - key**找到对应的**effect的fn**，然后执行即可。

**这就形成了我们看到的“响应式”**。

```ts
export function trigger(target: object, key: string, newValue: unknown) {
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    return
  }
  const dep: Dep | undefined = depsMap.get(key)

  if (!dep) {
    return
  }
  triggerEffects(dep)
}

function triggerEffects(dep: Dep) {
  const effects = [...dep]
  for (const effect of effects) {
    triggerEffect(effect)
  }
}

function triggerEffect(effect: ReactiveEffect) {
  effect.fn()
}
```



## 最后

​	到此为止，我们简易版的**reactive + effect**的全部源码就完成了，虽然**vue3**的源码很复杂，但是我们抽丝剥茧，仅保留核心逻辑，大幅降低**vue3**源码阅读的难度，让绝大多数的前端开发者都可以读懂核心实现~

​	最后，建议大家[clone](https://github.com/BlueDancers/vue3-mini/tree/reactive)源码到本地实际运行一下，静下心来一步一步调试，将简易版逻辑弄明白，有兴趣的可以在看看正式的vue3源码，然后在简历上留下浓墨重彩的一笔~

