---
title: pinia源码分析【4】- Pinia Methods
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

源码分析仓库：https://github.com/vkcyan/goto-pinia

​	上一章我们对store的最核心流程完成了分析，从而了解了一个`store`从定义到被使用的内部实现流程，但是`store`相关的方法，我们还未进行分析，本章我们就重点分析分析`store`自带的**Methods**



## $onAction

### 使用示例

订阅当前`store`所有`action`操作，每当`action`被执行的时候，便会触发该方法

```js
onMounted(() => {
  useCounter1.$onAction((option) => {
    let { after, onError, args, name, store } = option;
  });

  setInterval(() => {
    useCounter1.counter++;
    // useCounter1.increment();
  }, 1000);
});
```



### 源码分析

#### 订阅

在`$Action`声明的地方，我们可以看到一段这样的函数

> 第一个参数传`null`，则不改变this指向，并且在后续的调用依旧是该this。

```js
const partialStore = {
  $onAction: addSubscription.bind(null, actionSubscriptions), // action事件注册函数
}
```

​	也就是说，当我们使用`store.$Action`的时候实际上触发的是`addSubscription`函数，并将我们`$Action`中的回调函数传入`createSetupStore`中的`actionSubscriptions`中，**也就是订阅了我们的callback**

​	运行`store.$Action`后得到了`addSubscription`方法的返回值`removeSubscription`方法，让我们可以执行其返回值，达到取消订阅的目的。

```js
export function addSubscription<T extends _Method>(
  subscriptions: T[], // createSetupStore中的actionSubscriptions
  callback: T, // 我们传入的callback
  detached?: boolean, // 如果为true，则该$Action在页面销毁之后依旧有效
  onCleanup: () => void = noop
) {
  // 使用$Action的时候就会触发本函数
  subscriptions.push(callback)

  const removeSubscription = () => {
    const idx = subscriptions.indexOf(callback)
    if (idx > -1) {
      subscriptions.splice(idx, 1)
      onCleanup()
    }
  }

  if (!detached && getCurrentInstance()) {
    // 如果detached参数不存在，则在当前页面卸载的时候，去除该订阅事件
    onUnmounted(removeSubscription)
  }
  return removeSubscription
}
```



#### 触发订阅

​	在`useStore`中对`action`进行处理的逻辑中，存在这样的一段代码，这段代码中的hot在正常使用的业务场景下都是undefined，所以会走后面的逻辑。

```js
const actionValue =  wrapAction(key, prop) // hot为undefined的情况下
```

