---
title: Vue + webpack项目的移动端适配
categories:
  - 移动端
tags:
  - vue3
  - webpack5
  - 响应式
  - vw
toc: true
date: 2022-05-20
---

## 2022-5-20更新

技术栈：vue3 + webpack5

### 安装插件

```bash
npm i postcss-px-to-viewport -D
```

### 增加配置

新建配置文件`postcss.config.js`

```js
module.exports = () => {
  return {
    plugins: {
      autoprefixer: {},
      "postcss-px-to-viewport": {
        unitToConvert: "px", // 需要转换的单位，默认为"px"
        viewportWidth: 750, //  设计稿的视口宽度
        unitPrecision: 5, // 单位转换后保留的精度
        propList: ["*"], // 能转化为vw的属性列表
        viewportUnit: "vw", //  希望使用的视口单位
        fontViewportUnit: "vw", // 字体使用的视口单位
        selectorBlackList: [".ignore", ".hairlines", ".ig-"], // 需要忽略的CSS选择器
        minPixelValue: 1, // 最小的转换数值，如果为1的话，只有大于1的值会被转换
        mediaQuery: false, // 媒体查询里的单位是否需要转换单位
        replace: true, // 是否直接更换属性值，而不添加备用属性
        include: undefined, // 如果设置了include，那将只有匹配到的文件才会被转换，例如只转换 'src/mobile' 下的文件 (include: /\/src\/mobile\//)
        landscape: false, // 是否添加根据 landscapeWidth 生成的媒体查询条件 @media (orientation: landscape)
        landscapeUnit: "vw", // 横屏时使用的单位
        landscapeWidth: 568, // 横屏时使用的视口宽度
      },
    },
  };
};
```



## end

以下配置已经过时，请看最新内容



需要安装一下的插件

```
npm install postcss-aspect-ratio-mini postcss-px-to-viewport postcss-write-svg postcss-cssnext postcss-viewport-units cssnano cssnano-preset-advanced postcss-import postcss-url --S
```



`postcss.config.js`配置

```javascript
module.exports = {
  plugins: {
    'postcss-import': {},
    'postcss-url': {},
    'postcss-aspect-ratio-mini': {},
    'postcss-write-svg': { utf8: false },
    'postcss-cssnext': {},
    'postcss-px-to-viewport': {
      viewportWidth: 750, // 视窗的宽度，对应的是我们设计稿的宽度，一般是750
      viewportHeight: 1334, // 视窗的高度，根据750设备的宽度来指定，一般指定1334，也可以不配置
      unitPrecision: 3, // 指定`px`转换为视窗单位值的小数位数（很多时候无法整除）
      viewportUnit: 'vw', // 指定需要转换成的视窗单位，建议使用vw
      selectorBlackList: ['.ignore', '.hairlines'], // 指定不转换为视窗单位的类，可以自定义，可以无限添加,建议定义一至两个通用的类名
      minPixelValue: 0, // 小于或等于`1px`不转换为视窗单位，你也可以设置为你想要的值
      mediaQuery: false // 允许在媒体查询中转换`px`
    },
    'postcss-viewport-units': {
      filterRule: rule => rule.nodes.findIndex(i => i.prop === 'content') === -1
    },
    cssnano: {
      preset: 'advanced',
      autoprefixer: false,
      'postcss-zindex': false
    }
  }
};

```



> 这里注意假如生成的项目里面没有.postcssrc.js 说明写在package.json里面,记得把package里面的部分配置删除

````
"postcss": {
  "plugins": {
    "autoprefixer": {}
    }
  },
````



最后在index.html里面进行引入viewport-units-buggyfill解决兼容问题

````JavaScript
<script src="//g.alicdn.com/fdilab/lib3rd/viewport-units-buggyfill/0.6.2/??viewport-units-buggyfill.hacks.min.js,viewport-units-buggyfill.min.js"></script>
    <script>
      window.onload = function () { 
        window.viewportUnitsBuggyfill.init({ hacks: window.viewportUnitsBuggyfillHacks });
      }
    </script>
````

#### 注意

如果遇到图片无法正常显示

1.img图片不显示：

全局引入

```
img { 
	content: normal !important;
}
```

2.与第三方UI库兼容问题：

使用postcss-px-to-viewport-opt，然后使用exclude配置项，具体参考 [Vue+ts下的移动端vw适配（第三方库css问题）](https://zhuanlan.zhihu.com/p/36913200)









