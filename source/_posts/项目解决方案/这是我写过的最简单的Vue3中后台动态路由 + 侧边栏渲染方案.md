---
title: 这是我写过的最简单的Vue3中后台动态路由 + 侧边栏渲染方案
categories:
  - JavaScript-2023
tags:
  - 动态路由
   - vue3
toc: true
date: 2023-06-15
---

## 前言

​	在3年前发布了一篇[vue2后台管理系统动态路由方案](https://juejin.cn/post/6844903816593145864)，时至今日，**vue2**已经升级到了**vue3**，动态路由的实现方案也同步做出了一些升级迭代，帮助开发者们更高效的完成业务需求，然后摸鱼🐟。



​	本次逻辑的升级，主要聚焦于2点

1. **更加简单的实现逻辑**

2. **更加便捷的路由配置**





## 之前的动态路由方案

- 前端只存储基础的路由（**登录、首页、404**）

- 根据不同的登录角色，返回其对应的可访问路由树

- 从服务端获取路由树（**JSON**）并递归处理成**vue-router**可使用的数据结构，并通过**addRouters**拼接到基础路由树，完成动态路由

### 优点

- 相当安全，项目里面只有基础路由，业务路由全部来自接口

### 缺点

- 代码中只保存了基础路由，**业务路由的全部字段需要前端开发人员手动录入到超级管理系统之中，维护业务路由非常繁琐。**
- 客户端逻辑相对复杂，**addRouters**的逻辑必须在路由钩子**beforeRouter**中完成，这部分逻辑比较烧脑。



有没有一种办法，可以**既保留动态路由的特性，也保证代码逻辑的简单性，同时将路由配置回归前端项目呢？**



## 转变解决思路

​	通过以上分析，我们首先可以明确一点，动态路由的配置数据还是**需要放在前端项目**中，而不是将路由的配置录入到系统中。所以，我们可以简化系统后台的权限配置，由之前的超多字段简化为两个字段 **路由名称、路由标识，仅服务于为角色勾选路由，路由配置交还给前端项目**。

​	**路由的树形结构还是需要录入到系统中**，因为我们还需要**保留动态路由的核心逻辑**，给（角色/用户）勾选指定路由。

<img src="http://qiliu.vkcyan.top/Fmtj-mmyZ6NTcHap9GnkgQ6_YXSE.png" style="zoom:50%;" />

​	**Vue2**版本，我们通过**addRouters为项目路由树“做加法”实现动态路由**。

​	**Vue3**版本，我们则为项目路由树手动**“做减法”实现动态路由**。



## 做“减法”实现动态路由

首先依旧将我们的全部路由分成2部分，**基础路由数组，动态路由数组**

基础路由：无论什么角色都可以访问的路由（登录后的公共页面，比如工作台，没有可以为空）

动态路由：拥有权限的角色才可进行访问的路由

```js
/**
 * 基础业务路由
 */
export const baseRouter: RouteRecordRaw[] = [
  //.....
]

/**
 * 动态业务路由
 */
export const asyncRouter: RouteRecordRaw[] = [
  //.....
]


const router = createRouter({
  routes: [
    // 这一层的路由除了dashboard，其他都是业务无关的路由，比如登录 注册 404 500，不属于动态路由
    {
      path: '/login',
      name: 'login',
      meta: {
        title: '登录',
      },
      component: () => import('@/views/login/index.vue'),
    },
    {
      path: '/',
      redirect: '/dashboard',
      name: 'baseDashboard',
      meta: {
        title: '根路径',
      },
      component: layout,
      children: [...baseRouter, ...asyncRouter], // 全部注册到vue-router中
    }
    // ...
  ],
})
```

我们直接将全部路由都注册到**Vue**中，如果不存在鉴权，此时任意角色都可以访问所有页面。



为了实现动态路由的需求，我们只要解决两个问题

1. **如何实现路由拦截，拦截不允许被访问的页面。**
2. **如何做减法，筛选出指定的权限树，用于侧边菜单栏的展示。**



### 如何实现路由拦截

在用户登录的时候，我们会调用服务端的**`获取当前用户权限`**接口，获取到当前用户的权限数据

无论后端给我们返回的什么样的接口，嵌套也好、一维数组也罢，我们都将其处理成一维数组

```js
["index", "dashboard", "root", "goods", "goodsList", "goodsClass"]
```

并将这样的一维数组，保存到我们全局状态库**pinia**中，我们假设变量名称为**authList**。

然后我们再增加路由钩子的逻辑，每次跳转之前，都判断**next**的页面**name**在**authList**中是否存在，如果不存在，则直接**404**，如果存在，则**允许访问**

```js
router.beforeEach((to, from, next) => {
  const auth = useAuthStore()
	// ......
  // 登录后逻辑
  if (auth.isLogin) {
    // 判断权限是否通过
    if (auth.asyncRouter.includes(String(to.name))) {
      next()
    } else {
      next({ name: '404' })
    }
  } else {
    // 未登录逻辑...
  }
})
```

**经过beforeEach的逻辑之后，我们现在就已经实现了基本的鉴权，不允许访问的页面，都将404**。



### 递归筛选出菜单树

​	接下来，我们考虑第二个问题，用户登录之后，自身权限获取完毕，用户进入到管理系统内部，右侧显示功能侧边栏，**我们不能将不属于该用户的动态路由都显示出来**，所以我们需要根据服务端返回的权限数据，实现项目中的动态路由**asyncRouter**的筛选。

​	因为我们路由层级理论上是无限的，所以这里使用**递归**进行实现比较合理。

<img src="http://qiliu.vkcyan.top/Fjpi98MtbL0irT1Va_-kNy2BXSfw.png" style="zoom:50%;" />

实现思路如下

1. 递归遍历**asyncRouter**路由树，如果路由的**name**在权限数组内，并且该菜单可显示，则继续递归**children**，如果没有**children**则不做处理
2. 如果路由的**name**不在权限数组中，则将其**splice**，也就是**做减法**，并且循环下标后退一位，防止跳过下一个。

```js
/**
 * 获取可访问路由树
 * @param tree
 */
export function loopRouter(tree: RouteRecordRaw[], asyncRouter: RouteRecordName[]) {
  for (let i = 0, len = tree.length; i < len; i++) {
    let item = tree[i]
    if (asyncRouter.includes(item.name!) && item.meta!.showMenu) {
      if (item.children) {
        item.children = loopRouter(item.children!, asyncRouter)
      }
    } else {
      tree.splice(i, 1)
      len = tree.length // 刷新循环长度
      if (i < tree.length) {
        // 删除后,数组长度-1,数组的下一位前进了一位,所以一旦splice掉不存在的权限,便需要i--,否则会跳过下一位
        i--
      }
    }
  }
  return tree
}
```

经过以上代码后，我们的路由树便会过滤掉那些不允许被访问的路由

接下来，我们便可以进行菜单的渲染工作了，前端基操我就不做过多赘述了，大家可以看看我精心为大家准备的[源码案例](https://github.com/BlueDancers/vue3-dynamic-routing-admin)。

到此为止，我们的动态路由就实现了，是不是非常简单~





## 安全问题

有经验的小伙伴会问，用户登录后，刷新页面再访问的时候，我们如何处理权限呢？

在我看来这有2个方案

**方案1（相对简单）**：使用**pinia**的持久化插件**pinia-plugin-persistedstate**来持久化我们存储**pinia**的权限数据，无论用户如何刷新，我们的权限数据都一直有效，这样的实现非常简单，但是存在一定的安全隐患，就是心怀不轨的某些人，知道了不允许访问的菜单路由名称之后，可以通过手动修改当前页面的**localStorage**实现权限的突破。

**方案2（更加安全）**：在**pinia**中创建一个变量**isRouterInit**标识权限是否已经被初始化，如果没获取过权限数据，则为**false**，如果获取过则为**true**；每次**beforeEach**的时候，登录情况下都判断该值是否为**false**，如果为**false**，则请求当前角色的权限数据，并在请求完毕后，再进行相关路由拦截逻辑，并将变量**isRouterInit**改为**true**，表明权限已经被初始化。

​	以上2种方案都可以用，如果安全性要求不是特别高，建议方案1，如果对安全性、实时性要求比较高，则建议方案2。

![](http://qiliu.vkcyan.top/Fq4pwPLiWUPmGhd34tSFDfd8x4Ag.png)



## 菜单的排序问题

​	经过实操的小伙伴还会发现一个问题，在项目中的路由排序也许是个麻烦事。

​	可能角色A说：我要**acb**，角色b说：我要**bac**，但是我们路由表放在项目中，其排序是固定不变的，有什么办法可以实现项目左侧路由树按照后台返回的权限字段数据进行渲染呢？

​	其实这个问题很简单，我们针对递归后的动态路由的结果，加一个**sort**即可。

```js
/**
 * 获取可访问路由树
 * @param tree
 */
export function loopRouter(tree: RouteRecordRaw[], asyncRouter: RouteRecordName[]) {
  // .....
  // 将菜单按照当前拥有权限asyncRouter的顺序排序
  return tree.sort((a, b) => asyncRouter.indexOf(a.name) - asyncRouter.indexOf(b.name))
}
```

这样项目的右侧菜单树就会按照我们后端返回的顺序进行排序了。



## 不需要的路由可以删除吗？

​	vue3提供了动态路由的相关[API](https://router.vuejs.org/zh/guide/advanced/dynamic-routing.html)，其中有一个**removeRoute**，可以删除不需要的路由，那么，我们在根据登录角色动态删除剔除不需要在侧边栏显示的路由的时候，是否也同步将不需要的路由进行删除呢？

​	关于这个问题，我的建议是，不建议删除，首先我们的路由模块均为异步加载的，是否删除一定不会访问的路由，都不会影响项目的加载。而一旦真的删除了，用户在推出登录切换账号的时候，一定要**重载当前页面**，因为我们不知道用户下一次登录的角色是什么，会不会用到已经被删除的路由。



## 最后

​	B端项目通过动态路由实现角色鉴权，已经是一个非常成熟的方案，无论是使用**“加法方案”**实现，还是使用**“减法方案”**，都是可行的，理论上都是对权限的一次递归筛选。

​	大家主要根据项目规模、要求合理选择最适合的方案，在安全、便捷、开发难度、稳定性，等多角度做好权衡利弊。

