---
title: （5）vue3 reactivity-computed源码解析
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-02-03
---



## 前言

熟悉**computed**的同学都知道，**computed**会在依赖属性发生变化的时候自动更新结果。

他有一个重要的特点：**计算值是可缓存的，只有依赖项发生变化的时候，才会重新计算**

​	通过之前的文章，我们已经了解了**reactive**，**ref**的实现原理，已经对**vue**响应式机制有所了解，今天我们就来了解一下**computed**是如何实现的。

## 正文

### 测试代码

```ts
const obj = reactive({
  name: '张三'
})

const showName = computed(() => {
  return '我叫' + obj.name
})

effect(() => {
  document.querySelector('#app').innerText = showName.value
})

setTimeout(() => {
  obj.name = '李四'
}, 2000)
```

​	以上代码运行后，我们可以看到如下现象

- 页面显示：**我叫张三**
- 2s后，页面显示**我叫李四**

先让我们看看**computed**内部都做了些什么吧

### computed初始化

```ts
// 入口函数
export function computed<T>(
  getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>) {
  let getter: ComputedGetter<T>
  let setter: ComputedSetter<T>
  // 传入的是否是一个方法
  const onlyGetter = isFunction(getterOrOptions)
  if (onlyGetter) {
    // 如果是方法, 则直接赋值到getter
    // 同时屏蔽setter行为
    getter = getterOrOptions
    // dev环境下 set函数给予提示
    setter = __DEV__
      ? () => {
          console.warn('Write operation failed: computed value is readonly')
        }
      : NOOP
  } else {
    // 如果不是方法,则认为是对象,将对象中的get set分别赋值到getter setter中
    getter = getterOrOptions.get
    setter = getterOrOptions.set
  }
  const cRef = new ComputedRefImpl(getter, setter, onlyGetter || !setter, isSSR)
  return cRef
}
```

​	入口函数的逻辑还是非常简单的，如果传入的是一个匿名函数，这处理为**getter**，如果传入的是对象，这赋值**getter** **setter**，这部分逻辑符合Vue官方文档的描述。

​	抹平两种传参方式的差异后，**new ComputedRefImpl**，并返回，看来核心实现都在**ComputedRefImpl**中了，我们接下来就进入该类中看看吧。

```ts
// 计算属性的响应式也是通过class get set去实现的
export class ComputedRefImpl<T> {
  public dep?: Dep = undefined // 依赖收集处(effect)

  private _value!: T  // 存储计算属性结果的值
  public readonly effect: ReactiveEffect<T> // 存储依赖
  public readonly __v_isRef = true // 所有的计算属性也会被识别为ref
  public _dirty = true // 判断是否需要重新计算
  
  constructor(
    getter: ComputedGetter<T>,
    private readonly _setter: ComputedSetter<T>,
    isReadonly: boolean, // 是否只读,如果存在setter,则为false
  ) {
    // 将计算属性的识别为effect，初始化一个ReactiveEffect
    // 初始化阶段仅仅声明 但是却没有触发
    this.effect = new ReactiveEffect(getter, () => {
      // 脏变量（_dirty）的本质就是判断什么时候去触发依赖
      // 脏变量为false的时候才会触发  
      if (!this._dirty) { 
        this._dirty = true
        // 触发依赖
        triggerRefValue(this)
      }
    })
    this.effect.computed = this // 赋值ReactiveEffect中的computed为当前this
  }
  
  get value() {}
  set value(newValue: T) {}
}
```

​	在**ComputedRefImpl**初始化阶段，我们看到了非常熟悉的api，**ReactiveEffect**，在我们的effect源码分析中，我们使用这个api来完成关键步骤**依赖收集**，但是这里却又有点不同，完成声明却没有执行，并且还传入了第二个参数，一个匿名函数，现在还无法发挥它的作用，我们需要后面再说。

​	总的来说，初始化的时候，生成了一个**ReactiveEffect**并保存到当前类的**effect**变量中，忘记**ReactiveEffect**记得去**reactive**章节看看其作用哦。



