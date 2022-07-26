---
title: Pinia源码分析【2】- createPinia
categories:
  - JavaScript-2022
tags:
  - Pinia
toc: true
date: 2022-07-07
---

## 专栏导航

[分析pinia源码之前必须知道的API](https://juejin.cn/post/7124279061035089927)

[Pinia源码分析【1】- 源码分析环境搭建](https://juejin.cn/post/7117131804229763079)

[Pinia源码分析【2】- createPinia](https://juejin.cn/post/7119788423501578277)

[pinia源码分析【3】- defineStore](https://juejin.cn/post/7121661056044236831)

[pinia源码分析【4】- Pinia Methods](https://juejin.cn/post/7123504805892325406)

## 前言

本系列文章参考源码`pinia V2.0.14`

上一篇文章我们主要介绍了如何搭建一个pinia源码阅读环境；本文主要介绍pinia在vue3初始化阶段相关逻辑，以及如何构建pinia对象。



## 正文

根据官方文档，我们使用`pinia`首先需要是将他注册到`vue`中

```js
const pinia = createPinia();
app.use(pinia);
```

`createPinia`的阶段究竟做了什么，他又是如何被注册到vue中呢？我们要从`createPinia`中寻找答案。



### 源码地址

我们通过`pinia\src\index.ts`找到

```js
export { createPinia } from './createPinia' // `pinia\src\createPinia.ts为源码文件`
```



### createPinia函数

​	在函数的最开头，我们就可以看到通过`effectScope`声明了一个`ref`，并赋值给了**state**，这里的`effectScope`是高级API，未来会单独介绍，有兴趣的同学可以看一下[官方文档](https://vuejs.org/api/reactivity-advanced.html#effectscope)，我们将其**简单理解为声明了一个ref并赋值给state**。

```ts
export function createPinia(): Pinia {
    const scope = effectScope(true);
    const state = scope.run<Ref<Record<string, StateTree>>>(() =>
       ref<Record<string, StateTree>>({})
    )!;
    // 简化理解
    // const state = ref({})
    
   	// ...
}

```



​	`pinia`通过`markRaw`进行包装，将其**标记为不会转化为响应式**，最终`pinia`对象被`createPinia`函数返回，执行`vue.use(pinia)`的时候便会执行`pinia`对象中的`install`函数。

```js
export function createPinia(): Pinia {
  // ...
  let _p: Pinia["_p"] = []; // 所有需要安装的插件
  let toBeInstalled: PiniaPlugin[] = []; // install之前保存的待安装插件

  // 使用markRaw标记pinia使其不会被响应式
  const pinia: Pinia = markRaw({
    // vue.use实际执行逻辑
    install(app: App) {
      setActivePinia(pinia); // 设置当前使用的 pinia
      if (!isVue2) { // 如果是vue2，全局注册已经在PiniaVuePlugin完成，所以这段逻辑将跳过
        pinia._a = app; // 保存app实例
        app.provide(piniaSymbol, pinia); // 通过provide传递pinia实例，提供给后续使用
        app.config.globalProperties.$pinia = pinia; // 设置全局属性 $pinia
        toBeInstalled.forEach((plugin) => _p.push(plugin)); // 处理未执行插件
        toBeInstalled = [];
      }
    },
    use(plugin) {
      if (!this._a && !isVue2) { // 如果use阶段为初始化完成则暂存toBeInstalled中
        toBeInstalled.push(plugin);
      } else {
        _p.push(plugin);
      }
      return this;
    },
    _p, // 所有pinia的插件
    _a: null, // app实例，在install的时候会被设置
    _e: scope, // pinia的作用域对象，每个store都是单独的scope
    _s: new Map<string, StoreGeneric>(),  // store缓存 key为pinia的id value为pinia的对外暴露数据
    state, // pinia所有state的合集 key为pinia的id value为store下的所有state（所有可访问变量）
  });
  return pinia;
}
```



### 返回值的含义以及作用

![image-20220713153012540](https://www.vkcyan.top/image-20220713153012540.png)



​	初始化的逻辑相对比较简单，只需要了解`effectScope` `markRaw`便能完全读懂，`install`阶段组成的`pinia`对象被`setActivePinia`保存了下来，而这个对象贯穿`pinia`整个生命周期，每个字段的作用在后面的源码解读中都会有所体现。



## 关于Vue2

​	通过`pinia`官网，我们可以了解到`pinia`支持`vue2`，不过`vue2`环境需要在使用`createPinia`之前，预先安装插件`PiniaVuePlugin`，通过`pinia`的入口文件了解到`PiniaVuePlugin`的源码入口为`pinia\src\vue2-plugin.ts`

​	`PiniaVuePlugin`是`vue2`插件比较主流的实现方式，**获取Vue实例，通过mixin实现数据共享**。如果了解过`vuex`的源码，相信对以下代码会十分熟悉。

```js
export const PiniaVuePlugin: Plugin = function (_Vue) {
    // Equivalent of
    // app.config.globalProperties.$pinia = pinia
    // pinia在vue2中的注册逻辑与vuex核心逻辑几乎一致，
    // 注入全局mixin的beforeCreate
    _Vue.mixin({
        beforeCreate() {
            const options = this.$options;
            // 在根节点通过vue.use中注册了pinia
            if (options.pinia) {
                const pinia = options.pinia as Pinia;
                // defineProperty版provided实现
                if (!(this as any)._provided实现) {
                    const provideCache = {};
                    Object.defineProperty(this, "_provided", {
                        get: () => provideCache,
                        set: (v) => Object.assign(provideCache, v),
                    });
                }
                (this as any)._provided[piniaSymbol as any] = pinia;

                // 首次注册变量不存在，进行存储
                if (!this.$pinia) {
                    this.$pinia = pinia;
                }

                // 保存Vue实例
                pinia._a = this as any;
                if (IS_CLIENT) {
                    setActivePinia(pinia);
                }
            } else if (!this.$pinia && options.parent && options.parent.$pinia) {
                // 所有子组件/页面都继承上一层的pinia
                this.$pinia = options.parent.$pinia;
            }
        },
        destroyed() {
            delete this._pStores;
        },
    });
};
```



## 关于devTool

在`createPinia`中存在这样一段代码

```js
if (__DEV__ && IS_CLIENT && !__TEST__) {
    pinia.use(devtoolsPlugin);
}
```

如果是开发环境，并且是浏览器环境，并且不是测试环境，就会向pinia注册`devtoolsPlugin`，也就是将`pinia`注册到浏览器插件**Vue.js devtools**中。

![image-20220713170645056](https://www.vkcyan.top/image-20220713170645056.png)





## 结语 

​	**createPinia**的源码解读就全部结束了，现在我们已经了解初始化的具体流程，以及生成的pinia对象中存在什么参数，这些参数在运行阶段都会发挥它应用的价值。

​	下一章我们将要解析**创建以及使用pinia**的相关源码，`defindStore`函数实现逻辑，在`defindStore`中我们将会了解到`install`阶段每个字段的实际用途，以及pinia的核心响应原理。

