---
title: react和vue的区别
date: 2023-11-02 20:05:30
tags:
---

Vue和React作为两个大框架，总是被拿来问区别和选型。零零散散的，想到哪是哪。所以想着做一个收集的总和。

## 概述

#### Vue
Vue是一个用于构建用户界面的渐进式框架，自底向上的增量开发。它的定位是**降低前端开发的门槛**
> 渐进式框架：允许开发者逐渐引入它的核心库和附加的生态系统库。也就是说他可以逐渐引入所需要的功能，不必要的功能可以先不引入。

#### React
React主张**函数式编程的理念**，善于处理组件化的页面。

### 共同点
- 数据驱动视图
- 单一数据流，响应式开发
- 使用虚拟dom+diff算法
- 组件化思想

## 核心思想
从核心思想上捋一下，代码层面的问题。
### 写法差异
Vue：template模版，（Vue2以上引入vdom概念，也可以使用jsx）,拥抱经典的html+css+js，提供一些指令，数据劫持的方式进行监听，对组件的渲染更加精准
React：JSX all in js 更加灵活。setState的引入使得更新时自顶向下的，父组件props改变，子组件也会重新渲染，需要优化
### api差异
Vue：template+options api 需要理解一些自定义指令，slot，filter等。
React：api较少，知道setState就可以开发。
### 社区生态差异
Vue：有很多官方主导开发的解决方案，比如vuex，vue-router等等
React：只关注底层，上层应用解决方案基本不插手。大部分都丢给社区解决，导致社区比较繁荣。
### 响应式原理
Vue：数据监听，数据劫持
React：setState,状态机，数据不可变
### diff算法
    - 对比节点，节点元素相同，classname不同，
      - vue：认为是不同的类型，删除重建。
      - react：认为是同类型，只修改节点属性
    - 对比列表
      - vue：两段到中间
      - react：从左到右
### vuex和redux
    vuex：数据可变，原理通过getter/setter实现
    redux：数据不可变，通过diff比较差异

## 技术选型
### 浅浅的想象
原则上来说，从开发角度，学习成本，上手难易，以及性能去对比，差距不是很大。而且学习成本，上手难度来比较，很没有说服力。
通俗的约定是：react用大项目，vue用小项目。然而大小项目的区分没有标准。
### 深度的比较
#### 市场占比
从npm下载量来说，react较为领先。了解到react许可的问题，保留vue的技术栈总是没错滴。
#### 生态和ui组件库
生态上react比vue好一点，但是核心还是用代码这方面，并不能作为优势去考量。
组件库上大差不差，都很丰富。
都可以访问ref，操作dom
### 小程序框架开发
常见的框架
1. 芋头 react
2. wepy vue
3. uni-app vue
这些都是基于框架二次开发的。
vue的一些周边库，高度依赖于vue，和vue强绑定。而redux他可以再react用，也可以再vue上用。类似的问题很多，总结下就是，react的东西可以再vue上用，但是反过来vue不可以。
比如wepy 他不支持vuex 支持redux
### app生态
rn 和 weex
### 代码层面 jsx和template
#### 可测试性，重构
vue需要另起一个页面，react可以直接借助jsx语法实现一个函数方法。
```js
function Test(props: { hello: string }) {
  console.log(props);
  return <div>{props.hello}</div>
}
```
#### 一部分丢失
在vue的页面，封装一些函数，引入的时候需要在methods中重新申明一次。
#### this指针
在react里，函数组件这张是没有this指针的，在多人协作或者承接的时候，显得尤为简洁。

#### 参考文档
[vue和react深度比较](https://developer.aliyun.com/article/1207640)