### computed的get行为

当我们的实例中的api，**effect**初次执行的时候，我们会触发`showName.value`的**get**，也就是说，会触发**ComputedRefImpl**的**get**。

```ts
// 被读取的时候触发
get value() {
  // 依赖收集
  trackRefValue(this)
  // 判断是否需要更新，如果需要则进入函数
  if (this._dirty) {
    // 如果更新过，这下一次就不需要更新了，
    this._dirty = false
    // effect的run执行，也就是执行computed的fn，将会得到一次计算属性的结果
    this._value = this.effect.run()! 
  }
  // 返回computed的结果
  return this._value
}

export function trackRefValue(ref: RefBase<any>) {
  // 首次computed内部的dep是不存在的，会通过createDep生成一个Set
  trackEffects(ref.dep || (ref.dep = createDep()))
}

export function trackEffects(dep: Dep) {
  // 将activeEffect，此时是effect的fn，收集到computed的dep中
  dep.add(activeEffect!)
}
```

​	当我们触发**computed**的**get**的时候，首先会触发**trackRefValue**，将当前**activeEffect**收集到类中的**dep**中，这正是依赖收集，这里effect被收集到了computed的dep中，**建立起了computed与其被依赖项的联系**。

​	然后判断**_dirty**是否为**true**，默认是**true**，所以进入判断中，首先将**_dirty**改为**false**，下一次则不会进入判断，直接返回**computed**之前的结果，之后执行初始化阶段声明的ReactiveEffect，也就是我们computed本身。

​	**computed**的**effect.run**一旦触发，**activeEffect**将会被替换当前api **effect的fn** ，并且触发**computed**依赖项**obj.name**的**get**，再次触发proxy的依赖收集，于是**obj.name**成功收集到了**computed**内部的**effect**，**proxy与computed建立了联系**；同时返回了最新的computed结果。

​	**computed的get行为触发的时候，我们发现computed收集了effect，reactive收集了computed，三者之间建立起了联系。**

​	而且我们还发现了**_dirty**变量的含义，第一次**get**后，赋值为**false**，下一次就不需要再计算**computed**的结果，实现了数据缓存。

<img src="http://www.vkcyan.top/FulT2b9ii1-iTvws8zj2z8vEn0Hn.png" style="zoom:33%;" />



### obj.name触发set，计算属性触发

**2s**后，我们触发了**obj.name**的**set**，所以首先触发**obj.name**的依赖触发，此时我们将可以通过WeakMap会找到之前收集到**computed**，我们直接进入依赖触发的逻辑。

````ts
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)
  if (!depsMap) {
    return
  }
  let deps: (Dep | undefined)[] = []
  deps.push(depsMap.get(key))
  triggerEffects(deps[0]) // 找到了之前收集到的computed中的effect
}
// 按照常理来说，我们找到指定依赖之后，就是触发依赖，但是计算属性有所不同，因为计算属性存在“调度器”
// 还记得computed初始化阶段，new ReactiveEffect传递的第二个参数吗?
// 该参数将会被保存到ReactiveEffect的scheduler(调度器)中
// 所以此时的ReactiveEffect中，fn是computed的匿名函数，scheduler是computed初始化阶段new ReactiveEffect的第二个参数

export function triggerEffects(dep: Dep | ReactiveEffect[]) {
  const effects = isArray(dep) ? dep : [...dep]
  for (const effect of effects) {
    triggerEffect(effect, debuggerEventExtraInfo)
  }
}

function triggerEffect(effect: ReactiveEffect) {
	// 调度器的优先级大于run，所以此时会执行调度器逻辑
  if (effect.scheduler) {
    effect.scheduler()
  } else {
    effect.run()
  }
}

// 调度器代码
this.effect = new ReactiveEffect(getter, () => {
  // 还记得我们get之后将dirty改为false吗？
  // 此时computed的依赖发生变化，将_dirty改为true，表示下次重新计算
  if (!this._dirty) {
    this._dirty = true
    // 触发当前computed中收集了相关effect（依赖触发）
    triggerRefValue(this)
  }
})


