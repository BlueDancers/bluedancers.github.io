---
title: 浏览器读取xls并生成二维码下载到本地
categories:
  - JavaScript-2022
tags:
  - JavaScript
toc: true
date: 2022-3-15
---

## 需求

一次普通的技术需求会议

​	项目经理首先发言 我们技术这边需要将xls表格中的几千条数据变成二维码，并且中间镶嵌logo，图片底部放置编号，由于xls表格数据私密，不能通过第三方完成

​	平常这个事情都是后端处理的，前端就是来摸鱼的，但是这次一反常态，后端脸黑了，带样式搞不来，脚一蹬，直接装死

​	项目经理用期盼的眼神看着我，顿时我紧张了起来，眼神飘忽，我已经好多年没搞过node了啊！！会议室都沉默了，在项目经理不断精神攻击下，后端装死的情况下，看来注定要大前端来拯救世界了，毕竟JavaScript万能语言，俺来试试吧！



## 实现方案

​	以上情节纯属虚构，但是需求确实是这样的，虽然好几年没碰过node，好歹年轻记性好，用过的基本都还记得，调研实现方案上没出现太多问题，有如下方案

### puppeteer 

地址：https://github.com/puppeteer/puppeteer

​	使用基于node环境的puppeteer，进行二维码绘制，图片绘制，是JavaScript开发者面对此类需求的主流选择

### node-canvas

地址：https://github.com/Automattic/node-canvas

​	同样是在服务端完成渲染，但是这个库依赖node-gyp，如果不安装python2，那安装过程懂得都懂，不过这也是很不错的方案

### 浏览器

​	通过浏览器canvas绘制，然后下载下来，会有刷刷刷下载图片的炫酷效果



很明显有刷刷刷下载图片炫酷效果的方案更好，所以就选择你了 **浏览器**方案！



## 问题分解

确定了技术方案，就要考虑具体实现了

- JavaScript读取execl文件，并处理成理想格式
- 将读取到的execl中的网址字段生成一张二维码
- 将二维码写入canvas，在其中间加上logo，并在底部加一行文字
- 将canva转化为DataURL，下载它
- 不断递归生成，直到xls数据全部处理完毕



**理论存在，实践开始**！

## 具体实现

### 启动一个本地服务器

首先我们通过VScode **Live Server** 启动一个本地服务器

这里有好奇宝宝要问了，为啥第一步是这？

答：因为浏览器是访问不了电脑的文件系统的，所以只能通过启动一个本地服务器的方案，来读取我们的资源文件



### 创建html，引入资源库



分析需要用到的第三方开源库

- 解析xls https://github.com/sheetjs/sheetjs
- 生成QRcode https://github.com/soldair/node-qrcode

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>生成二维码</title>
    <script src="./qrcode.js"></script>
	<script src="./xlsx.full.min.js"></script>
  </head>

  <body>
    <!-- 用于生成载体 最终生成的图片大小，按自己的需求来 -->
    <canvas width="260" height="310" id="canvas"></canvas>
  </body>
    <script>
    const ctx = initCanvas(); // 获取ctx实例
    // 初始化画布
    function initCanvas() {
      const canvas = document.getElementById("canvas");
      const ctx = canvas.getContext("2d");
      ctx.fillStyle = "#fff";
      ctx.fillRect(0, 0, 260, 310);
      return ctx;
    }
    </script>
</html>

```



### 解析xls文件

```js
readWorkbookFromRemoteFile().then((res) => {
    // res 为实际解析代码 [{key:'xxxx',value:'xxxx'},....]
});

