+++
title = "谈一谈前端路由"
date = "2019-04-14"
author = "naeco"
[taxonomies]
tags = ["javascript"]
+++

## 前言
  在最初的多页面应用中，每一个页面往往都会对应一个具体的url，每当我们切换一次页面，浏览器都会重新发起一次对服务器的请求，然后再重新渲染HTML。后来随着Ajax等技术的发展，SPA逐渐开始流行起来。所谓SPA呢，指的是只加载单个HTML页面的应用，后续的交互只会向服务器请求数据，然后前端通过操作dom，来切换数据相应的视图。但是SPA也存在着一些缺点，由于SPA不会改变url，所以万一不小心刷新了页面，所有的状态都会被销毁，页面又会回到最初的状态。所以我们迫切需要一种机制，把应用的层次结构映射到url中，于是乎，前端路由就诞生了。

### 何谓前端路由
  说白了，前端路由就是根据不同的url，动态地切换需要显示的视图。但是一般如果我们直接改变url，浏览器会刷新页面。所以一开始的路由利用了url上的hash，因为hash的改变不会导致页面刷新，再加上浏览器又提供了`onhashchange`这样的事件接口，所以hash路由开始大行其道。但是hash这种模式的路由有一个缺陷，就是url上出现一个#字符，显得很突兀，有人可能会觉得很丑。后面HTML5新增了popstate的接口，我们可以通过history的方法改变url而不刷新页面，然后在`onpopstate`的事件中执行相应的处理逻辑。所以目前主流的路由是有两种模式，一种是hash，兼容性强但不美观；一张是history，美观但不兼容一些低版本浏览器，而且后端需要额外的处理逻辑。

### 实现路由
下面我们实现一个简单的hash路由：
```javascript
class Router {
  constructor () {
    this.routes = {}
    window.addEventListener('hashchange', this.change.bind(this))
    window.addEventListener('load', this.change.bind(this))
  }
  
  addRoute(path = '/', match) {
    this.routes[path] = match
  }

  change () {
    const currentUrl = location.hash.slice(1) || '/'
    const match = this.routes[currentUrl]
    match && match()
  }
}
```
原理很简单，就是根据url的变化来执行对应url的回调函数。如果我们还想要更精细的操作，我们可以在Router内部维护一个记录url变化的数组，这样就可以实现一些编程式的导航方法，比如go，back，push和replace等等。
再来说一下history路由，history是浏览器提供的一个接口，上面有pushState和replaceState两个方法，其实两个方法差不多，只不过replaceState是取代当前state。这两个方法最关键的是可以无刷新的改变url，就和改变hash值差不多。但是有一点问题就是这两个方法不会触发popstate事件，所以如果利用这点去做编程式的路由跳转，需要去包装这两个方法，在跳转的同时需要触发相应path的回调。除此之外，history还提供了go，back，forward等接口，可以通过JavaScript模拟页面的前进后退等功能。其实大体的实现原理是和hash差不多的，只不过hisotry原生支持了很多功能，而hash需要hack。
