---
title: pinia源码分析【4】- Pinia Methods
categories:
  - JavaScript-2022
tags:
  - Pinia
toc: true
date: 2022-07-18
---

## 前言

本系列文章参考源码`pinia V2.0.14`

源码分析记录：https://github.com/vkcyan/goto-pinia

上一章我们对store的最核心流程完成了分析，从而了解了一个store从定义到被使用的内部实现流程，但是store相关的方法，我们还未进行分析，本章我们就重点分析分析store自带的**Methods**



## $onAction

### 使用示例

订阅当前store所有action操作，每当action被执行的时候，便会触发该方法

```js
onMounted(() => {
  useCounter1.$onAction((option) => {
    let { after, onError, args, name, store } = option;;
  });

  setInterval(() => {
    useCounter1.counter++;
    // useCounter1.increment();
  }, 1000);
});
```



### 源码分析

#### 订阅

在$Action声明的地方，我们可以看到一段这样的函数

> 第一个参数传`null`，则不改变this指向，并且在后续的调用依旧是该this。

```js
const partialStore = {
  $onAction: addSubscription.bind(null, actionSubscriptions), // action事件注册函数
}
```

​	也就是说，当我们使用store.$Action的时候实际上触发的是addSubscription函数，并将我们$Action中的回调函数传入createSetupStore中的actionSubscriptions中，**也就是订阅了我们的callback**

​	运行store.$Action后得到了addSubscription方法的返回值removeSubscription方法，让我们可以执行其返回值，达到取消订阅的目的。

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

​	在useStore中对action进行处理的逻辑中，存在这样的一段代码，这段代码中的hot在正常使用的业务场景下都是undefined，所以会走后面的逻辑。

```js
const actionValue =  wrapAction(key, prop) // hot为undefined的情况下
```

![image-20220720184122095](https://www.vkcyan.top/image-20220720184122095.png)

​	所有的action在初始化阶段都会被wrapAction方法，也就代表我们执行action的时候，实际上执行的是wrapAction函数，那就让我们就看看，在wrapAction中究竟发生了什么

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

​	首先可以看到，之前在$Action中都的回调函数在此处发挥了作用，每当一个action触发的都会遍历一遍之前订阅的所有$Action的回调函数，而是执行action，action执行正常在执行after的callback，执行异常则触发onError的callback。

![image-20220721143852948](https://www.vkcyan.top/image-20220721143852948.png)



### 小结

本质上来说$Action就是一个订阅发布模式，

**$Action 订阅者**

**store.action 发布者**

**actionSubscriptions - 事件注册中心**

**triggerSubscriptions - 调度中心**

​	通过订阅者（$Action）把对发布者（action）的订阅注册到事件注册中心（actionSubscriptions）中，当发布者（action）触发时，通知调度中心（triggerSubscriptions），调度中心（triggerSubscriptions）触发事件注册中心中的所有订阅。



## $subscribe

### 使用示例

订阅当前store中的state的变化，state发生任意更改都会触发其回调函数，他还会返回一个用来删除的回调

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

当我们使用$subscribe并传入callback的时候，首先会将当前的callback加入注册中心中

```js
const removeSubscription = addSubscription(
    subscriptions, // 事件注册中心
    callback, // $subscribe传入的callback
    options.detached, // 页面卸载的时候是否取消监听
    () => stopWatcher() // 执行stopWatcher实际上执行的是scope.run返回的watch，而执行watch的返回函数，也就是停止当前watch
);
```

​	前三个参数经过对$Action的分析后已经比较熟悉，这里我们重点说明一下第四个参数

​	stopWatcher是当前store中的effectScope，我们将对当前state的watch放入scope中，以便于销毁store的时候统一处理。

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



![image-20220722170139137](https://www.vkcyan.top/image-20220722170139137.png)



### 小结

​	`$subscribe`主要依赖`vue3`的`watch`进行实现，在`subscriptions`中注册`callback`，但是注册的callback不通过triggerSubscriptions进行触发，仅仅作为保存，watch的触发函数中通过闭包触发$subscribe中的callback，达到store中任意值发生变化的时候都执行callback

​	在addSubscription的返回值removeSubscription中，不仅会再注册中心subscriptions删除订阅，同时也会执行() => stopWatcher()，停止watch监听。达到完全停止监听的目的。



## $dispose



## $patch



## $reset



## 





