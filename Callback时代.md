# Callback 时代
## 混沌
在 `Promise` 的概念还没普及前，比如ES5，异步编程是回调的形式，所以需要通过函数封装，通过闭包原理，将要处理的结果值往作用域外层传递，所以 `callback` 回调函数模式很盛行 ，比如 `jquery`里`ajax` 函数(后续 `jquery` 支持 `Promise`，甚至 `ajax` 函数执行的参数列表里有参数标示同步执行)
```js
$(function() {
	// 定义请求函数
	function request(uri, callback) {
		$.ajax({
			url: uri,
			method: 'get',
			success: function(data) {
				callback(null, data)
			},
			error: function(xhr, textStatus, errorThrown) {
				callback(errorThrown)
			}
		})
	}
	
	// callback函数，参数列表规定，第一个参数为错误抛出，第二个参数为响应值
	request('http://baidu.com', function (err, data) {
		if (err) {
		// handle error
			return
		}
		// handle after request logic
		console.log(data)
	})
})
```

## 流程写法崩坏，回调黑洞
人们为了能清晰处理 `callback`，而封装出许多**回调流程控制函数**，来控制出现回调黑洞，并行回调，串行回调。在 `nodejs` 编写的 `web` 应用，在处理无相关的 `i/o` 操作里，都应该并行回调，让一次请求快速完成响应，否则响应过久，在高负载下，导致的请求连接堆积，降低服务的负载能力，qps值也会下降(全是`pending`)，当然这不是一个原因(还有`gc`导致js逻辑执行延后，cpu执行打满，无力再执行新的js逻辑等等...)。还有并行回调对于目标地址，也是有成本的，比如请求数据库服务，导致数据库服务压力上升，这时候要考虑队列化并发请求...还有单线程的并发能力很有限，是一段段js逻辑调用底层线程池，这点`golang`这样的`goroutine`很厉害。
```js
// 回调黑洞
db.find('condtion_1', function() {
  db.find('condtion_2', function() {
    db.find('condition_3', function() {
      db.find('condition_4', function() {
        //...//
      })
    })
  })
})
```

## 回调流程控制函数的实现

早期 `callback` 风格的时候，`npm` 社区里有很多优秀的回调流程控制函数库，比如 `async库`, `eventproxy库`，这里简单的介绍这两者源码里的实现

以下为 `async` 异步回调流程控制库的几个方法，原理上的简单实现

#### callback 并发流程控制
定义一个函数 parellel, 对两个没有先后关联异步操作做流程控制，在完成操作后，执行后续逻辑
```js
// 一个异步操作回调封装
function foo(t, callback) {
	return function (callback) {
		setTimeout(function() {
			// 闭包 t 变量
			callback(null, t)
		}, t)
	}
}

parellel([
  foo(500)
  foo(1000)
], function(err, res) {
  // 总计 1000ms 完成
  // 且保证返回值顺序与执行顺序500, 1000对应
  // next... 执行后续逻辑
})

// 函数实现
function parellel(tasks, callback) {
  var result = [] // 返回值存储
  var quit = 0 // 第三方变量控制器(关键)
  var done = false // 判断完成，用于不重复 callback

  for (var i = 0; i < tasks.length; i++) {
    // 注意使用IIFE自执行函数来创建块作用域，来保证闭包里i的值正常传递。
    // 不然就要使用 let i = 0 定义即可
    (function(i) {
      quit++ // 开启一次异步操作
      
      // 执行，传入要执行的回调
      tasks[i].call(null, function(err, res) {
        if (err && !done) {
          // 当一个异步操作有错误，停止回调抛错， done目的只让一次错误抛出，不让总callback被多次执行
          done = true
          // 总回调传出错误
          callback(err)
          return
        }

        // 正常，减掉控制器
        quit--
        result[i] = res // 保存一次的异步的值，按照顺序，即操作的排列与返回值的排列一样
        if (quit <= 0) {
          // 判断控制器全部完成
          done = true
          // 总回调传出执行结果
          callback(null, result)
        }
      })
    })(i)
  }
}
```

#### callback 串行流程控制一: series
定义一个函数 series, 对两个有前后关系的异步操作做串行处理，在串行完成后，执行后续逻辑
```js
var fn1 = foo(500),
var fn2 = foo(1000)
series([
  fn1,
  fn2
], function(err, res) {
  // 第一个 fn1 执行完成后执行第二个 fn2
  // 总时长1500ms
  if (err) {
    return 
  }
  // next 后续逻辑
})

// 函数实现
function series(tasks, callback) {
  var result = [] // 最终结果值，数组顺序与执行逻辑一样
  var len = tasks.length // 总异步任务长度
  let i = 0 // 定义控制变量

  if (!Array.isArray(tasks) || len === 0) {
  	 // 判空
    callback(null, [])
    return
  }

	// 定义每个task的回调函数内容
  function next(err, res) {
    // 异常处理
    if (err) {
      callback(err)
      return
    }
    // 上一个task完成，将结果值传入result
    result.push(res)
    i++ // 控制变量自增

    // 判断控制变量是否已经达到全部异步回调执行完成,如果未完成，递归自身，否则执行总回调，传出最终执行完毕的result
    if (i < len) {
      tasks[i].call(null, next)
    } else {
      callback(null, result)
    }
  }

  // 初始执行 i=0, next为单个task的回调的内容
  tasks[i].call(null, next)
}
```

