---
title: （4）vue3 ref源码解析
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-02-03
---



## 前言

​	在上一篇文章中，我们了解了**reactive**的实现原理，也发现了**reactive**在使用上的局限性，也因为此**Vue3**提供了一个新的API **ref**，面对**proxy**无法代理基础类型数据的问题，**ref**是如何解决的呢？

本文我们带着问题一起走进ref的世界。



## 前置知识

​	为了降低大家理解**ref**源码的难度，我们在正式源码阅读之前，先学习一下JavaScript的 **class**以及修饰符**get set**相关知识点

```js
class Obj {
  _value = '张三'
  get value() {
    console.log('value的get行为触发')
    return this._value
  }
  set value(val) {
    console.log('value的set行为触发', val)
    this._value = val
  }
}

let obj = new Obj()
```

get： 被get修饰的方法，允许通过**属性读取**的方式，触发方法

set： 被set修饰的方法，允许通过**属性赋值**的方式，触发方法

当访问`obj.value`的时候

会触发被**get**修饰的**value()**，打印log，并得到返回值‘张三’

当为`obj.value`赋值的时候

将会触发被**set**修饰的**value()**方法，打印log，并完成变量_value的赋值



​	看到这里，大家是否有点似曾相识的感觉，**访问与赋值触发get set**，和我们被**proxy**代理的对象很相似，大家能理解到这一点就足够了。

​	接下来，我们正式开始源码分析。



## Ref基础类型

​	使用过**vue3**的都知道，例如**string number**这样的基础类型，必须使用**ref**完成响应式，不过复杂类型**ref**也可以完成响应式。

​	而这2种情况，ref的内部逻辑是存在一些差别的，我们先从相对简单的基础类型开始。

### 案例

```js
let { ref, effect } = Vue

const name = ref('卖鱼强')
effect(() => {
  document.querySelector('#app').innerText = name.value
})

setTimeout(() => {
  name.value = '狂飙强'
}, 2000)
```

​	上述代码现象：页面初始化的时候显示“卖鱼强”，2s之后，依赖触发，变成了“狂飙强”。通过现象与我们之前分析reactive的经验，这个我们可以将ref分为三大模块 **初始化** **读取（依赖收集）** **赋值（依赖触发）**

### 初始化

`packages/reactivity/src/ref.ts`

````ts
export function ref(value?: unknown) {
  // ref 实际上就是createRef
  return createRef(value, false)
}

function createRef(rawValue: unknown, shallow: boolean) {
  // 如果已经是ref，则直接返回
  if (isRef(rawValue)) {
    return rawValue
  }
  // ref API 参数shallow 为 false 含义是 代理是否是浅层的,浅层则只会代理第一层数据
  // ref 就是RefImpl的实例
  return new RefImpl(rawValue, shallow)
}

class RefImpl<T> {
  private _value: T // 被代理对象
  private _rawValue: T // 原始对象

  public dep?: Dep = undefined // Dep是reative阶段声明的Set, 内部存放的是ReactiveEffect
  public readonly __v_isRef = true // 将RefImpl实例默认为true, 未来的isRef判断就一定为true

  constructor(value: T, public readonly __v_isShallow: boolean) { 
    // 寻找原始类型，如果是基础类型不会做任何处理
    this._rawValue = toRaw(value) 
    
    // 如果value是基础类型，toReactive内部不会做任何处理
    this._value = toReactive(value)
  }

  get value() {
    return this._value
  }

  set value(newVal) {
    newVal = toRaw(newVal)
    // 判断新旧值是否一致，不一致进入if
    if (hasChanged(newVal, this._rawValue)) {
      // 每次value的值发生修改的时候，都保存一下原始对象
      this._rawValue = newVal
     	// 如果value是基础类型 toReactive不会做任何处理
     	// 如果value是复杂类型，则重新进行proxy处理
      this._value = toReactive(newVal)
      
      // 依赖触发，后面单独说
    }
  }
}
````

​	通过源码分析，我们可以发现我们使用ref的本质就是`new RefImpl`，在RefImpl的内部，如果是基础类型，则通过class本身的get set完成对读取与赋值的操作的监听，然后更新RefImpl的_value，实现响应式



### 读取

调用`name.value`的时候，会触发**RefImpl**的**get value()**，方法内部返回最新的_value，完成读取。

```ts
get value() {
  // trackRefValue(this) // 依赖收集，后面单独说
  
  return this._value
}
```

### 赋值

`name.value`发生赋值的时候，会触发**RefImpl**的**set value()**方法，方法内部进行_value的赋值，完成数据更新。

