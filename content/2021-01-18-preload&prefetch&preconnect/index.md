+++
title = "preload、prefetch和preconnect的简单介绍"
date = "2021-02-28"
author = "naeco"
[taxonomies]
tags = ["html"]
+++



### 简介

​	得益于浏览器技术的不断发展，现代网页应用体验已经不断接近原生应用，许多大公司也将原生应用迁移到浏览器，甚至更极端的，放弃原生应用，只提供网页应用。随之而来的，网页应用的代码量在不断增加，如果还像以前的样子，将所有代码放到一两个文件中，势必会影响网页加载和渲染的速度。所以网页应用一般都会利用构建工具，如webpack、gulp和rollup等，将代码按模块、路由或者命名空间进行分割，然后生成一个个比较小的js、css和html文件。但是问题又随之而来了，在如此众多的资源文件中，该如何决定加载的先后顺序和优先级呢？幸运的是，浏览器提供了preload、prefetch、preconnect和prerender等指令用来帮助网页优化资源的加载。这些指令用于`<link>`标签中，可以用来加载图像、css、js和字体等关键资源。

### preload

```html
<link rel="prelaod" as="script" href="https://xxx.xxx.com/xxx.js">
```

​	指定了preload的link标签将会告诉浏览器，提高该资源的加载优先级，必须而且要提前进行资源加载，不管该资源是否被使用。同时，preload不会阻塞页面的渲染过程，避免了下载资源对页面渲染造成的延迟。preload主要用于网页必须的关键资源加载，对于不确定是否使用的资源，使用preload可能会造成带宽的浪费，以及性能损耗。

### prefetch

```html
<link rel="prefetch" as="script" href="https://xxx.xxx.com/xxx.js">
```

​	与preload不同，prefetch是一个低优先级的资源提示，它的作用是告诉浏览器加载可能会用到的资源，比如其他网页、继续滚动才会加载的资源等等。

​	prefetch有prefetch和dns-prefetch两种

#### prefetch

```html
<link rel="prefetch" href="https://fonts.gstatic.com/" >
```

​	提前加载未来可能会用到的资源，浏览器将会在空闲时获取资源，获取完成后，将会存储在浏览器缓存中，等到真正使用时，直接从内存中读取即可。

#### dns-prefetch

```html
<link rel="dns-prefetch" href="https://fonts.gstatic.com/" >
```

​	标识了dns-prefetch的资源，在真正获取之前，将会提前进行dns解析，可以加快请求的速度。一般dns-prefetch针对的是跨域资源，同域资源其实是无效的。道理也很简单，同域的资源在请求html页面的时候已经解析完成了，所以dns-prefetch一般用于CDN、以及请求第三方资源，比如google font、google analytics等。

### preconnect

```html
<link rel="preconnect" href="https://cdn.example.com">
```

​	preconnect 允许浏览器在一个 HTTP 请求正式发给服务器前预先执行一些操作，这包括 DNS 解析，TLS 协商，TCP 握手，这消除了往返延迟并为用户节省了时间。可以理解为preconnect是升级版的dns-prefetch，预执行更多动作，同时也消耗更多的性能，请谨慎使用。

### prerender

```html
<link rel="prerender" href="http://example.com">
```

​	prerender，也就是预渲染，将会下载完整的网页资源，然后在后台进行渲染，这是会创建DOM结构，执行CSS和JavaScript，结果将会被放置在内存中。注意此指令将会加载链接包含的所有资源，会消耗大量的网络带宽、内存和cpu，建议只用于用户肯定会访问的页面链接。

