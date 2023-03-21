---
title: （1.1）搭建属于自己的vue3
categories:
  - Javascript-2023
tags:
  - Vue
toc: true
date: 2023-01-29
---



### 前言

​	我们本次源码的目的是最终完成一个简化版的vue3，我们将他称为vue3-mini，本节我们就开始项目的搭建工作。

### 生成项目

```
npm init
```

### 引入ts

```
tsc -init
```

tsconfig.json范本

```json
{
  // 编辑器配置 
  "compilerOptions": {
    // 根目录
    "rootDir": ".",
    // 严格模式标志
    "strict": true,
    // 指定类型脚本如何从给定的模块说明符查找文件。
    "moduleResolution": "node",
    // https://www.typescriptlang.org/tsconfig#esModuleInterop 
    "esModuleInterop": true,
    // JS 语言版本
    "target": "es5",
    // 允许未读取局部变量
    "noUnusedLocals": false,
    // 允许未读取的参数
    "noUnusedParameters": false,
    // 允许解析 json
    "resolveJsonModule": true,
    // 支持语法迭代:https://www.typescriptlang.org/tsconfig#downlevelIteration 
    "downlevelIteration": true,
    // 允许使用隐式的 any 类型(这样有助于我们简化 ts 的复杂度，从而更加专注于逻辑本身 
    "noImplicitAny": false,
    // 模块化
    "module": "esnext",
    // 转换为 JavaScript 时从 TypeScript 文件中删除所有注释。
    "removeComments": false,
    // 禁用 sourceMap
    "sourceMap": false,
    // https://www.typescriptlang.org/tsconfig#lib
    "lib": [
      "esnext",
      "dom"
    ],
    "baseUrl": ".",
    // 设置路径映射
    // 设置后ts在打包过程中也会自动完成路径映射,需要其他地方再次设置
    "paths": {
      "@vue/*": [
        "packages/*/src"
      ]
    }
  },
  "include": [
    "packages/*/src"
  ],
}
```

### 引入代码格式化

vscode下载插件prettier

创建文件.prettierrc.js

```js
module.exports = {
  singleQuote: true, // 单引号
  trailingComma: 'es5', // 对象属性最后有 ","
  semi: false, // 是否需要分号
  printWidth: 110, // 一行最多120
  jsxSingleQuote: true, // jsx使用单引号
  tabWidth: 2, // 一个tab代表几个空格数，默认就是2
  useTabs: false, // 不使用缩进符，而使用空格
  jsxBracketSameLine: true,
}
```

创建文件.prettierignore

```
# Ignore artifacts:
dist
build
coverage
common
tsconfig.json
README.md
```

后续的代码格式化工具上选择prettier即可

我们这里不使用eslint，不做非常强制的代码校验



### 创建相关文件

按照vue3源码中的结构进行创建，暂时只创建packages文件夹



### 引入打包工具

全局安装rollup

```
npm install --global rollup
```

项目创建rollup配置文件rollup.config.js

> output中的name暂时不生效

```js
import resolve from '@rollup/plugin-node-resolve'
import commonjs from '@rollup/plugin-commonjs'
import typescript from '@rollup/plugin-typescript'

/**
 * 默认导出一个数组,数组中,每个对象都是独立导出项
 */
export default [
  {
    input: 'packages/vue/src/index.ts',
    output: [
      // 导出iife的包(自动执行 适用于script标签)
      {
        format: 'iife',
        sourcemap: true,
        file: './packages/vue/dist/vue.js',
        name: 'Vue', // 指定打包后的全局变量名（如果被打包代码，没有任何导出，将不存在导出名称）
      },
    ],
    // 插件
    plugins: [
      // 让rollup 支持打包ts代码,并可以指定ts代码打包过程中的相关配置
      typescript({
        sourceMap: true,
      }),
      // 与webpack不同的是,rollup并不知道如何寻找路径以外的依赖,比如node_module中的
      // 帮助程序可以在项目依赖中找到对应文件
      resolve(),
      // rollup默认仅支持es6的模块,但是还存在很多基于commonjs的npm模块,这就需要改插件来完成读取工作
      commonjs(),
    ],
  },
]
```



package.json中增加打包命令

```bash
"build": "rollup -c"
```



执行打包命令

```bash
npm run build
```

然后我们在`packages/vue/src/index.ts`编写测试代码

```
import { isArray } from '@vue/shared' // ts部分我们配置了paths选项

console.log(isArray([]))
```

不出意外的话，这里肯定是正常打包了，并且会生成sourceMap文件。



### 热更新

package的script中增加一个命令

```
"dev": "rollup -c -w"
```



至此，基础的vue3框架环境我们就搭建完成了。

