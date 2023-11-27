---
title: Vue3硬核源码解析系列（5）ref源码解析
categories:
  - JavaScript-2023
tags:
  - Vue
   - 源码解读
toc: true
date: 2023-02-03
---



![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07d7fdd1b1a44560ae1fd052db6e37e0~tplv-k3u1fbpfcp-zoom-crop-mark:1512:1512:1512:851.awebp?)



## 前言

​	本文是[**Vue3硬核源码解析系列**](https://juejin.cn/column/7199826518570172472)的第五篇文章，在之前文章中，我们了解到了**reactive effect**的源码实现原理，并抽丝剥茧输出了[mini版本的reactive + effect](https://juejin.cn/post/7209967260898033722)，带领大家充分理解**reactive**的实现原理，同时我们也发现了**reactive**在使用上的一些局限性，比如无法代理基础类型。

​	正因为此，**Vue3**提供了另一个API **ref**，面对**proxy**无法代理基础类型数据的问题，**ref**又是如何实现其响应式的呢，本文将带领大家一起走进vue3源码世界，看看**ref**的实现原理



## 逻辑图

因为**ref**既可以传入**基础类型**，也可以传入**复杂类型**，所以其实现逻辑要比**reactive**更加复杂，并且依赖**reactive**。

![](https://www.vkcyan.top/FkmdQj_dyMoiD6Rg7PaH-Lf7FdHO.png)



## 前置知识

> 如果关于class get set已经很了解，请跳过前置知识

为了降低大家理解**ref**源码的难度，我们在正式阅读源码之前，先学习一下JavaScript的 **class**以及修饰符**get set**相关知识点

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

当访问`obj.value`的时候，会执行被**get**修饰的**value()**，打印log，并得到返回值**‘张三’**

当我们执行`obj.value = ’李四‘`，进行赋值的时候，将会执行被**set**修饰的**value()**方法，打印log，并完成变量_value的赋值



​	看到这里，大家是否有点似曾相识的感觉，**访问与赋值触发get set**，和**proxy**代理的对象的**get set**很相似，大家能理解到这一点就足够了。

​	因为ref可以代理**简单类型**，同时也可以代理**复杂类型**，并且这两种情况下的响应式实现逻辑是完全不同的。

​	所以接下来，我们从这两个角度分别解读ref的源码实现，以及其核心逻辑。

​	首先我们看相对简单的基础类型场景，从源码的角度去了解ref是如何实现响应式的。



## 基础类型场景

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

​	上述代码现象：

1. 页面初始化的时候显示“卖鱼强”

2. 2s之后，**name**发生改变，变成了“狂飙强”。

通过**现象**与我们之前分析**reactive**的经验，这个我们可以将**ref**的实现分为三大模块

1.  **初始化**
2.  **读取**（依赖收集）
3.  **赋值**（依赖触发）

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

通过源码分析，我们可以发现，**ref**的本质就是**new RefImpl**

我们ref传入的参数 原始对象被保存到_rawValue，同时将参数（“卖鱼强”）保存到-value中，便于后续的get set



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

到此为止，**ref**的基础逻辑就完成，我们已经具备给**ref**赋值、读取的能力。

但是还不具备响应式的能力，接下来就让我们看看，ref的响应式系统是如何实现的。



### 依赖收集（trackRefValue）

​	根据我们解读**reactive**的源码经验，我们可以猜到，**ref**一定是在**get**中完成依赖收集的，事实也是如此。

​	而第一次**ref**的**get**是何时触发的呢？

​	答案是初始化时期的**effect**，**effect**触发后，内部**fn**被保存到**activeEffect**中，并触发**fn**，**fn**访问了`name.value`，触发了**ref**的**get**行为，所以接下来我们前往**RefImpl**的**get**中，看看**ref**是如何完成依赖收集的。

```ts
get value() {
  // 依赖收集函数 将当前RefImpl实例传入方法
  trackRefValue(this)
  return this._value
}

export function trackRefValue(ref) {
  // shouldTrack一定为true，activeEffect在effect执行阶段保存了fn，所以一定存在
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

通过以上源码，我们可以发现，他们都公用了**activeEffect**部分的逻辑，但是**ref**收集依赖的方式与**reactive**是存在一些差别的

- **reactive**的依赖收集通过**WeakMap**完成，实现**属性、变量与effect fn**的绑定关系
- **ref**则通过自身实例内部的**dep**变量来保存所有相关的**effect fn**



### 依赖触发（triggerRefValue）

若干时间后，`name.value`的值被修改，触发**RefImpl**的**set value**

```ts
set value(newVal) {
  // 判断传入值是否与原始值不一致
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

依赖触发的逻辑就非常简单了，**set value**的同时，获取当前**ref**的**dep**，并遍历**dep**中的依赖，依次执行，完成依赖触发。



### 小结

​	到此为止，我们基础类型场景的**ref**源码解读就结束了，我们简单做一下总结，

​	相比较于**reactive**，该场景下的逻辑要稍微简单一点，相关依赖**（effect fn）**被实例本身的**dep**管理，没有构建复杂的**WeakMap**对象。

**ref**与**reactive**的收集与触发的逻辑也不相同

- ref实际上是一个**class** **RefImpl**的实例
- 数据响应并不是通过**proxy**实现，而是通过**class** 的**get** **set**修饰符实现
- 依赖收集、触发并不是通过**WeakMap**实现，而是通过**RefImpl**实例中的变量**dep**实现



## 复杂类型场景

​	大家都知道**ref**不仅可以实现基础类型的响应式，还可以实现复杂类型的响应式，我们可以说**ref**是**reactive**的超集，那**ref**是如何实现既支持基础类型也支持复杂类型的呢？

​	接下来就让我们看看复杂类型场景下的**ref**是如何完成响应式的吧。

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

首先依旧是进入**ref**函数中，开始**new RefImpl**，前面流程完全一致，所以直接我们进入**RefImpl**内部

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

​	在**constructor**逻辑中，我们可以看到**this._value = toReactive(value)**，而**toReactive**函数中，会首先识别**value**类型，如果不是**object**，原路返回，如果是**object**，将会被**reactive**函数处理，所以在该场景下，**value**将被**reactive**函数处理成**proxy**对象。

​	也就是说，此时**ref**内部的**_value**实际上成了**reactive**类型。



### 读取

​	初始化阶段，**effect**触发的时候，将会读取**obj.value.name**，，首先会访问量**obj.value**，触发**ref**的**get**方法。

​	**obj.value**获取完成后，继续去获取**obj.value.name**，而**name**已经在初始化阶段，被**toReactive**处理成了**proxy**，所以接下来，会再触发**reactive**的**get**，来获取`name`

​	也就是说，读取阶段，实际上触发了2次**get**，一次是**ref**的**get value**，一次是**proxy**的**get**，进而完成了变量的读取。

```ts
get value() {
  // trackRefValue(this) // 依赖收集，后面单独说
  return this._value // 获取到proxy类型的{name: '张三'}，进而再次触发proxy的get方法
}
```



### 赋值

若干时间后，**obj.value.name**发生**set**行为，首先依旧会触发**ref**的**get**，获取`obj.value`，然后再触发**reactive**的**set**方法，完成**name**的赋值。

整个赋值过程，实际上分别触发了ref的**get value**，和proxy的**set**，进而完成变量的赋值

```ts
//ref 本身的set在value为object，并且没有直接修改ref.value的情况下，不会被触发
set value(newVal) {}
```



到此为止，我们了解了ref在处理复杂对象时候的读取与赋值的逻辑。

读取：**先触发ref的get，再触发proxy的get**

赋值：**先触发ref的get，再触发proxy的set**





### 依赖收集

依赖收集是在**get**阶段进行完成，而通过上面的分析我们可以了解到，**ref**的**get**实际上其内部是两次**get**事件，所以我们分开来看。



#### ref的依赖收集（trackRefValue）

effect初始化阶段执行的时候，会读取`obj.value.name`，首先会触发**ref**的**get**方法

```ts
get value() {
  // 依赖收集函数 将当前ref本身传入方法
  trackRefValue(this)
  return this._value
}
```

**ref**的**get**方法触发了**trackRefValue**，会在当前**ref**的**dep**中收集到**effect**，此处逻辑与**ref**为基础类型的逻辑一致。



#### proxy的依赖收集（track）

​	**ref**的的**get**完成后，紧接着触发了**reactive**的**get**，然后**get**内部通过**WeakMap**再次完成依赖收集（相关逻辑参考[Vue3硬核源码解析系列（3） reactive + effect源码解析](https://juejin.cn/post/7202132390549553211)）。

​	我们会发现，在该阶段，我们内部实际上**触发了2次依赖收集**，**effect fn**被**ref**收集的同时，也被**proxy**收集了。



### 依赖触发

因为ref内部是一个对象，所以赋值也存在多种方式，这依赖触发存在多种方式



#### 对象属性触发依赖

```ts
obj.value.name = '狂飙强'
```

这种**不会破坏RefImpl初始化阶段其内部构建的proxy**，仅修改已有**proxy**内部变量的值。

首先触发的是**obj.value**的**get**行为（此时没有**effet**在执行，不会发生依赖收集）。然后**ref**的**get**函数返回**proxy**对象 `{name:'卖鱼强'} `，紧接着触发**proxy**的**set**，并完成依赖触发（proxy的依赖触发请看这里[Vue3硬核源码解析系列（3） reactive + effect源码解析](https://juejin.cn/post/7202132390549553211)）。



#### 对象触发依赖

```ts
obj.value = {
  name: '狂飙强'
}
```

第二种方式首先触发**obj.value**的**set**行为，同时替换掉ref的值，**注意这会破坏RefImpl初始化构建的_value的proxy**，进而导致**WeakMap**中已有的**依赖关系断裂**

然后执行**triggerRefValue**，触发，ref本身在get阶段收集了相关effect fn，。

effect fn被触发后，再次触发**ref的get**，**proxy的get**，并帮助**proxy**又重建了与**effect fn**之间的依赖关系。

这就是为什么存在依赖收集2次的原因。



到此为止，我们的**ref**核心源码分析就全部完毕了。



## 关于ref的一些问题

**Q：为啥一定要.value，不能干掉吗？**

A：非常遗憾，value是去不掉的，因为ref依赖class get set 进行实现，在当前实现的场景下，可以简写为v，但是无法去除

**Q：我是不是可以完全使用ref，不用reactive？**

A：是的，可以完全使用ref，因为ref会根据你传入的类型，自动识别内部是否需要使用reactive，但是读过源码的同学知道ref在处理响应式系统中，存在重复收集依赖的场景，如果你有极致的性能要求，建议复杂类型依旧使用reactive完成，业务开发场景则无所谓。



如果还有其他问题，请评论区提问~



## 总结

​	通过对ref源码的阅读，我们可以察觉到，如果仅仅聚焦基础类型的ref，其实底层实现还是比较简单的，所以建议有兴趣的同学渐进式的阅读源码，先完成基础类型场景的源码解读，再进行复杂类型的源码解读，这样事半功倍~

​	如果有任何问题，请评论区留言~

​	下一个阶段，我将手摸手带大家完成**mini版本vue3 ref API**，帮助大家深入理解**ref**~

​	
