+++
title = "petite-vue 框架分析"
date = "2022-08-01"
author = "naeco"
[taxonomies]
tags = ["javascript", "vue"]
+++

## petite-vue 介绍

[petite-vue](https://github.com/vuejs/petite-vue) 是基于 DOM 和 [@vue/reactivity]([@vue/reactivity - npm (npmjs.com)](https://www.npmjs.com/package/@vue/reactivity)) 驱动的 MVVM框架，提供了和 Vue 框架相似的语法和响应式模型，由 Vue 作者 Evan You 开发，编译后的大小只有 6kb，十分适合小项目的使用。`petite-vue` 并不是为了替代 Vue，不能用于生产环境，`petite-vue` 的作用是提供一种渐进式增强的框架给开发者使用，无需构建和编译，也不需要复杂的配置文件，只需要一个 `script` 标签，引入 CDN 文件，便能使用该框架来构造我们的视图：

```html
<script src="https://unpkg.com/petite-vue" defer init></script>

<!-- anywhere on the page -->
<div v-scope="{ count: 0 }">
  {{ count }}
  <button @click="count++">inc</button>
</div>
```

怎么样？是不是很简单...



## petite-vue 使用

### 基本用法

```html
<script type="module">
  import { createApp } from 'petite-vue';

  createApp({
    count: 0,
    increment() {
      this.count++;
    },
    decrement() {
      this.count--;
    }
  }).mount('#app');
</script>

<div id="app" v-scope>
  <p>{{ count }}</p>
  <button @click="decrement"> - </button>
  <button @click="increment"> + </button>
</div>
```
语法其实很简单，`createApp` 用于创建一个应用，传入一个对象，可以传入响应式的数据和方法，这里为了简单，并没有Vue 的 data 和 methods 等选项，所有数据都是混在一起的。

然后，在 dom 元素中指定 `v-scope`，表明有 `petite-vue` 控制，`mount` 的时候就会从该节点开始递归解析，包括所有子孙节点，渲染数据和绑定事件。



### 生命周期

`petite-vue` 只支持 mount 和 unmount 生命周期：
```html
<div
  @vue:mounted="console.log('mounted on: ', $el)"
  @vue:unmounted="console.log('unmounted: ', $el)"
>
  {{ msg }}
</div>
```



### 组件

`petite-vue` 的组件和我们所理解的组件不太一样，更多的是逻辑上的封装：
```html
<script type="module">
  import { createApp } from 'petite-vue'

  // Counter 组件
  function Counter(props) {
    return {
      count: props.initialCount ?? 0,
      increment() {
        this.count++
      },
      mounted() {
        console.log(`I'm mounted!`)
      }
    }
  }

  createApp({
    Counter
  }).mount()
</script>

<!-- 使用组件 -->
<div v-scope="Counter({ initialCount: 1 })" @vue:mounted="mounted">
  <p>{{ count }}</p>
  <button @click="increment">increment</button>
</div>
<!-- 使用组件 -->
<div v-scope="Counter({ initialCount: 2 })">
  <p>{{ count }}</p>
  <button @click="increment">increment</button>
</div>
```

组件同样支持模板：
```html
<script type="module">
  import { createApp } from 'petite-vue'

  // Counter 组件
  function Counter(props) {
    return {
      template: '#counter-template'
      count: props.initialCount ?? 0,
      increment() {
        this.count++
      }
    }
  }

  createApp({
    Counter
  }).mount()
</script>

<!-- 定义模板 -->
<template id="counter-template">
  <p>{{ count }}</p>
  <button @click="increment">increment</button>
</template>
<!-- 使用组件 -->
<div v-scope="Counter({ initialCount: 1 })">
<div v-scope="Counter({ initialCount: 2 })">
```



### 指令

`petite-vue` 提供了 v-if、v-show、v-for、v-bind、v-on、v-html、v-model 等指令：
```html
<script type="module" defer>
  import { createApp } from './petite-vue.es.js'
  
  createApp({
    count: 1,
    msg: 'Hello World!',
    htmlMsg: '<b>Hello</b> World!'
  }).mount()
</script>

<div id="app">
  <p>{{ msg }}</p>
  <p v-text="htmlMsg"></p>
  <p v-html="htmlMsg"></p>
  <input type="text" v-bind:value="msg" >
  <p v-if="count > 1">count > 1</p>
  <p v-else>count <= 1</p>
  <input type="number" v-model="count" >