![image-20220720184122095](https://www.vkcyan.top/image-20220720184122095.png)

​	所有的`action`在初始化阶段都会被`wrapAction`方法，也就代表我们执行`action`的时候，实际上执行的是`wrapAction`函数，那就让我们就看看，在`wrapAction`中究竟发生了什么

```js
/**
* 包装一个action来处理订阅。
*
* @param name - store的名称
* @param action - 需要被包装的action
* @returns a wrapped action to handle subscriptions
*/
function wrapAction(name: string, action: _Method) {
    return function (this: any) {
        setActivePinia(pinia);
        // 当前action的参数
        const args = Array.from(arguments);

        const afterCallbackList: Array<(resolvedReturn: any) => any> = [];
        const onErrorCallbackList: Array<(error: unknown) => unknown> = [];
        // 声明after方法
        function after(callback: _ArrayType<typeof afterCallbackList>) {
            // 将after的call放入list中
            afterCallbackList.push(callback);
        }
        // 声明error方法
        function onError(callback: _ArrayType<typeof onErrorCallbackList>) {
            onErrorCallbackList.push(callback);
        }

        // @ts-expect-error
        // 触发actionSubscriptions中订阅的store.$Action的全部回调函数,并将参数传入
        // 此时store.$Action的callback已经执行,但是after onError的回调函数尚未执行
        triggerSubscriptions(actionSubscriptions, {
            args,
            name,
            store,
            after,
            onError,
        });

        let ret: any; // ret为action的返回值
        try {
            ret = action.apply(this && this.$id === $id ? this : store, args);
            // handle sync errors
        } catch (error) {
            // 如果action执行出错,则直接执行错误回调,终止函数
            triggerSubscriptions(onErrorCallbackList, error);
            throw error;
        }
        // 如果ret是promise,则当前结果未知，会通过上方的try catch，但是会在action结尾增加then catch进行结果捕捉
        if (ret instanceof Promise) {
            return ret
                .then((value) => {
                triggerSubscriptions(afterCallbackList, value);
                return value;
            })
                .catch((error) => {
                triggerSubscriptions(onErrorCallbackList, error);
                return Promise.reject(error);
            });
        }

        // allow the afterCallback to override the return value
        // 如果try catch 通过，并且当前action不是Promise，则逻辑进行到此处，触发所有 触发真正的after函数，并将当前action的返回值传入其中，至此完成对action触发的监听。
        triggerSubscriptions(afterCallbackList, ret);
        return ret;
    };
}
```

​	之前在`$Action`中的回调函数在此处发挥了作用，每当一个`action`触发的都会遍历一遍之前订阅的所有`$Action`的回调函数，其内部执行`action`方法，`action`执行正常在触发`after`的`callback`，执行异常则触发`onError`的`callback`。





### 小结

![image-20220721143852948](https://www.vkcyan.top/image-20220721143852948.png)

本质上来说$Action就是一个订阅发布模式。

**$Action 订阅者**

**store.action 发布者**

**actionSubscriptions - 事件注册中心**

**triggerSubscriptions - 调度中心**

​	通过**订阅者（$Action）**把对**发布者（action）**的订阅注册到**事件注册中心（actionSubscriptions）**中，当**发布者（action）**触发时，通知**调度中心（triggerSubscriptions）**，**调度中心（triggerSubscriptions）**触发事件注册中心中的所有订阅。



## $subscribe

### 使用示例

订阅当前`store`中的`state`的变化，`state`发生任意更改都会触发其回调函数，他还会返回一个用来删除的回调

```js
let abc = useCounter1.$subscribe(
    (option, state) => {
        // 通过store.num = xxxx修改，type为direct
        // 通过store.$patch({ num: 'xxx' })修改，type为directpatchObject
        // 通过store.$patch((state) => num.name='xxx')修改，type为patchFunction

        // storeId为当前store的id
        // events 当前改动说明
        let { events, storeId, type } = option;
        console.log(events, storeId, type, state);
    },
    { detached: false }
);
```



### 源码分析

当我们使用`$subscribe`并传入`callback`的时候，首先会将当前的`callback`加入注册中心中

```js
const removeSubscription = addSubscription(
    subscriptions, // 事件注册中心
    callback, // $subscribe传入的callback
    options.detached, // 页面卸载的时候是否取消监听
    () => stopWatcher() // 执行stopWatcher实际上执行的是scope.run返回的watch，而执行watch的返回函数，也就是停止当前watch
);
```

​	前三个参数经过对`$Action`的分析后已经比较熟悉，这里我们重点说明一下第四个参数

​	`stopWatcher`是当前`store`中的`effectScope`，我们将对当前`state`的`watch`放入`scope`中，以便于销毁`store`的时候统一处理。

```typescript
const stopWatcher = scope.run(() =>
    watch(
        () => pinia.state.value[$id] as UnwrapRef<S>, // 监听state的变化
        (state) => {
            // 在不使用$patch的情况下，则两个参数都为true，callback一定会执行
            if (options.flush === "sync" ? isSyncListening : isListening) {
                callback(
                    {
                        storeId: $id, // 
                        type: MutationType.direct,
                        events: debuggerEvents as DebuggerEvent,
                    },
                    state
                );
            }
        },
        assign({}, $subscribeOptions, options)
    )  
)
```



### 小结

![image-20220722170139137](https://www.vkcyan.top/image-20220722170139137.png)

​	`$subscribe`主要依赖`vue3`的`watch`进行实现，在`subscriptions`中注册`callback`，但是注册的`callback`不通过`triggerSubscriptions`进行触发，仅仅作为保存，`watch`的触发函数中通过闭包触发`$subscribe`中的`callback`，达到`store`中任意值发生变化的时候都执行`callback`的目的

​	在`addSubscription`的返回值`removeSubscription`中，不仅会在`subscriptions`(注册中心)删除订阅，同时也会执行`() => stopWatcher()`，停止`watch`监听。达到完全停止监听的目的。



## $patch

### 使用示例

直接更新当前`state`，可以通过传入**对象**与**callback**两种方式进行`state`更新，允许传递嵌套值

```js
// 对象
useCounter1.$patch({ counter: 2 });
// function
useCounter1.$patch((state) => {
    state.counter = 2;
});
```



### 源码分析

​	`$patch`的主体逻辑不算很复杂，针对不同的参数类型进行分别处理，其中`partialStateOrMutator`是传入的方法，我们将当前`store`传入其中，通过其`callback`直接完成`state`的修改，而传入类型为`object`的时候，则依赖`mergeReactiveObjects`进行处理。

```typescript
function $patch(stateMutation: (state: UnwrapRef<S>) => void): void; // Fun传参
function $patch(partialState: _DeepPartial<UnwrapRef<S>>): void; // 对象传参
function $patch(
partialStateOrMutator:
 | _DeepPartial<UnwrapRef<S>>
 | ((state: UnwrapRef<S>) => void)
): void {
    let subscriptionMutation: SubscriptionCallbackMutation<S>;
    isListening = isSyncListening = false;
    // reset the debugger events since patches are sync
    /* istanbul ignore else */
    if (__DEV__) {
        debuggerEvents = [];
    }
    // 如果参数是方法，走以下处理逻辑
    if (typeof partialStateOrMutator === "function") {
        partialStateOrMutator(pinia.state.value[$id] as UnwrapRef<S>);
        subscriptionMutation = {
            type: MutationType.patchFunction,
            storeId: $id,
            events: debuggerEvents as DebuggerEvent[],
        };
    } else {
    // 如果参数是对象，走以下处理逻辑
        mergeReactiveObjects(pinia.state.value[$id], partialStateOrMutator);
        subscriptionMutation = {
            type: MutationType.patchObject,
            payload: partialStateOrMutator,
            storeId: $id,
            events: debuggerEvents as DebuggerEvent[],
        };
    }
    const myListenerId = (activeListener = Symbol());
    nextTick().then(() => {
        if (activeListener === myListenerId) {
            isListening = true;
        }
    });
    isSyncListening = true;
    // 在上方逻辑中，我们将isListening isSyncListening 重置为false，不会触发$subscribe中的callback，所以需要手动进行订阅发布
    triggerSubscriptions(
        subscriptions,
        subscriptionMutation,
        pinia.state.value[$id] as UnwrapRef<S>
    );
}


```

```typescript
// $patch传入参数为Object的处理逻辑
function mergeReactiveObjects<T extends StateTree>(
  target: T,
  patchToApply: _DeepPartial<T>
): T {
  // no need to go through symbols because they cannot be serialized anyway
  for (const key in patchToApply) {
    if (!patchToApply.hasOwnProperty(key)) continue;
    const subPatch = patchToApply[key];
    const targetValue = target[key];
    if (
      isPlainObject(targetValue) &&
      isPlainObject(subPatch) &&
      target.hasOwnProperty(key) &&
      !isRef(subPatch) &&
      !isReactive(subPatch)
    ) {
      // 如果被修改的值 修改前修改后都是object类型并且不是Function类型、并且不是ref 不是isReactive，则递归mergeReactiveObjects达到修改嵌套object的目的
      target[key] = mergeReactiveObjects(targetValue, subPatch);
    } else {
      // @ts-expect-error: subPatch is a valid value
      // 如果是简单类型 则直接进行state的修改，这里的target为pinia.state.value[$id]
      // 按我们的示例来实际分析：pinia.state.value[$id].counter = 2
      target[key] = subPatch;
    }
  }
  return target;
}
```

​	完成对`mergeReactiveObjects`的分析后，`$patch`的核心逻辑就全部结束了，但是还有一点我们没完成，就是通过`$patch`修改的`state`，`$subscribe`是否可以监听到。



### $patch触发$subscribe

​	在`$patch`执行的中，我们会修改当前`store`中的`state`，`$subscribe`中的`watch`在`flush='sync'`的情况下可以立刻监听到，但是也无法执行`callback`，因为`$patch`函数最开始的地方将`isListening，isSyncListening`置为`false`

​	在对值完成修改后，我们将`isSyncListening`置为true，并且手动订阅`$subscribe`的`callback`，达到通过`$patch`修改`state`也能被`$subscribe`监听到的目的。



### 小结

​	`$patch`的源码相对来说比较简单，但是关于触发`$subscribe`的部分代码逻辑比较复杂，尤其是当`$subscribe` `option`设置中的`flush`为sync的时候，修改`state`立刻就会触发`$subscribe`的`watch`，虽然最终呈现出来的结果是一致的，但是内部对不同情况的兼容有大学问。

![image-20220723162542035](https://www.vkcyan.top/image-20220723162542035.png)



## $dispose

​	调用该方法后将会注销当前`store`

`	scope`中存储当前`store`中的相关反应，当前`state`的`watch`，`ref`，等等`effect`都通过`scope.run`创建，就是为了方便统一处理，这里调用`scope.stop()`所有的effect全部被注销了。

```js
  function $dispose() {
    scope.stop();
    subscriptions = []; // $subscribe注册中心
    actionSubscriptions = []; // $Action的注册中心
    pinia._s.delete($id); // 删除effectMap结构
  }
```



## $reset

调用该方法可以将当前`state`重置为初始化时候的状态

但是又有点需要注意，如果`defineStore`通过`setup类型`声明，则无法调用该函数

````js
const $reset = __DEV__
	? () => {
        throw new Error(
            `🍍: Store "${$id}" is built using the setup syntax and does not implement $reset().`
        );
	}
	: noop; // noop为空函数
````

如果通过`option类型`进行声明，则会**重写$reset**方法

```js
store.$reset = function $reset() {
    // state通过闭包机制获得最初state定义的状态
    const newState = state ? state() : {};
    // 通过$patch完成对state中数据的更新
    this.$patch(($state) => {
        assign($state, newState);
    });
};
```



## 总结

​	至此，我们就完成了对pinia所有方法的源码解读，而pinia源码解读系列文章也将告一段落，我们从pinia的初始化到使用action修改state，最后完成对自带metnods的解读，已经完成理解了其核心实现，最后我们将会实现一个简易版的pinia，一来降低源码阅读门槛，而来也是检验是否真的读懂了其核心实现。

 
