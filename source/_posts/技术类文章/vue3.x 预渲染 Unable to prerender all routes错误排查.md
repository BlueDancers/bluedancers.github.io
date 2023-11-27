---
title: Vue3.x 预渲染 Unable to prerender all routes错误排查
categories:
  - JavaScript-2022
tags:
  - JavaScript
toc: true
date: 2022-03-15
---

## 前言

​	BOOS最近对前几年做的公司官网不太满意，觉得没有有效体现公司的优势，表明随着公司这几年的努力发展，我们将会接触到更大规模的合作伙伴，自然要展示更好的企业形象，所以官网重做。



## 需求分析

- 没有交互的静态页面，但是存在大量动画
- 需要支持良好的SEO

​	最早期的官网是`vue2.x` + `webpack3.x` + `vue-cli-plugin-prerender-spa`进行实现的，效果挺不错，很快各大搜索引擎就收录了我们的网站，所以这次我们打算沿用此方案，不过使用最新技术栈；

### 为什么不用vite

​	查阅vite的生态后，未找到类似**prerender-spa**的plugin，没办法支持预渲染，所以vite就被淘汰了。

### 为什么不用unxtjs

​	我们的官网不具备大量的接口交互，用**Nnxtjs**多少有点杀鸡用牛刀了，并且还需要使用**pm2**部署代码，付出于收获不成正比，被淘汰。



### 最终方案

​	我们部门是vue技术栈，团队不考虑react，通过以上排除法，只能使用`vue3.x` + `webpack5.x` + `prerender-spa`进行业务实现了。

## 技术实现

### 基础模板

我们使用最新的`vue-cli`进行项目搭建，选择vue3版本，最近的cli默认就是webpack5



### 安装预渲染插件

```bash
npm i prerender-spa-plugin -D
```



### 增加配置

```js
const { defineConfig } = require('@vue/cli-service')
const PrerenderSPAPlugin = require('prerender-spa-plugin')
const path = require('path')

module.exports = defineConfig({
  transpileDependencies: true,
  configureWebpack: (config) => {
    if (process.env.NODE_ENV === 'production') {
      config.plugins.push(
        new PrerenderSPAPlugin({
          staticDir: path.join(__dirname, 'dist'),
          routes: ['/xxx'],
        })
      )
    }
  },
})

```



### 打包测试

```bash
npm run build
```

然后就出现一个错误

```bash
[prerender-spa-plugin] Unable to prerender all routes!
```

让我们一起抽丝剥茧，看看报错的具体原因。

## 错误排查

因为报错提示很模糊，我们打开他的源码，在源码line144发生错误的地方增加log，了解具体报错。

<img src="http://www.vkcyan.top/image-20220520134031605.png" alt="image-20220520134031605" style="zoom:50%;" />



再次执行`npm run build`，得到真正的错误。

```
Building for production...error TypeError: compilerFS.mkdirp is not a function
```



​	我们继续最终源码发现 **compilerFS** 由**webpack**进行提供，我们带着错误前往**webpack**官网查询错误，于是就找到了[Filesystems](https://webpack.js.org/blog/2020-10-10-webpack-5-release/#filesystems)，因为这个插件已经好几年没有更新，而我们当前使用的是webpack5，出现了API变更的情况。

​	于此同时，根据错误提示，我们也在该库的issues中找到了历史讨论。

<img src="http://www.vkcyan.top/image-20220520135141080.png" alt="image-20220520135141080" style="zoom:67%;" />



在讨论中，找到了两种解决方案

1. **修改node_modules源码，使其兼容webpack5**

```js
 // From https://github.com/ahmadnassri/mkdirp-promise/blob/master/lib/index.js
  const mkdirp = function (dir, opts) {
    return new Promise((resolve, reject) => {
      console.log('\ndir', dir, opts, '\n');
      try {
        compilerFS.mkdirp(dir, opts, (err, made) => err === null ? resolve(made) : reject(err))
      } catch(e) {
        compilerFS.mkdir(dir, opts, (err, made) => err === null ? resolve(made) : reject(err))
      }
    })
  }
```



2. **使用已经被修改的库，感谢这位大哥**

![image-20220520135437289](http://www.vkcyan.top/image-20220520135437289.png)



````js
npm i @dreysolano/prerender-spa-plugin
````



我们使用第二种方案，重新修改**vue.config.js**

````js
- const PrerenderSPAPlugin = require('prerender-spa-plugin')
+ const PrerenderSPAPlugin = require('@dreysolano/prerender-spa-plugin')
````



然后再次打包测试

![image-20220520135659280](http://www.vkcyan.top/image-20220520135659280.png)



打包成功，通过启动本地服务器**curl**命令测试得知，SEO功能正常，未发现问题。

## 总结

​		使用**prerender-spa-plugin**打包出现报错`[prerender-spa-plugin] Unable to prerender all routes!`，更换库为**@dreysolano/prerender-spa-plugin**，即可解决问题。

