---
title: vue3相对于vue2做了什么？
categories:
  - JavaScript-2022
tags:
  - Vue
  - 源码解读
toc: true
date: 2022-09-18
---



vue3.0更加注重模块上的拆分，在2.0版本中所有vue相关的逻辑都相互耦合在一起，就算仅仅使用vue的一小部分，也需要引入完成的vue，造成了空间的浪费，而vue3则在模块层面上进行拆分，通过tree-shaking实现按需导入，减少用户打包体积，同时每个项目单独管理，单独发布，更加具备稳定性

​	模块拆分成npm包，独立使用，独立发布



虽然底层出现大量改动，但是顶层设计理念没有发生改变，依旧是声明式架构。

 

###  Monorepo：

- 一个仓库下可以维护多个模块
- 方便版本管理，依赖管理，模块间引用。

![](https://www.vkcyan.top/image-20220914110426629.png)



## 搭建vue3仓库

#### 通过pnpm初始化项目

```bash
pnpm init
```

#### 创建Monorepo仓库环境

新建文件夹`packages`

新建文件`pnpm-workspace.yaml`

```
packages:
  - 'packages/*' // 含义为怕package目录下的每个文件都是一个单独的仓库
```



安装公共依赖

项目根目录执行

```bash
pnpm install vue -w // w 为workspace-root的缩写，代表该包为全局依赖
```



幽灵依赖

​	vue依赖了abc包，我们下载vue的时候abc包就被下载到项目，我们可以直接使用abc，但是vue可能在某一个版本就不再使用abc，这就会造成依赖丢失，这些依赖就被成为幽灵依赖



## 生成相关配置信息

创建packages内部的package环境

通过

```
pnpm tsc --init
```

生成ts默认配置文件

并且增加配置

```js
"baseUrl": ".",
"paths": {
    "@vue/*": ["packages/*/src"] // 引入库关系映射
}
```



## 实现构建流程

1. 编写每个组件的package.json
2. 编写公共打包文件，可以打包packages中的所有库
3. 编写esbuild打包代码





