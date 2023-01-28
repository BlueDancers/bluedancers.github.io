---
title: vue3 开始源码解读（2）
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-01-28
---



### 下载vue3源码

仓库地址：https://github.com/vuejs/core

克隆仓库地址

```
git clone https://github.com/vuejs/core
```



### 打包与运行vue3源码

vue3采用monorepo进行包管理，而monorepo由pnpm提供，所以需要我们预先安装pnpm

```
npm install -g pnpm
```



pnpm安装完成后，开始安装vue3的依赖

```
pnpm install
```



依赖安装完成后，开始vue3代码的打包工作，该步骤可能花费较长时间

```
npm run build
```



打包完成后，将会在`packages/vue/dist`，路径下生成打包后的vue3的代码，接下来我们去`packages/vue/examples`官方提供的案例中运行打包后代码，在目标html文件中启动live server，完成Vue3源码的运行。



### 打包vue3源码注意事项

Error: Command failed with exit code 128: git rev-parse HEAD

运行`build`之后，出现以上错误，原因是因为build的过程中，读取了当前git的commit id，如果没有.git，相关逻辑就会出错，所以

注释掉`scripts/build.js`中的

```
line34 const commit = execa.sync('git', ['rev-parse', 'HEAD']).stdout.slice(0, 7)
line97 `COMMIT:${commit}`
```

跳过获取commitID相关逻辑代码，即可正确打包。



### 为打包后的源码增加sourceMap，正式开始源码解读



















