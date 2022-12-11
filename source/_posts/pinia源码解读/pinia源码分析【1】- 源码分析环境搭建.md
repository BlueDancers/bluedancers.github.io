---
title: Pinia源码分析【1】- 源码分析环境搭建
categories:
  - JavaScript-2022
tags:
  - Pinia
toc: true
date: 2022-07-06
---



## 专栏导航

[分析pinia源码之前必须知道的API](https://juejin.cn/post/7124279061035089927)

[Pinia源码分析【1】- 源码分析环境搭建](https://juejin.cn/post/7117131804229763079)

[Pinia源码分析【2】- createPinia](https://juejin.cn/post/7119788423501578277)

[pinia源码分析【3】- defineStore](https://juejin.cn/post/7121661056044236831)

[pinia源码分析【4】- Pinia Methods](https://juejin.cn/post/7123504805892325406)

## 前言

​	经过`vue3`从`beta版本`走向默认版本，`vue2.x`更新最后一个版本`Naruto`，全局状态管理也从`vuex`慢慢走向更加易用更加契合`vue3`的`pinia`。

本系列文章将会带领大家从**源码角度**去理解下一代vue全局状态管理库 pinia 的**实现原理**。



参考源码`pinia V2.0.14`

源码分析仓库：https://github.com/vkcyan/goto-pinia



工欲善其事，必先利其器，我们应该如何去阅读pinia的源码呢？

本文将手摸手教大家如何**在vue3环境搭建pinia的源码阅读环境。**



## 创建源码分析环境

我们使用vue3开箱即用的CLI初始化一个项目，创建项目非常简单，不做过多赘述。

https://vuejs.org/guide/quick-start.html

```bash
npm init vue@latest
```



## pinia源码入口分析

先去`pinia`的官方仓库下载源码

源码地址：https://github.com/vuejs/pinia，我们将其clone到本地

````
git clone https://github.com/vuejs/pinia.git
````

首先分析`pinia`仓库的打包文件，寻找源代码位置



### 源码地址

​	在`pinia/packages/package.json`中，我们找到了打包命令，打包命令中可以得知，打包文件为`../../rollup.config.js`

![image-20220706112404074](https://www.vkcyan.top/image-20220706112404074.png)

![image-20220706112113981](https://www.vkcyan.top/image-20220706112113981.png)

​	在打包文件中，我们找到了被打包源码的入口文件，即为`pinia/packages/pinia/src/index.ts`



### 仓库依赖

​	在打包文件`rollup.config.js`中`line121`标注了依赖文件，不过我们通过CLI生成的项目中已经包含了以下依赖文件，所以这一步我们不需要额外操作。

![image-20220706100633575](https://www.vkcyan.top/image-20220706100633575.png)



### 环境变量

​	源码中存在大量环境变量注入代码，具体配置在`rollup.config.js`中`line121`；如果缺失环境变量声明，会导致源码无法正常运行。

​	所以在源码阅读环境，我们需要添加合适的环境变量，让源码正常运行起来。

![image-20220706102739081](https://www.vkcyan.top/image-20220706102739081.png)



## 环境适配

将`pinia/packages/pinia`目录下的所有文件复制到我们之前通过CLI生成项目的`/src`中。

并根据我们通过源码入口分析获取的信息进行环境变量补全。



### 在vite.config.ts增加环境变量

```js
define: {
	__DEV__: "true", // 是否开发环境
	__TEST__: "false", // 是否测试环境
},
```



### 全局环境变量报错

​	我们在vite的配置文件中向运行环境注入了pinia必要的环境变量，但是`TypeScript`并不认识相关全局变量，便会发出警告。

![image-20220706113750507](https://www.vkcyan.top/image-20220706113750507.png)

​	我们需要将源码中的`pinia/packages/pinia/src/global.d.ts`文件内的声明复制到项目中的`env.d.ts`即可。

```js
/// <reference types="vite/client" />

// Global compile-time constants
declare var __DEV__: boolean;
declare var __TEST__: boolean;
declare var __FEATURE_PROD_DEVTOOLS__: boolean;
declare var __BROWSER__: boolean;
declare var __CI__: boolean;
declare var __VUE_DEVTOOLS_TOAST__: (
  message: string,
  type?: "normal" | "error" | "warn"
) => void;
```

至此，我们解决了项目静态检查阶段中所有报错信息，接下来我们启动项目。



### 浏览器控制台报错'...ComputedRef'

​	启动项目后，在浏览器控制台收获了一个报错信息

![image-20220706104253811](https://www.vkcyan.top/image-20220706104253811.png)

我们找到报错代码`type.ts`进行分析

![image-20220706104358605](https://www.vkcyan.top/image-20220706104358605.png)

​	错误提示已经非常贴心，可以得知是`tsconfig.json`配置问题，我们根据报错信息修改配置

> 出现报错的原因是因为这里vue CLI生成的代码自带一份配置文件，此文件与pinia源码的tsconfig部分配置发生了冲突。

```json
{
  "extends": "@vue/tsconfig/tsconfig.web.json",
  "include": ["env.d.ts", "src/**/*", "src/**/*.vue"],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    "importsNotUsedAsValues": "remove", // 默认被设置为error error情况下类型导入必须增加前缀type 以区分类型 改成remove即可
    "preserveValueImports": false // 对于类型，在同时启用了 "preserveValueImports" 和 "isolatedModules" 时，必须使用仅类型导入进行导入；改成false即可
  },

  "references": [
    {
      "path": "./tsconfig.config.json"
    }
  ]
}
```



### 源码分析环境测试

在源码处增加打印，测试pinia源码是否正常运行。

![image-20220706105740320](https://www.vkcyan.top/image-20220706105740320.png)

![image-20220706105829188](https://www.vkcyan.top/image-20220706105829188.png)

log被正常打印，说明pinia源码已经被正常运行。



​	如果感觉搭建环境过于繁琐，又想阅读pinia源码，可以直接clone项目，https://github.com/vkcyan/goto-pinia，开箱即用~

## 结语

​	至此，我们便完成了万里长征第一步；植入的源码文件被正常运行，我们便可以通过**log debug**的方式来进行逻辑观测，接下来我们将正式开始pinia核心实现的探索之旅~

