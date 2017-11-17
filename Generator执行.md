# Generator 执行

流程再次进化，去除链式效果，出现肉眼上的同步函数

Generator 星函数 `node v4.8` 加入。结合 Promise 可用于美化异步回调

## 简单使用
```js
function* gen() {
  yield 1
  yield 2
  yield 3
}
var g = gen()
// g.next() // return { done: boolean, value: 1 } 每一段 next 都会抛出 对应 yield 语法里的值
// 每执行一次 g.next() 相当于将函数体里的执行位置移至下一次 yield
// 如果执行到return或者无yield后，done值为true

for (value of g) {
  console.log(value) // 1, 2, 3
}

var g2 = gen()
console.log(...g2) // 1 2 3 迭代器的使用, g2有Symbol.iterator属性

// g2[Symbol.iterator] = function* () {} 后话。 (for ... of array ) (...array) 对于 js 数组对象可以实现数组遍历，因为数组对象实现了Symbol.iterator属性 Array.prototype[@@iterator]


function gen3() {
  yield* gen()
  yield 4
}
var g3 = gen3()
// g3.next() // 会优先进入gen来对应执行 next
for (value of g3) {
  console.log(value) // 1,2,3,4
}
```

## 为什么在结合Promise下，能用于控制异步回调流程
`generaotr`函数执行之后，返回一个遍历器，规定了一种协议：`g.next()`执行结果为 `{ done: boolean, value: {...} }`，指向遍历位置所处的 `yield || return ...` 位置 ，`value` 表示当前遍历位置的值，`done` 表示当前遍历是否结束。`next` 函数执行如果传递值的话 `g.next('something')`，会生成上一个 `yield` 表达式的返回值。所以，如果这样操作，`g.next().value.then(res => g.next(res))`，则星函数内部可以写成 `var res = yield Promise.resolve('something')`，出现看似同步的函数执行起来。

```js
function *gen() {
  let res = yield Promise.resolve(0)
  res = yield Promise.resolve(res)
  res = yield Promise.resolve(res)
  res = yield Promise.resolve(res)
  return res
}


// 执行器 co 的 Promise resolve reject 态下的简单实现
function co(gen) {
  let g = gen()

  return new Promise((resolve, reject) => {
    next(g.next())

    function next(ret) {
    	// 判断完毕
      if (ret.done) return resolve(ret.value)
      ret.value.then(function (res) {
        ret = g.next(res)
        // 递归执行自己
        next(ret)
      }, function(err) {
      	// yield Promise.reject reject 态处理
      	try {
      		ret = g.throw(err)
      		next(ret)
      	} catch(e) {
      		reject(e)
      	}
      })
    }
  })
}

co(gen).then(d => { })
```


### 看似同步的错误捕获 try ... catch ...

`Promise.reject` 态的错误捕获。`Generator` 在迭代器执行时, 通过判断 `g.next().value` `Promise` 态为 `reject`， 执行 `g.throw(err)` 创建一个 `Error` 抛出在上一次 `yield` 表达式的位置处，除此之外和 `g.next()` 一样，将迭代位置下移。因此可以通过在内部函数 try ... catch ... 指定 `yield` 执行位置，捕获那段`Promise` 的 `reject`

原理如下
```js
function *gen() {
  try {
    yield 1
  } catch (e) {
    console.error(e)
  }
}
var g = gen()
g.next() // -> 执行到 yield 1
g.throw('something error') // -> 使得 yield 1 部分抛出错误，被 try {...} catch {} 补获
```

所以 Koa 在处理中间件或者路由时可以这样处理
```js
// co 和 koa 都实现了这样的逻辑
app.use(function* (next) {
  // next 为下一个中间件函数
  try {
    yield next
  } catch (e) {
    this.body = {
      error: '500' || '401' || '400'
    }
  }
})

app.use(function* () {
  yield asyncFn_1()
  yield asyncFn_2()
})
```

执行器错误捕获原理
```js
// co 处理reject态下
function *gen() {
  try {
    return Promise.reject('error')
  } catch (e) {
    console.error(e)
  }
}

// 错误的核心处理部分
function co(gen) {
  return new Promise((resolve, reject) => {
    let g = gen()

    g.next().then(() => {

    }).catch(e => {
      // 对应指定的迭代位置抛出生成的捕获的 reject 态
      // 同时 try ... catch ... 自身，防止指定位置处，没有 try ... catch.. 捕获，自身捕获
      // 通过总 reject 抛出外部的 .catch() 回调
      try {
        g.throw(e)    
      } catch(e) {
        reject(e)
      }
    })
  })
}

co(gen).catch(e => { })
```
目前 `next` 为同步函数，有提案让 `next` 也为 `Promise`, 异步遍历器（后话）



## 总结
co库 与 koa 对 Generator 的处理, 核心就是上述 resolve 态，只用记住自己封装 Promise 或者直接使用 function* () {} (co库的额外特性，会递归的执行自己，包装成 Promise)

```js
function asyncFn() {
  return new Promise(resolve => {
    setTimeout(resolve, 1000)
  })
}

app.use(function* () {
  yield asyncFn()
  console.log('next')
})
```

并行，与 Promise.all 一样的逻辑, 见 co , koa 的处理

```js
// return Promise
function asyncFn() {
  return new Promise(resolve => {
    setTimeout(resolve, 1000)
  })
}

console.time()
co(function* () {
  yield [
    asyncFn(),
    asyncFn()
  ]

  // 或者(推荐)
  yield Promise.all([
    asyncFn(),
    asyncFn()
  ])

  console.timeEnd() // 2000ms
})

// yield* 在 co koa 中 没什么用
```

## 题外话
值得一提，使用 `Generator` 函数和后续的 `async` 函数对于轮询爬虫，控制接口访问顺序和延缓访问蛮好用的
```js
const co = require('co')

function sleep(T) {
  return new Promise(resolve => {
    setTimeout(resolve, T)
  })
}

const uri = ['http://baidu.com', 'http://baidu.com', 'http://baidu.com']

co(function* () {
  for (let i = 0; i < uri.length; i++) {
    try {
    	yield request(uri[i])
    } catch (e) { }
    // 串型访问，保证每次间隔1000s
    yield sleep(1000)
  }
}).catch(e => {
  console.log(e)
})
```

[下一章: async-await模式](async-await模式.md)