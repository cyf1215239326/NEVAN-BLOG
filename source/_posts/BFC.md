---
title: BFC
top: true
cover: false
toc: true
mathjax: true
date: 2022-05-16 10:15:04
password:
summary:
tags:
	- css
	- html
	- bfc
categories: BFC
---

# 完全弄懂 BFC

BFC 全称是 Block Formatting Context，即块级格式化上下文。

BFC 最初被定义在 css2.1 规范的 Visual formatting model 中。要想明白 BFC 到底是什么，首先看看什么是 visual formatting model

## 视觉格式化模型（Visual Formatting Model）

视觉格式化模型是用来处理文档并将它显示在视觉媒体上得机制，它让视觉媒体知道如何吃力文档。（视觉媒体-user agent 通常指的是浏览器。）

在时候也格式化模型中铭文当属的每个元素根据盒模型生成零个或多个盒子。这些盒子的布局受一下因素控制：盒子得尺寸和类型、定位方案（普通文档流、浮动文档流和绝对定位流）、文档树中元素间的关系、外部因素（如视口大小、图像本身的尺寸等）

## 普通文档流

普通文档流是一种定位方案。
在 css2.1 中，普通文档流包括：块级盒子的块级格式化上下文、内联级盒子的内联格式化上下文、块级和内联级盒子的相对定位

在普通文档流的盒子属于格式化上下文（formatting context）。可以属于块级或者内联级，但不能同时属于。块级盒子属于块级格式化上下文。内联盒子属于内联格式化上下文。

## 格式化上下文（formatting context）

Formatting Context,即格式化上下文。用于决定如何渲染文档的一个区域

不同的盒子使用不同的格式化上下文来布局

每个格式化上下文都拥有一套不同的渲染规则，他决定了其子元素将如何定位，以及和其他元素的关系和相互作用。

简单理解为格式化上下文就是**为盒子准备的一套渲染规则**。

常见的格式化上下文有这样几种：

【Block formatting context】(BFC)

【Inline formatting context】(IFC)

【Grid formatting context】(GFC)

【Flex formatting context】(FFC)

## 什么是 BFC

BFC 即 Block Formatting Context（块级格式化上下文）。

先来看看 W3C 对于 BFC

> Floats, absolutely positioned elements, block containers (such as inline-blocks, table-cells, and table-captions) that are not block boxes, and block boxes with 'overflow' other than 'visible' (except when that value has been propagated to the viewport) establish new block formatting contexts for their contents.

也就是，有这几种情况会创建 BFC：

- 根元素（html）或者其他包含它的元素
- 浮动元素
- 绝对定位元素
- 非块级盒子的块级容器（inline-block，table-cells，table-captions 等）
- overflow 部位 visible 的块级盒子

BFC 的范围

> A block formatting context contains everything inside of the element creating it that is not also inside a descendant element that creates a new block formatting context.

一个 BFC 包含创建该上下文元素的所有子元素，但不包括创建了新 BFC 的子元素的内部元素。

换句话说，一个元素不能同时存在两个 BFC 中。

## BFC 的特性

- 盒子从顶部开始垂直排列
- 两个相邻的盒子之间的垂直距离由外边距（即 margin）决定
- 块级格式化上下文中相邻的盒子之间在垂直边距折叠
- 每个盒子的左外边与容器的左边接触（从右到左的格式化则相反），即使存在浮动也是如此，除非盒子建立了新的块格式化上下文
- 形成了 BFC 的区域不会与 float box 重叠
- 计算 BFC 的高度时，浮动子元素也参与计算

## 从实际代码来分析 BFC

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>BFC</title>
    <style>
      * {
        margin: 0;
        padding: 0;
      }

      .green {
        background: #73de80;
        opacity: 0.5;
        border: 3px solid #f31264;
        width: 200px;
        height: 200px;
        float: left;
      }

      .red {
        background: #ef5be2;
        opacity: 0.5;
        border: 3px solid #f31264;
        width: 400px;
        min-height: 100px;
      }

      .gray {
        background: #888;
        height: 100%;
        margin-left: 50px;
      }
    </style>
  </head>
  <body>
    <div class="gray">
      <div class="green"></div>
      <div class="red"></div>
    </div>
  </body>
</html>
```

![BFC1](BFC1.png)

在这个例子中，构建出BFC的只有class名为green的盒子（浮动元素）

green盒子由于浮动，脱离普通文档流,形成浮动流。他好像跟其他两个盒子不在同一个世界一样。

现在普通文档流中只有gray和red盒子，所以gray的高度只被red撑起来。红色盒子也无视绿色盒子的存在，跑到了最左边。

## 实例二
如果想要让灰色框包裹住绿色，最简单的方式就是给gray盒子构建出BFC
```html
.gray{
  background:#888;
  height: 100%;

  overflow: hidden;
}
```
![BFC2](BFC2.png)

还记得BFC的特性吗？当我们计算BFC的高度时，浮动子元素也参与计算。这样一来我们就能灰色盒子的高度被绿色盒子撑开了。

## 实例三
我们再来看看如何让红色盒子“接受”绿色盒子

```html
.red {
  background: #EF5BE2;
  opacity: 0.5;
  border: 3px solid #F31264;
  width: 400px;
  min-height: 100px;

  overflow: hidden;
}
```
![BFC3](BFC3.png)

我们将红色盒子也构建出BFC，根据特性，形成BFC区域与float box不会发生重叠。于是这里红色成功“接受”了绿色盒子。

