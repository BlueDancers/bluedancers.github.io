---
title: Pinia源码分析【2】- createPinia
categories:
  - JavaScript-2022
tags:
  - Pinia
toc: true
date: 2022-07-07
---

## 前言

本系列文章参考源码`pinia V2.0.14`

源码分析记录：https://github.com/vkcyan/goto-pinia

本文主要介绍pinia中**createPinia**的实现



## 正文

使用过`vue3`+`pinia`都知道，我们使用`pinia`首先便需要是将他注册到`vue`中

```js
const pinia = createPinia();
app.use(pinia);
```

`createPinia`的阶段究竟做了什么，他又是如何被注册到vue中呢？我们要从源码中找到答案。



### 源文件地址

我们通过`pinia\src\index.ts`找到

```js
export { createPinia } from './createPinia'
```

`pinia\src\createPinia.ts`就是本次重点分析的源码文件。



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



​	`pinia`通过`markRaw`进行包装，将其**标记为不会转化为响应式**，并且最终`pinia`对象作为`createPinia`函数的返回值，在执行`vue.use(pinia)`的时候会执行`install`函数。

```js
export function createPinia(): Pinia {
  // ...
  let _p: Pinia["_p"] = []; // 所有需要安装的插件
  let toBeInstalled: PiniaPlugin[] = []; // install之前保存的待安装插件

  // 使用markRaw标记pinia使其不会被响应式
  const pinia: Pinia = markRaw({
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



​	到此位置，`createPinia`函数分析完毕，`install`时期组成的的`pinia`对象被保存了下来，而这个对象贯穿整个`pinia`的生命周期，每个字段的作用在后面的源码解读中都会有所体现。



## 关于Vue2

​	通过`pinia`官网，我们可以了解到`pinia`支持`vue2`，不过`vue2`环境需要在使用`createPinia`之前，预先安装插件`PiniaVuePlugin`，通过`pinia`的入口文件可以看到`PiniaVuePlugin`的源码入口为`pinia\src\vue2-plugin.ts`

​	`PiniaVuePlugin`函数是`vue2`插件比较主流的实现方式，**获取Vue实例，通过mixin实现数据共享**。如果了解过vuex的源码，相信对以下代码会十分熟悉。

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

如果是开发环境，并且是浏览器环境，并且不是测试环境，就会向pinia注册devtoolsPlugin，也就是将pinia注册到浏览器插件**Vue.js devtools**中。

![image-20220713170645056](https://www.vkcyan.top/image-20220713170645056.png)





## 结语 

​	到此为止，createPina的源码解读就全部结束了，现在我们已经了解初始化阶段准备好的每个参数的含义。

​	下一章是`pinia`源码最核心也是最复杂的`defindStore`具体实现，在`defindStore`我们将会了解到`install`阶段每个字段的实际使用，以及pinia的核心响应原理。

