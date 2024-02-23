+++
title = "60行代码实现一个状态管理库"
date = "2019-05-05"
author = "naeco"
[taxonomies]
tags = ["vue"]
+++

> 先把成品放出来：[vuebx](https://github.com/naecoo/vuebx)

## 前言

当我们使用Vue开发应用时，随着应用的功能增多，不可避免地组件也会增加，不同组件之间的状态重用就会越来越困难。这个时候我们就需要一个全局状态管理工具帮助我们管理状态，Vue官方提供了Vuex给我们使用，但是很多时候我们的项目只是几个页面，根本不需要用Vuex这种复杂度比较高的解决方案，所以我们迫切需要一个轻量、高可用的状态管理库。
## 正文
有人可能会说用`EventBus`不就行了吗？，事实上`EventBus`实质上是个跨组件的消息传递工具，根本算不上状态管理，而且写起来也很麻烦。其实问题不用弄得这么复杂，我们可以利用Vue自带的响应性去解决我们的问题，Vue 2.6.0版本发布了一个新的API [Vue.observable( object )](https://cn.vuejs.org/v2/api/#Vue-observable "Vue.observable( object )"),这个方法可以让一个对象变得可响应，Vue内部就是用这样的方式处理data函数返回的对象，现在Vue把这样的能力暴露出来给我们使用。这个返回的对象可以直接用于渲染函数和计算属性，并且会在发生改变时触发相应的更新。
```javascript
const state = Vue.observable({ count: 0 })

const Demo = {
  render(h) {
    return h('button', {
      on: { click: () => { state.count++ }}
    }, `count is: ${state.count}`)
  }
}
```
你看，Vue已经帮我们把脏活累活都干好了，距离我们的目标就差一步了，我们只需要封装一下，提供一个获取数据和更新数据的接口就ok了。所以，话不多说，请看代码：
```javascript
import Vue from 'vue'
import { cloneDeep } from 'lodash'

function _updateCreator(state) {
  return (newState) => {
    if (typeof newState === 'function') {
      const oldState = cloneDeep(state)
      newState = newState.call(null, oldState)
    }
    Object.keys(newState).forEach(key => {
      if (!(key in state)) {
        throw new Error(`unknown state: ${key}`)
      }
      state[key] = newState[key]
    })
    return new Promise(resolve => {
      Vue.nextTick(() => {
        resolve(cloneDeep(state))
      })
    })
  }
}

function _mapGetters (state) {
  return (getters) => {
    const res = {}
    normalize(getters).forEach(({key, val}) => {
      res[key] = function () {
        if (! (val in state)) {
          throw new Error(`unknown state: ${val}`)
        }
        return state[val]
      }
    })  
    return res
  }
}

function normalize(map) {
  return Array.isArray(map) ?
    map.map(key => ({
      key,
      val: key
    })) :
    Object.keys(map).map(key => ({
      key,
      val: map[key]
    }))
}

function Vuebx (defaultValue = {}) {
  const state = Vue.observable(defaultValue)

  const getState = _mapGetters(state)
  const setState = _updateCreator(state)

  return [getState, setState]
}

export default Vuebx
```
虽然说代码只有60行，但看起来也挺长的，我们不妨将代码分解一下：
```javascript
function Vuebx (defaultValue = {}) {
  const state = Vue.observable(defaultValue)

  const getState = _mapGetters(state)
  const setState = _updateCreator(state)

  return [getState, setState]
}

export default Vuebx
```
这是我们的主要模块，主要的作用就是接受一个对象，然后用`Vue.observable`生成一个`observable`对象充当`state`，最后利用闭包生成`state`的`getter`和`setter`。然后我们再来看一下具体是怎么生成的：
```javascript
function _mapGetters (state) {
  return (getters) => {
    const res = {}
    normalize(getters).forEach(({key, val}) => {
      res[key] = function () {
        if (! (val in state)) {
          throw new Error(`unknown state: ${val}`)
        }
        return state[val]
      }
    })  
    return res
  }
}

function normalize(map) {
  return Array.isArray(map) ?
    map.map(key => ({
      key,
      val: key
    })) :
    Object.keys(map).map(key => ({
      key,
      val: map[key]
    }))
}
```
其实这和Vuex的mapGetter实现方式是一样的，所以使用方式一模一样，作用都是将`state`映射到组件的计算属性中。这个方法的实现原理是将传进来的数组或者对象序列化后，然后再从`state`中提取出具体的字段包装成函数，与在组件中定义`computed`的函数是一样的。最后，我们看一下`setter`是怎么实现的:
```javascript
function _updateCreator(state) {
  return (newState) => {
    if (typeof newState === 'function') {
      const oldState = cloneDeep(state)
      newState = newState.call(null, oldState)
    }
    Object.keys(newState).forEach(key => {
      if (!(key in state)) {
        throw new Error(`unknown state: ${key}`)
      }
      state[key] = newState[key]
    })
    return new Promise(resolve => {
      Vue.nextTick(() => {
        resolve(cloneDeep(state))
      })
    })
  }
}
```
这是一个高阶函数，作用是利用闭包绑定`state`，返回的函数接受一个对象或者一个可以返回对象的函数来更新`state`的值，最后函数会返回一个`promise`，promise会在`state`更新后resolve，所以可以在`then`中获取到最新的`state`。
ok，一个简单的状态管理工具就完成了，实现的代码加起来一共只有60行，包大小连1K都没有，可以说是相当轻量了。而且接口定义非常简洁，只有两个方法，用起来十分清爽。
```javascript
// store/index.js
import vuebx from 'vuebx'

const state = {
  count: 1
}
const [getter, setter] = vuebx(state)
export default {
  getter,
  setter
}

// Counter.vue
<template>
  <p>{{ count }}<p>
  <p>
    <button v-on:click="increment">-</button>
    <button v-on:click="decrement">+</button>
  </p>
</template>
<script>
  import { getter, setter } from './store'
  export default {
    name: 'Counter',
    computed: {
      ...getter(['count'])
    },
    methods: {
      increment () {
        setter((state) => {
          return {
            count: state.count + 1
          }
        })
      },
      decrement () {
        setter((state) => {
          return {
            count: state.count - 1
          }
        })
      }
    }
  }
</script>
```
## 总结
我们利用了Vue的响应性的能力，创建了一个轻量级的状态管理工具，使用起来非常方便，适用于小型的Vue项目，当然中大型的项目还是推荐使用Vuex，毕竟是官方负责维护的，质量有保证。