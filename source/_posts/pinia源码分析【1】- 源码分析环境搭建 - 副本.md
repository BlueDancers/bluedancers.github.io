---
title: Pinia源码分析【1】- 源码分析环境搭建
categories:
  - JavaScript-2022
tags:
  - Pinia
toc: true
date: 2022-07-06
---

## 前言

​	这段时间在`pinia`与`vuex`之间切换开发，深感`pinia`的简单高效，遂对`pinia`的内部实现有很大兴趣，打算好好研究一下`pinia`的实现原理。

本系列文章参考源码`pinia V2.0.14`

源码分析记录：https://github.com/vkcyan/goto-pinia

本文主要介绍分析源码前的**环境搭建**

## 创建源码分析环境

我们使用vue3开箱即用的CLI，初始化一个项目作为分析环境，创建项目非常简单，不做过多赘述。

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

![image-20220706102739081](https://www.vkcyan.top/image-20220706102739081.png)



## 源码适配

​	将`pinia/packages/pinia`目录下的所有文件复制到我们之前通过CLI生成的空项目之中。并根据我们通过源码入口分析获取的信息进行环境补全。



### 在vite.config.ts增加环境变量

```js
define: {
	__DEV__: "true", // 是否开发环境
	__TEST__: "false", // 是否测试环境
},
```



### 全局环境变量报错

​	我们在vite的配置文件中向运行环境注入了**_DEV_**环境变量，但是TS并不知道我们注入了，便会发出警告。

![image-20220706113750507](https://www.vkcyan.top/image-20220706113750507.png)

​	我们只需要将源码中的`pinia/packages/pinia/src/global.d.ts`文件内的声明复制到项目中的`env.d.ts`即可。

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

至此，我们解决了项目中所有报错信息，接下来我们启动项目。



### 浏览器控制台报错'...ComputedRef'

​	启动项目后，在浏览器控制台收获了一个报错信息

![image-20220706104253811](https://www.vkcyan.top/image-20220706104253811.png)

我们找到报错代码`type.ts`进行分析

![image-20220706104358605](https://www.vkcyan.top/image-20220706104358605.png)

​	错误提示已经非常贴心，可以得知是`tsconfig.json`配置问题，我们根据报错信息修改配置

> 出现报错的原因是因为这里vue CLI生成的代码自带一份配置文件，此文件与pinia源码冲突。

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

log被正常打印，说明pinpa源码已经被正常运行。

## 结语

至此pinia源码分析环境搭建全部结束，接下来我们将开始逐步分析pinia的核心实现逻辑。