// 读取xls信息，并处理
function readWorkbookFromRemoteFile() {
    return new Promise((resolve, reject) => {
        var xhr = new XMLHttpRequest();
        xhr.open("get", "http://127.0.0.1:5500/xls.xls", true);
        xhr.responseType = "arraybuffer";
        xhr.onload = (e) => {
            if (xhr.status == 200) {
                var data = new Uint8Array(xhr.response);
                var workbook = XLSX.read(data, { type: "array" });

                // 获取实际表格长度（去除表头）
                let carryLen = 0;
                for (const key in workbook.Sheets["Sheet"]) {
                    const ele = workbook.Sheets["Sheet"][key];
                    if (key.includes("A")) {
                        carryLen++;
                    }
                }
                // 解析数据
                let xls = [];
                for (let i = 2; i <= carryLen; i++) {
                    let data = workbook.Sheets["Sheet"];
                    xls.push({
                        key: data["A" + i].w,
                        value: data["B" + i].w,
                    });
                }
                resolve(xls);
            }
        };
        xhr.send();
    });
}
```

看到这里肯定也有细心的好奇宝宝问，为啥循环体中的`i`为2呢?

答案：因为表格中的A1，B1为表格的第一行，而第一行是表头，要去除



#### 将链接生成为二维码

````js
new Promise((resolve, reject) => {
    // 生成二维码
    QRCode.toDataURL(
        'xxxxxxx',
        {
            width: 260,
            height: 260,
            margin: 3,
        },
        (error, url) => {
            if (error) console.error(error);
            const code = new Image();
            code.src = url;
            code.onload = () => {
                ctx.drawImage(code, 0, 0);
                resolve(code);
            };
        }
    );
````



### 写入中间logo

```js
return new Promise((resolve, reject) => {
    const code = new Image();
    code.src = "http://127.0.0.1:5500/logo.jpeg";
    code.onload = () => {
        ctx.drawImage(code, 260 / 2 - 20, 260 / 2 - 20, 40, 40);
        resolve();
    };
});
```



#### 写入底部文字

```js
// 写入编号
ctx.font = "24px Arial";
ctx.fillStyle = "#000";
ctx.textAlign = "center";
ctx.textBaseline = "middle";
ctx.fillText(xls[index].value, 130, 270);
```



### canvas转化为图片，并下载到本地

```js
// 用于预览
let url = document.getElementById("canvas").toDataURL("image/png");
var a = document.createElement("a"); // 生成一个a元素
var event = new MouseEvent("click"); // 创建一个单击事件
a.download = xls[index].value; // 将a的download属性设置为我们想要下载的图片名称，若name不存在则使用‘下载图片名称’作为默认名称
a.href = url; // 将生成的URL设置为a.href属性
a.dispatchEvent(event); // 触发a的单击事件
```



第一张图片，完成生成

<img src="http://www.vkcyan.top/image-20220424154056800.png" alt="image-20220424154056800" style="zoom: 67%;" />

### 递归调用

我们修改发起逻辑代码，逻辑尾部增加递归调用就好啦

```js
readWorkbookFromRemoteFile().then((res) => {
    createImg(res, 0); // 递归生成
});
// ......
// 实际生成逻辑
function createImg(xls, index) {
    new Promise((resolve, reject) => {
        // 生成二维码
    })
        .then((res) => {
        // 生成中间logo
    })
        .then(() => {
        // 写入编号
    })
        .then(() => {
        // 下载图片
    })
        .then(() => {
        setTimeout(() => {
            if (xls.length > index + 1) {
                ctx.fillStyle = "#fff"; 
                ctx.fillRect(0, 0, 260, 310); // 初始化画布
                createImg(xls, index + 1);
            }
        }, 20); // 爱惜机器，加个延时，也可以去掉延时，体会机器的极致速度
    });
}
```



## 最终效果

![8my3l-a8ef0](http://www.vkcyan.top/8my3l-a8ef0.gif)

至此，终于实现了刷刷刷下载图片炫酷效果，此时可以脑部一段很快的rap，如果华佗再世，崇洋可以医治，外邦来学汉字...............



最终生成的文件

<img src="http://www.vkcyan.top/image-20220424160255476.png" alt="image-20220424160255476" style="zoom:67%;" />



## 最终代码地址

> 一定要针对该项目启动一个本地服务器，否则资源无法访问

[web-Output-QRcode](https://github.com/vkcyan/web-Output-QRcode)



## 结语

​	首先纠正一点，JavaScript开发者针对生成二维码类似的任务，首选肯定是`puppeteer`，使用浏览器绕个弯这种实现方案，多少带点科研味道，长期项目自然是不推荐的

​	带着学习的态度去完成需求，并且不断优化代码、总结问题，将遇到的未知知识点学会，（比如创建a链接，自动触发点击事件），这才是本文的目的。

​	感谢阅读，觉得还不错就点个赞吧~

​	QQ交流群：530496237 大佬解答疑惑~（内有微信群二维码）

​	