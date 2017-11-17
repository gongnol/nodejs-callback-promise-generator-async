# Promise/A 设定

Promise 将回调嵌套转换成了链式操作，形象上，更符合人脑逻辑了。

Promise/A 作为 `callback` 的升级方案，后面`ES`的升级方案都是基于 Promise，神奇的Promise。由于 nodejs 底层的异步逻辑，在js层面上是通过 `event.on/ event.emit` 来建立 `i/o` 调用和收到 `i/o` 返回值的，这里还是函数回调，需要通过 `Promise` 处理这个异步逻辑。
```javascript
// Promise 三种状态 pending(等待resolve) fulfilled(完成resolve) reject(拒绝catch)
function readFile(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, 'utf8', function(err, res) {
      if (err) {
        reject(err)
      } else {
        resolve(res)
      }
    })
  })
}

readFile('./test.txt').then(content => {
  console.log(content)
}).catch(e => {
  console.error(e)
})
```

## 语法使用
Promise对象将回调里的错误标记为`reject`态，通过外部`.catch()`方法捕获。正常为 `resolve` 态，传递给 `.then()` 方法捕获，将原来回调的嵌套逻辑，改为了链条式操作，让肉眼感觉出了一条一条同步的感觉
```javascript
// 串行链
readFile('./test_1.txt').then(() => {
  return readFile('./test_2.txt') // 可以将两个返回值为Promise的函数，通过链路上return接上，然后统一在catch中处理错误
}).then(() => {
  return 1 // 返回值如果不是Promise 会被封装成Promise
}).catch(e => {

})
```
```js
// 并行链
Promise.all([
  readFile('./test_2.txt'),
  readFile('./test_2.txt')
]).then((d) => {
  // d array 顺序与all里Promise对象执行的顺序一致
}).catch(e => {
  // 有一个Promise标记为reject状态就会在 catch 回调里执行，但是其他异步逻辑还会继续走，比如数据库插入操作
})
```

```js
// 争夺型
Promise.race([
  // 只有有一个Promise 标记为resolve，则整体标记为resolve，走then回调
]).then()

// 额外特性 bluebird 实现
Promise.reduce Promise.map // ... 
```

**注意** 在 `Promise` 实例化对象来封装异步操作即 `new` 时，异步操作已经在执行了，Promise 只是保证了异步操作的执行态，来保证后续逻辑在异步操作完成后执行。
```js
// 一些技巧，比如套入多个 Promise 对象。
var o = [1, 2, 3]
var prs = o.map((item) => {
  // 每段map执行异步逻辑，返回封装的Promise对象
  return new Promise(resolve => {
    setTimeout(() => {
      resolve(i)
    }, 1000)
  })
})

// 用all来装填，加载好的Promise对象数组，保证后续逻辑在前者全部完成
Promise.all(prs).then(d => {
  console.log(d) // [1, 2, 3]
})

```

## Polyfill 实现
当js执行器 `Promise` 还未支持时候，需要使用 `callback` 来实现 `polyfill` 。 

现在 `nodejs` 里支持的 `Promise` 理论上来说直接在编译中就支持了 `Promise` 的实现，而不是用 `callback` 封装，否则封装里执行代码变多，运行性能会变差。

#### 一段简单的单链Polyfill
`es-promise`库，原理上的简单实现
```js
// 实现效果
var o = new Promise(function(resolve) {
  resolve('1')
})
o.then(function(arg) {
  console.log(arg)
})

// 异步操作的封装
function sleep() {
  return new Promise(function(resolve, reject) {
    setTimeout(function() {
      resolve()
    }, 1000)
  })
}
sleep().then(function() {
  // complete at 1000ms
}).catch(function(e) {
  // 错误处理
})
```

```js
// 构造函数
function Promise(callback) {
  var self = this
  var scheduleFlush = process.nextTick // 使用nextTick强制 event-loop 下次执行js，达到异步执行效果，防止 resolve 同步函数

  // resolve 函数
  function onFulfillment (arg) {
    scheduleFlush(function() {
      try {
        // 捕获 .then() 回调执行中 throw 的错误
        self._complete(arg)
      } catch (e) {
        // 执行catch
        self._error(e)
      }
    })
  }
  
  // reject 函数
  function onRejection (e) {
    scheduleFlush(function() {
      self._error(e)
    })
  }

  callback(onFulfillment, onRejection)
}

Promise.prototype.then = function(callback) {
  // resolve态 回调
  this._complete = callback
  return this // 这里为了单点链式 使用 return this。 实际中不是，实际是多点队列数组, 每次返回一个新的 Promise，对应减少队列元素，蛮复杂的
}

Promise.prototype.catch = function(callback) {
  // 定义错误处理
  this._error = callback
}
```
#### 一段简单的并行Polyfill
```js
Promise.all = function(tasks) {
	var quit = 0 // 定义控制变量
	var result = [] // 存储返回值
	
	return new Promise(function(resolve) {
		for (var i = 0; i < tasks.length; i++) {
			// IIFE 构建块作用域
			(function(i) {
				quit++ // 执行+1
				tasks[i].then(function(r) {
					quit-- // 执行完成-1
					result[i] = r // 返回值与执行位置一致
					if (quit <= 0) {
						// 控制变量监控全部回调完成
						resolve(result)
					}
				})
				
			})(i)
		}
	})
}

Promise.all([
  sleep('S'),
  sleep('T')
]).then(d => {
  console.log(d) // ['S', 'T']
})

function sleep(param) {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve(param)
    }, 1000)
  })
}
```

## 题外话
值得一提: `ES` 规范中 `Promise` 构造函数里 resolve 或 reject 方法只有第一次执行有效，多次调用失效，即 `Promise` 状态一旦改变则不能再变。
```js
const promise = new Promise((resolve, reject) => {
  resolve('success1')
  reject('error')
  resolve('success2')
})

promise
  .then((res) => {
    console.log('then: ', res) // 只会执行第一次 resolve 回调 success1
  })
  .catch((err) => {
    console.log('catch: ', err)
  })
```

[下一章: Generator执行](Generator执行.md)