</div>
```

支持自定义指令，可以使用 `directive` 方法配置指令。比如，实现一个 `v-red` 指令，将文本变为红色：
```html
<script type="module" defer>
  import { createApp } from './petite-vue.es.js'
  
  const app = createApp({
    msg: 'Hello World!'
  })
  // 定义指令
  app.directive('red', ({ el, get, effect }) => {
    effect(() => {
      el.innerHTML = `<span style="color: red;">${get()}</span>`
    })
  });

  app.mount()
</script>

<div id="app">
  <p>{{ msg }}</p>
  <p v-red="msg"></p>
</div>
```





## petite-vue 原理
大家是不是很好奇，`petite-vue` 是如何实现的呢，这么多功能，是不是需要很复杂的代码和模块？其实不然，`petite-vue` 源代码不到 1000 行，只依赖 `@vue/reactivity` 一个库，代码实现非常简单。

简而言之，`petite-vue` 就是**基于 Vue3 提供的响应式能力，创建组件的上下文，然后在挂载的时候，直接遍历 DOM 节点，解析DOM节点上的属性和表达式，注册成一系列的指令，使用 Vue3 的 effect API 订阅数据变更，从而驱动视图变更**。

### 响应式
`@vue/reactivity` 是 Vue 3.0 提供的响应式模块，对外暴露了 `ref`、`reactive`、`watch`、`computed` 和 `effect` 等 API。在这些 API 中，比较关键的是 [`effect`](https://github.com/vuejs/core/blob/main/packages/reactivity/src/effect.ts)，这个 API 接受一个函数，当函数执行的时候，会自动收集依赖，当这些依赖更新时，会再次执行函数。watch、computed等API也是基于 effect 实现的，感兴趣的读者可以阅读 vue3 的 [源码](https://github.com/vuejs/core/blob/main/packages/reactivity/src/computed.ts)。

effect 的使用可以参照下面的例子：

```js
import { effect, ref } from '@vue/reactivity' 


const count = ref(0);
const foo = ref('')
effect(() => {
    console.log(count.value);
});
// 0

count.value++;
// 1

foo.value = ''; // 这里就不会影响到effect里面的函数，因为没有收集到这个依赖
```

基于 effect，`petite-vue` 可以感知到数据的变更，从而驱动视图重新进行渲染。



### 模板解析

`petite-vue` 和 vue 最大的不同在于这里，vue 是基于虚拟 DOM 设计的，而 `petite-vue` 没有虚拟 DOM，直接操作 DOM 结构。所以模板解析的逻辑非常简单，但相对的，性能也降低了很多，不支持服务端渲染。

`petite-vue` 解析模板的逻辑也很简单，采用了深度优先遍历的方式遍历DOM节点，逐个解析指令、属性、props和表达式，比方说下面的代码结构：

```html
<div id="app" v-scope>
  <p>{{ count }}</p>
  <button @click="decrement"> - </button>
  <button @click="increment"> + </button>
</div>
```

渲染成DOM结构就是这样子：

![image-20220814230329082](https://image.naeco.top/blog/1660489410197.png)

解析的步骤就是：

1. div#app 节点，识别到v-scope关键词，创建上下文
2. 解析 p 节点，没有属性，解析p的子节点
3. 解析文本节点，识别到表达式，自动绑定v-text指令
4. 解析button节点，识别到click事件，绑定decrement事件
5. 解析文本节点，不是表达式，无需绑定指令
6. 解析button接待你，识别到click事件，绑定increment事件
7. 解析文本节点...



## petite-vue 源码分析

### 创建应用

了解了大概的原理后，就可以分析源代码了，首先看一下创建应用的过程：

```typescript
// /src/index.ts

export { createApp } from './app'
export { nextTick } from './scheduler'
export { reactive } from '@vue/reactivity'

import { createApp } from './app'

const s = document.currentScript
// petite-vue 会自动挂载，如果 script 标签带有 init 属性
// 比如：<script src="https://unpkg.com/petite-vue" init></script>
if (s && s.hasAttribute('init')) {
  createApp().mount()
}

```

```typescript
// /src/app.ts

// ...

