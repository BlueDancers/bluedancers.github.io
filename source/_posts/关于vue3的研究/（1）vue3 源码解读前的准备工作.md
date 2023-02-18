---
title: （1）vue3 源码解读前的准备工作
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-01-28
---



### 前言

​	在22年下半年就有想阅读vue3源码的想法了，但是因为很多不可抗力原因，一直在不断拖延。

​	23年年初下定决心，一定要在半年内完成vue3的核心源码的解读，所有源码的阅读记录我都讲输出到本专栏中，目测可能有10片以上的文章，在输出中，我会尽力保证文字的简单易懂；

​	话不多说，我们直接开始吧！



### 下载vue3源码

仓库地址：https://github.com/vuejs/core

本专栏版本为3.2.37，地址：https://github.com/vuejs/core/releases/tag/v3.2.37

克隆仓库地址

```
git clone https://github.com/vuejs/core
```



### 打包与运行vue3源码

**vue3**采用**monorepo**进行包管理，而**monorepo**由**pnpm**提供，所以需要一定要预先安装**pnpm**

```
npm install -g pnpm
```



**pnpm**安装完成后，开始安装**vue3**的依赖

```
pnpm install
```



依赖安装完成后，开始**vue3**源码的打包工作，该步骤可能花费较长时间

```
npm run build
```

​	打包完成后，将会在`packages/vue/dist`，路径下生成打包后的vue3的代码，接下来我们去`packages/vue/examples`官方提供的案例中运行打包后代码，在目标html文件通过**vscode启动live server**，即可完成vue3示例的运行。



### 打包vue3源码可能遇到的问题

#### Error: Command failed with exit code 128: git rev-parse HEAD

运行`build`之后，出现以上错误，原因是因为**build**的过程中，会读取了当前**git**的**commit id**，如果当前目录下没有**.git**文件，相关逻辑就会出错，所以需要注释掉`scripts/build.js`中以下代码

```
line34 const commit = execa.sync('git', ['rev-parse', 'HEAD']).stdout.slice(0, 7)
line97 `COMMIT:${commit}`
```

**跳过获取commitID相关逻辑代码，即可正确打包。**



#### vue3示例中源码未支持SourceMap

当我们完成上文的打包后，我们运行一个**example**，运行后就会发现一个问题，我们使用Vue3源码是打包后的代码，没有sourceMap，这样是无法调试源码的。

**vue3**源码内提供了打开**sourceMap**的能力，修改打包命令`package.json`中的line7，即可

```
node scripts/build.js -s
```



为什么这个改动呢？我们还要从`rollup.config.js`入手

在`rollup.config.js`的**line94** 我们可以看到

```
output.sourcemap = !!process.env.SOURCE_MAP // 反向取反，获取其对应的boolean类型的值
```

要开启sourcemap首先需要修改process.env.SOURCE_MAP，而这个值来源于`scripts/build`

`scripts/build.js`中的**line103**

```
sourceMap ? `SOURCE_MAP:true` : ``
```

而这里的sourceMap来源于命令行后缀

```
const args = require('minimist')(process.argv.slice(2))
const sourceMap = args.sourcemap || args.s
```

**所以我们只需要在脚本命令后增加`-s`即可开启sourcemap。**



至此为止，我们便可以在vue源码环境中进行阅读与调试了





