# async/await 模式
再次升级 ...

作为 `ES` 规范中的定义，是 `Generator` 异步流程控制的升级版本，让本身(自带执行器 `Generator` 的语法糖)支持，不再需要像 `co` 这样的执行器，`node.js v7.6.0` 开始支持，同理基于 `Promise`，更加完美，更加简洁，升级 `lts 8` 版本，解放码农。
## 语法使用
```js
function readFile(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, 'utf8', (err, res) => {
      if (err) {
        reject(err)
      } else {
        resolve(res)
      }
    })
  })
}

(async function() {
  // await 返回的Promise 让后续逻辑在Promise 执行完成
  let res = await readFile('./test.txt')

  // ...
  console.log(res)
})().catch(e => {
  // return Promise
})

// 并发
(async function() {
  let [t1, t2] = await Promise.all([
    readFile('./test.txt'),
    readFile('./test.txt')
  ])

  // ...
  
  // 或者写成，下面同样是并发，不过语法有点怪而已，因为Promise对象被定义的时候已经被执行了
  let pr1 = readFile('./test.txt')
  let pr2 = readFile('./test.txt')
  await pr1
  await pr2
})().catch(e => {

})

// 使用 async/await 合理语意下，执行 reduce each map 变得简单
```
### 实现原理
`async/await` 的实现，相当于是 `Generator` 的内封装执行器，所以 `Generator` 的异步操作和错误处理的写法，在 `async/await` 里都受用
```js
async function foo(...args) {

}

// 等同于

function foo(...args) {
	// 返回Promise
	return co(function*() {
	
	})
	
	// 定义执行器
	function co(gen) {
	}
}
```

## 总结
`async/await` 的用法与 `generator` 差不多，只是不需要像 `co` 这样的执行器了，只用记住 `await` 执行封装好的 `Promise` 对象即可, `async` 函数本身执行完成返回 `Promise`
```js
// format 返回值
app.use(async (ctx, next) => {
	try {
		await next()
		ctx.body = {
			status: '200',
			msg: 'ok',
			result: ctx.body
		}
	} catch(e) {
		// handle 中间件的错误
		ctx.body = {
			status: '500',
			msg: e
		}
	}
})

app.use(async (ctx) => {
	let res = await db.find('condition')
	ctx.body = res
})

// 或者不要try ... catch ...
await next().catch(e => {
	// 效果一样
})
```
### async 后话
前面说的：目前 `next` 为同步函数，有提案让 `next` 也为 `Promise`, 异步遍历器
```js
// next 异步，则需要异步Generator函数
async function* gen() {
	yield 'index'
}

let g = gen()

// 而不是 g.next().value
g.next().then(value => {
	// value === 'index'
})

// 后话 还在提案中 目的是让同步 Generator 和异步 Generator 表现形式一致
async function* readLines(path) {
  let file = await fileOpen(path);

  try {
    while (!file.EOF) {
      yield await file.readLine(); // yield 后为 await 执行的 Promise ，不再需要手动封装Promise
    }
  } finally {
    await file.close();
  }
}

// 执行 用 for await ... of 来遍历异步
(async function () {
  for await (const line of readLines(filePath)) {
    console.log(line);
  }
})()
```




