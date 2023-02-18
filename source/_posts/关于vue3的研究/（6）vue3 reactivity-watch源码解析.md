---
title: （6）vue3 reactivity-watch源码解析
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-02-14
---

## 前言

​	上一章我们说完了computed的实现逻辑，今天就让我们来看看他的好兄弟watch是如何实现的；

watch即为监听的意思：监听响应式数据，每当状态发生变化，就会触发回调函数。



## 正文

watch的源码并不在reactivity中，而是在runtime-core中

### watch初始化



### 监听值触发set事件

1. watch初始化，生成ReactiveEffect，并将watch的匿名函数放（cb）入ReactiveEffect的scheduler中
2. 触发一次watch的第一个参数，完成依赖收集（proxy与watch建立联系）
3. proxy发生变化，触发watch的ReactiveEffect的scheduler，也就是watch初始化事情传入的cb
4. cb开始执行，完成watch监听函数的触发。



watch的复杂并不在于核心逻辑的复杂性，而是其内部进行了大量兼容性判断，导致代码的复杂度极高。