```ts
set value(newVal) {
  // 判断新旧值是否一致，不一致进入if
  if (hasChanged(newVal, this._rawValue)) {
    // 如果value是基础类型 toReactive不会做任何处理
    this._value = toReactive(newVal)

    // triggerRefValue(this)// 依赖触发，后面单独说
  }
}
```



### 响应

#### 依赖收集

​	根据我们解读**reactive**的源码经验，我们可以猜到，**ref**一定是在**get**中完成依赖收集；**effect**触发，内部**fn**被保存到**activeEffect**中，并触发**fn**，**fn**访问了`name.value`，触发了**ref**的**get**行为，所以接下来我们前往**RefImpl**实例中的**get**中，其内部是符合完成依赖收集的。

```ts
get value() {
  // 依赖收集函数 将当前RefImpl实例传入方法
  trackRefValue(this)
  return this._value
}

export function trackRefValue(ref) {
  // shouldTrack一定为true，activeEffect在effect执行阶段保存了fn，所以也一定为true
  if (shouldTrack && activeEffect) {
    // createDep我们在reactive中见过，含义为创建一个Set
    // 所以这个实际函数是给RefImpl实例的dep赋值为Set，然后在传入trackEffects方法
  	trackEffects(ref.dep || (ref.dep = createDep()))
  }
}

export function trackEffects(dep: Dep,) {
  // 将当前activeEffect，也就是effect的fn，保存到当前RefImpl实例的dep中，effect成功被ref依赖收集到实例的dep中
 	dep.add(activeEffect)
}
```

通过以上源码，我们可以发现，他们都公用了**activeEffect**部分的逻辑，但是**ref**依赖收集的方式与**reactive**是存在一些差别的

- **reactive**的依赖收集通过**WeakMap**，完成**属性、变量与effect**的绑定
- **ref**则通过将相关的**effect**保存到自身实例内部的dep变量中，进而完成依赖收集



#### 依赖触发

​	若干时间后，`name.value`的值修改，触发**RefImpl**实例**set**

```ts
set value(newVal) {
  if (hasChanged(newVal, this._rawValue)) {
    // 完成赋值
    this._value = toReactive(newVal)
    // 依赖触发
    triggerRefValue(this)
  }
}

export function triggerRefValue(ref: RefBase<any>) {
  if (ref.dep) { // dep为依赖收集阶段收集到的依赖，内部为effect的fn
    triggerEffects(ref.dep)
  }
}

export function triggerEffects(dep: Dep) {
  const effects = isArray(dep) ? dep : [...dep] // 转为数组
  for (const effect of effects) {
    	// 进入依赖触发函数
      triggerEffect(effect)
  }
}

function triggerEffect(effect: ReactiveEffect) {
  // 依次通过run触发被收集的effect的fn，至此完成依赖触发工作
  effect.run()
}

```

​	到此为止，我们完成了**ref**为基础类型的情况下，响应式相关逻辑。相比较于**reactive**，逻辑要稍微简单一点，因为该场景下，相关**effect**函数被实例本身的**dep**管理，没有构建复杂的**WeakMap**对象。

​	**ref**与**reactive**的收集与触发的逻辑也不相同，其依赖收集与依赖触发的上层逻辑有所差别，**ref通过set value手动触发**、**reactive通过字段的赋值自动触发set**，然后找到需要触发的依赖，完成依赖触发。



### 小结

​	到此为止，我们基础类型场景的**ref**源码解读就结束了，我们简单做一下总结

- ref实际上是一个**class** **RefImpl**的实例
- 数据响应并不是通过**proxy**实现，而是通过**class** 的**get** **set**修饰符实现
- 依赖收集、触发并不是通过**WeakMap**实现，而是通过**RefImpl**实例的变量**dep**实现



## Ref复杂类型

​	我们就想使用reactiv一样使用ref即可

### 案例

```ts
let { ref, effect } = Vue

const obj = ref({
  name: '卖鱼强'
})

effect(() => {
  document.querySelector('#app').innerText = obj.value.name
})

setTimeout(() => {
  obj.value.name = '狂飙强'
}, 4000)
```



### Ref初始化

首先依旧是进入**ref**函数中，开始**new RefImpl**，前面流程一致，所以我们进入**RefImpl**内部

```ts
class RefImpl<T> {
  private _value: T // 被代理对象
  private _rawValue: T

  public dep?: Dep = undefined // Dep是reative阶段声明的Set,内部存放的是ReactiveEffect
  public readonly __v_isRef = true // 将RefImpl的实例全部置为true,下次isRef判断就会为true

  constructor(value: T, public readonly __v_isShallow: boolean) {
    this._rawValue = toRaw(value) // toRaw 获取原始数据
    this._value = toReactive(value) // 跳转到toReactive函数中 并且最终会获取到一个proxy对象
  }

  get value() {}

  set value(newVal) {}
}

export const toReactive = <T extends unknown>(value: T): T =>
  isObject(value) ? reactive(value) : value // value为object，进入reactive(value)逻辑 最终返回一个proxy的对象
```

