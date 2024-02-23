+++
title = "懒加载原理"
date = "2019-08-26"
author = "naeco"
[taxonomies]
tags = ["javascript"]
+++


> 特别声明：此篇文章内容来源于[@Jeremy Wagner](https://developers.google.com/web/resources/contributors/jeremywagner)
的[《Lazy Loading Images and Video》](https://developers.google.com/web/fundamentals/performance/lazy-loading-guidance/images-and-video/)一文。著作权归作者所有。

作为网页内容的一部分，图像和视频通常要消耗很多资源加载。要提高网页应用的性能，如何避免资源浪费在加载图像和视频上就很重要了。但是，很多时候我们都不愿意减少网页上的媒体资源，所以我们经常无从下手。幸运的是，我们有懒加载这个绝招，它可以帮助我们减少加载时间和降低负载，而不在内容上偷工减料。

### 什么是懒加载？
懒加载是一种在页面加载时延迟加载一些非关键资源的技术，换句话说就是按需加载。对于图片来说，非关键通常意味着离屏。如果你有使用过Lighthouse并且做过一些性能调优，你可能已经见过一些离屏图片的应用。[(offscreen-images)](https://developers.google.com/web/tools/lighthouse/audits/offscreen-images)![](https://upload-images.jianshu.io/upload_images/5070211-a85350f6ac063ddc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们之前看到的懒加载一般是这样的形式：
1. 浏览一个网页，准备往下拖动滚动条
2. 拖动一个占位图片到视窗
3. 占位图片被瞬间替换成最终的图片

我们可以在[Medium](https://medium.com)中看到懒加载是如何使用的，网页首先用一张轻量级的图片占位，当占位图片被拖动到视窗，瞬间加载目标图片，然后替换占位图片。
![](https://upload-images.jianshu.io/upload_images/5070211-b4120bdbd5933573.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果你不是很熟悉懒加载技术，你或许会疑问它有什么用，能为我们带来什么好处，下面我们将会探讨这个问题。

### 为什么要懒加载而不直接加载？
- 浪费流量。在不计流量收费的网络，这可能不重要；在按流量收费的网络中，毫无疑问，一次性加载大量图片就是在浪费用户的钱。
- 消耗额外的电量和其他的系统资源，并且延长了浏览器解析的时间。因为媒体资源在被下载完成后，浏览器必须对它进行解码，然后渲染在视窗上，这些操作都需要一定的时间。

懒加载图片和视频，可以减少页面加载的时间、页面的大小和降低系统资源的占用，这些对于性能都有显著地提升。在这里，我们将会提到一些懒加载技术和使用方法，还有一些常用的[懒加载库](https://developers.google.com/web/fundamentals/performance/lazy-loading-guidance/images-and-video/#lazy_loading_libraries)。

### 懒加载图片
图片懒加载在技术上实现很简单，不过对于细节要求比较严格。目前有很多实现懒加载的方法，先从懒加载内联图片说起吧。
###### 内联图片
最常见的懒加载方式就是利用<img>标签。懒加载图片时，我们利用JavaScript检查<img>标签是否在视窗中。如果在，<img>的src（有时候是srcset）就会设置为目标图片的url。
###### 利用intersection observer
如果你之前用过懒加载，你很可能是通过监听一些事件比如scroll或者resize来检测元素出现在视窗，这种方法很成熟，能够兼容大部分的浏览器。但是，现代浏览器提供了一个更好的方法给我们([the intersection observer API](https://developers.google.com/web/updates/2016/04/intersectionobserver))
>*注意：Intersection observer目前只能在Chrome63+和firefox58+使用*

比起事件监听，IntersectionObserver用起来比较简单，可阅读性也大大提高。开发者只需要注册一个observer去监控元素而不是写一大堆乱七八糟的视窗检测代码。注册observer之后我们只需要做的就是当元素可见时改变它的行为。举个例子吧：
```html
<img class="lazy" src="placeholder-image.jpg" data-src="image-to-lazy-load-1x.jpg" data-srcset="image-to-lazy-load-2x.jpg 2x, image-to-lazy-load-1x.jpg 1x" alt="I'm an image!">
```
需要注意三个相关的属性
- class：用于在JavaScript中关联元素
- src属性：指向了一张占位图片，图片在页面的第一次加载会显现
- data-src和data-srcset属性：这是占位属性，里面放的是目标图片的url

ok，看一下怎么在JavaScript中使用Intersection observer吧：
```javascript
document.addEventListener("DOMContentLoaded", function() {
  var lazyImages = [].slice.call(document.querySelectorAll("img.lazy"));

  if ("IntersectionObserver" in window) {
    let lazyImageObserver = new IntersectionObserver(function(entries, observer) {
      entries.forEach(function(entry) {
        if (entry.isIntersecting) {
          let lazyImage = entry.target;
          lazyImage.src = lazyImage.dataset.src;
          lazyImage.srcset = lazyImage.dataset.srcset;
          lazyImage.classList.remove("lazy");
          lazyImageObserver.unobserve(lazyImage);
        }
      });
    });

    lazyImages.forEach(function(lazyImage) {
      lazyImageObserver.observe(lazyImage);
    });
  } else {
    // Possibly fall back to a more compatible method here
  }
});
```
当DOMContentLoaded触发后，js会查询class为lazy的img元素。然后我们检测浏览器支不支持intersection observer，如果可以用，先创建一个observer，然后传入回调函数，回调函数将会在元素可见性变化时被调用。具体的代码可以在[这里](https://codepen.io/malchata/pen/YeMyrQ)查看。
最后比较麻烦的是处理兼容性，在不支持intersection observer的浏览器，你需要引入[polyfill](https://github.com/w3c/IntersectionObserver/tree/master/polyfill)，或者回退到更安全的方法。

###### 利用事件
当你选择使用intersection observer来实现懒加载时，你要考虑它的兼容性，当然你可以使用polyfill，实际上这也非常简单。事实上你也可以针对低版本的浏览器使用事件来完成更安全地回退。你可以使用scroll、resize和orientationchange事件，再配合getBoundingClientRectAPI就可以实现懒加载了。
和上面一样的例子，现在JavaScript程序变成了这样：
```javascript
document.addEventListener("DOMContentLoaded", function() {
  let lazyImages = [].slice.call(document.querySelectorAll("img.lazy"));
  let active = false;

  const lazyLoad = function() {
    if (active === false) {
      active = true;

      setTimeout(function() {
        lazyImages.forEach(function(lazyImage) {
          if ((lazyImage.getBoundingClientRect().top <= window.innerHeight && lazyImage.getBoundingClientRect().bottom >= 0) && getComputedStyle(lazyImage).display !== "none") {
            lazyImage.src = lazyImage.dataset.src;
            lazyImage.srcset = lazyImage.dataset.srcset;
            lazyImage.classList.remove("lazy");

            lazyImages = lazyImages.filter(function(image) {
              return image !== lazyImage;
            });

            if (lazyImages.length === 0) {
              document.removeEventListener("scroll", lazyLoad);
              window.removeEventListener("resize", lazyLoad);
              window.removeEventListener("orientationchange", lazyLoad);
            }
          }
        });

        active = false;
      }, 200);
    }
  };

  document.addEventListener("scroll", lazyLoad);
  window.addEventListener("resize", lazyLoad);
  window.addEventListener("orientationchange", lazyLoad);
});
```
上面的代码用了getBoundingClientRect，在scroll事件中检测img是否在视窗。setTimeout用于延迟执行操作，active变量代表了处理状态防止同时响应。当图片被懒加载完成后，事件处理程序将被移除，具体请看[这里](https://codepen.io/malchata/pen/mXoZGx)。
尽管上面这段代码可以在绝大部分的浏览器上运行，但存在显著的性能损耗。在此示例中，无论在视口中是否存在图像，文档滚动或窗口大小调整时都会每200毫秒执行一次检查。 另外，跟踪有多少元素留给延迟加载和解除事件处理程序的繁琐工作也留给了开发者。
>*建议：尽可能使用intersection observer，如果应用要求兼容低版本的浏览器才考虑利用事件*
###### CSS图像
展示图像不是<img>标签的特权，CSS利用background-image也可以做到。相比较而言，CSS加载图片比较容易控制。当文档对象模型、CSS对象模型和渲染树被构造完成后，开始请求外部资源之前，浏览器会检测CSS规则是怎么应用到DOM上的。如果浏览器检测到CSS引用的外部资源并没有应用到已存在的DOM节点上，浏览器就不会请求这些资源。
这个行为可用于延迟CSS图片资源的加载，思路是通过JavaScript检测到元素处于视窗中时，加一个class类名，这个class就引用了外部图片资源。
这可以实现图片按需加载而不是一次性全部加载。给个例子：
```javascript
<div class="lazy-background">
  <h1>Here's a hero heading to get your attention!</h1>
  <p>Here's hero copy to convince you to buy a thing!</p>
  <a href="/buy-a-thing">Buy a thing!</a>
</div>
```
这个div.lazy-background元素会正常地显示CSS规则加载的占位图片。当元素处于可见状态时，我们可以添加一个类名完成懒加载：
```css
.lazy-background {
  background-image: url("hero-placeholder.jpg"); /* 占位图片 */
}

.lazy-background.visible {
  background-image: url("hero.jpg"); /* 真正的图片 */
}
```
下面是利用JavaScript去检测元素是否处于视窗（intersection observer），如果可见就为它加上一个visible的类名。
```javascript
document.addEventListener("DOMContentLoaded", function() {
  var lazyBackgrounds = [].slice.call(document.querySelectorAll(".lazy-background"));

  if ("IntersectionObserver" in window) {
    let lazyBackgroundObserver = new IntersectionObserver(function(entries, observer) {
      entries.forEach(function(entry) {
        if (entry.isIntersecting) {
          entry.target.classList.add("visible");
          lazyBackgroundObserver.unobserve(entry.target);
        }
      });
    });

    lazyBackgrounds.forEach(function(lazyBackground) {
      lazyBackgroundObserver.observe(lazyBackground);
    });
  }
});
```
### 懒加载视频
就像图片一样，我们同样可以懒加载视频，播放视频会用到<video>标签。如何懒加载视频取决于特定的场景，先来讨论几个需要不同解决方案的场景。

###### 视频不需要自动播放

```
<video controls preload="none" poster="one-does-not-simply-placeholder.jpg">
  <source src="one-does-not-simply.webm" type="video/webm">
  <source src="one-does-not-simply.mp4" type="video/mp4">
</video>
```
我们还需要添加一个poster属性给<video>标签，这相当于一个占位符。preload属性则规定是否在页面加载后载入视频。鉴于浏览器之间的preload默认值差异，显式定义会更具兼容性。在这种情况下，当用户点击播放视频时，视频才会被加载，预加载视频简单地实现了。不幸的是，当我们想用视频替代GIF动画时，这个方法就行不通了。

###### 用视频模拟GIF
GIF在很多地方都不及视频，特别是文件大小方面。在相同质量下，视频的尺寸通常会比GIF文件小得多。当然，利用视频取代GIF并不是直接用<video>标签取代<img>标签那么简单。因为GIF图片有三种要注意的行为：
1. 加载完后自动播放
2. 不停地循环播放
3. 没有音轨
要实现这些，HTML是这样的：
```html
<video autoplay muted loop playsinline>
  <source src="one-does-not-simply.webm" type="video/webm">
  <source src="one-does-not-simply.mp4" type="video/mp4">
</video>
```
autoplay、muted和loop的作用是为了实现上述三个功能，playsinline是为了兼容IOS的autoplay。现在我们已经有了一个跨平台的视频模版用于取代GIF图片了。接下来怎么进行懒加载呢？Chrome会帮我们自动完成这项工作，但你不能保证所有浏览器都能做到这个。所以，还是动手实现一下吧。首先，修改<video>标签的属性：
```html
<video autoplay muted loop playsinline width="610" height="254" poster="one-does-not-simply.jpg">
  <source data-src="one-does-not-simply.webm" type="video/webm">
  <source data-src="one-does-not-simply.mp4" type="video/mp4">
</video>
```
注意到了吗？有一个奇怪的poster属性。这个属性其实是一个占位符，在被懒加载之前，poster里面指定的内容会在<video>标签中显现。和之前的图片懒加载一样，我们指定真正的video url藏于每个<source>的data-src中。下一步，JavaScript程序该出场了：
```javascript
document.addEventListener("DOMContentLoaded", function() {
  var lazyVideos = [].slice.call(document.querySelectorAll("video.lazy"));

  if ("IntersectionObserver" in window) {
    var lazyVideoObserver = new IntersectionObserver(function(entries, observer) {
      entries.forEach(function(video) {
        if (video.isIntersecting) {
          for (var source in video.target.children) {
            var videoSource = video.target.children[source];
            if (typeof videoSource.tagName === "string" && videoSource.tagName === "SOURCE") {
              videoSource.src = videoSource.dataset.src;
            }
          }

          video.target.load();
          video.target.classList.remove("lazy");
          lazyVideoObserver.unobserve(video.target);
        }
      });
    });

    lazyVideos.forEach(function(lazyVideo) {
      lazyVideoObserver.observe(lazyVideo);
    });
  }
});

当懒加载一个视频的时，首先要迭代<video>标签里面的每一个<source>，然后将data-src中的url分配给src属性。然后调用元素的load方法，现在视频就可以自动播放了。
通过这个方法，我们有了一个模拟GIF动画的视频解决方案，不会消耗带宽加载不必要的媒体资源，而且还能实现懒加载。
```
### 懒加载库
如果你不关心懒加载背后是如何实现的，你只是想找一个库去实现这个功能，可供选择的有：
- [lazysizes](https://github.com/aFarkas/lazysizes) 是一个功能十分强大的懒加载库，主要用于加载图片和iframes。你只需要指定data-src/data-srcset属性，lazysizes会帮你自动懒加载内容。值得注意的是，lazysizes基于intersection observer，因此你需要一个polyfill。你还可以通过一些插件扩展库的功能以用于懒加载视频。
-  [lozad.js](https://github.com/ApoorvSaxena/lozad.js)是一个轻量级、高性能的懒加载库，基于intersection observer，你同样需要提供一个相关的polyfill。
- [blazy](https://github.com/dinbror/blazy)是一个轻量级的懒加载库，大小仅为1.4KB。相对于lazysizes，它不需要任何的外部依赖，并且兼容IE7+。你可能猜测到了，blazy不支持intersection observer，性能相对较差。
- [yall.js](https://github.com/malchata/yall.js)是作者本人写的一个懒加载库，基于IntersectionObserver和事件，兼容IE11和大部分的浏览器。
- 如果你想寻找一个基于React的懒加载工具，[react-lazyload](https://github.com/jasonslyvia/react-lazyload)可能是你的选择。

上述每个懒加载库的文档都写得很好，同时提供了大量的标记模式。如果你不想深究懒加载的技术细节，就选择任意一个去使用，这能节省你很多的时间和功夫。

### 容易出错的地方
看到有那么多的库可以实现懒加载，你可能会以为这是一项很轻松的工作。但是，懒加载一旦出现错误，会导致意想不到的后果。为了避免出错，下面的建议你最好熟读于心：

###### 布局偏移和占位符
如果你没有使用占位符，懒加载会导致页面布局的偏移。除了让用户感到困惑之外，还会导致不必要的浏览器reflow，性能大幅下降。因此，你至少也要使用一张固定的图片填充img标签，或者使用像[LQIP](http://www.guypo.com/introducing-lqip-low-quality-image-placeholders/)和[SQIP](https://github.com/technopagan/sqip)这样的技术在加载之前提示媒体资源的内容。
对于<img>标签，src属性初始化应该指向一张占位图片，最终会被替换成目标图片。对于<video>，可以使用poster属性指定占位符。除此之外，对于<img>和<video>来说，显式声明其width和height属性都是十分必要的，这可以保证从占位符切换到目标资源的过程中不会导致浏览器reflow。
###### 延迟图片解码
用JavaScript加载一些比较大的图片会阻塞线程，导致网页短暂地失去交互能力。如果你不乐意这样的情况出现，用decode方法异步解码图片是一个很好的选择，这可以避免阻塞线程，下面展示一下例子：
```javascript
var newImage = new Image();
newImage.src = "my-awesome-image.jpg";

if ("decode" in newImage) {
  // Fancy decoding logic
  newImage.decode().then(function() {
    imageContainer.appendChild(newImage);
  });
} else {
  // Regular image load
  imageContainer.appendChild(newImage);
}
```
上面的代码主要是用了Image.decode()方法，具体的请看清这里([this CodePen link](https://codepen.io/malchata/pen/WzeZGW))。如果你的图片不是太大，可以选择直接同步加载。
###### 加载失败
有时候媒体资源会加载失败，假象有这种情况：一个网页设置了HTML短时间的缓存（大概5分钟），然后用户打开了一个tab几小时后再浏览这个网页。在这个过程的某个时间，缓存会重新部署，基于hash的版本号会改变或者丢失。如果这时候懒加载图片，会导致失败。
虽然这种情况有点极端，但你必须要有策略来应对图片加载失败这种状况。比如你可以用一个按钮代替图片填充，用户可以点击这个按钮重新加载图片，或者提示用户发生了错误。
无论多小概率的错误可能会出现，所以说不管怎样，及时提示用户网页加载错误或者提供一个重新加载的按钮总是值得的。

### 总结
懒加载可以减少页面加载的时间、降低页面负载。在用户浏览的时候，网页不会加载那些用户看不到的内容，但如果用户愿意，用户依然可以正常地浏览这些内容。
就改进性能方面，懒加载是无可争议的，合理的一项技术。如果网页应用中出现了大量的图片，懒加载可以完美地限制不必要的加载，对于用户体验，这是巨大的提升，相信我，你的用户和老板会感谢你的。

---