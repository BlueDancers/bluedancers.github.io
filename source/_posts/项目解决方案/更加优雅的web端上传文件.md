---
title: 更加优雅的web端上传文件 
categories:
  - JavaScript-2023
tags:
  - JavaScript
toc: true
date: 2023-01-03
---



## 前言

​	大家一起回想一下，我们在使用Element、Antd的时候，是如何实现图片、文件上传功能。

<img src="http://www.vkcyan.top/FkmvkuGX712E-A7FFkPMnBuwOUkg.png" style="zoom: 33%;" />

​	antd就不截图了，使用方法差不多，相信写过相关代码的同学，都会感觉比较繁琐。

- 既要改html，还需要写JavaScript，开发效率低。
- 组件API复杂，很多时候，并不需要那么多功能，心智负担重。
- 等等

​	这时候，写过微信小程序的同学，心里肯定会想，小程序的上传图片API开发体验还可以，逻辑上也很直观，通过事件触发函数，函数内实现上传图片的逻辑，我们web端能否也能拥有类似的开发体验呢？



## npm地址

**以下内容为实现思路以及关键代码，如果您仅仅想使用的话，请直接到[npmjs](https://www.npmjs.com/package/choose-to-file)**

文档地址：https://github.com/vkcyan/choose-to-file#readme

```bash
npm i choose-to-file
```



## 需求分析

首先，web端使用文件上传功能，一般使用input标签进行实现

```html
<input type ="file" onchange/>
```

最后在`onchange`callback中，获取本次的结果。

我们会发现，上传文件必须通过特定的标签才能触发。这么来看似乎与函数式相违背。

但这并不是死路一条，我们可以通过`input.click()`，自动指定点击事件，核心逻辑就是这样。



## 实现思路

- 创建`input`标签
- 触发`<input />`的`click`事件
- 监听上传结果，并作为结果返回，同时对临时数据进行销毁。



## 具体实现

实现不与具体框架做绑定，我们基于原生逻辑进行开发，天然兼容web端框架。

**理想使用方式**

```js
async function uploadImg() {
	try {
		let res = await chooseToFile()
		console.log('file'， res)
	} catch (err) {
		console.log(err)
	}
}
```

接下来开始实现`chooseToFile`，首先我们创建input标签

```js
let input = document.createElement('input')
input.type = 'file'
```

创建完成后，立刻执行自动点击事件

````js
	input.click()
````

然后监听`input`的`onchange` callback，并通过`Promise resolve`进行返回。

```js
input.onchange = (evt) => {
  removeInput(input) // 删除dom
  let { files } = evt.target as HTMLInputElement
  return resolve(files)
}
```

至此，我们的核心逻辑就完成了，还是很简单的~



## 一些问题

### 初始化阶段调用无效，并且会弹出警告。

```
File chooser dialog can only be shown with a user activation. 
```

​	这是因为浏览器的安全限制，不允许用户在没有任何”激活行为“的情况下，JavaScript调用窗口，针对此问题官网有详细的说明[user-activation](https://developer.chrome.com/blog/user-activation/)。

​	也就是说这是不允许的，大家开发中要规避这种行为。



### 无取消上传callback

​	当用户点击文件进行上传的时候，我们可以通过onchange组件进行获取，但是如果用户关闭了上传文件弹窗，或者点击”取消“按钮，input并未提供响应的回调函数。

​	如果无法监听取消上传，逻辑将不知道何时销毁临时标签input，查阅了一些资料后，找到了解决方案。

​	无论用户是否上传，只要当前用户有操作，上传行为结束，都会重新聚焦到body本身，也就会触发全局的focus方法，如果用户上传了文件则在onchange callback中将fileCancle赋值为false；

​	之后focus事件触发的时候则不会进入if内逻辑。

```js
let fileCancle = true // 是否未上传文件
window.addEventListener(
  'focus',
  () => {
    setTimeout(() => {
      if (fileCancle) {
        removeInput(input)
        reject('upload canceled')
      }
    }, 500)
  }
)
```



## 最终效果

- 调用函数，打开文件管理器

  - 用户上传文件：.then触发，收到上传的文件信息

  - 用户取消上传：.catch触发，收到无文件上传错误



## 最后

​	功能的实现并不复杂，因为作者工作上vue用的较多，所以可以保证Vue是没问题，理论上React也是没问题的，欢迎大家体验，在使用中有任何问题，请评论区留言。

​	祝你开发愉快。

















