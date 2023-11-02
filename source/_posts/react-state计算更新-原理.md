---
title: react state计算更新 原理
date: 2023-11-02 15:44:59
tags:
---

# react中的计算状态

## 概述
一旦用户的交互产生了更新，就会生成一个update对象去承接新的状态。多次更新（多次调用setState）会产生多个update对象，链接成一个环形链表：**updateQueue**，挂载在fiber节点上，然后在该fiber上的beginWorker阶段循环updateQueue，依次处理。在React中，类组件和根组件用一类update对象，函数组件用另一类。
**useEffect, useImperativeHandle, useLayoutEffect,只有这三个hook会产生副作用，updateQueue同时也要收集这些副作用。**
#### update对象结构
```js
const update ={
    eventTime, // update的产生时间，如果因为低优先级则会报超时，react会调用他一次
    lane,// update的优先级
    suspenseConfig,// 任务挂起相关
    tag,// 更新的类型 updatestate, replacestate, forceUpdate, captureUpdate
    payload,// 更新携带的状态  分为类组件和根组件 根组件中是react.element
    callback,// setstate的回调
    next,// 指向下一个update的指针
}
```
#### updateQueue的结构
```js
const queue ={
    baseState,//前一次更新计算得出的state
    firstBaseUpdate,//前一次更新时updateQueue第一个被跳过的update对象
    lastBaseUpdate,//以first为起点到最后一个update的队列中最后一个
    shared:{
        pending:null//存储这本次更新的update队列
    },
    effects:null//数组，保存update.callBack不为空的update
}
```


> 为什么更新队列是环状
> > 因为方便定位到链表的第一个元素，链表最后一个指针指向第一个update，否则需要遍历。
> > 环状链表，只需记住尾部，无需遍历所有操作

#### firstBaseUpdate， lastBaseUpdate的概念及作用
> A1 -> B1 -> C2 -> D1 -> E2

举例： 字母为状态， 数字为优先级（越小越高）
第一次渲染队列 以 A1->B1->D1 ，遇到C2跳过

**本次更新完成之后，firstBaseUpdate:C2 lastBaseUpdate:E2, baseState:AB**

> 为什么baseState中只有AB,没有D呢
> 1. 因为baseState的定义是 被跳过的update之前那些update所计算出的状态。
> 2. 这样目的是为了保障最终的updateQueue所有优先级的update处理完之后和预期结果一直。也就是说，虽然第一次计算结果为ABD，但是最终的结果一定是ABCDE

### 更新的处理机制
- 准备阶段
- 处理阶段
- 完成阶段

#### 准备阶段
主要是整理updateQueue，可能有两条队列
1. 上次遗留的 从first-last
2. 本次新增的
将这两条合并起来，但不需要合并成环型，方便从头到尾遍历。

**另外，这次的操作都在workInProgress中，因此需要在current节点上操作一次，保持同步**

#### 处理阶段
循环处理上一个整理好的队列
1. 本次更新依然取决于优先度（update.lane和 renderLanes 渲染优先度）
2. 本次的结果基于baseState

##### 优先级不足的情况
update被跳过
- 将被跳过的整理到first和last中
- 记录baseState，只在第一次跳过时记录，因为低优先级任务重做时，会从第一个被跳过的开始执行
- 记录被跳过的update优先级，更新即将结束之后放进workInProgress.lanes,得以再次发起，重做低优先级。

> 第一次更新的baseState 是空字符串，更新队列如下，字母表示state，数字表示优先级。优先级是1 > 2的
A1 - B1 - C2 - D1 - E2 
第一次的渲染优先级（renderLanes）为 1，Updates是本次会被处理的队列:
 Base state: ''
 Updates: [A1, B1, D1]      <- 第一个被跳过的update为C2，此时的baseUpdate队列为[C2, D1, E2]，
                             它之前所有被处理的update的结果是AB。此时记录下baseState = 'AB'
                             注意！再次跳过低优先级的update(E2)时，则不会记录baseState
Result state: 'ABD'--------------------------------------------------------------------------------------------------
第二次的渲染优先级（renderLanes）为 2，Updates是本次会被处理的队列:
 Base state: 'AB'           <- 再次发起调度时，取出上次更新遗留的baseUpdate队列，基于baseState
                             计算结果。
Updates: [C2, D1, E2] Result state: 'ABCDE'

##### 优先级足够的情况
- 如果baseUpdate不为空，放入baseUpdate队列
- 处理更新，计算新状态 
主要是为了让最终全部更新完成的结果与预期的结果一致。

##### 完成阶段
完成赋值和优先级标记
- 赋值baseState，first，last，updateQueue
- 如果任务完成，代表没有优先级被跳过，意外着本次的update都处理完了，lanes清空；如果不是，则重新将低优先级的update放入lanes
- 更新workInProgress节点上的memoizedState

## 总结
对更新的处理都是围绕优先级。processUpdateQueue函数的主要目的就是处理更新。
**优先级被跳过时，需要记住他的状态和此时的优先级之后的更新队列，还要将队列放到current节点中，做备份。**
目的是不乱序，完整处理。
参考文章：[扒一扒React计算状态的原理](https://segmentfault.com/a/1190000039008910)