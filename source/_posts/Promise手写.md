---
title: Promise手写
top: true
cover: false
toc: true
mathjax: true
date: 2022-05-13 10:11:03
password:
summary:
tags:
	- promise
	- javascript
categories: Promise
---

## 什么是 Promise

- 语法上：`promise` 是一个构造函数，返回一个带有状态的对象
- 功能上：`promise` 用于解决异步函数并根据结果做出不同的应对
- 结果上：`promise` 是一个带有 `then` 方法的对象(js 中函数也是对象)

---

## 为什么要使用 promise

前端最令人头疼的就是处理异步请求

```javascript
function load() {
	$.ajax({
		url: '/xxx/xxx'
		data:{},
		success:function(res) {
			init(res, function(res) {
				render(res, function(res) {
					...
				})
			})
		}
	})
}
```

代码层级多，可读性差且难以维护，形成回调地狱。

有了 promise，我们可以用同步的操作流程写异步操作，解决的了层层嵌套的回调函数的困扰。

```javascript
new Promise(function (resolve, reject) {
  // 一些处理逻辑
  resolve('成功')
  // or
  reject('失败')
})
  .then(
    res => {
      console.log(res)
    } // 成功
  )
  .catch(err => {
    console.log(err)
  }) //失败
```

当然 `promise` 也有缺点

- 无法取消 promise，一旦新建就会立即执行，无法中途取消
- 如果不设置回调函数，无法抛出 Promise 内部错误到外部
- 当处于 Pending 状态时，无法得知目前运行的状态，是刚开始还是快结束

## promise 的状态

`promise`有以下三种状态

```javascript
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'
```

状态只能由`pending`向`fulfilled`或`rejected`转变，且只有在执行环境堆栈包含平台代码时转变一次，成为状态凝固，并保存一个参数表明结果。

```javascript
this.value = value // fulfilled状态，保存终值
this.reason = reason // rejected状态，保存据因
```

promise 构造函数接受一个函数作为参数，我们称该函数为`executor`，待`promise`执行时，会向`executor`函数传入两个参数分别为`resolve`和`reject`，它们只做 3 件事：

- 改变`promise`状态
- 保存`value/reason`结果
- 执行`onFulfilled/onRejected`回调函数

其中第三条即为`then`方法中配置的回调函数，这里暂先不讨论，先看前两条，只需要两行代码即可

```javascript
this.state = state
this.value = value / this.reason = reason
```

我们先手撸一个简单的构造函数

```javascript
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'
class promise {
  constructor(executor) {
    this.state = PENDING
    this.value = null
    this.reason = null

    this.onFulfilledCallbacks = [] // 成功回调队列
    this.onRejectedCallbacks = [] // 失败回调队列
    // 定义 resolve 函数
    // 这里使用箭头函数以解决 this 的指向
    const resolve = value => {
      // 保证状态只能改变一次
      if (this.state === PENDING) {
        this.state = FULFILLED
        this.value = value
      }
    }
    const reject = reason => {
      // 保证状态只能改变一次
      if (this.state === PENDING) {
        this.state = REJECTED
        this.reason = reason
      }
    }

    // executor函数会出现异常，需要铺货并调用rejext函数表示执行失败
    try {
      executor(resolve, reject)
    } catch (e) {
      reject(e)
    }
  }
}
```

看上去还不错，大概的流程已经完成了。还记得之前说过，状态的改变是出于主线程空闲时，这里使用`setTimeout`来模拟，以及`resolve/reject`还剩下第三件事，现在让我们一起完善它吧

```javascript
const resolve = value => {
  // setTimeout模拟
  // 注意即便是判断状态是否为pending 也是要在主线程空闲时执行

  setTimeout(() => {
    if (this.state === PENDING) {
      this.state = FULFILLED
      this.value = value
      // 若是使用foreEach回调函数有可能不按顺序执行
      this.onFulfilledCallbacks.map(cb => cb(this.value))
    }
  })
}

// reject同上
```

好啦，一个完整的构造及函数就写完了

接下来是重头戏`then`方法，`then`方法接受两个函数参数，分别是`onFulfilled/onRejected`，用来配置 promise 状态改变后的回调函数。
其有两个重点：

1. 返回一个**promise2**，以实现链式调用

   - 其中 promise2 的状态必须要凝固
   - 通过 resolvePromise 函数以及 onFulfilled/onRejected 的返回值来实现 promise2 的状态凝固

