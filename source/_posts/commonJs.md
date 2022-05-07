---
title: commonJs
top: false
cover: false
toc: true
mathjax: true
date: 2022-05-07 14:12:36
password:
summary: commonJs-javascript模块化规范
tags:
	- commonJs
	- javascript
	- nodeJs
	- 模块化
categories: javascript
---
### CommonJS
长久以来Javascript都不支持模块化，但随着前端工程越来越庞大、复杂，模块的需求也越来越高。为此，社区推出了各种模块化的实现和规范，比如AMD规范、CMD规范和CommonJs规范等。Node.js使用的CommonJs规范，它通过module.export导出模块。演示代码如下：
```javascript
	// 模块文件：module.js
    module.export = {
        title:'My name is module'
        say:function(){
            console.log('log from module.js')
        }
    }
    // 入口文件
    var myModule = require('./module.js')
    myModule.say();
    console.log(myModule.title)
```
把以上代码放到用以目录，在该目录下打开命令行，在命令行执行如下指令：
```shell
	> node main
```
最终程序输出
```shell
	> log from module.js
	> My name is module
```
这说明入口程序已经加载了模块module.js,并且能访问此模块下导出的内容，这就是Node.js为开发者提供的模块机制。
一旦一个模块被导入运行环境中，就会被缓存。当再次尝试导入这个模块是，就会读取缓存中的内容，而不会重新加载一边这个模块的代码。这种机制不仅避免了重复导入相同模块冲突的问题，还保证了程序的执行效率。

