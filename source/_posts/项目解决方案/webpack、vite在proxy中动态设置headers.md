---
title: 通过webpack、vite实现proxy headers的动态设置（高某强看后都要请我吃鱼）
categories:
  - 日常开发
tags:
  - 关于proxy
toc: true
date: 2023-02-17
---



# 通过webpack、vite实现proxy headers的动态设置（高某强看后都要请我吃鱼）



## 前言

大家都知道，利用**webpack、vite**的**proxy**可以解决开发环境的跨域问题。

但是在真实开发场景下，我们可能不仅要面对跨域问题，还有可能面对**动态header**的情况。



让我们来看如下案例



## 案例

强盛集团开发了一套**多店铺的H5商城系统**，此时小安 小龙 小虎，都想利用这个系统开一个线上商城，我们如何区分他们的店铺呢？

聪明的同学肯定已经想到了答案，用二级域名进行区分

```
小安 -> xa.shop.com
小龙 -> xl.shop.com
小虎 -> xh.shop.com
```

接下来让我们把视角聚焦到强盛集团的技术部门。



面对这样的多店铺商城系统，开发环境肯定无法用ip直接访问了，因为ip无法识别具体是什么店铺。

这个问题其实也很好解决，修改本地host即可

```
127.0.0.1 xa.shop.com
127.0.0.1 xl.shop.com
127.0.0.1 xh.shop.com
```

假设我们开发环境的端口是8082，我们想在开发环境访问小安的店铺，则通过`xa.shop.com:8082`进行访问。



​	直到某一天，安全部门发现了沙海集团的的scrf恶意攻击，决定限制该产品请求发起方的**Origin Header**与**Referer Header**，并且限制了必须是80端口。

> 这两个请求头的含义是标记来源域名，可以起到防止scrf攻击的目的。



​	此时前端开发就麻烦了，因为在开发环境，我们的**Origin Header**与**Referer Header**都是**x.shop.com:8082**，这会被服务端识别为不合法的请求来源，同时因为浏览器安全限制，前端是不具备直接修改**referer**头的能力， 除非将项目端口号改为80端口，但是这并不是一个好办法。



**如果你是强盛集团的前端开发，你会怎么解决以上问题呢？**



## 后端高某强提供的思路

​	后端开发高某强这时候提供了一个想法，通过本地nginx代理8082端口就好了呀。



1. 在开发机器本地启动一个**nginx**
2. 通过**nginx**将**x.shop.com**指向**127.0.0.1:8082**
3. 同时配合**host**的修改，实现开发环境去端口的诉求。



后续，我尝试了这个方案，确实是可以实现的，也在团队中推广并使用了一段时间；

但是长期使用下就暴露了一些问题

1. 每次新增一个站点，都需要同时增加host、nginx中的配置，流程复杂。
2. 并不是每个前端都了解nginx，初级开发非常容易出问题，增加团队内耗。



## proxy解决方案

### Vite解决方案

​	直到某一天，我在**vite**的文档中突然发现一个细节，**server.proxy**的实现依赖[node-http-proxy](https://github.com/http-party/node-http-proxy)，而这个库具备**设置请求头的能力**

![](https://www.vkcyan.top/Fl8GqobFWoDlUT-UZTrUN43npq1H.png)

​	如果是这样，我是否可以在开发环境通过proxy代理请求接口，同时覆写**Origin Header**与**Referer Header**的方式来解决我们遇到的多域名+端口限制问题呢？进而在开发环境规避掉nginx。

​	通过**vite**的问题可以了解到参数**configure**可以编写**http-proxy**相关逻辑，再结合**http-proxy**文档，我们便可以完成相关代码。

```ts
  server: {
    // .....
    proxy: {
      '/client': {
        target: 'https://api.xxxx.com', // 需要代理的地址
        changeOrigin: true,
        secure: true, // 如果是https接口，需要配置这个参数
        rewrite: (path) => path.replace(/^\/client/, ''),
        configure: (proxy) => {
          proxy.on('proxyReq', (proxyReq, req, res) => {
            // req是当前真实请求的地址 开发环境为：a.shop.com:8082
            let host = req.headers.host!.split(':')[0] // a.shop.com 动态获取当前请求地址，并去除端口
            proxyReq.setHeader('referer', `http://${host}`)
            proxyReq.setHeader('origin', `http://${host}`)
          })
        },
      },
    },
   	// .....
  },
```

​	测试结果符合预期，**web**端请求被**proxy**代理，并在代理请求中完成了端口的去除，符合了服务端对**Origin Header**与**Referer Header**的要求。



### Webpack解决方案

​	**vite**测试成功后，我们便开始对**webpack**的**vue2.x**项目**proxy**动态**headers**进行评估；通过**webpack4**的文档，我们可以了解到**webpack4**的**proxy**是基于[http-proxy-middleware](https://github.com/chimurai/http-proxy-middleware) 进行实现。

​	接下来我们也顺利在**http-proxy-middleware**文档中找到相关配置

![](https://www.vkcyan.top/Fh7xKiiJnus0ciu_JRlUfqO7Wyk9.png)



我们基于**webpack**与**http-proxy-middleware**的文档，就可以很顺利的做产出了。

```js
  devServer: {
		// ....
    proxy: {
      '/client': {
        target: 'https://api.xxxx.com', // 需要代理的地址
        changeOrigin: true, //是否跨域
        secure: true, // 如果是https接口，需要配置这个参数
        pathRewrite: { '^/client': '' },
        onProxyReq: (proxyReq, req, res) => {
          let host = req.headers.host.split(':')[0]
          proxyReq.setHeader('referer', `http://${host}`) //添加请求头
          proxyReq.setHeader('origin', `http://${host}`) //添加请求头
        },
      },
    },
    // .....
  },
```



​	项目配置完成后，我们便可以在开发环境利用proxy的动态headers完成**a.shop.com:8082**正常访问线上端口了，只需要在本地配置host即可。



## 结语

高某强了解到前端部门使用**proxy + 动态headers方案**后，连连称赞，表示请我去他家吃鱼。

如果你也遇到了类似的问题，快来试试proxy的解决方案吧~