2. 监听或执行对应的 onFulfilled/onRejected 的回调函数

   - 若是执行则需放入 event-loop
   - 监听只需推入回调函数数组中

上述的`resolvePromise`我们先不理会，只要知道它是用来决定`promise2`的状态即可。

首先，`then` 需要返回一个`promise2`：

```javascript
then(onFulfilled,onRejected){
	let promise2

	return promise2 = new Promise((resolve,reject)=>{

	})
}
```

其次 `then`函数的目的是配置或执行对应 onFulfilled/onRejected 回调函数：

```javascript
then(onFulfilled,onRejected){
	let promise2

	return promise2 = new Promise((resolve,reject)=>{
		// 将回调函数配置好并推入对应callback数组中
		this.onFulfilledCallbacks.push(value => {
			// 配置第一步：执行callback函数，并保存返回值x
			let x = onFulfilled(value)
			// 配置第二步：通过resolvePromise决定resolvePromise函数决定promise2的状态
			resolvePromise(promise2,x,resolve,reject)
		})
		// onRejected 同上
		this.onRejectedCallbacks.push(value => {
			let x = onRejected(value)
			resolvePromise(promise2,x,resolve,reject)
		})
	})
}
```

在这里可以大概了解到`resolvePromise`是如何改变`promise2`状态的，它接受`promise2`的`resolve/reject` 由于箭头函数的原因，`resolve/reject`的`this`指向依旧指向`promise2`，从而可以通过 resolvePromise 来改变状态。
万一`onFulfilled/onRejected`出错怎么办？我们需要将他捕获并将 promise2 的状态改为`rejected`，我们将代码再做修改：

```javascript
then(onFulfilled,onRejected){
	let promise2

	return promise2 = new Promise((resolve,reject)=>{
		// 将回调函数配置好并推入对应callback数组中
		this.onFulfilledCallbacks.push(value => {
			try{
				let x = onFulfilled(value)
				resolvePromise(promise2,x,resolve,reject)
			}catch(e){
				reject(e)
			}

		})
		// onRejected 同上
		this.onRejectedCallbacks.push(value => {
			try{
				let x = onRejected(value)
				resolvePromise(promise2,x,resolve,reject)
			}catch(e){
				reject(e)
			}
		})
	})
}
```

如果调用 `then` 方法的是已经状态凝固的 `promise` 呢，也要推入 `callbacks` 数组吗？答案当然不是，而是直接将配置好的 `onFulfilled/onRejected` 扔入 `event-loop` 中，就不劳烦 `resolve/reject` 了：

```javascript
then(onFulfilled, onRejected){
    // fulfilled 状态，将配置好的回调函数扔入 event-loop
    if (this.state === FULFILLED) {
      return (promise2 = new MyPromise((resolve, reject) => {
        setTimeout(() => {
          try {
            let x = onFulfilled(this.value)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }
    // rejected 状态同上
    if (this.state === REJECTED) {
      return (promise2 = new MyPromise((resolve, reject) => {
        setTimeout(() => {
          try {
            let x = onRejected(this.reason)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }
    // pending 状态则交由 resolve/reject 来决定
    if (this.state === PENDING) {
      return (promise2 = new MyPromise((resolve, reject) => {
        this.onFulfilledCallbacks.push(value => {
          try {
            let x = onFulfilled(value)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })

        this.onRejectedCallbacks.push(reason => {
          try {
            let x = onRejected(reason)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }
  }
}
```

看上去完美了，不过还差一件小事，假如 promise 使用者不按套路出牌，传入的 `onFulfilled/onRejected` 不是一个函数怎么办？这里我们就直接将之作为返回值直接返回：

```javascript
then(onFulfilled, onRejected){
    let promise2
    // 确保 onFulfilled/onRejected 为函数
    // 若非函数，则转换为函数并且返回值为自身
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value
    onRejected = typeof onRejected === 'function' ? onRejected : reason => {
      throw reason
    }

    if (this.state === FULFILLED) {
      return (promise2 = new MyPromise((resolve, reject) => {
        setTimeout(() => {
          try {
            let x = onFulfilled(this.value)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }

    if (this.state === REJECTED) {
      return (promise2 = new MyPromise((resolve, reject) => {
        setTimeout(() => {
          try {
            let x = onRejected(this.reason)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }

    if (this.state === PENDING) {
      return (promise2 = new MyPromise((resolve, reject) => {
        this.onFulfilledCallbacks.push(value => {
          try {
            let x = onFulfilled(value)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })

        this.onRejectedCallbacks.push(reason => {
          try {
            let x = onRejected(reason)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }
  }
}
```

