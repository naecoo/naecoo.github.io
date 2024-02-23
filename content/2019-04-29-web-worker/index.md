+++
title = "让你的函数运行在web worker中"
date = "2019-04-27"
author = "naeco"
[taxonomies]
tags = ["javascript", "html"]
+++

### 前言
我们都知道JavaScript在浏览器中执行采用的是单线程模型，也就是说，在同一时间下所有的任务都只能在一个线程上完成，只有上一件任务完成才能开始下一件任务。如果javascript的代码计算量太大，执行会耗费很长的时间，会影响了其他任务的执行，严重的还可能阻塞UI线程的渲染，导致页面出现卡顿等情况。而且现在很多的cpu都是多核的，单线程执行会浪费很多cpu的性能。所以浏览器厂商提供的`web worker`的接口，就是为了提供让`javascript`代码运行在多个线程的环境。`web worker`的工作线程和主线程是分开的，两者互不干扰，相互间通过事件接口进行通信。不过`web worker`提供的接口太原始了，不是很方便我们使用，每次实例化`worker`之后都要预定义单独的`javascript`脚本文件，而且还要单独维护一套通信的方法。
所以我们得想一个办法让`worker`用起来优雅顺手一点，最好是提供一个接口可以像函数一样调用，比如像这样：
```javascript
  const work = someWorker()
  work.add(function count(n){
    return n + 1
  })
  work.count(1).then(res => {
    console.log(res) // 2
  })
```
这样看上去是不是比较直观一点，配合`async`和`await`用起来就非常优雅了，完全屏蔽了js主线程和`worker`之间通信的细节。
### 实现方法
其实具体通信的解决方法很简单，我们只需要在函数调用的时候，`postMessage`要调用的函数和参数到`worker`里面去，再监听`worker`返回的结果。`worker`内部也是一样的道理，监听主线程传过来的消息，再执行相应的函数，用`postMessage`返回执行结果。
```javascript
// js主线程
function invoke (method = '', params = []) {
  const promise = new Promise((resolve, reject)=> {
    worker.onmessage = (e) => {
      resolve(JSON.parse(e.data))
    }
    worker.onerror = (e) => {
      reject(e)
    }
  })
  worker.postMessage(JSON.stringify({
    method,
    params
  }))
  return promise
}
```
```javascript
// worker
self.onmessage = (e) => {
  const {method, params} = JSON.parse(e).data
  const result = self[method].apply(null, params)
  postMessage(JSON.stringify(result))
}
// self是worker全局环境的引用，和window差不多
// 所以调用self的方法就是调用全局环境下注册的方法
```
这样我们就实现了一个简单的`worker`通信模型，只需要在传入`worker`的js脚本中提前定义好函数，就可以在主线程通过`invoke`调用函数了。但这和我们的想法还是有点不一样，我们的模型的可以动态地往worker中添加函数，而且函数可以定义在主线程中，这样可以获得更好的灵活性和可维护性。
那么问题来了，实例化`worker`需要传入js文件的地址，而且这个地址不能是`file://`开头的，意味着不能访问本地的文件，所以worker的脚本必须加载至网络。那么有没有一种好的方法可以动态生成js代码片段，而且能够包装成`worker`可以接受的类型呢？
其实是有的，浏览器厂商提供了一个`URL`的对象，这个对象有一个[`create​ObjectURL`](https://developer.mozilla.org/zh-CN/docs/Web/API/URL/createObjectURL)方法，这个方法可以接受一个二进制对象生成URL，所以我们还需要`Blob`类来生成二进制数据，我们的问题就可以完美解决了。
```javascript
let funcStr = ''
// 我们把函数名和函数引用以key-value的方式用Map储存起来
for (const [name, func] of methods.entries()) {
  let str = ''
  if (isArrowFunc(func)) {
    str = `;var ${name} = ${Function.prototype.toString.call(func)}`
  } else {
    str = `;${Function.prototype.toString.call(func)}`
  }
  funcStr += str
}

const code = `${code};\n${worker_scheduler}`
const url = URL.createObjectURL(new Blob([code]))
const worker = new Worker(url)
```
其实原理也很简单，我们通过`Function.toString`这个方法得到函数的定义，相当于把函数定义复制到了`worker`脚本。这里需要提醒一下的是，es6箭头函数的函数定义和普通的函数有一定的区别，我们需要分别处理。
```javascript
const fn1 = () => {}
function fn2 () {}
Function.prototype.toString.call(fn1)
// () => {}
Function.prototype.toString.call(fn2)
// function fn2 () {}
```
我们可以看到箭头函数的定义没有定义的名字，不过我们可以通过`Function.name`获取到函数定义名。
```
fn1.name //  "fn1"
fn2.name //   "fn2"
```
接下来的东西都很简单了，我们自己在内部维护这样的一套机制，只需要对外暴露`add`和`invoke`两个接口就可以让主线程定义的函数跑在`worker`当中了。
所以我顺手实现了一个库[funcwork](https://github.com/naecoo/funcwork)，内部实现代码只要一百多行，对外暴露了3个方法，用起来很方便。
```javascript
import funcwork form 'funcwork'

const { add, invoke, terminate } = funcwork()

function sayName (name) {
  return `Hello ${name}!`
}

const sayHi () {
  return 'Hi!'
}

async function requestInfo (url, id) {
    return fetch(url, {id})
}

add(sayName, sayHi)

await invoke('sayName', ['naeco'])    // Hello naeco!
await invoke('sayHi')                // Hi!
await invoke('requestInfo', ['api/getUserInfo', 'xxx123456']) //  user info...

// 不用的时候记得销毁
terminate()
```
大家觉得不错的可以顺手给个star😂😂😂
### 后续
其实web worker这个东西出现了也挺久了，现在浏览器支持度已经很不错了，但是我发现实际项目还是很少人用到。个人认为主要原因有两个：
1. 接口不友好
2. 使用场景有限

针对第一点我们可以自己进行封装，可以让`web worker`用起来像`promise`一样顺手。第二点要看我们具体的业务场景了，一些计算量比较大的工作可以尝试交给`web worker`，比如`canvas`和图片的计算，服务器轮询和上传文件等等场景。