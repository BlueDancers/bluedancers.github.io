---
title: pinia源码分析【5】- 150行代码实现mini版pinia
categories:
  - JavaScript-2022
tags:
  - Pinia
toc: true
date: 2022-07-29
---



## 源码解析系列文章

[分析pinia源码之前必须知道的API](https://juejin.cn/post/7124279061035089927)

[Pinia源码分析【1】- 源码分析环境搭建](https://juejin.cn/post/7117131804229763079)

[Pinia源码分析【2】- createPinia](https://juejin.cn/post/7119788423501578277)

[pinia源码分析【3】- defineStore](https://juejin.cn/post/7121661056044236831)

[pinia源码分析【4】- Pinia Methods](https://juejin.cn/post/7123504805892325406)

## 前言

> 别人还在学习使用pinia，看过文章的你直接了解核心原理，无论是实际使用，还是面试都将更上一层楼~

​	前段时间完成了对`pinia`核心源码的解读，因为源码存在难度，也间接到了分析文章具有较高的阅读门槛，为了解决这一问题，可以让更多人参与到pinia的源码阅读中，所以今天给大家带来一个mini版pinia的核心实现，核心代码压缩到100行左右，极大了降低了源码阅读难度。

​	mini版pinia实现了**state，getters，action，$patch，$reset，$dispose**；居家旅行**面试**常备~

​	同时为了降低阅读门槛，方便TypeScript不熟练的同学，本版本全部使用any，话不多说我们直接开始！

​	mini版pinia开源地址：https://github.com/vkcyan/mini-pinia



## mini版逻辑流程图

![image-20220729092737403](https://www.vkcyan.top/image-20220729092737403.png)

## 简单版实现

我们在代码结构上尽量与正式源码保持一致，仅仅做一些逻辑上的简化与压缩，保证核心实现的质量。

### 注册到vue

> 这里主要参照官方实现，如果不清楚effectScope，请看[分析pinia源码之前必须知道的API](https://juejin.cn/post/7124279061035089927)，如果想深入了解createPinia，请看[Pinia源码分析【2】- createPinia](https://juejin.cn/post/7119788423501578277)

```typescript
/**
 * 创建Pinia
 */
export function createPinia() {
  // 创建响应空间
  const scope = effectScope(true);
  const state = scope.run<Ref<Record<string, StateTree>>>(() =>
    ref<Record<string, StateTree>>({})
  )!;
  // markRaw使其不具备响应式
  const pinia = markRaw({
    install(app: App) {
      // 注入pinia
      app.provide(piniaSymbol, pinia);
    },
    use() {},
    _s: new Map<string, StoreGeneric>(), // 保存处理后的store数据全部数据
    state, // 保存可访问state
    _e: scope, // 相应空间
  });
  return pinia;
}

```



### 实现defineStore

实现一个基础功能的pinia，简单来说，我们只需要做最核心的两件事

1. **将state转为ref，使其具有响应式**
2. **将getters处理为computed**
3. 如果需要实现$Action还需要对action中所有事件进行拦截处理（mini版不实现$Action）

### defineStore

> defineStore中的useStore主要做一些初始化判断，如果是store第一次被使用，则需要初始化，进入createOptionsStore，非第一次直接获取_s中已被处理好的缓存。

```js
/**
 * 创建store（仅支持单对象创建方式）
 * @param options
 * @returns
 */
export function defineStore(options: {
  id: string;
  state: any;
  getters: any;
  actions: any;
}) {
  let { id } = options;
  // 实际运行函数
  function useStore() {
    const currentInstance = getCurrentInstance(); // 获取实例
    let pinia: any;
    if (currentInstance) {
      pinia = inject(piniaSymbol); // 获取install阶段的pinia
    }
    if (!pinia) {
      throw new Error("super-mini-pinia在mian中注册了吗?");
    }
    if (!pinia._s.has(id)) {
      // 第一次会不存在，单例模式
      createOptionsStore(id, options, pinia);
    }
    const store = pinia._s.get(id); // 获取当前store的全部数据
    return store;
  }
  useStore.$id = id;
  return useStore;
}
```

### createOptionsStore

> **使用ref处理state，使用computed处理getters**，但是此处尚未运行，将setup函数作为参数传值到createSetupStore。

```js
/**
 * 处理state getters
 * @param id
 * @param options
 * @param pinia
 */
function createOptionsStore(id: string, options: any, pinia: any) {
  const { state, actions, getters } = options;
  function setup() {
    pinia.state.value[id] = state ? state() : {}; // pinia.state是Ref
    const localState = toRefs(pinia.state.value[id]);
    return Object.assign(
      localState, // 被ref处理后的state
      actions, // store的action
      Object.keys(getters || {}).reduce((computedGetters, name) => {
        computedGetters[name] = markRaw(
          computed(() => {
            const store = pinia._s.get(id)!;
            return getters![name].call(store, store);
          })
        );
        return computedGetters;
      }, {} as Record<string, ComputedRef>) // 将getters处理为computed
    );
  }
  let store = createSetupStore(id, setup, pinia);
  return store;
}
```

### createSetupStore

> ​	声明当前store的方法，并且运行上一个函数组建的setup函数，其中包含state，getters，我们将其响应式存储到pinia._e中，便于后面对数据变化进行监听，以及统一管理。
>
> ​	最后将setup返回的对象与存放方法的partialStore对象进行assign，完成store的全部初始化逻辑，并将其加入_s，下次使用该store则直接取值，最后返回当前store。全部逻辑结束。

```js
/**
 * 处理action以及配套API将其加入store
 * @param $id
 * @param setup
 * @param pinia
 */
function createSetupStore($id: string, setup: any, pinia: any) {
  // 所有pinia的methods
  let partialStore = {
    _p: pinia,
    $id,
    $reset: () => console.log("reset"), // 该版本不实现
    $patch: () => console.log("patch"), // 该版本不实现
    $onAction: () => console.log("onAction"), // 该版本不实现
    $subscribe: () => console.log("subscribe"), // 该版本不实现
    $dispose: () => console.log("dispose"), // 该版本不实现
  };

  // 将effect数据存放如pinia._e、setupStore
  let scope!: EffectScope;
  const setupStore = pinia._e.run(() => {
    scope = effectScope();
    return scope.run(() => setup());
  });

  // 合并methods与store
  const store: any = reactive(
    Object.assign(toRaw({}), partialStore, setupStore)
  );
  // 将其加入pinia
  pinia._s.set($id, store);
  
  return store;
}
```

​	我们nimi版pinia的核心实现便完成了，真实的pinia源码中存在许多边际判断，为了方便阅读作者仅仅保留核心逻辑，剔除ts，简化分叉流程，极大的降低了了解pinia核心实现的门槛。



### 增加一些方法

> $Action $subscribe因为涉及到**订阅发布模块**，所以代码量比较大，mini版就忽略了，对其原理有兴趣的请看[pinia源码分析【4】- Pinia Methods](https://juejin.cn/post/7123504805892325406)



#### $patch

> 将状态补丁应用于当前状态

```js
function $patch(partialStateOrMutator: any) {
    // mini版实现仅支持传入function
    if (typeof partialStateOrMutator === "function") {
        partialStateOrMutator(pinia.state.value[$id]);
    }
}
```



#### $reset

> 初始化state

```js
store.$reset = function $reset() {
    const newState = state ? state() : {}; // 通过闭包获取最初定义的state
    this.$patch(($state: any) => { // 借用$patch完成state数据的替换
        Object.assign($state, newState);
    });
};
```



#### $dispose

> 停止store的所有effect，并且删除其注册信息

```js
function $dispose() {
    scope.stop(); // effect作用于停止
    pinia._s.delete($id); // 删除effectMap结构
}
```



### 测试使用

我们首先将实现的函数导出出去

`src\super-mini-pinia\index.ts`

```
import { createPinia } from "./createPinia";
import { defineStore } from "./store";

export { createPinia as myCreatePinia, defineStore };
```

在项目中的main.ts进行注册

```js
import { createApp } from "vue";
import { myCreatePinia } from "./super-mini-pinia/index";
import App from "./App-super-mini.vue";
const app = createApp(App);
app.use(myCreatePinia());
app.mount("#app");
```

在页面增加一些测试代码

```vue
<template>
  <div>
    <div>state.num:{{ useStore.num }}</div>
    <div>getters.dnum:{{ useStore.dnum }}</div>
    <button @click="addNum">增加</button>
  </div>
</template>

<script setup lang="ts">
import { watchEffect } from "vue";
import { useCounterStore } from "./super-mini-store/counter";

const useStore = useCounterStore();

watchEffect(() => {
  console.log(useStore.num);
});

function addNum() {
  useStore.addNum();
}
</script>
```

预期效果

1. action正常触发
2. num与dnum随着action的触发更新UI



## mini版pinia测试

![59mhu-59cji](https://www.vkcyan.top/59mhu-59cji.gif)



​	 到此为止，我们便完成了mini版pinia的开发，代码虽少，但是核心逻辑五脏俱全，看懂了mini版pinia便是了解了pinia最核心的实现逻辑。

​	 我已将mini版pinia的开源到[github](https://github.com/vkcyan/mini-pinia)，如果你对pinia核心实现有兴趣，欢迎fock、clone，有任何问题请评论区留言。

## 结语

​	到此为止pinia源码解读系列便全部结束了，总体来说难度不算太大，作者前前后后花费了半个月时间，从零开始搭建环境，逐步深入阅读，读懂pinia源码的也让作者vue3 reactivity核心响应机制，闭包，订阅发布有了更深入的理解，值得阅读；也欢迎大家一起阅读源码，交流讨论~
