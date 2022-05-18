---
title: js中arguments
top: false
cover: false
toc: true
mathjax: true
date: 2022-05-17 10:14:23
password:
summary:
tags:
	- arguments
	- javascript
categories: Javascript
---

## JS 中的 arguments 对象

### 1.在调用函数时，浏览器每次都会传递两个隐匿的参数

- 函数的上下文对象
- 封装实参的对象 arguments

### 2.arguments 是一个类数组对象，也可以获取长度

- 在调用函数时，传递的实参都会保存子 arguments 中，arguments.length 就是实参的个数。
- 即使不在函数中定义形参，也可以通过 arguments 来使用实参，**不过使用起来比较麻烦**

```javascript
function list() {
  console.log(arguements instanceof Array) //检查arguements是不是数组
  console.log(Array.isArray(arguements)) //使用Array的isArray()方法来检查arguements是不是一个数组
  console.log(arguements.length) //这里的arguements.length就是传递进来的实参的长度（个数）
  console.log(arguements[1]) //输出索引为1的实参

  console.log(arguements.callee) //输出的结果就是当前执行的函数，与console.log(list());相同
}

list(1, 2, 3)
```

### 3.它里面有一个属性叫callee，这个属性对应一个函数对象，就是当前正在执行的函数对象
