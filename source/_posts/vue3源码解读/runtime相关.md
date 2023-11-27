---
title: runtime运行时
categories:
  - JavaScript-2023
tags:
  - Vue
  - 源码解读
toc: true
date: 2023-09-29
---

虚拟dom存在2个非常核心的概念

1. 挂载：mount
2. 更新：patch



h函数作用，生成vnode

render函数的作用，解析vnode并生成真实dom



# h函数

源码地址在：**packages/runtime-core/src/h.ts**

首先进入**h**函数

h函数存在3个参数

```
type：当前节点类型

propsOrChildren：props或者children

children：子节点
```

该函数的作用就是处理好以上三个函数的传参的多种情况。

然后再将处理好的**type，props，children**传入到**createVNode**函数中

createVNode最终会执行_createVNode函数，只不过开发环境会做一些额外的处理

对于初始化的组件来说，_createVNode的主要目的就是给当前组件增加组件类型标识shapeFlag

然后进入createBaseVNode，在该出构建了完成的VNode，并根据children的字段，重新计算shapeFlag，最终返回vnode



小结：**h - createVNode - _createVNode - createBaseVNode**

1. h函数 处理入参，使其标准化的进入到vnode创建流程
2. 根据type类型预先赋值shapeFlag，并增强处理class与style
3. 构建VNode基础数据
4. 根据children的类型，再次刷新ShapeFalg，完成对组件类型的全部标识。



h函数最终就是要生成可用于render函数渲染的vnode数据



# render函数

在h函数中处理过去的vnode，将会在render函数中被渲染为真实dom

render的核心代码在runtime-core中进行实现，核心代码与平台式无关的，runtime-dom中存放所有dom渲染相关的代码

在导出render之前，首先会将dom相关方法放入render函数，我们使用的render其实已经被处理过了

代码执行流程大概如下

render：传入vnode与挂载阶段

patch：判断阶段类型，进入对应渲染函数

processElement：文本组件进入这里，首先判断是更新还是新增

mountElement：新增文本组件进入该函数 （创建节点 填充数据 设置props 插入dom）

