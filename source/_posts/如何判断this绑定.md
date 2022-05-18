---
title: 如何判断this绑定
top: false
cover: false
toc: true
mathjax: true
date: 2022-05-18 09:30:55
password:
summary:
tags:
	- this
	- javascript
categories: Javascript
---

### 如何判断一个运行中函数this的绑定

可以按照下面的顺序来进行判断：

1. 由new创建？this直接绑定新创建的对象

   var bar = new foo()
	
2. 由call或者apply（或者bind）调用？绑定到指定的对象

	 var bar = foo.call(obj)

3. 由上下文对象调用？ 绑定到那个上下文对象。

	 var bar = obj.foo()

4. 默认：在严格模式下绑定到undefined否则绑定到全局对象。
   
	 var far = foo()

一定要注意，有些调用可能无意中使用默认绑定规则。如果想“更安全”地忽略this绑定，你可以使用一个DMZ对象，比如ø = object.create(null),以保护全局对象。

ES6中的箭头函数并不会使用四条标准的绑定规则，而是根据当前的词法作用域来决定this，具体来说，箭头函数会继承外层函数调用的this绑定（无论this绑定到什么）