大功告成！
最后只剩下一个 resolvePromise 方法，先介绍一下它的功能：根据回调的返回值 x 决定 promise2 的最终状态：

- 如果 x 为 thenable 对象，即带 then 方法的对象
  - 有，因其不一定符合 promise 得标准，多做一些准备
  - 无，当作普通值执行
  - 使用 called 变量使得其状态改变只能发生一次
  - 监听异常
  - 递归调用 resolvePromise 以防止出现套娃
  - 如果 x 为 promise，则递归调用，直到返回值为普通值为止
  - 如果 x 为函数或对象，判断其有无 then 方法
- x 为普通值
  - 直接返回

```javascript
function resolvePromise(promise2, x, resolve, reject) {
  // 先从 x 为 thenable 对象开始
  // 如果 x === promise2 需要抛出循环引用错误，否则会死循环
  if (x === promise2) {
    reject(newTypeError('循环引用'))
  }
  // 如果 x 就是 promise
  // 根据 x 的状态来决定 promise2 的状态
  if (x instanceof MyPromise) {
    // x 状态为 PENDING 时
    // 当 x 被 resolve 会调用新的 resolvePromise
    // 因为怕 resolve 保存的终值还是 promise 继续套娃
    // 所以一定要递归调用 resolvePromise 保证最终返回的一定是普通值
    // 失败直接调用 reject 即可
    if (x.state === PENDING) {
      x.then(
        y => {
          resolvePromise(promise2, y, resolve, reject)
        },
        r => {
          reject(r)
        }
      )
    } else {
      // x 状态凝固，直接配置即可
      // 不过这里有个疑问
      // 如果之前 resolve 保存的终值还是 promise 呢
      // 该怎样预防这一问题，后续将会讲到
      x.then(resolve, reject)
    }
  }
}
```

现在把应对 `x` 的值为 `promise` 的代码书写完毕，但这还不够，我们要面对的不只是 `promise，而是一个` `thenable` 对象，所以还要继续判断：

```javascript
function resolvePromise(promise2, x, resolve, reject) {
  if (x === promise2) {
    reject(newTypeError('循环引用'))
  }
  if (x instanceof MyPromise) {
     // 前面的代码不再赘述
  } elseif (x && (typeof x === 'function' || typeof x === 'object')) {
      // 因为不一定是规范的 promise 对象
      // 我们需要保证状态的改变只发生一次
      // 加入一个 called 变量来加锁
      let called = false
      // 还是因为不一定是规范的 promise 对象
      // 需要保证运行时异常能够被捕获
      try {
          // 注意，前面不加 try/catch
          // 仅仅下面这一行代码也有可能会报错而无法被捕获
          let then = x.then
          // 假如 x.then 存在并为函数
          if (typeof then === 'function') {
              // 使用 call 方法保证 then 的调用对象为 x
              then.call{
                  x,
                  y => {
                      // 假如状态凝固便不再执行
                      if (called) return
                      called = true
                      // 防止出现 resolve 保存 promise 的情况
                      resolvePromise(promise2, y, resolve, reject)
                  },
                  r => {
                      // 同上
                      if (called) return
                      called = true
                      reject(r)
                  }
              }
         } else {
            // 如果 x.then 不是函数
            // 即为普通值，直接 resolve 就好
            resolve(x)
        }
       } catch (e) {
          // 若调用一个不正规的 thenalbe 对象出错
          // 抛出异常
          // 这里要注意，这里出现错误很有可能是执行了 x.then 方法，而之前也说过，其不一定正规，可能状态已经凝固，需要多加一重保险
          if (called) return
          called = true
          reject(e)
        }
     } else {
       // 不是 thenable 对象，那就是普通值
       // 直接 resolve
       resolve(x)
    }
}
```