// 创建应用，笼统来说，应用就是一个组件
export const createApp = (initialData?: any) => {
  // 创建组件上下文
  const ctx = createContext()

  // 对传入的数据做处理，使其变成响应式的数据
  if (initialData) {
    ctx.scope = reactive(initialData)
    // 绑定方法的上下文
    bindContextMethods(ctx.scope)
  }

  // ...

  // 创建block
  let rootBlocks: Block[]

  // 返回一个对象，对外暴露新建指令、挂载和卸载方法
  return {
    directive(name: string, def?: Directive) {
      if (def) {
        ctx.dirs[name] = def
        return this
      } else {

        return ctx.dirs[name]
      }
    },

    mount(el?: string | Element | null) {
      if (typeof el === 'string') {
        el = document.querySelector(el)
      }

      el = el || document.documentElement
      let roots: Element[]

      // 寻找带有 v-scope 属性的元素
      if (el.hasAttribute('v-scope')) {
        roots = [el]
      } else {
        roots = [...el.querySelectorAll(`[v-scope]`)].filter(
          (root) => !root.matches(`[v-scope] [v-scope]`)
        )
      }
      if (!roots.length) {
        roots = [el]
      }

      // 生成block
      rootBlocks = roots.map((el) => new Block(el, ctx, true))
      return this
    },

    unmount() {
      // 逐个卸载block
      rootBlocks.forEach((block) => block.teardown())
    }
  }
}
```
创建应用的过程非常简单：
1. 创建上下文，初始化响应式数据

2. 调用 mount 方法的时候，生成 block

那么问题又来了，这里的上下文(Context)和 block 到底是啥？

### Context

在 `petite-vue` 中 context 就是组件的上下文：

```typescript
// /src/context.ts

interface Context {
  key?: any // 组件key
  scope: Record<string, any> // 响应式数据和方法
  dirs: Record<string, Directive> // 指令
  blocks: Block[] // blocks
  effect: typeof rawEffect // effect函数
  effects: ReactiveEffectRunner[] // 组件产生的effect函数集合
  cleanups: (() => void)[] // 调用指令后的清理函数，卸载组件的时候会调用
}
```

创建上下文：

```typescript
// /src/context.ts

// 创建上下文
const createContext = (parent?: Context): Context => {
  const ctx: Context = {
    ...parent, // 父组件
    // 如果有父组件，直接引用父组件的数据
    scope: parent ? parent.scope : reactive({}),
    dirs: parent ? parent.dirs : {},
    effects: [],
    blocks: [],
    cleanups: [],
    // 组件effect实际上是 vue effect函数的封装
    effect: (fn) => {
      // v-once 指令处理，渲染后就不会再更新，哪怕响应式数据修改了
      if (inOnce) {
        queueJob(fn)
        return fn as any
      }
      // 调用 vue3 的effect函数
      const e: ReactiveEffectRunner = rawEffect(fn, {
        scheduler: () => queueJob(e)
      })
      ctx.effects.push(e)
      return e
    }
  }
  return ctx
}

```

queueJob就是一个调度的程序，和vue2的机制差不多：

```typescript
// /src/scheduler.ts

let queued = false
const queue: Function[] = []
const p = Promise.resolve()

// 简单nextTick实现，就是创建一个微任务程序
export const nextTick = (fn: () => void) => p.then(fn)

// 如果不存在该任务，添加新的任务到任务队列
// 然后调用 nextTick 清空任务队列
export const queueJob = (job: Function) => {
  if (!queue.includes(job)) queue.push(job)
  if (!queued) {
    queued = true
    nextTick(flushJobs)
  }
}

const flushJobs = () => {
  // 这里和 vue2 的机制不一样，执行任务的过程中创建了新的任务也会再一个时机同时清空
  for (const job of queue) {
    job() // 执行任务
  }
    
  // 初始化任务队列
  queue.length = 0
  queued = false
}

```

总结一下：

1. 上下文其实就是 `petite-vue` 的实例，保持了scope 响应式数据、refs、指令、blocks等数据
2.  `petite-vue` 内部响应式数据更新依赖 vue3 的 effect API，同时内部也有类似 vue2 的微任务队列，所以 `petite-vue` 也是异步更新的
3. 创建上下文后不会主动渲染，需要手动调用 mount 方法进行block的实例化，然后挂载元素



### Block

当调用 mount 函数的时候，应用会生成block，渲染页面。block其实就是 dom节点模板，在 `petite-vue` 中就是组件的 template、v-scope挂载的节点、v-if （v-else、v-else-if等）节点或者是v-for指令创建的分支。

block的定义如下：

```typescript
/// /src/block.ts

