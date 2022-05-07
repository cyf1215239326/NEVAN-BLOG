---
title: remote模块
top: false
cover: false
toc: true
mathjax: true
date: 2022-05-07 14:40:57
password:
summary:
tags:
	- remote
	- Electron
categories: Electron
---

# remote模块
## 说明

remote在electron > 10的版本中已被弃用，最终将被删除

@electron/remote是一个`electron`模块，它将JavaScript对象从主进程桥接到渲染器进程，这使得我们可以访问主进程的对象，就像它们在渲染器进程中可用一样。

> ⚠警告!:这个模块有许多微妙的缺陷。有更好的解决方案比使用此模块更好的方法来完成您的任务。例如`ipcRender.invoke`可以服务于许多常见的用例

## 使用此模块的基本步骤
1. 你需要安装它
```shell
npm install --save @electron/remote
```
2. 初始化在它被渲染进程使用之前
```shell
// in the main process:
require('@electron/remote/main').initialize()
```
3. require('electron').remote 替换成 require('@electron/remote')
```shell
// in the renderer process:
// Before
const { BrowserWindow } = require('electron').remote
// After
const { BrowserWindow } = require('@electron/remote')
```
> 注意：由于这需要通过 npm 使用模块而不是内置模块，因此如果您remote从沙盒进程中使用，则需要适当配置您的打包器以打包@electron/remote 预加载脚本中的代码。当然，`使用@electron/remote会使沙箱的效率大大降低。`

> 注意：在 中`electron >= 14.0.0`，您必须使用新的`enable`API 来`WebContents`分别为每个所需的启用远程模块：require("@electron/remote/main").enable(webContents


在 中electron < 14.0.0，@electron/remote尊重WebPreferences的enableRemoteModule 值。您必须传递`{ webPreferences: { enableRemoteModule: true } }`给BrowserWindow应该被授予使用权限 的构造函数@electron/remote。




