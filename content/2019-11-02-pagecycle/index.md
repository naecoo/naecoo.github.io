+++
title = "网页生命周期(Page lifecycle API)"
date = "2019-11-02"
author = "naeco"
[taxonomies]
tags = ["browser", "html", "javascript"]
+++

### 背景
apps的生命周期对于操作系统管理系统资源来说至关重要。在Android、IOS和Windows等的平台上，系统可以随意启动和中止apps的运行，这使得这些平台能够重新分配资源，以提供最好的操作体验给用户。

但对于web应用，并没有类似的生命周期机制。随着打开网页数量增加，内存、CPU、网络吞吐等系统关键资源都被过度使用，导致系统性能下降，降低了用户的操作体验。

虽然浏览器很早之前就有提供关于生命周期的事件，比如`load`、`unload`和`visibilitychange`等，但是这些事件只允许开发人员响应用户自行发起的生命周期状态更改。为了让网页更可靠地运行，尤其是在低功耗的设备（手机、智能手表等），浏览器需要一种主动回收和分配系统资源的机制。

[Page Lifecycle API](https://wicg.github.io/page-lifecycle/spec.html)就是为了解决这些问题而提出的方案，目标主要有三点：

1. 在web上引入并标准化生命周期状态的概念
2. 定义新的应用状态，允许浏览器限制网页持续占据系统资源
3. 创建新的API和事件，允许开发人员响应这些应用状态之间的转换

`Page Lifecycle API`已经在Chrome 68上得到支持，下面将会进行详细介绍。

+++

### 概览
`Page Lifecycle API`定义了规范化的app生命周期状态，且每个页面在一个阶段只能处于一个状态，每个状态的改变都会有相应的事件被触发。话不多说，先上图：

![page lifecycle](https://user-gold-cdn.xitu.io/2019/11/2/16e2a10c43ce5690?w=3280&h=2218&f=png&s=181596)

#### 状态

页面状态有6个，下面将分别展开描述，from代表的是前一个页面状态，to代表的是下一阶段可能会变更到的状态。

1. `Active`

如果页面可见并具有输入焦点，则该页面处于`Active`状态

from：`passive`(focus事件)

to: `passive`(blur事件)



2. `Passive`

如果页面可见并不具有输入焦点，则该页面处于`Passive`状态

from: `avtive`(blur事件)、`hidden`(visibilitychange事件)

to: `avtive`(focus事件)、`hidden`(visibilitychange事件)



3. `Hidden`

如果页面不可见且不处于`frozen`状态，则该页面处于`hidden`状态

from: `passive`(visibilitychange事件)

to:  `passive`(visibilitychange事件)、`frozen`(freeze事件)、`terminated`(pagehide事件)



4. `Frozen`

如果网页处于`frozen`状态下，浏览器将暂停执行页面任务队列中的可冻结任务，直到页面被解除冻结。这意味着像定时器和回调函数这样的任务不会运行。

from: `hidden`(freeze事件)

to:  `active`(resume和pageshow事件)、`passive`(resume和pageshow事件)、`hidden`(resume事件)



5. `Terminated`

如果页面开始被浏览器卸载并从内存中清除，它就处于`terminated`状态。在此状态下不能启动任何新任务，如果现有运行时间太长，可能会被提前终止。

from: `hidden`(pagehide事件)

to: None



6. `Discarded`

当浏览器为了节省资源而卸载页面时，它处于`discarded`状态。任何类型的任务、事件回调或JavaScript代码都不能在这种状态下运行，因为丢弃通常发生在资源约束下，可以理解为页面被动关闭，浏览器主动释放资源。

from: `frozen`(无事件触发)

to: None 



#### 事件

<span style="color:red;">*</span>代表的是`Page Lifecycle API`新提供的事件

1. foucs

   DOM元素获取焦点，前一个状态一般是`passive`，当前状态是`avtive`

   

2. blur

   DOM元素失去焦点，前一个状态一般是`active`，当前状态是`passive`

   

3. visibilitychange

   `document`的`visibilitySatate`属性发生变更，当用户导航到新页面、切换选项卡、关闭选项卡、最小化或关闭浏览器、或切换移动操作系统上的应用程序时，会触发事件。前一个状态是`passive`或`hidden`,，当前状态是`passive`或者`hidden`

   

4. freeze<span style="color:red;">*</span>

   页面被冻结，任务队列中的可冻结任务都停止运行。前一个状态是`hidden`，当前状态是`frozen`

   

5. resume<span style="color:red;">*</span>

   页面解除冻结状态，前一个状态是`frozen`，当前状态是`active`、`passive`或者`hidden`

   

6. pageshow

   当一条会话历史记录被执行的时候将会触发事件，包括了后退(前进)按钮操作，同时也会在`onload`事件触发后初始化页面时触发。前一个状态可能为`frozen`，当前状态为`active`、`passive`或`hidden`

   

7. pagehide

   与`pageshow`类似，不同的就是导航离开当前网页时触发。先前的状态可能为`hidden`，当前状态可能为`frozen`或者`terminated`

   

8. beforeunload

   窗口、文档及其资源即将被卸载。文档仍然可见，此时事件仍然可以取消。前一个状态可能为`hidden`，当前状态为`terminated`

   

9. unload

   卸载页面时触发，前一个状态可能为`hidden`，当前状态为`terminated`




#### 新增功能

上面的图表显示了两种系统触发的页面状态：`frozen `和`discard`，这种页面状态和用户触发的不一样，开发者无法主动感知状态变更。但是在Chrome  68上，开发者可以监听`freeze`和`resume`两个事件处理页面状态变更。

```javascript
document.addEventListener('freeze', (event) => {
  // 页面处于冻结状态
});

document.addEventListener('resume', (event) => {
  // 页面解冻
});
```

同时`document`对象新增了`wasDiscarded`属性，这个属性代表页面是否处于`discarded`状态，开发者可以根据这个属性的值处理不同的逻辑

```javascript
if (document.wasDiscarded) {
  // 页面被浏览器丢弃
}
```

+++

### 观察状态

```javascript
// active、passive和hidden三种状态
const getState = () => {
  if (document.visibilityState === 'hidden') {
    return 'hidden';
  }
  if (document.hasFocus()) {
    return 'active';
  }
  return 'passive';
};
```

另外，`frozen`和和`terminated`的状态变更可以通过监听`freeze`和`pagehide`事件获取。



#### 封装状态观察器

```javascript
// 保存初始页面状态
let state = getState();

// 记录状态变更并打印在控制台
// 更新当前页面状态
const logStateChange = (nextState) => {
  const prevState = state;
  if (nextState !== prevState) {
    console.log(`State change: ${prevState} >>> ${nextState}`);
    state = nextState;
  }
};

// 监听生命周期事件，保持页面状态更新
['pageshow', 'focus', 'blur', 'visibilitychange', 'resume'].forEach((type) => {
  window.addEventListener(type, () => logStateChange(getState()), {capture: true});
});

// 监听freeze事件
window.addEventListener('freeze', () => {
  // 页面状态变更为frozen
  logStateChange('frozen');
}, {capture: true});

window.addEventListener('pagehide', (event) => {
  if (event.persisted) {
    // event.persisted为true意味着页面是从缓存中加载的，所以是frozen状态
    logStateChange('frozen');
  } else {
    // terminated状态
    logStateChange('terminated');
  }
}, {capture: true});
```

#### 跨平台

`Page Lifecycle`这个标准刚刚引入，并没有得到全部浏览器平台的支持。有些浏览器可能在切换标签的时候不会触发`blur`事件，有些浏览器没有实现`freeze`和`resume`事件，IE10以下版本不支持`visibilitychange`事件等等...

为了让开发者更容易上手处理跨平台兼容的问题，谷歌开发了[PageLifecycle.js](https://github.com/GoogleChromeLabs/page-lifecycle)这个库，用于观察页面生命周期状态的变化。PageLifecycle.js按事件触发顺序规范化处理了跨浏览器的差异，保证状态变更和标准规范保存一致。



+++

### 最佳实践

对于开发者来说，理解页面生命状态并懂得根据页面状态不懂执行不同业务逻辑是非常重要的，下面介绍各个状态下的最佳实践：

1. Active

   `active`状态是用户最关键的时间，因此也是页面响应用户输入最重要的时间，任何可能阻塞主线程执行的非UI渲染任务都可以放到`requestidlecallback`或者`web worker`执行

   

2. Passive

   在此状态下，用户没有和页面进行交互，但页面仍然处于可视状态，所以UI和动画需要保持渲染状态，从`active`状态过渡到`passive`是一个保存页面状态的好时机，比如一些表单值。

   

3. Hidden

   页面被隐藏或者关闭，停止所有与用户交互、UI渲染有关的任务，并及时保存应用状态

   

4. Frozen

   在`frozen`状态下，[可冻结任务](<https://html.spec.whatwg.org/multipage/webappapis.html#queue-a-task>)将会被挂起直到页面解冻(有可能永远不会发生）。在这个状态下，开发者要做以下几点：

   1. 关闭IndexedDB连接
   2. 关闭[BroadcastChannel](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API) 连接
   3. 关闭WebRTC连接
   4. 停止http轮询和websocket连接
   5. 停止定时器

   

5. Terminated

   通常在此状态下不需要处理任何任务，因为这个阶段的任务不能保存可靠执行，有可能被强行终止。但你也可以做一些状态持久化或者埋点分析的任务



+++

### 相关资料

1. [developers-google-web-page-lifecycle-api](https://developers.google.com/web/updates/2018/07/page-lifecycle-api)
2. [Page Lifecycle API](https://wicg.github.io/page-lifecycle/spec.html)
3. [windows-event-pageshow](<https://developer.mozilla.org/zh-CN/docs/Web/Events/pageshow>)