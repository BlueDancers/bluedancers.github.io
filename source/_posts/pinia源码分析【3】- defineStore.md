---
title: pinia源码分析【3】- defineStore
categories:
  - JavaScript-2022
tags:
  - Pinia
toc: true
date: 2022-07-08
---

## 前言

本系列文章参考源码`pinia V2.0.14`

源码分析记录：https://github.com/vkcyan/goto-pinia

本文主要介绍我们pinia中**defineStore**的实现原理

## 关于store的初始化

### 三种创建方法

![image-20220708105442886](https://www.vkcyan.top/image-20220708105442886.png)

这里便解释了为何我们可以用以上三种方式创建，总的来说在defineStore声明中，我们需要传入三种类型的参数

- id：定义store的唯一id，单独传参或者options.id进行传参
- options：具体配置信息，state，getters，action，无论那种创建方法可以可以传值，但是如果是第三种，则智能传入actions。
- storeSetup：直接传入setup逻辑

### defineStore执行逻辑

```js
export function defineStore(
  // TODO: add proper types from above
  idOrOptions: any,
  setup?: any,
  setupOptions?: any
): StoreDefinition {
  let id: string;
  let options:
    | DefineStoreOptions<
        string,
        StateTree,
        _GettersTree<StateTree>,
        _ActionsTree
      >
    | DefineSetupStoreOptions<
        string,
        StateTree,
        _GettersTree<StateTree>,
        _ActionsTree
      >;
  // 对三种store创建形式进行兼容。
  const isSetupStore = typeof setup === "function";
  if (typeof idOrOptions === "string") {
    id = idOrOptions;
    options = isSetupStore ? setupOptions : setup;
  } else {
    options = idOrOptions;
    id = idOrOptions.id;
  }
  function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {}
  useStore.$id = id;
  // 将函数useStore返回，再该store在引用之前被返回函数不会被执行，所以defineStore早于pinia在Vue中注册之前也不会出现错误，因为store内逻辑未被执行
  return useStore;
}
```

​	通过对defineStore的源码进行分析可以得知，只有在store被执行的时候才会运行被返回的函数useStore。这才是核心逻辑，我们接下来便分析其实现原理。



## useStore逻辑分析

### useStore之前的逻辑执行顺序

我们在App.vue中使用我们创建的store

```js
<script setup lang="ts">
	const useCounter1 = useCounterStore1();
</script>
```

在main createPinia defineStore useStore初始化处增加log

![image-20220708145541424](https://www.vkcyan.top/image-20220708145541424.png)

打印结果符合我们的想法，首先执行defineStore初始化，然后开始执行main.ts，紧接着createPina，vue.use pinia。最后执行页面逻辑，开始执行真正的store逻辑，也就是我们将要分析的userStore



### useStore准备工作

```js
// useStore接受两个参数，一个是pinia的实例，另一个与热更新相关。
function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
		// 首先会通过getCurrentInstance获取当前组件实例，并处理参数pinia，在非测试环境，并且组件实例被正常获取则会通过inject(piniaSymbol)获取pinia实例（在install阶段保存的）。
    const currentInstance = getCurrentInstance();
    pinia =
        (__TEST__ && activePinia && activePinia._testing ? null : pinia) ||
        (currentInstance && inject(piniaSymbol));
    // 设置当前活跃的pinia，如果存在多个pinia实例，方便后续逻辑获取当前pinia实例
    if (pinia) setActivePinia(pinia);
    // 在dev环境并且全局变量activePinia获取不到当前pinia实例，则说明未全局注册，抛出错误
    if (__DEV__ && !activePinia) {
        throw new Error(
            `[🍍]: getActivePinia was called with no active Pinia. Did you forget to install pinia?\n` +
            `\tconst pinia = createPinia()\n` +
            `\tapp.use(pinia)\n` +
            `This will fail in production.`
        );
    }
    // 获取最新pinia，并断言pinia一定存在（猜测这里主要为了断言，此时两个变量本身就是一个值）
    pinia = activePinia!;
	// ....
}
```



### 核心store创建

```js
function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
    // .....
    // 如果是第一次使用store则创建effect，后面则跳过
    if (!pinia._s.has(id)) {
        // 如果defineStore的时候第二个参数是函数则为true，否则为false
        if (isSetupStore) {
            createSetupStore(id, setup, options, pinia);
        } else {
            createOptionsStore(id, options as any, pinia);
        }
    }
    // 从_s中获取当前id对应的store信息
    const store: StoreGeneric = pinia._s.get(id)!;
	// 这里返回的值实际上就是我们实际获取到值
    return store as any;
}
```



### createOptionsStore

defineStore的第二个参数使用非Function进行声明将会走入该逻辑

```js
function createOptionsStore<
  Id extends string,
  S extends StateTree,
  G extends _GettersTree<S>,
  A extends _ActionsTree
>(
  id: Id, // storeid
  options: DefineStoreOptions<Id, S, G, A>, // state action getters
  pinia: Pinia, // 当前store实例
  hot?: boolean
): Store<Id, S, G, A> {
  const { state, actions, getters } = options;
  // 获取state中是否已经存在该store实例
  const initialState: StateTree | undefined = pinia.state.value[id];
  console.log("initialState", initialState);
  let store: Store<Id, S, G, A>;
  
  function setup() {
      // .....
  }
  // 使用createSetupStore创建store
  store = createSetupStore(id, setup, options, pinia, hot, true);
  
  // 重写$store方法
  store.$reset = function $reset() {
    const newState = state ? state() : {};
    // 我们使用补丁将所有更改分组到一个订阅中
    this.$patch(($state) => {
      assign($state, newState);
    });
  };
  return store as any;
}
```

createOptionsStore函数在获取defineStore声明的数据后，将其转化为函数型，具体转化逻辑交给了setup进行处理，内部最终还是通过createSetupStore进行store的获取，将要分析createSetupStore的实现逻辑。

### createSetupStore

> 除了defineStore第二个参数传入Function以外，createOptionsStore内部最终也会走向该函数完成最终store的创建；而createSetupStore也是最核心代码。
>
> 这一块代码实在是复杂，我们向梳理总流程，再分别看细节实现。

```
```



​	




## 结语 

至此pinia源码分析环境搭建全部结束，接下来我们将开始逐步分析pinia的核心实现逻辑。

