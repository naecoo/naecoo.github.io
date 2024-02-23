+++
title = "从0开始手写一个Vue消息弹窗组件"
date = "2019-04-25"
author = "naeco"
[taxonomies]
tags = ["vue"]
+++

### 前言
近期闲来无事，就想折腾一点东西，无奈技术水平太低，做不了什么高深复杂的项目，所以还是先从简单的做起吧。平时我们做项目的时候或多或少都会用到一些组件库，比如element、muse之类的，所以这次我想自己手动写一个组件，封装成一个包，可以暴露到外部去调用。

### 效果预览
废话不多说，这次写的是一个Vue的消息弹窗组建，借鉴了element的message组件，大家可以去看一看element的[源码](<https://github.com/ElemeFE/element>)，里面一些组件的实现方式很巧妙，非常值得去学习。话不多说，先来看看最终的实现效果：

> 图片迁移，暂时无法查看

***
### 代码

第一步应该是写一个模板：

```html
<template>
  <transition name="msg-fade" @after-leave="afterLeave">
    <div :class="classes" class="msg-container" v-show="visible" role="alert" @mouseenter="clearTimer" @mouseleave="startTimer">
      <span>{{ message }}</span>
      <i class="msg-icon-wrapper" v-if="showCloseButton" @click="close">
        <img class="msg-icon" src="./close.svg" alt="close" />
      </i>
    </div>
  </transition>
</template>
```

可以看到，html内容是很简单的，主要是几个标签和一些props，还有几个事件，然后我们再来看js：

```javascript
<script>
export default {
  name: 'Message',
  props: {
    // 要显示的消息
    message: {
      type: String,
      require: true
    },
    // 主题
    type: {
      type: String,
      default: 'normal'
      // normal success error warning
    },
    // 是否显示关闭按钮
    showCloseButton: {
      type: Boolean,
      default: false
    },
    // 组件出现的位置
    location: {
      type: String,
      default: 'top-center'
      // top-center top-right top-left bottom-center bottom-right bottom-left
    },
    // 自定义样式名
    className: {
      type: String,
      default: ''
    },
    // 持续时间
    duration: {
      type: Number,
      default: 2000
    }
  },
  data () {
    return {
      // 动态修改样式
      classes: [
        {
          'msg-icon-padding': this.showCloseButton
        }, 
        `msg-${this.type}`,
        `msg-${this.location}`,
        this.className
      ],
       // 控制组件显示
      visible: false,
       // 是否关闭
      closed: false,
       // 定时器
      timer: null
   }
  },
  watch: {
    closed (newVal) {
      if (newVal) {
        this.visible = false
      }
    }
  },
  methods: {
    close () {
      this.closed = true
    },
    afterLeave () {
      this.$destroy(true)
      this.$el.parentNode.removeChild(this.$el)
    },
    clearTimer () {
      clearTimeout(this.timer)
    },
    startTimer () {
      const timeout = this.duration
      if (timeout > 0) {
        this.timer = setTimeout(() => {
          if (!this.closed) {
            this.close()
          }
        }, timeout);
      }
    }
  },
  mounted () {
    this.startTimer()
    this.visible = true
  },
  beforeDestroy () {
    this.clearTimer()
  }
}
</script>
```

可以看到组件的代码逻辑也是非常简单的，props的处理我就不多细说了，可能每一个人的需求都一样。重点要讲一讲的是组件的创建和销毁过程。在组件挂载后，组件会调用startTimer方法生成一个定时器，按照duration指定的时间后调用close方法，close方法会设定data的closed为true，visible监听到closed变为true后就变为false，所以根据v-show这个指令，组件的display样式属性就会变成none。最后Vue官方内置的过渡组件transition会调用一系列钩子函数，我们直接在afterLeave这个钩子上销毁组件和清除dom。看到这里，可能有同学会有疑问了？为啥要这样销毁组件啊，直接用一个prop控制组件显示不就可以了吗？其实对于一般的组件我们是没必要弄成这样，但是我们得思考一下消息弹窗的使用场景啊。一般情况下，消息弹窗都是在完成了某种操作后，弹出来告诉用户操作成功与否，所以我们通常用js控制消息弹窗显示。因此，我们的消息弹窗组件不仅仅是声明式的，更应该是命令式的，直接用js控制显示与否。

很多同学应该都使用过element，element在全局引入的时候会向vue上挂载一个message的方法，一般我们都是`this.$message`这样来操作的，我的目标也是做成这样子。

```javascript
import Vue from 'vue'
import Message from './Message.vue'

const messageConstructor = Vue.extend(Message)
let count = 1

const notify = (options = {}) => {
  // 如果Vue运行在服务端，放弃调用
  if (Vue.prototype.$isServer) {
    return
  }
  
  if (typeof options === 'string') {
    options = {
      message: options
    }
  }

  // 创造实例
  const instance = new messageConstructor({
    propsData: options
  })

  // 渲染并挂载到body上
  const vm = instance.$mount()
  document.body.appendChild(vm.$el)
  vm.$el.style.zIndex = count

  // 返回一个主动关闭的方法
  const close = () => {
    vm.closed = true
  }
  return close
}

export default notify
```

这里我们用了Vue的一个方法，`vue.extend`，这个方法接受一些配置参数，返回一个可以生产vue实例的构造函数，其实我们平时使用的组件一般都是这些构造函数实例化的对象。代码的逻辑很简单，当notify方法被调用的时候会创造一个message组件的实例对象，同时传入props，最后渲染成dom并挂载到document.body上。最后我们提供一个install方法将notify和message组件全局安装到Vue中：

```javascript
class Plugin {
  static install (Vue) {
    Vue.prototype.notify = notify
    Vue.mixin({
      components: {
        'oce-message': Message
      }
    })
  }
}
export default Plugin


// 全局注册
import oce-message from 'oce-message'
import Vue from 'vue'

Vue.use(oce-message)

// 使用
<oce-message ... />
this.notify(...)
```

好了，一个简单的消息弹窗组件就完成了，具体的代码和使用方法在这里[oce-message](https://www.github.com/naeco/oce-message)可以找到，欢迎star和提issue。

### 后续

这个组件有点简陋，其实有一些可以优化的地方：

1. 样式
2. 过渡动画
3. 性能（实例是否可以做成一个单例？）