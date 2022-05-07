---
title: Context-isolation
top: false
cover: true
toc: true
mathjax: true
date: 2022-05-07 14:19:29
password:
summary:
tags:
	- contextIsolation
	- 隔离
	- node
	- Electron
categories: Electron
---

## contextIsolation 上下文隔离
上下文隔离是一项功能，可确保你的`preload`脚本和electron的内部逻辑都在与您加载到`webContents`，这对于安全目的很重要，因为它有助于防止网站访问Electron内部或您的预加载脚本可以访问强大API。

这意味着`window`您的预加载脚本有权访问的对象实际上与网站有权访问的对象不同。例如，如果您`window.hello = 'wave'`在预加载脚本中设置并启用了上下文隔离，`window.hello`则在网站尝试访问它时将是未定义的。

自 Electron 12 以来，上下文隔离已默认启用，它是所有应用程序的推荐安全设置。

### 迁移
#### 之前：情境隔离已禁用
将预加载脚本中的 API 暴露给渲染器进程中加载的网站是一个常见用例。禁用上下文隔离后，您的预加载脚本将window与渲染器共享一个公共全局对象。然后，您可以将任意属性附加到预加载脚本：
```javascript
	// preload.js
	// preload with contextIsolation disabled
	window.myApi= {
		doAThing:()=>{}
	}
```
`doAThing()`然后就可以在渲染进程中使用该函数：

```javascript
	// render.js
	// use the exposed API in the renderer
	window.myApi.doAThing()
```

#### 之后：上下文启用隔离
Electron中有一个专用模块可以帮助您轻松完成此操作。该`contextBridge`模块可用于将API从预加载脚本的隔离上下文安全的公开到网站运行的上下文。API也可以`window.myAPI`像以前一样从网站访问。
```javascript
	// preload with contextIsolation enabled
	const { contextBridge } = require('electron')
    contextBridge.exposeInMainWorld('myAPI', {
    	doAThing: () => {}
    })
```
```javascript
	// render.js
	// use the exposed API in the renderer
	window.myApi.doAThing()
```
#### 安全注意事项
仅启用contextIsolation和使用contextBridge并不自动意味着您所做的一切都是安全的。例如，这段代码是不安全的。
```javascript
	// ❌ Bad code
    contextBridge.exposeInMainWorld('myAPI', {
    	send: ipcRenderer.send
    })

```
它直接公开了一个强大的 API，没有任何类型的参数过滤。这将允许任何网站发送您不希望成为可能的任意 IPC 消息。公开基于 IPC 的 API 的正确方法是为每个 IPC 消息提供一种方法。
```javascript
    // ✅ Good code
    contextBridge.exposeInMainWorld('myAPI', {
    	loadPreferences: () => ipcRenderer.invoke('load-prefs')
    })
```