export function triggerRefValue(ref: RefBase<any>, newVal?: any) {
  // 公共依赖触发逻辑
  triggerEffects(ref.dep)
}

// computed的dep中收集的effect触发，再次触发computed的get
get value() {
  // 依赖项发生变化的时候activeEffect不存在，所以此处收集不到任何依赖
  trackRefValue(this)
  // 刚才依赖项发生了变化，所以dirty为true，表示本次需要更新计算属性的结果
  if (this._dirty) {
    // 计算后dirty改为false 除非依赖项发生变化，否则将不会再重新计算。
    this._dirty = false
    // 重新计算 computed的结果
    this._value = this.effect.run()! 
  }
  return this._value
}
````

​	计算属性的触发逻辑还是非常复杂的，首先**proxy**的set，触发**computed**的**scheduler（调度器）**，**scheduler**通过**computed**的**dep**找到相关**effect**，**effect的fn**执行又会触发**computed**的**get**，**并与首次完成computed的计算，同时缓存最新的computed的结果**，进而再完成effect的全部逻辑。

<img src="https://www.vkcyan.top/Fu-QElucOgKlCPvYBJuMIQst0Fuo.png" style="zoom:33%;" />



## 代码执行流程

### 依赖收集阶段

1. **computed**初始化阶段，通过**ReactiveEffect**进行初始化，并且生成**scheduler（调度器）**
2. **effect**初始化，触发**computed**的**get**，将当前**activeEffect（effect）**收集到**computed**的**dep**中**（computed将effect收集）**
3. 执行**computed**自身逻辑，刷新全局**activeEffect**
4. 进而触发**proxy**的**get**事件触发，将当前**activeEffect（computed）**收集到**WeakMap**中**（proxy将computed收集）**
5. **proxy**的返回值返回**computed**，完成**computed**的计算逻辑
6. 获取到**computed**结果，完成**effect**



### 依赖触发阶段

1. 触发**proxy**的**set**，**set**行为中触发依赖，触发之前保存的**computed**的**调度器scheduler**（proxy找到computed）
2. **调度器scheduler**触发，**dirty**改为**true**，同时触发**computed**中保存的依赖，其中都是相关**effec**的**fn**。（computed找到effect）
3. **effect**触发，**fn**执行，触发**computed**的**get**行为
4. **dirty**为**true**，首次进行计算属性的重新计算（除非依赖项改变，否则下次不会重新计算），返回最新的**computed**结果，
5. **effect**执行完成



## 回答一些问题

#### computed如何实现高性能缓存的？

​	通过**调度器scheduler** + **脏值检查_dirty**，实现依赖项不变化，不进行重新计算，依赖项变化后仅执行一次的逻辑，进而实现高性能缓存。

#### 为什么访问computed需要.value

​	因为我们访问**computed**实际上是访问**ComputedRefImpl**这个**Class**的实例，他的内部通过**get value**返回被访问值，所以我们必须通过**.value**来访问

#### 简述computed的实现原理？

> vue的响应式api都可以从依赖收集 依赖触发2个角度出发阐述其原理实现

依赖收集阶段：computed通过首次get的完成相关effect的依赖收集，首次计算的时候proxy完成computed的依赖收集。

依赖触发阶段：computed的依赖项发生变化后，会通过proxy找到computed的调度器 scheduler，触发所有effect，effct中再出发computed的get，首次get将进行一次结果运算（后续不在运算，除非computed依赖项发生变化），effect触发完成



## 总结

​	到此为止，我们**computed**的核心源码就解读完毕了，虽然总体依旧可以从**依赖收集**和**依赖触发**两个角度去理解实现原理，但是新增加的**scheduler（调度器）**与**_dirty（脏值检查）**机制，让逻辑复杂了很多。

​	大家在理解computed源码的时候，一定要多走几遍流程，多捋几遍逻辑。

