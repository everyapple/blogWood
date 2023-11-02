---
title: hooks 链表
date: 2023-11-02 17:58:31
tags:
---

# Hooks 单向链表

初始挂载和更新的 hooks，是不同的。
初始化的 useEffect，和 更新渲染后的 useEffect，是在两个不同的阶段调用。本质上是两个函数。
**本质上来源于函数组件要维护一个 hooks 的链表，初始化创建链表，更新的时候更新链表**

分属于两个过程的 hook 函数会在各自的过程中被赋值到*ReactCurrentDispatcher*的 current 属性上。所以在调用函数组件之前，当务之急是根据当前所处的阶段来决定 ReactCurrentDispatcher 的 current，这样才可以在正确的阶段调用到正确的 hook 函数。_HooksDispatcherOnMount_ ，_HooksDispatcherOnUpdate_ 这俩内部是不一样的，不同阶段对 hook 的处理不一样

```js
const HooksDispatcherOnMount: Dispatcher = {
  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  ...
};

const HooksDispatcherOnUpdate: Dispatcher = {
  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  ...
};
```

## 认识 hooks 链表

hook 的结构

```js
const hook = {
  memoizedState: null,
  baseQueue: null,

  next: {
    memoizedState: null,
    baseQueue: null,
    next: null,
  },
};
```
无论是挂载还是更新，只要调用一次hooks函数，就会生成一个hook对象。依次排列，形成链表 存在<font style="color:#d63384">fiber.memoizedState</font>.有一个指针workInProgressHook会记录当前生成的hook对象，可以反应出调用到哪个hook。

### 组件挂载
初次挂载时，组件上没有任何hooks的信息，所以，这个过程主要是在fiber上创建hooks链表。挂载调用的是mountWorkInProgressHook，它会创建hook并将他们连接成链表，同时更新workInProgressHook，最终返回新创建的hook，也就是hooks链表。

### 组件更新
由于存在current树，也会存在currentHook。自然就是有两条hook链表，分别存在workInProgress和current节点的memoizedSate。

**useContext不会被挂到hook链表上**
> 所以为什么 react要用hooks链表呢

如果有多个同个hook调用，还需要有调用的先后顺序，使用next串联所有的hook。

> react16做了哪些更新

> react hooks的本质

> useContext为什么不会被挂到hook链表上

因为在初始化和更新时会有两套不同的函数执行，mount 和 update，但是useContext只有一套代码，都是readContext.所以不需要挂载链表上。
原理类似于观察者模式。Provider上的值发生变化，通知给context和consumer

>useContext 和 redux 的区别

> 为什么只能在函数最外层调用HOOK，不能在循环，条件里调用

因为