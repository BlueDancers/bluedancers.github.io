```
title: 分析pinia源码之前必须知道的API
categories:
  - JavaScript-2022
tags:
  - JavaScript
toc: true
date: 2022-03-15
```



## 专栏导航

[分析pinia源码之前必须知道的API](https://juejin.cn/post/7124279061035089927)

[Pinia源码分析【1】- 源码分析环境搭建](https://juejin.cn/post/7117131804229763079)

[Pinia源码分析【2】- createPinia](https://juejin.cn/post/7119788423501578277)

[pinia源码分析【3】- defineStore](https://juejin.cn/post/7121661056044236831)

[pinia源码分析【4】- Pinia Methods](https://juejin.cn/post/7123504805892325406)



## 前言

​	在pinia源码中有一些业务场景下不常用的vue3 api，如果没有预先了解将会给源码解读带来较大困难，建议先搞清楚相关API，阅读代码将会事半功倍~

## effectScope

​	在`createPinia`中的遇到的第一行就是不认识的vue3 API，打开官网看了一下，最上方info中写道 **effect作用域是一个高阶API，专为库作者服务**。

​	他的作用是创建一片单独的`effect`空间，该空间内的`effect`将可以一起被处理，有点类似`docker`与`k8s`的关系，例如`ref computed watchEffect` 都是`docker`中的容器，而`effectScope`就是`k8s`，它可以统一管理`effect`集群。

类型：

```typescript
function effectScope(detached?: boolean): EffectScope

interface EffectScope {
  run<T>(fn: () => T): T | undefined // 如果这个域不活跃则为 undefined
  stop(): void
}
```

​	通过官网的类型可以看到，`effectScope`存在一个`boolean`类型的参数，但是在`vue3`文档中并未找到参数说明，而在[RFC](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0041-reactivity-effect-scope.md)中找到了更加详细的文档。接下来为`effectScope`的相关API说明。

### run

> 接受一个函数并返回该函数的返回值

```js
const scope = effectScope();
const counter = ref(1);
const setupStore = scope.run(() => {
  const doubled = computed(() => counter.value * 2);
  watch(doubled, () => console.log('doubled',doubled.value));
  watchEffect(() => console.log("count:", doubled.value));
  return {
    doubled,
  };
});

// 打印 'count 2' watchEffect触发
console.log(setupStore!.counter.value); // 打印'1',可以正常访问返回值
setupStore!.counter.value = 2; // 打印 'doubled 4' 'count: 4'  counter修改触发watch与watchEffect
```

### stop

> 递归结束所有effect，包括后代effectScope

```js
const setupStore = scope.run(() => {
  const counter = ref(1);
  const doubled = computed(() => counter.value * 2);
  nestedScope = effectScope(true /* detached */);
  nestedScope.run(() => {
    watch(counter, () => console.log("doubled", counter.value * 2));
    watchEffect(() => console.log("count:", counter.value*2));
  });
  return {
    counter,
    doubled,
  };
});

scope.stop();
setupStore!.counter.value = 2;
// 打印 doubled 4 count: 4 
// 因为nestedScope被指定为true，所以就算父级被销毁，nestedScope依旧存在反应。
// 如果想结束nestedScope，需要手动进行销毁nestedScope.stop()
```

### detached

表示是否在分离模式下创建，该参数默认为`false`；当为`true`的时候，父级被停止，子集也不会受影响。

```js
const scope = effectScope();
let nestedScope: any;

const setupStore = scope.run(() => {
  const counter = ref(1);
  const doubled = computed(() => counter.value * 2);
  nestedScope = effectScope(true /* detached */);
  nestedScope.run(() => {
    watch(counter, () => console.log("doubled", counter.value * 2));
    watchEffect(() => console.log("count:", counter.value * 2));
  });
  return {
    counter,
    doubled,
  };
});

scope.stop();
setupStore!.counter.value = 2;
// 打印 doubled 4 count: 4
// 因为nestedScope被指定为true，所以就算父级被销毁，nestedScope依旧存在反应。
// 如果想结束nestedScope，需要手动进行销毁nestedScope.stop()
nestedScope.stop();
setupStore!.counter.value = 3;
// 不会出现任何打印
```

## markRaw

标记一个对象，使其永远不会转换为 `proxy`。返回对象本身。

```js
const foo = markRaw({})
console.log(isReactive(reactive(foo))) // false

// 嵌套在其他响应式对象中时也可以使用
const bar = reactive({ foo })
console.log(isReactive(bar.foo)) // false
```

`markRaw`在`pinia`源码中非常常见，主要用于优化pinia的自身性能。

## toRaw

> toRaw可以获取一个响应式对象的原始属性

```js
const foo = {};
const reactiveFoo = reactive(foo);
console.log("toRaw", toRaw(reactiveFoo) === foo); // true

const foo1 = {};
const refFoo1 = ref(foo1);
console.log("toRaw", toRaw(refFoo1.value) === foo1); // true
```

在`pinia`源码中用于获取`reactive`的原始数据，并添加字段到其中

## toRefs

​	`toRefs`比较常见，简单来说：结果中的每个对象都指向原始属性；在实际开发中常用于reactive的解构。

​	在`pinia`的源码中，针对`store`中的`state`的处理用到了`toRefs`，不过它解构的是`state（ref类型）`对象，如果解构的是普通对象将不具备响应式。



## 结语

​	以上就是`pinia`源码中使用较多的`vue3 api`，还有些非常基础的例如`ref reactive`，就不做过多赘述了。