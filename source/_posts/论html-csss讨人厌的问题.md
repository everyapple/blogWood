---
title: 论html/csss讨人厌的问题
date: 2023-11-19 19:56:54
tags:
---
<meta name="referrer" content="no-referrer" />

## 浏览器究竟是如何渲染页面的
通俗的来说，答案无非是构建dom树，构建cssom树，合成render树，进行布局绘制。
那究竟是js的加载阻塞了页面的渲染，还是js的执行阻塞了页面的渲染？
所以回过头把浏览器渲染的机制从头看一遍。

那如何将html渲染到屏幕上呢？

### 关键渲染路径
在浏览器接搜到一个html文档是，会经历一个叫**关键渲染路径**的步骤。
{% note primary%}
构造文档对象模型（dom），构造css对象模型（cssom），生成渲染树，布局，绘制
{% endnote %}
#### 构造dom树
1. 转化 ： 浏览器从磁盘（缓存）读取html的原始字节，将他们转化为单个字符。
2. 分词（词法分析）： 将上一步骤得到的字符串转为Token。
3. 分析（语法分析）： 将token转为对象nodes，定义了他们的属性和规则
4. 构造dom树：html定义了不同标签之间的关系，所以最终浏览器得到的对象是一个树形结构。
整个过程会将 HTML 大概处理成为上述类型的树状结构, 最终输出文档对象模型 (DOM)，之后浏览器会使用该对象模型进行其他处理。
#### 构造cssom树
和上述类似，Css 文件经过转化为字符，然后进行分词、转化为节点最终拼接为一个树状的 Cssom。
{% note success %}
Q: 为什么css也是树状结构
A: 因为css规则是向下级联的嵌套方案，浏览器在计算任何节点的时候，会从适用于该节点的最通用的规则开始计算。比如，如果需要计算的节点是 body 元素的子元素，那么它会应用 body 的样式，之后会一层一层进行递归该过程从而得到该节点最终的样式。
{% endnote %}

特殊提问？
> css解析和dom解析是否是并行呢？

 css 如果写在style标签里，还是会被html parse执行的。大部分情况下，css文件会用外联加载css文件。
 **当使用link标签引入css文件时**
 cssom的生成时机不会影响dom的改变（js文件因为可能会对dom进行操作，所以会影响），所以当html parse遇到link标签的导入时，他不会去等待这个文件的下载和解析，然后再去解析后续dom，而是在加载style脚本的同时，继续解析dom。这个就是所谓的并行关系（也就是非阻塞关系）
 当加载完样式脚本后，需要去解析link中的样式内容生成cssom树上的节点，这个过程就是parse stylesheet，主线程依然需要他去完成。
 这个过程必然会和parse html抢占主线程资源。
 因此也就是说 **css加载不会阻塞dom的解析，但是会阻塞dom的渲染，也会阻塞后面js的执行**
 存在特例，"<head>"尾部的 CSS 会阻塞位于其后紧跟的 DOM 的解析
 这种情况被称为 FOUC(Flash of Unstyled Content), 样式闪烁, 因为 First Plain 机制的存在, 会将已经解析的 DOM 和 CSSOM 结合生成渲染树并进行部分渲染, 当该 CSS 加载完成后, 才会继续解析 DOM, 而且之前已经渲染的内容将会根据解析完成的 CSS 进行重绘.
 在 Firefox, Chrome, Edge 中，如果 <link rel="stylesheet" href="xxx"> 后面跟着 <script>, 则 CSS 加载完成后, 才能触发 DOMContentLoaded
对于 JS ，在交互上，想要正确的操作 DOM，将 JS 放在 HTML 尾部，当解析到 JS 时，虽然没有触发 DOMCOntentLoaded，但是已经可以正确操作 DOM，减少加载时间，将 JS 异步化，对于外部 JS，采用添加 defer 属性或者 async 引入
而 CSS 在头部不会阻塞 DOM 的解析，但是跟在其后的 JS 在 CSS 加载的这段时间是不能够被执行的, 即使 JS 下载完成时间比 CSS 要快, 优先加载 CSS 避免 FOUC 的出现 

