---
title: 关于我使用Hexo搭建个人博客这档子事
categories:
  - JavaScript-2022
tags:
  - Hexo
  - 个人博客
toc: true
date: 2022-06-30
---



## 前言

​	在我刚工作的那年，便搭建过一次Hexo静态站点到github Pages，那时候输出不多，几乎全都是技术类文章，于是我便把学习记录、文章都放在github仓库中，也会把精品文章发布到掘金；这样的情况下，自建博客没有被维护下去。

​	时间来到2022年，随着输出类型越来越多，自然便需要一个空间来存储所有的输出，调研了很多在线文档工具，比如语雀，飞书，Notion，都是非常不错的，不过我个人非常喜欢用Typora，也不太习惯其他文档工具，同时作者也是比较喜欢自由的人，思来想去还是自建博客吧，这便是再次搭建github Pages的原因。

​	博客搭建过程也花费了一些时间，遇到了若干小问题，希望后来人不要踩前人已经踩过的坑，这也是这篇文章的由来，希望借此可以帮到大家。

​	接下来就让我们正式开始吧。



## 适合人群

- 爱折腾的开发者
- 爱自由的开发者
- 需要一个美观的空间来存储各类文章的开发者

## Hexo介绍

​	[Hexo](https://link.zhihu.com/?target=https%3A//hexo.io/zh-cn/)是一个快速、简洁且高效的博客框架。Hexo 使用 [Markdown](https://link.zhihu.com/?target=http%3A//daringfireball.net/projects/markdown/)（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。即把用户的markdown文件，按照指定的主题解析成静态网页。



## githubPages介绍

​	GitHub Pages 是一种静态站点托管服务，它直接从 GitHub 上的存储库获取 HTML、CSS 和 JavaScript 文件，可选择通过构建过程运行文件，然后发布网站。您可以在[GitHub Pages 示例集合](https://github.com/collections/github-pages-examples)中查看 GitHub Pages 站点示例。



​	通过上面的介绍我们便可以得到一个结论：**githubPages + Hexo = 个人博客**，下面让我们正式开始搭建个人博客吧！



## 环境要求

- nodejs环境
- 安装git并
- 注册github账号，并已经将当前电脑的ssh秘钥添加到了Github（相当于告诉github，这台电脑有权操作你的github数据）



## 搭建github Pages

![](https://www.vkcyan.top/FtXLW2s18F2De4HdFl9GZn-0Y2Vz.png)

仓库创建完成后，点击

![](https://www.vkcyan.top/FsurWnTJagwBZAGN6n77r7Rti1Ob.png)



之后点击pages菜单，这里正常情况下会自动开启，这边得到了一个访问地址，这就是个人博客的链接了。

![](https://www.vkcyan.top/Ft06OhJmhttzFocFQp6OwJ-XNeC5.png)

​	关于分支请各位根据自己的情况来选择，只需要在后面配置hexo相关配置的时候别填写错误即可。

​	到此为止githubPages便设置完成了，到这一步已经成功一半了！上图的链接已经可以访问，只是现在里面还没有内容，接下来便是增加站点内容。



## 配置hexo

### 搭建环境

首先全局安装hexo

```
npm install -g hexo-cli
```

然后创建存放站点资源的文件夹，在你认为合适的目录下执行

```bash
hexo init <folder>
cd <folder>
npm install
```

install完成后，在该文件夹下执行

```bash
hexo s // hexo server
```

正常情况下会看到依一下信息

![](http://www.vkcyan.top/Fr5GDqqz86brApdn8L_jCkRQETWT.png)

然后我们访问`http://localhost:4000/`,便可以看到hexo生成的站点

![](https://www.vkcyan.top/FnYUcxkH02d8-kUCfbjfA6wISWdw.png)

能到这一步就算搭建成功了，接下来我们需要认识hexo生成文件中的配置，来个性化定制我们的个人站点

### hexo文件说明

####  _config.yml

非常重要的文件，配置网站的基本信息信息，比如名称 描述 分页 主题等信息。具体含义看这里 [配置参数](https://hexo.io/zh-cn/docs/configuration)

#### themes

​	第三方网站主题存放目录，后面我们会用到

#### source

​	该文件夹是存放站点文章的地方，_post目录为文章存放处，除 `_posts` 文件夹之外，开头命名为 `_` (下划线)的文件 / 文件夹和隐藏的文件将会被忽略。

#### public

​	所有打包生产的文件都会被输出到public中，其中`MD` `html`文件将会被解析再存放在其中，其他文件会被拷贝进去。



### 编写一篇文章

我们在`_post`中增加一个`md`文件，然后文章开头处写入以下信息，使用过`wordPress`的同学应该很熟悉，这是对当前文章的属性描述，具体配置可以看[官方配置](https://hexo.io/zh-cn/docs/front-matter)，之后便是填入文章内容，还记得我们通过`hexo s`启动的本地服务器吗，文章编写完成后，让我们去看看是是否存在变化。

![image-20220628134758790](https://www.vkcyan.top/image-20220628134758790.png)

能够看到文章便说明成功了

在文章头部编写的**categories**，**tags**字段，hexo会按照关键字自动生成索引。

![image-20220628135401927](https://www.vkcyan.top/image-20220628135401927.png)

下一步我们便要将其发布到github Pages中，让所有人都可以访问到。

## 将hexo发布到githubPages

​	我们之前已经配置好git本地环境与仓库的创建，所以接下来部署便十分简单

1. 安装快速部署脚本

该脚本会在你的目录下生成`.deploy_git/`执行更新命令的时候就会将该文件夹下的所有文件上传到指定的仓库中。

```
npm install hexo-deployer-git --save
```

2. 填写相关配置

```yml
deploy:
  type: git
  repo: https://github.com/username/username.github.io // 替换这里的username
  branch: pages // 填写部署的GitHub分支，默认为gh-pages
  message: '' // 提交信息，存在默认值，可以不填写
```

3. 一键部署

```
hexo clean // 清除缓存文件 db.json与public/
hexo d //  deploy的缩写，读取yml中deploy的配置并部署
```

![](http://www.vkcyan.top/image-20220628143934942.png)

看到以上日志则说明部署成功



接下来我们前往仓库查看是否正常上传，以及github Pages是否正常工作

![](http://www.vkcyan.top/image-20220628144123522.png)



点击`Action`查看部署任务是否正常完成，正常情况下`Action`会在60s左右完成，部署完成后我们便可以访问线上站点了

https://username.github.io/

如果样式，内容正常显示则代表部署成功~（可能访问速度还有些慢，我们后面再优化）

![image-20220628145010327](https://www.vkcyan.top/image-20220628145010327.png)

如图所见，我们便部署成功了~

这里肯定有好奇的小伙伴要问了，为什么你的样子和我部署的不一样?

哎，这就是下面要说的，自定义主题

## 自定义主题

​	一个风格优美的博客可以为读者带来更好的体验，也是作者个性的一种表达，这也是自建博客的优势所在， hexo中存在大量官方提供 + 个人维护的[Themes](https://hexo.io/themes/)

![image-20220628182610145](https://www.vkcyan.top/image-20220628182610145.png)

​	如果出现无法正常访问的情况可以在github中搜索[hexo-theme](https://github.com/search?q=hexo-theme)，同样可以看到受欢迎的hexo主题，每个主题都会有使用文档，只需要仔细阅读文档并进行配置即可使用主题。

​	主题文件都存在在themes文件夹中，并且需要将根目录的_config.yml的theme字段改为主题名称。



## 常见问题

### 站点很慢

​	github Pages访问很慢，绝大部分原因都是因为JS脚本加载阻塞问题，在不加任何异步属性的情况下，script的下载和执行都会阻塞dom的渲染。

​	例如项目依赖了jQuery，但是其CDN因为某些原因加载缓慢甚至超时失败，都会导致页面的长时间白屏，这会非常影响访问者的用户体验，以及作者的创作热情。

​	如果出现此类情况，我们就打开**Chrome devTool**查看**NetWork**中**加载缓慢**的脚本文件，然后复制其名称，在hexo源码中找到，再百度搜索国内的高速CDN替换掉即可。（注意不要修改打包后代码）

### 点击菜单404

​	有些第三方主题自带的一些菜单点击可能会出现白屏的情况，这是因为没有建立标签页

![image-20220628163232358](https://www.vkcyan.top/image-20220628163232358.png)

执行以下命令建立不存在的页面即可，新建的md具体内容还是需要参考使用的主题文档

```
hexo new page books
```



### 缓存问题

​	修改了一篇文章，但是打包到github Pages依旧没有被删除，类似这样的缓存问题怎么办？

​	我们只需要在每次push到github之前清除hexo生成的缓存文件即可

```bash
hexo clean
```



### 部署报错

执行`hexo d`，得到错误：

```bash
fatal: unable to access 'https://github.com/username/username.github.io/': OpenSSL SSL_read: Connection was reset, errno 10054
FATAL {
  err: Error: Spawn failed
      at ChildProcess.<anonymous> (xxx\username.github.io\node_modules\hexo-util\lib\spawn.js:51:21)
      at ChildProcess.emit (node:events:390:28)
      at ChildProcess.cp.emit (xxx\username.github.io\node_modules\cross-spawn\lib\enoent.js:34:29)
      at Process.ChildProcess._handle.onexit (node:internal/child_process:290:12) {
    code: 128
  }
} Something's wrong. Maybe you can find the solution here: %s https://hexo.io/docs/troubleshooting.html
```

​	网络问题，重试即可。



## 其他静态构建工具

### Vuepress

​		大Vue出品的静态站点生成工具，用来写文档非常不错，但是如果你对个性化要求较高，可能他不是很适合你。

### jekyll

​	非常老牌的静态网站生成工具，使用Ruby语言进行开发，这就意味着使用jekyll需要安装Ruby环境，环境的事情劝退我了，并且相对于Hexo未发现明显优势，喜欢的小伙伴可以试用一下。

### Hugo

​	基于Go语言开发的静态网站构建工具，不过它并不需要Go环境，而相对于hexo需要node环境与前端开发者的天然适配性，也是存在一些劣势，但是对于非前端开发者还是不错的选择，因为他号称最快的构建框架；在搭建完hexo后我才看到了hugo感觉有点意思，喜欢折腾的朋友可以试用一下，也是非常棒的工具，目前拥有59.8k的Star，是静态构建工具中最受欢迎的项目





## 结语

​	搭建好github Pages是一件相对简单的事情，不断输出却是一件有难度的事情，如果通过这篇文章成功搭建起了博客，非常恭喜，你完成了万里长征非常重要的第一步，希望你可以坚持输出，从小的知识点开始做起，保持对可输出点的敏感嗅觉，每月输出几篇，搭建起自己的知识库。

​	如果一个正在被使用的技术方案作为开发者的你却无法通过输出教会别人，这便不算掌握了该技术方案，浅尝辄止乃是技术岗位的大忌。在输出的过程中，你会带着问题再深入学习一遍，会发现一些潜在问题，而为了顺利输出，便需要自发的弄明白遇到的疑问；

​	用输入驱动输出，用输出倒逼输入，将复杂的知识通过简单平实的话说出来，并教会别人，才是完全掌握。





