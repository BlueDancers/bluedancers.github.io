---
title: pinia源码分析【3】- defineStore
categories:
  - JavaScript-2022
tags:
  - Pinia
toc: true
date: 2022-07-18
---

## 专栏导航

[分析pinia源码之前必须知道的API](https://juejin.cn/post/7124279061035089927)

[Pinia源码分析【1】- 源码分析环境搭建](https://juejin.cn/post/7117131804229763079)

[Pinia源码分析【2】- createPinia](https://juejin.cn/post/7119788423501578277)

[pinia源码分析【3】- defineStore](https://juejin.cn/post/7121661056044236831)

[pinia源码分析【4】- Pinia Methods](https://juejin.cn/post/7123504805892325406)

## 前言

本系列文章参考源码`pinia V2.0.14`

源码分析记录：https://github.com/vkcyan/goto-pinia

在上一节，我们完成了`createPinia`相关逻辑的源码解读，了解了`pinia`注册到`vue`的阶段具体做了哪些工作，以及`pinia`核心对象的生成逻辑，本文我们就要一起解读`pinia`中最重要的方法**defineStore**的实现原理

![image-20220718114312824](https://www.vkcyan.top/image-20220718114312824.png)

## 关于store的初始化

### 三种创建方法

![image-20220708105442886](https://www.vkcyan.top/image-20220708105442886.png)

源码中对`defineStore`的三种类型描述便解释了为何我们可以用以上三种方式创建。

在`defineStore`声明中，我们需要传入三种的参数。

- **id**：定义`store`的唯一id，单独传参或者通过`options.id`进行传参
- **options**：具体配置信息包含如果传参是对象，则可以传，`state`，`getters`，`action`，`id`，例如上图1 2 种声明方式；如果传参是`Function`，则自主声明变量方法，例如上图第三种声明方式
- **storeSetup**：仅限第三种`store`的声明方式，传入函数

### defineStore执行逻辑

```js
export function defineStore(
  // TODO: add proper types from above
  idOrOptions: any,
  setup?: any,
  setupOptions?: any
): StoreDefinition {
  let id: string;
  let options: // ....
  
  // 对三种store创建形式进行兼容。
  const isSetupStore = typeof setup === "function";
  if (typeof idOrOptions === "string") {
    id = idOrOptions;
    options = isSetupStore ? setupOptions : setup;
  } else {
    options = idOrOptions;
    id = idOrOptions.id;
  }
  function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
      //.....下面单独分析
  }
  useStore.$id = id;
  // 将useStore执行结果返回，在该store在使用之前被返回函数不会执行。
  // 所以defineStore早于在Vue种注册pinia也不会出现错误。
  return useStore;
}
```

​	通过对`defineStore`的源码大致分析可以得知，只有在`store`被执行的时候才会运行被返回的函数`useStore`，`useStore`才是核心store的创建逻辑，我们接下便要重点分析其实现原理。



## useStore逻辑分析

### useStore之前的逻辑执行顺序

我们在`App.vue`中使用我们创建的`store`

```html
<script setup lang="ts">
	const useCounter1 = useCounterStore1();
</script>
```

在`main` `createPinia` `defineStore` `useStore`初始化处增加日志

![image-20220708145541424](https://www.vkcyan.top/image-20220708145541424.png)

1.  `defineStore`初始化
2. main.ts -> createPinia -> vue.use -> install（注册逻辑）
3. 执行useStore（页面逻辑）

代码执行与我们想象的一致，`defineStore`是一个函数，会在引用阶段执行，并返回未执行函数`useStore`，之后便是一连串的初始化，最后是页面中使用`pinia`而运行的`useStore`。



### useStore准备工作

![image-20220718112122831](https://www.vkcyan.top/image-20220718112122831.png)

```js
// useStore接受两个参数，一个是pinia的实例，另一个与热更新相关。
function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
		// 首先会通过getCurrentInstance获取当前组件实例，并处理参数pinia，组件实例可以被正常获取，接下来通过inject(piniaSymbol)获取pinia实例（在install阶段保存）。
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
    // 获取最新pinia，并断言pinia一定存在（猜测这里主要为了断言，此时两个变量就是一个值）
    pinia = activePinia!;
	// ....
}
```



### 核心store创建

![image-20220718112358146](https://www.vkcyan.top/image-20220718112358146.png)

​	当我们第一次运行`store`的时候，才会进行相关逻辑的执行，通过单例模式创建，未来再次使用该`store`将会直接从`pinia._s`中获取已经被处理过的store并返回。

```js
function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
    // .....
    // 如果是第一次使用创建store逻辑，后面则跳过
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

`useStore`的大致逻辑比较简单，我们假设第一次使用，并且通过非Function进行传参，进入**createOptionsStore**函数。

### createOptionsStore

`defineStore`的第二个参数使用非`Function`进行声明将会走入该逻辑

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
    if (!initialState && (!__DEV__ || !hot)) {
      if (isVue2) {
        set(pinia.state.value, id, state ? state() : {});
      } else {
        // 将数据存储到state中，因为state时通过ref进行创建
        pinia.state.value[id] = state ? state() : {};
      }
    }

    // 避免在 pinia.state.value 中创建状态
    console.log(11, pinia.state.value[id]);
    console.log(22, toRefs(pinia.state.value[id]));

    const localState =
      __DEV__ && hot
        ? // 使用 ref() 来解开状态 TODO 中的 refs：检查这是否仍然是必要的
          toRefs(ref(state ? state() : {}).value)
        : toRefs(pinia.state.value[id]);
    // 经过toRefs的处理后，localState.xx.value 就等同于给state中的xx赋值
    let aa = assign(
      localState, // state => Refs(state)
      actions, //
      Object.keys(getters || {}).reduce((computedGetters, name) => {
        if (__DEV__ && name in localState) {
          // 如果getters名称与state中的名称相同，则抛出错误
          console.warn(
            `[🍍]: A getter cannot have the same name as another state property. Rename one of them. Found with "${name}" in store "${id}".`
          );
        }
        // markRow 防止对象被重复代理
        computedGetters[name] = markRaw(
          // 使用计算属性处理getters的距离逻辑，并且通过call处理this指向问题
          computed(() => {
            setActivePinia(pinia);
            // 它是在之前创建的
            const store = pinia._s.get(id)!;

            // allow cross using stores
            /* istanbul ignore next */
            if (isVue2 && !store._r) return;

            // @ts-expect-error
            // return getters![name].call(context, context)
            // TODO: avoid reading the getter while assigning with a global variable
            // 将store的this指向getters中实现getters中this的正常使用
            return getters![name].call(store, store);
          })
        );

        return computedGetters;
      }, {} as Record<string, ComputedRef>)
    );
    console.log("aa", aa);
    return aa;
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

​	`createOptionsStore`函数在获取`defineStore`声明的数据后，在其内部构建了**setup函数**，该函数将**option形式的state**与**getters**分别转化为**ref**与**computed**，这样就与**setup形式**声明的`store`保持一致。

​	**这一块代码非常核心，初步解释了为何state具备响应式，为何getters具备computed的效果**

​	最后不论是**option**方式创建还是**setup**的形式创建，最后都统一通过`createSetupStore`完成对`store`最后的处理



### createSetupStore

![image-20220718112449354](https://www.vkcyan.top/image-20220718112449354.png)

> 无论是何种`defineStore`创建方式，最终都会走向`createSetupStore`，在这里进行最终store的生成以及相关methods的实现。
>
> 注：这一块代码实在是复杂，关于$reset $patch等API，我们放下一个系列文章

​	经过`createOptionsStore`的处理，已经将**option**形式的字段全部转化为**setup**形式进行返回，现在无论何种创建方式，执行此处的setup函数，都会得到同一个结果。

![image-20220715170700824](https://www.vkcyan.top/image-20220715170700824.png)

以上三种创建方式，内部运行setup函数都会得到如下结果

![image-20220715170712733](https://www.vkcyan.top/image-20220715170712733.png)

接下来，我们就需要对其数据进行处理，获取到所有变量与方法，并对action通过wrapAction进行处理，便于实现后续的订阅发布模式 methods`$Action`

```js
const setupStore = pinia._e.run(() => {
    scope = effectScope();
    return scope.run(() => setup());
})!;

for (const key in setupStore) {
    const prop = setupStore[key];
    if ((isRef(prop) && !isComputed(prop)) || isReactive(prop)) {
    	// 如果当前props是ref并且不是计算属性与reative
        if (!isOptionsStore) {
			// option结构已经在createOptionsStore将其加入pinia
            if (isVue2) {
                set(pinia.state.value[$id], key, prop);
            } else {
                pinia.state.value[$id][key] = prop;
            }
        }
    } else if (typeof prop === "function") {
        // 如果当前函数是fun
        // wrapAction 会将当前prop也就是函数增加调用错误与正常的回调函数
        const actionValue = __DEV__ && hot ? prop : wrapAction(key, prop);
        if (isVue2) {
            set(setupStore, key, actionValue);
        } else {
            setupStore[key] = actionValue;
        }
        // 将其函数同步到optionsForPlugin中
        optionsForPlugin.actions[key] = prop;
    }
}

```

​	经过以上逻辑处理后，`setupStore`方式进行创建的`store`也会被添加到`pinia.state`中，而所有的`function`都会被`wrapAction`进行包装处理。

​	对state，action进行处理的同时，还需要对当前`store`可调用API进行处理，例如`$reset`，`$patch`

```js
const partialStore = {
    _p:pinia,
    $id,
    $Action,
    $patch,
    $reset,
    $subscribe,
    $dispose
}
const store: Store<Id, S, G, A> = reactive(
    assign(
        __DEV__ && IS_CLIENT
        ? // devtools custom properties
        {
            _customProperties: markRaw(new Set<string>()),
            _hmrPayload,
        }
        : {},
        partialStore
        // must be added later
        // setupStore
    )
) as unknown as Store<Id, S, G, A>;

// ...将变量 方法合并到store中
assign(toRaw(store), setupStore);
```

​	最终将相关`methods`与`store`内的数据进行合并，存储以当前store的id为key的`Map`中，`createSetupStore`的核心逻辑便全部结束了。



##  useStore后续逻辑

我们再回到`defineStore`的逻辑中，获取到`createSetupStore`最后放入`pinia._s`中的当前store被处理后的对象。

```js
// ....
// 从_s中获取当前store的effect数据
const store: StoreGeneric = pinia._s.get(id)!;
// StoreGeneric cannot be casted towards Store
return store as any;
```

​	最后将通过`createSetupStore`处理后的数据进行返回，我们便得到了使用当前`store`中变量与方法以及各种方法的能力。

![img](https://www.vkcyan.top/312434234234.png)





## 拓展：为什么访问defineStore创建的state不需要.value

​	通过以上源码分析可以得知，state的数据都会被处理为ref，那访问ref自然是需要.value，但是我们日常使用pinia似乎从来没有.value。

我们先看一个小例子

```js
let name = ref("张三");
let age = ref("24");

const info = reactive({ name, age });

console.log(info.name); // 张三
console.log(info.age); // 24
```

​	简单来说就是reactive中嵌套ref的时候，修改reactive内的值不需要.value

​	在官方文档（https://vuejs.org/api/reactivity-core.html#reactive）中，我们也能找到相关说明

> 注意：reactive嵌套ref的场景下，对象与数组格式存在差异，有兴趣可以了解一下

![image-20220716160727154](https://www.vkcyan.top/image-20220716160727154.png)

根据文档我们简单的翻阅了一下vuejs/core/.../baseHandlers.ts的源码

> 源码地址：https://github.com/vuejs/core/blob/main/packages/reactivity/src/baseHandlers.ts

**line 131 - 134 createGetter()**

![image-20220716161515875](https://www.vkcyan.top/image-20220716161515875.png)

**line 131 - 134 createSetter()**

![image-20220716161502977](https://www.vkcyan.top/image-20220716161502977.png)

可以发现，逻辑实现与文档描述相符。



最后再看一下我们的pinia源码中`createSetupStore`函数中`store`声明的那一段函数，这便解释了为什么在使用`pinia`修改值、读取值的时候都不需要进行.value了。

```js
const store: Store<Id, S, G, A> = reactive(
    assign(
	  // ...
      partialStore
    )
  ) as unknown as Store<Id, S, G, A>;

// .....
if (isVue2) {
    // ...
} else {
    assign(toRaw(store), setupStore); // 将defineStore的数据合并到reactive声明的store中
}
```



## 结语

​	**虽然代码量比较大，但是核心逻辑就是将state处理为ref，将getters处理为computed，将action进行二次封装，提供若干方法，最后组合对象存储到Pinia中。**

​	在这一章我们完成了最核心`store`创建流程的源码分析，但是通过`partialStore`增加的方法我们还没有一一了解。下一篇我们将会重点介绍`store`相关`Methods`的具体实现。



