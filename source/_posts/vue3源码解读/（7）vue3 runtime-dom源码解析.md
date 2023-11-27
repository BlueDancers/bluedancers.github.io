---
title: （7）vue3 runtime-dom源码解析
categories:
  - JavaScript-2023
tags:
  - Vue
  - 源码解读
toc: true
date: 2023-02-18
---



html叫做DOM节点数

vdom是正式dom的JavaScript数据结构的描述



在运行时runtime中，渲染器rerender会遍历整个虚拟dom树，并根据此结构构建正式dom树，这个过程我们称之为mount

当vnode发生变化的时候，，我们会对比旧的vnode与新的vnode，找出他们的区别，并应用于真实dom上，这个过程我们称之为patch。