当**ref**的**value**是一个**object**的时候，我们可以发现，在**RefImpl**的**constructor**逻辑中，**value**被**reactive API**处理成了**proxy**对象，也就是说，此时ref内部的**_value**实际上就是一个**reactive**。



### 读取

调用`obj.value.name`的时候

​	首先会触发**ref**的**get**方法，获取`obj.value`，而object类型的value在创建实例阶段，已经被toReactive处理成了proxy，所以接下来，会再触发**reactive**的**get**方法，获取`name`

​	也就是说，读取阶段，实际上触发了2次**get**，一次是**ref**本身的，一次是**proxy**的**get**，最终完成变量的读取。

```ts
get value() {
  // trackRefValue(this) // 依赖收集，后面单独说
  
  return this._value // 获取到proxy类型的{name: '张三'}，进而再次触发proxy的get方法
}
```



### 赋值

`obj.value.name`发生赋值的时候

​	首先会触发**ref**的**get**，获取`obj.value`，然后触发**reactive**的**set**方法，完成name的赋值。

​	整个赋值过程，实际上触发了一次**get**，一次**set**，**get**是**ref**的，目的是读取`obj.value`，**set**是**proxy**的，目的是为`name`赋值

```ts
//ref 本身的set在value为object，并且没有直接修改ref.value的情况下，不会被触发
set value(newVal) {}
```



### 响应

#### 依赖收集

​	effect第一次执行的时候，会读取`obj.value.name`，首先会触发**ref**的**get**方法

```ts
get value() {
  // 依赖收集函数 将当前ref本身传入方法
  trackRefValue(this)
  return this._value
}
```

​	**ref**的**get**方法触发了**trackRefValue**，会在当前**ref**的**dep**中收集到**effect**，此处逻辑与ref为基础类型的逻辑一致。

​	**ref**的依赖收集完成后，紧接着触发了**reactive**的**get**，然后**get**内部通过**WeakMap**再次完成依赖收集。

​	我们会发现，在该阶段，我们内部实际上**触发了2次依赖收集**，一次是**ref**本身收集，将依赖存储在自身的**dep**中，第二次是reactive出发**get**事件，依赖存放在**WeakMap**中。



#### 依赖触发

以下2种赋值方式，从现象看似乎一致，但是内部流程是完全不一样的

##### 第一种情况

```ts
obj.value.name = '狂飙强'
```

第一种**不会破坏RefImpl初始化构建的proxy**，仅修改的是已有的**proxy**内部的变量。

1. 首先触发的是`obj.value`的**get**行为（此时没有**effet**在执行，所以**ReactiveEffect**一定为空，不会发生依赖收集）。

2. 然后**ref**的**get**函数返回**proxy**对象 `{name:'卖鱼强'} `，紧接着触发**proxy**的**set**，在set中，完成依赖触发。

   

##### 第二种情况

```ts
obj.value = {
  name: '狂飙强'
}
```

​	第二种方式，因为引用类型本身发生了变化，**会破坏RefImpl初始化构建的_value的proxy**，进而导致**WeakMap**中已有的依赖关系断掉，然后再通过**ref**本身的**set**去触发effect，进而触发`obj.value.name`的get，重建**ref**内部的**proxy**与**effect**的联系，**这也是为什么value为object的时候，依赖会被收集2次**

1. 首先触发`obj.value`的**set**行为
2. 新的**value**重新通过**reactive**函数重新生成为**proxy**，之前的proxy被抛弃。
3. 触发**set**逻辑中的**triggerRefValue**，进而触发**ref**中**dep**中存放的**ReactiveEffect**，完成依赖触发。
4. 依赖触发中，再次访问刚才被**reactive**处理过的新的`obj.value`，新的`obj.value`与**effect**再次建立联系，完成依赖收集。



## 总结

​	到此为止，我们的**ref**核心源码分析就全部完毕了，**针对基础类型proxy无法进行代理的情况，ref实际上是使用class的get set修饰符进行完成，复杂类型的ref，实际上是通过class + reactive完成的**。

​	如果`ref.value`是基础类型，读取会触发**class** **get**，赋值会触发**class** **set**。

​	如果`ref.value`是复杂类型

​		读取`res.value.xxx`会先触发**class** **get**再触发**proxy** **get**，赋值会先触发**class** **get**再触发**proxy** **set**。

​		读取`res.value`会触发**class** **get**，赋值会触发**class** **set**，同时完成新的**value**与**effect**完成依赖收集

​	

​	ref的源码解读完成后，我们下一个阶段将开始分析watch与computed。

