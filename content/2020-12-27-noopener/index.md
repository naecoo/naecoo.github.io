+++
title = "noopener、noreferrer和nofollow的作用"
date = "2020-12-27"
author = "naeco"
[taxonomies]
tags = ["html"]
+++

### 简介

​   HTML中，a标签可以设置多个rel属性，其中包括noopener、noreferrer和nofollow，相信很多人这三个属性都不太了解，但是这三个属性关乎到网页的安全性和SEO等方面，每个开发者都应该了解其作用。



### noopener

```html
<a href="https://www.baidu.com" target="_blank">baidu</a>
```

​   当a标签设置了`target="_blank"`，点击链接，浏览器将会打开一个新的tab页，但是由于两个网页依然存在关联性，很多开发者都会忽略其中的安全和性能问题：

1. 新打开的网页可能与原网页运行在同一进程，如果新网页运行大量的js脚本，原网页性能也会受到影响
2. 新打开的网页可以通过`window.opener`属性访问原网页，所以新网页可以将原网页重定向到其他URL，比如一些黑客会将网页重定向到一个钓鱼网站，来获取用户的私密信息。
  通过设置`rel="noopener"`属性可以避免这个问题，所以，a标签每次设置`target="_blank"`时，都必须设置rel="noopener"：

```html
<a href="https://www.baidu.com" target="_blank" rel="noopener">baidu</a>
```



### noreferrer

​	该属性和noopener是一样的作用，你可以挑选任意一个。但有一点不同的是，noreferrer会设置请求头为`referrer=no-referrer`，同时，noreferrer可以得到更低版本浏览器的支持。

```html
<a href="https://www.baidu.com" target="_blank" rel="noopener">baidu</a>
<!-- 或者 -->
<a href="https://www.baidu.com" target="_blank" rel="noreferrer">baidu</a>
<!-- 也可以这样，最佳实践 -->
<a href="https://www.baidu.com" target="_blank" rel="noreferrer noopener">baidu</a>
```

​	很多人可能用js打开新网页，而不是a标签，比如利用以下js方法：

```javascript
window.open('https://www.baidu.com', '_blank')
```

但是这个方法并没有提供设置rel属性的参数，所以，为了设置rel，可以采用这个方案：

```js
function openPage (url, target) {
    const a = document.createElement('a');
    a.href = url;
    a.target = target;
    a.rel = 'noreferrer noopener';
    a.click();
    a = null;
}
openPage('https://www.baidu.com', '_blank')
```



### nofollow

​	通过设置nofollow这个属性，我们可以告知搜索引擎不要追踪此网页上的链接或不要追踪此特定链接。这个属性会影响搜索引擎对网页的打分，所以要谨慎使用，最好是要先学习一定的SEO知识再去使用，关于更多请看参考<sup>3</sup>。



### 参考
1. [link-type-noreferrer](https://html.spec.whatwg.org/multipage/links.html#link-type-noreferrer)
2. [external-anchors-use-rel-noopener](https://web.dev/external-anchors-use-rel-noopener/)
3. [Nofollow链接 VS Follow链接：所有你需要了解的知识](https://ahrefs.com/blog/zh/nofollow-links/)