#### 生成render树
浏览器合并dom和cssom，组成一个具有所有可见节点样式和内容的render tree
也因为合成render树这一过程，需要dom树和cssom树都生成完毕，因此css加载会堵塞dom的渲染。
#### 布局
布局过程的输出是一个“盒子模型”，它精确地捕获视口内每个元素的确切位置和大小：所有相对测量值都转换为屏幕上的绝对像素。
#### 绘制
既然浏览器已经明确的知道哪些节点是可见的，以及它们的样式和几何形状，我们可以将此信息传递到最后阶段，它将 RenderTree 中的每个节点转换为屏幕上的实际像素，此步骤通常称为“Paint”。
![过程](https://picx.zhimg.com/80/v2-d567cb08f77f85f8b462ea610bdb6d8b_1440w.webp?source=2c26e567)

### js会不会影响渲染呢

**外部脚本链接的加载和执行只会影响后续 Dom 的解析和渲染，对于脚本之前的的 Dom 并不会阻塞它的解析以及渲染，这也就是为什么我们常说将 js 放在底部。**
而内联 肯定影响 别想了
defer & async 不会阻塞渲染
defer会在domcontentloaded之前执行
## 结论

1. 普通情况下css 脚本的加载并不会阻塞后续 Dom Tree 的生成。（Css 文件加载不阻塞解析特性） 
2. 同时 css 脚本的加载是会阻塞 RenderTree 的合成，从而阻塞页面的渲染（Css 文件加载渲染阻塞特性）。
3. 特殊情况下，比如 link 的样式脚本后存在 JS 文件，那么此时 Css 代码的加载是会阻塞后续 JS 脚本的执行从而 JS 脚本会阻塞后续 Dom 的解析，从而变相相当于阻塞了 Js 代码之后的 Dom 解析。


## 浏览器字体默认最小12px，如何变成10px
1. zoom
2. scale
3. css -webkit-text-size-adjust
transform 属性 scale zoom
主要原因是浏览器做了最小的限制
#### zoom
zoom是变焦，可以用百分比，normal，数值。可以改变页面上元素的尺寸，会引起页面的重排。缩放相对于左上角。
#### scale
对元素的缩放，transfrorm:scale()
**scale属性只对可以定义宽高的元素起效**因此需要将元素转为行内元素
```css
.span10{
    font-size:12px
    display:inline-block
    transform:scale(0.5)
}
```
不改变页面布局，不会引起重排，不可以是百分比和normal，但可以是负数。
同时也可以决定某个维度，scalex，scaley
若为负数，就是对那个元素的翻转。
缩放相对于居中。
#### css属性  -webkit-text-size-adjust：none
设定文字大小是否根据设备浏览器大小调整
- auto：「默认」，字体大小会根据设备/浏览器来自动调整；
- percentage：字体显示的大小
- none:字体大小不会自动调整
存在兼容性问题

## link和import的区别
- link可以加载css，还可以定义rel等属性，import只能加载css文件。
- link可以使文件和html一起加载，但import需要等页面加载完再执行，会「影响浏览器的并行下载」，使得页面在加载时增加额外的延迟，增添了额外的往返耗时
- link 兼容性好
- link可以通过操作dom，js插入，import因为是基于文档的所以不可以
- link的权重也高于import

## 回流重绘
页面渲染的步骤是什么呢？
{% note primary%}
构建dom树
计算样式
布局定位
图层分层
图层绘制
合成显示
{% endnote %}
在css渲染页面的时候，会发生**重排**，**重绘**，**直接合成**这三种事件。
重排：回流，也就是说元素的**位置**，**大小**，导致其他元素发生变化，需要重新计算。
重绘：颜色 阴影等发生变化
直接合成：transform, opacity等属性，只需要将多个图层合并，直接生成**位图**。

#### 触发时机
![触发时机.png](https://s2.loli.net/2023/11/21/V7qLBCRt1fgxM8c.png)
e.g: margin-left position transform
{% note success%}
margin-left: 会引发重排，也会影响其他元素
position： 会引起自己的重排，但不会影响其他元素
transform：不会引起重排重绘。
ps：
合成阶段：就是更改了一个既不要布局也不要绘制的属性，那么渲染引擎会跳过布局和绘制，直接执行后续的合成操作，这个过程就叫合成。
{% endnote %}

**提升合成层的最好方式是使用 CSS 的 will-change 属性**
will-change:transform
就是告诉浏览器，我要变形了！
噗，通俗上来说，就是告诉浏览器要启动GPU渲染图层，是关于哪一块这样子。
缺点是：耗电，所以不能瞎用
也就是说，如下
1. 结合伪元素
```js
.will-change-parent:hover .will-change {
  will-change: transform;
}
.will-change {
  transition: transform 0.3s;
}
.will-change:hover {
  transform: scale(1.5);
}
```
2. 如果使用js，要移除
如果使用JS添加will-change, 事件或动画完毕，一定要及时remove.

## webp降级
1. canvas.toDataUrl() 是否返回image/webp 
2. 服务响应头判断accept 接受一个imagep/webp 返回
3. node端直出所有图片都为webp。在端上做一次webp的check，不支持则降级到原jpg或者png。
<https://segmentfault.com/a/1190000040044459>