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

本文主要介绍我们使用pinia中的**createPinia**的实现

## 关于store的三种创建方法

![image-20220708105442886](https://www.vkcyan.top/image-20220708105442886.png)

这里便解释了为何我们可以用以上三种方式创建，总的来说在defineStore声明中，我们需要传入三种类型的参数

- id：定义store的唯一id，单独传参或者options.id进行传参
- options：具体配置信息，state，getters，action，无论那种创建方法可以可以传值，但是如果是第三种，则智能传入actions。
- storeSetup：直接传入setup逻辑









## 结语 

至此pinia源码分析环境搭建全部结束，接下来我们将开始逐步分析pinia的核心实现逻辑。