一套行云流水的代码写下来，我们的 `promise` 就完成了，不过还记得之前代码里留了个疑问吗？当 `x` 为 `promise` 且状态凝固时，如果确定它保存的终值的不是 `promise` 呢？其实只要最开始的 `resolve` 函数多加一重判断即可：

```javascript
const resolve = value => {
  if (value instanceof MyPromise) {
    return value.then(resolve, reject)
  }

  setTimeout(() => {
    if (this.state === PENDING) {
      this.state = FULFILLED
      this.value = value
      this.onFulfilledCallbacks.map(cb => cb(this.value))
    }
  })
}
```
再次防止套娃！

好啦，也许你会问，我怎么知道这个手写的 promise 就一定是正确的呢？接下来将一步步带你验证！

首先找到一个空文件夹，在命令行输入：
```bash
npm init -y

// 下载 promise 测试工具
npm install promises-aplus-tests -D
```

新建 `promise.js` 文件，并将你实现的 `promise` 复制于此，并在下方加入一下代码：

```javascript
MyPromise.deferred = function() {
  let defer = {}
  defer.promise = new MyPromise((resolve, reject) => {
    defer.resolve = resolve
    defer.reject = reject
  });
  return defer
}

module.exports = MyPromise
```
再修改 package.json 文件如下：
```json
{
  "name": "promise",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "promises-aplus-tests ./promise.js"
  },
  "devDependencies": {
    "promises-aplus-tests": "^2.1.2"
  }
}
```
最后一步：
```bash
npm run test
```
最后附上完整实现代码
```javascript
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

class MyPromise {
  constructor(executor) {
    this.state = PENDING
    this.value = null
    this.reason = null

    this.onFulfilledCallbacks = []
    this.onRejectedCallbacks = []

    const resolve = value => {
      if (value instanceof MyPromise) {
        return value.then(resolve, reject)
      }

      setTimeout(() => {
        if (this.state === PENDING) {
          this.state = FULFILLED
          this.value = value
          this.onFulfilledCallbacks.map(cb => cb(this.value))
        }
      })
    }

    const reject = reason => {
      setTimeout(() => {
        if (this.state === PENDING) {
          this.state = REJECTED
          this.reason = reason
          this.onRejectedCallbacks.map(cb => cb(this.reason))
        }
      })
    }

    try {
      executor(resolve, reject)
    } catch (e) {
      reject(e)
    }
  }

  then(onFulfilled, onRejected) {
    let promise2

    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value
    onRejected = typeof onRejected === 'function' ? onRejected : reason => {
      throw reason
    }

    if (this.state === FULFILLED) {
      return (promise2 = new MyPromise((resolve, reject) => {
        setTimeout(() => {
          try {
            let x = onFulfilled(this.value)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }

    if (this.state === REJECTED) {
      return (promise2 = new MyPromise((resolve, reject) => {
        setTimeout(() => {
          try {
            let x = onRejected(this.reason)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }

    if (this.state === PENDING) {
      return (promise2 = new MyPromise((resolve, reject) => {
        this.onFulfilledCallbacks.push(value => {
          try {
            let x = onFulfilled(value)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })

        this.onRejectedCallbacks.push(reason => {
          try {
            let x = onRejected(reason)
            resolvePromise(promise2, x, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }
  }
}
function resolvePromise(promise2, x, resolve, reject) {
  if (x === promise2) {
    reject(newTypeError('循环引用'))
  }

  if (x instanceof MyPromise) {
    if (x.state === PENDING) {
      x.then(
        y => {
          resolvePromise(promise2, y, resolve, reject)
        },
        r => {
          reject(r)
        }
      )
    } else {
      x.then(resolve, reject)
    }
  } elseif (x && (typeof x === 'function' || typeof x === 'object')) {
    let called = false
    try {
      let then = x.then
      if (typeof then === 'function') {
        then.call(
          x,
          y => {
            if (called) return
            called = true
            resolvePromise(promise2, y, resolve, reject)
          },
          r => {
            if (called) return
            called = true
            reject(r)
          }
        )
      } else {
        resolve(x)
      }
    } catch (e) {
      if (called) return
      called = true
      reject(e)
    }
  } else {
    resolve(x)
  }
}

MyPromise.deferred = function() {
  let defer = {}
  defer.promise = new MyPromise((resolve, reject) => {
    defer.resolve = resolve
    defer.reject = reject
  });
  return defer
};

module.exports = MyPromise;

```