import { Context, createContext } from './context'
import { walk } from './walk'
import { remove } from '@vue/shared'
import { stop } from '@vue/reactivity'

// Block 对象
export class Block {
  template: Element | DocumentFragment // 模板  
  ctx: Context // block所对应组件的实例
  key?: any // 对应的key，用于v-for
  parentCtx?: Context // 父组件实例，如果有

  isFragment: boolean // 是否为 template 元素
  start?: Text // 文本节点，用于定位插入位置（模板需要）
  end?: Text // 文本节点，用于定位插入位置（模板需要）

  constructor(template: Element, parentCtx: Context, isRoot = false) {
    this.isFragment = template instanceof HTMLTemplateElement

    // 获取模板
    if (isRoot) {
      this.template = template
    } else if (this.isFragment) {
      this.template = (template as HTMLTemplateElement).content.cloneNode(
        true
      ) as DocumentFragment
    } else {
      this.template = template.cloneNode(true) as Element
    }

    if (isRoot) {
      this.ctx = parentCtx
    } else {
      // create child context
      this.parentCtx = parentCtx
      parentCtx.blocks.push(this)
      // 创建block的上下文
      this.ctx = createContext(parentCtx) 
    }

    // 开始遍历模板
    walk(this.template, this.ctx)
  }

  // 插入元素
  insert(parent: Element, anchor: Node | null = null) {
    if (this.isFragment) {
      if (this.start) {
        // 已经插入了，相当于移动位置
        let node: Node | null = this.start
        let next: Node | null
        while (node) {
          next = node.nextSibling
          parent.insertBefore(node, anchor)
          if (node === this.end) break
          node = next
        }
      } else {
        // 首次插入
        // 先插入定位的元素
        this.start = new Text('')
        this.end = new Text('')
        parent.insertBefore(this.end, anchor)
        parent.insertBefore(this.start, this.end)
        parent.insertBefore(this.template, this.end)
      }
    } else {
      // 如果不是模板， 直接插入父元素中即可
      parent.insertBefore(this.template, anchor)
    }
  }

  // 移除元素
  remove() {
    if (this.parentCtx) {
      remove(this.parentCtx.blocks, this)
    }
    if (this.start) {
      const parent = this.start.parentNode!
      let node: Node | null = this.start
      let next: Node | null
      while (node) {
        next = node.nextSibling
        parent.removeChild(node)
        if (node === this.end) break
        node = next
      }
    } else {
      this.template.parentNode!.removeChild(this.template)
    }
    
    this.teardown()
  }

  // 卸载block
  teardown() {
    this.ctx.blocks.forEach((child) => {
      child.teardown()
    })
    this.ctx.effects.forEach(stop)
    this.ctx.cleanups.forEach((fn) => fn())
  }
}

```

总结一下：

1. block就是组件、v-if、v-for等指令所对应的dom节点，`petite-vue` 封装了Block对象，用于插入、更新、删除这些dom节点



### walk

创建 block 之后，就开始遍历 block 对应的模板，这里基本上就是 DOM 节点的遍历了，具体操作有解析节点上的属性、时间、指令和渲染数据。对应的逻辑是：

```typescript
// /src/walk.ts

