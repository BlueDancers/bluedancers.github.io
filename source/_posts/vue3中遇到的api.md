```
title: 解读pinia源码必须知道的API
categories:
  - JavaScript-2022
tags:
  - JavaScript
toc: true
date: 2022-03-15
```

## 前言

​	在pinia源码中又很多业务场景下不常用的v3 api，因此专写一篇文章来进行记录。

## effectScope

​	在阅读pinia的createPinia中的遇到的第一行就是不认识的API，打开官网看了一下，最上方info中写道 effect作用域是一个高阶API，专为库作者服务。

​	他的作用是创建一篇单独的effect空间，该空间内的effect将可以被一起被处理，有点类似与docker与k8s的关系，例如ref computed watchEffect 都是docker中的容器，而effectScope就是k8s，它可以统一管理effect集群。

类型：

```typescript
function effectScope(detached?: boolean): EffectScope

interface EffectScope {
  run<T>(fn: () => T): T | undefined // 如果这个域不活跃则为 undefined
  stop(): void
}
```

​	通过官网的类型可以看到，effectScope函数存在一个boolean类型的参数，但是在文档中并未找到参数说明，但是再rfc找到了更加详细的文档。

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

表示是否在分离模式下创建，该参数默认为false，当为true的时候，则父级被stop，子集也不会收到影响。

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

​	

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

## toRefs





再更新数据的时候，可以设置使其不更新UI，达到优化性能的目的，其实也就是更新原始值，但是不触发effect	