#### callback 串行流程控制二: reduce
定义一个函数 reduce, 用做异步串行控制，不同于上述，将上一个异步回调的值传入下一个，最终总结抛出，类似 Array.prototype.reduce
```js
// 封装异步操作函数 bar
function bar(num, callback) {
  if (typeof num !== 'number') num = 0
  setTimeout(function() {
    callback(null, ++num)
  }, 1000)
}

reduce([bar, bar, bar], null, function(preValue, currentFunc, callback) {
  // preValue 上一次回调的结果，初始值为第二个参数
  // currentFunc 当前执行的函数
  currentFunc(preValue, function(err, res) {
    // 处理完毕，传递下次
    callback(null, res)
  })
}, function(err, result) {
  // 串行结果传递 最终 0 -> 1 -> 2 -> 3
  // 最终执行时长3000ms
  console.log(result) // 3
})

// 函数实现
function reduce(list, init, task, callback) {
  var quit = 0 // 定义控制变量

  function next(err, result) {
    quit++ // 回调完成 控制变量自增

    // 判断控制变量量为执行完成的次数，如果是完成执行，执行总回调，将结果值传出
    if (quit === list.length) {
      callback(null, result)
    } else {
      // 次数未达标，递归自己，将上次回调的结果最为初始值传入，实现串行传递
      task(result, list[quit], next)
    }
  }

  // 初始执行异步流程的第一个函数, next为回调
  task(init, list[quit], next)
}
```

## 基于 nodejs 的 event 标准库来实现异步回调流程控制

核心还是回调，不过 `event` 做了封装 `on，emit` 来对应订阅/发布，执行回调

`event`标准库实现原理
```js
// 简单实现
var event = {}
var c = {}

event.on = function(channel, emit) {
	c[channel] = emit
}

event.emit = function(channel) {
	if (typeof c[channel] === 'function') {
		c[channel].apply(null, Array.prototype.slice.call(arguments, 1))
	}
}

event.on('foo', function(say) {
	console.log(say)
})

event.emit('foo', 1)  // log 1
```

#### 简单的通过 event 机制来控制异步回调流程并行
以下为 `eventproxy(jacksontian)` 的几个方法原理上的简单实现

```
// 实现效果
// 定义一个自定义的事件构造函数，实例化出一个事件对象
var ep = new evCustom()

// 注册回调完成后执行的函数
ep._$on('c', 'b', function (c, b) {
  // 返回参数顺序与订阅事件顺序一致
  console.log(c)
  console.log(b)

  // 处理后续逻辑
  // next...
})

// 注册错误处理，如果在 callback 里 emit 了错误，_$fail注册的错误处理，将捕获执行后续错误逻辑
ep._$fail(function (err) {
  console.log(err)
  
  // 处理后续错误
  // next err...
})


setTimeout(() => {
  ep.emit('c', 'c')
}, 1000);

setTimeout(() => {
  ep.emit('b', 'b')
}, 1000);
```

```js
var event = require('events')
var util = require('util')

// 自定义事件订阅发布构造函数，继承标准库的所有方法
// 也可以使用 es6 class 即 class evCustom extends event {}
function evCustom() {
	// 用来放置实例化后的事件列表
  this.eventsList = []
	
	// 继承自有属性
  event.call(this)
}

// 自定义并行监听
evCustom.prototype._$on = function () {
  var self = this
  var result = [] // 存储返回值
  
  var len = Object.keys(arguments).length - 1 // 使用传递的事件数量，作为监听第三方变量
  var callback = arguments[len] // 取出最后一位的callback

  function foo(idx) {
    // idx 确保返回位置一样
    return function (res) {
      result[idx] = res
      len-- // 第三方监控变量递减
      if (len <= 0) {
        // 判断第三方变量监控完成调用回调
        callback.apply(null, result)
        removeListener()
      }
    }
  }

  function removeListener() {
    for (var i = 0; i < len; i++) {
      // 错误发生时候移除对应的事件监听
      self.removeAllListeners(arguments[i])
    }
  }

  for (var i = 0; i < len; i++) {
    // 除去callback后, 存储的订阅事件的事件名，用于后续 fail 里的removeListener
    this.eventsList.push(arguments[i])
    this.on(arguments[i], foo(i))
  }
}

// 自定义错误处理监听
evCustom.prototype._$fail = function (callback) {
  var self = this

  if (this.listenerCount('error')) {
    // 判断是否存在错误监听
    return
  }

  this.on('error', function (err) {
    for (var i = 0; i < self.eventsList; i++) {
      // 错误发生时候移除对应的事件监听
      self.removeAllListeners(self.eventsList[i])
    }

    callback(err)
  })
}
// 继承原型部分
// 上述 .call(this) 和这里的 .inherits(...) evCustom 就继承 event 的全部方法
util.inherits(evCustom, event)
```


[下一章: Promise-A设定](Promise-A设定.md)