const dirRE = /^(?:v-|:|@)/
export let inOnce = false
// 遍历模板
export const walk = (node: Node, ctx: Context): ChildNode | null | void => {
  const type = node.nodeType
  // 识别 dom 元素类型
  if (type === 1) {
    // 普通元素
    const el = node as Element
    // v-pre 指令的元素不解析
    if (el.hasAttribute('v-pre')) {
      return
    }

    let exp: string | null

    // v-if 指令
    if ((exp = checkAttr(el, 'v-if'))) {
      return _if(el, exp, ctx)
    }

    // v-for 指令
    if ((exp = checkAttr(el, 'v-for'))) {
      return _for(el, exp, ctx)
    }

    // v-scope 指令, 可以读取表达式执行（适用于函数组件）
    if ((exp = checkAttr(el, 'v-scope')) || exp === '') {
      const scope = exp ? evaluate(ctx.scope, exp) : {}
      // 创建嵌套的上下文
      ctx = createScopedContext(ctx, scope)
      if (scope.$template) {
        // 解析模板
        resolveTemplate(el, scope.$template)
      }
    }

    // v-once 指令
    const hasVOnce = checkAttr(el, 'v-once') != null
    if (hasVOnce) {
      // 只解析一次，不做响应式的处理，创建上下文的时候会用到
      inOnce = true
    }

    // 解析 ref
    if ((exp = checkAttr(el, 'ref'))) {
      applyDirective(el, ref, `"${exp}"`, ctx)
    }

    // 解析子节点
    walkChildren(el, ctx)

    // 
    const deferred: [string, string][] = []
    // 解析其它指令，包括v-bind、v-on、v-html、v-text等，遍历所有节点的所有属性
    for (const { name, value } of [...el.attributes]) {
      // 用正则判断是否为指令
      if (dirRE.test(name) && name !== 'v-cloak') {
        if (name === 'v-model') {
		  // v-model 依赖 v-bind 指令，所以放到后面再解析
          deferred.unshift([name, value])
        } else if (name[0] === '@' || /^v-on\b/.test(name)) {
          // v-on 事件指令同理，但后于 v-model 指令
          deferred.push([name, value])
        } else {
          // 其它的指令
          processDirective(el, name, value, ctx)
        }
      }
    }
 
    // 解析延迟的指令
    for (const [name, value] of deferred) {  
      processDirective(el, name, value, ctx)
    }

   	// 重置 flag
    if (hasVOnce) {
      inOnce = false
    }
  } else if (type === 3) {
    // 文本节点，主要是 {{}} 这种文本
    const data = (node as Text).data
    if (data.includes(ctx.delimiters[0])) {
      let segments: string[] = []
      let lastIndex = 0
      let match
      while ((match = ctx.delimitersRE.exec(data))) {
        const leading = data.slice(lastIndex, match.index)
        if (leading) segments.push(JSON.stringify(leading))
        // $s = displayString 函数
        segments.push(`$s(${match[1]})`)
        lastIndex = match.index + match[0].length
      }
      if (lastIndex < data.length) {
        segments.push(JSON.stringify(data.slice(lastIndex)))
      }
      // 应用文本指令
      applyDirective(node, text, segments.join('+'), ctx)
    }
  } else if (type === 11) {
    // fragment 类型，跳过该节点，直接解析其子节点
    walkChildren(node as DocumentFragment, ctx)
  }
}

// 遍历子节点列表
const walkChildren = (node: Element | DocumentFragment, ctx: Context) => {
  let child = node.firstChild
  while (child) {
    child = walk(child, ctx) || child.nextSibling
  }
}

// 解析组件模板
const resolveTemplate = (el: Element, template: string) => {
  if (template[0] === '#') {
    // #开头，说明是传入的是节点id
    const templateEl = document.querySelector(template)
    // 模板内容深拷贝一份，插入 el 中
    el.appendChild((templateEl as HTMLTemplateElement).content.cloneNode(true))
    return
  }
  // 其它情况，直接当作富文本处理
  el.innerHTML = template
}
```

然后是指令的处理

```typescript
// /src/walk.ts

const modifierRE = /\.([\w-]+)/g

// 处理指令
const processDirective = (
  el: Element,
  raw: string,
  exp: string,
  ctx: Context
) => {
  let dir: Directive
  
  // 指令的参数
  // 比如v-bind:value => value
  let arg: string | undefined
  // 指令的修饰符，v-bind.sync => sync
  let modifiers: Record<string, true> | undefined

  // 获取修饰符
  raw = raw.replace(modifierRE, (_, m) => {
    ;(modifiers || (modifiers = {}))[m] = true
    return ''
  })

  if (raw[0] === ':') {
    // 第一个为“:”，v-bind指令
    dir = bind
    arg = raw.slice(1)
  } else if (raw[0] === '@') {
    // 第一个字符为“@”，v-on指令
    dir = on
    arg = raw.slice(1)
  } else {
    const argIndex = raw.indexOf(':')
    const dirName = argIndex > 0 ? raw.slice(2, argIndex) : raw.slice(2)
    // 获取指令函数，首先是从内置的指令中获取，其次是组件定义的指令
    dir = builtInDirectives[dirName] || ctx.dirs[dirName]
    arg = argIndex > 0 ? raw.slice(argIndex + 1) : undefined
  }
  
  if (dir) {
    // :ref 是 ref 指令
    if (dir === bind && arg === 'ref') dir = ref
    
    // 应用指令
    applyDirective(el, dir, exp, ctx, arg, modifiers)
    // 指令处理完毕，移除属性
    el.removeAttribute(raw)
  }
}

// 应用指令
const applyDirective = (
  el: Node, // 指令对应的节点
  dir: Directive<any>, // 指令函数
  exp: string, // 指令表达式
  ctx: Context, // 指令对应的组件上下文
  arg?: string, // 指令参数
  modifiers?: Record<string, true> // 指令修饰符
) => {
  // 封装 get 方法，get函数是exp表达式的求值
  const get = (e = exp) => evaluate(ctx.scope, e, el)
  // 执行指令函数
  const cleanup = dir({
    el,
    get,
    effect: ctx.effect,
    ctx,
    exp,
    arg,
    modifiers
  })
  // 保持指令返回的cleanup函数
  if (cleanup) {
    ctx.cleanups.push(cleanup)
  }
}

```

指令的处理其实很简单，指令由**指令名称+修饰符+参数+表达式**组成，首先需要解析出这四个结构，然后调用具体的指令函数即可。特别的，看一下 `evaluate` 函数：

```typescript
// /src/eval.ts

// 缓存
const evalCache: Record<string, Function> = Object.create(null)

// 将表达式转化为函数
const toFunction = (exp: string): Function => {
  try {
    return new Function(`$data`, `$el`, `with($data){${exp}}`)
  } catch (e) {
    console.error(`${(e as Error).message} in expression: ${exp}`)
    return () => {}
  }
}

// 在 scope 作用域下执行 exp 表达式
const execute = (scope: any, exp: string, el?: Node) => {
  // 首先检查缓存，没有才新建一个
  const fn = evalCache[exp] || (evalCache[exp] = toFunction(exp))
  try {
    // 直接执行函数，返回结果
    return fn(scope, el)
  } catch (e) {
    if (import.meta.env.DEV) {
      console.warn(`Error when evaluating expression "${exp}":`)
    }
    console.error(e)
  }
}

// 表达式求值
export const evaluate = (scope: any, exp: string, el?: Node) =>
	execute(scope, `return(${exp})`, el)

```

通过 evaluate，指令和 "{{}}" 里面的表达式可以绑定组件上下文进行求值，比如 `v-if="visible"` 会被封装成这样的 get 函数：

```javascript
function fn($data, $el) {
    with($data) {
        return visible;
    }
}

const get = (ctx, el) => fn(ctx, el); 
```

ok，我们终于知道 `petite-vue` 执行模板中的表达式的原理了，其实都是封装成了函数，绑定了组件的上下文。

最后总结一下：

1. walk 其实就是对 dom 节点的深度优先遍历，会解析每个节点的属性和内容，再应用指令来接管渲染的流程
2. v-if 和 v-for 和一般的指令不一样，涉及到 dom 节点的变更，需要特殊处理
3. 模板中的表达式都是通过 `evaluate` 转化成函数再执行的，从而绑定了对应组件的上下文

### 指令

指令也是 `petite-vue` 的一个核心机制，本质上指令是一个函数，在 walk 阶段，会检查 dom 节点上的属性，执行对应的指令函数。然后，指令就可以通过 effect 函数，在数据更新后，执行对应的操作。

指令的定义：

```typescript
// 指令函数
export interface Directive<T = Element> {
  (ctx: DirectiveContext<T>): (() => void) | void
}

// 指令函数的参数
export interface DirectiveContext<T = Element> {
  el: T // 指令所对应的 dom 节点
  get: (exp?: string) => any // 指令的表达式所转化成的函数，已经绑定上下文
  effect: typeof rawEffect // 组件封装好的 effect 函数，响应式数据更新后会调用
  exp: string // raw 表达式
  arg?: string // 指令参数
  modifiers?: Record<string, true> // 指令修饰符
  ctx: Context // 指令对应组件实例
}

```

`petite-vue` 提供了一些内置指令，我们简单介绍一下：

```typescript
export const builtInDirectives: Record<string, Directive<any>> = {
  bind, // v-on :value="value"  props 或者属性绑定
  on, // v-bind @click="xxx" 事件绑定
  show, // v-show
  text, // v-text, {{}} 双引号表达式处理
  html, // v-html
  model, // v-model
  effect // v-effect
}

```

#### show

```typescript
// /src/directives/show.ts

export const show: Directive<HTMLElement> = ({ el, get, effect }) => {
  // 获取原始的 display 属性
  const initialDisplay = el.style.display
  effect(() => {
    // 根据表达式执行的结果判断，如果为false，则设置dispaly为none
    el.style.display = get() ? initialDisplay : 'none'
  })
}

```

#### ref

```typescript
// /src/directives/ref.ts

export const ref: Directive = ({
  el,
  ctx: {
    scope: { $refs }
  },
  get,
  effect
}) => {
  let prevRef: any
  effect(() => {
    // 获取 ref key
    const ref = get()
    // 在组件上下文中设置 ref
    $refs[ref] = el
    // 如果和之前设置的值不一样，删除之前设置的值
    if (prevRef && ref !== prevRef) {
      delete $refs[prevRef]
    }
    prevRef = ref
  })
  // cleanup 函数，组件卸载后执行，删除设置的 ref
  return () => {
    prevRef && delete $refs[prevRef]
  }
}

```

#### if

```typescript
// /src/directives/if.ts

// v-if、v-else-if、v-else 对应的节点和表达式
interface Branch {
  exp?: string | null
  el: Element
}

// v-if 指令函数
export const _if = (el: Element, exp: string, ctx: Context) => {
  const parent = el.parentElement! // 父节点
  const anchor = new Comment('v-if') // 锚点，用于定位
  parent.insertBefore(anchor, el) // 插入锚点

  // 分支列表
  const branches: Branch[] = [
    {
      exp,
      el
    }
  ]

  // 在后续的兄弟节点中，寻找 v-else-if 或者 v-else
  let elseEl: Element | null
  let elseExp: string | null
  while ((elseEl = el.nextElementSibling)) {
    elseExp = null
    // 找到 v-else-if 或者 v-else（v-else的表达式为空字符串）
    if (
      checkAttr(elseEl, 'v-else') === '' ||
      (elseExp = checkAttr(elseEl, 'v-else-if'))
    ) {
      parent.removeChild(elseEl)
      branches.push({ exp: elseExp, el: elseEl })
    } else {
      // 如果找不到，退出，v-if、v-else-if、v-else 节点必须相邻，中间不能有其它节点
      break
    }
  }
  
  const nextNode = el.nextSibling
  
  // 移除v-if节点，因为需要判别表达式是否为真值
  parent.removeChild(el)

  // 当前命中分支对应的 block
  let block: Block | undefined
  // 保持当前命中分支索引
  let activeBranchIndex: number = -1

  // 移除当前命中的分支 block
  const removeActiveBlock = () => {
    if (block) {
      // 先插入锚点
      parent.insertBefore(anchor, block.el)
      block.remove()
      block = undefined
    }
  }

  ctx.effect(() => {
    // 遍历分支
    for (let i = 0; i < branches.length; i++) {
      const { exp, el } = branches[i]
      if (!exp || evaluate(ctx.scope, exp)) {
        // 命中该分支
        if (i !== activeBranchIndex) {
          // 清除之前命中的分支
          removeActiveBlock()
          // 实例化新的分支所对应的 block
          block = new Block(el, ctx)
          // 插入元素
          block.insert(parent, anchor)
          // 删除锚点
          parent.removeChild(anchor)
          // 保存索引
          activeBranchIndex = i
        }
        // 命中一个分支就可以介绍流程
        return
      }
    }
    
    // 没有命中的分支
    activeBranchIndex = -1
    removeActiveBlock()
  })

  return nextNode
}

```

关于指令的源码分析就到这里吧，还有一部分指令，譬如 v-on、v-bind、v-for 原理都大同小异，都是基于 effect API，侦测响应式数据变更，执行对应 DOM 变更，感兴趣的读者可以自行去阅读。



## 总结

通过源码分析，我们了解了 `petite-vue` 内部的工作机制，内部实现虽然简单，但是很巧妙，非常值得我们学习。

