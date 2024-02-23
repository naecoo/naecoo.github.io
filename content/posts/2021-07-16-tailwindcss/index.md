+++
title = "对tailwindcss的一些简单看法"
date = "2021-07-16"
author = "naeco"
[taxonomies]
tags = ["css"]
+++





>本文是笔者对于Tailwind的一些简单看法，并不涉及Tailwind的介绍和学习等内容，对于后者而言，网上的资源是非常丰富的，本文不会复述这些内容。

## 前言

[Tailwind]([tailwindcss.com](https://tailwindcss.com/))是一个用于快速开发界面，原子类优先的css框架，在过去的一年里非常流行，网上的讨论也十分多。如果你没听说过Tailwind和原子css，我们可以看一个例子：

```html
<div class="bg-gray-100 flex-1 p-8 w-1 h-1">Hello World</div>
```

Tailwind提供了包含多个预定义类的css集合，开发者不再需要编写CSS，而是在HTML中直接引用预先定义好的类名，上述的例子就采用了这些类名：

- 背景 (bg-gray-100)
- flex布局 (flex-1)
- 内边距 (p-8)
- 尺寸 (w-1, h-1)

Tailwind提供了成千上万的预设类名，尽可能包括了所有基础css特性，同时提供了响应式、hover和focus状态、深色模式等等特性。所以开发者不再需要管理css，只需要像乐高积木那样，通过丰富的类名，组合成自己想要的效果



## 优点

### 可定制化程度高

如果不喜欢Tailwind的默认配置，开发者可以在tailwind.config.js文件中修改，定制主题、响应式断点、颜色、间距等等的配置，甚至，还可以引入第三方的插件和预设，覆盖默认配置。

### 不需要管理CSS样式

使用了Taliwind之后，你不再需要编写新的css，也不用再管理css，不用在为class的起名感到烦恼。

### 提高了开发的效率

Taliwind提供了丰富的原子class，有点像乐高积木，开发者可以通过这些class拼装自己的模型，而不用再手动编写css样式。

### 丰富的内置特性

Taliwind内置了许多有用的特性，比如响应式、动画和深色模式等等，开发者可以基于这些特性快速地开发出UI页面。如果不使用Tailwind，自己手动编写css代码，复杂度会相当高。

### 减少css文件体积

Taliwind提供了purge的功能，也就是tree-shaking，可以清除未使用的类名，一般的项目只有10kb左右的css文件。



## 缺点

### 不美观

```html
<div class="min-w-0 flex-auto space-y-0.5">
  <p class="text-lime-600 dark:text-lime-400 text-sm sm:text-base lg:text-sm xl:text-base font-semibold uppercase">
    <abbr title="Episode">Ep.</abbr> 128
  </p>
  <h2 class="text-black dark:text-white text-base sm:text-xl lg:text-base xl:text-xl font-semibold truncate">
    Scaling CSS at Heroku with Utility Classes
  </h2>
  <p class="text-gray-500 dark:text-gray-400 text-base sm:text-lg lg:text-base xl:text-lg font-medium">
    Full Stack Radio
  </p>
</div>
```

使用了Taliwind的标签看起来非常凌乱，丑陋...

### 需要对UI设计规范有一定的理解

前面我们提到Taliwind的可定制化程度高，可以通过配置文件生成不同的预设，但是前提是UI设计要有一定的规范，否则很容易出现预设值无法覆盖所有尺寸的问题，比如设计师让一张图片圆角设为3px，这时候可能需要单独编写css了，但这也违反了Taliwind的原则。所以，使用Taliwind前，必须要有一套完善的UI设计规范，尽可能保证预设值能覆盖所有属性。

### 有一定的上手成本

尽管Taliwind提供了Tailwind CSS IntelliSense插件，可以帮助开发者输入Tailwind类名，但是Tailwind有成千上百个关键词，对于新人来说仍然有一定的上手成本。同时，Tailwind在不断的迭代，关键词也会出现变动，开发者需要保持关注。

### 增加了维护成本

因为每个class都是独立的，如果要修改，每一个都要去修改，大大增加了维护成本。



## 总结

总体来说，尽管存在着那么多的缺点，但是瑕不掩瑜，Tailwind仍然是一个十分优秀的css框架，超过800K的周下载量可以说明这一点。

![image-20210725144720847](https://image.naeco.top/blog/1627195641622.png)

我的建议是，一方面，对于UI还原没那么重视的项目可以用起来了，比如to B行业的项目、公司内部的项目、私人小项目等等，使用Tailwind开发，可以极大的提高效率；另一方面，Tailwind可以作为UI框架的基础css框架，结合React、Vue、甚至是Web Components等组件化技术，不仅提高了开发效率，而且可以最大化压缩UI框架体积，提高极限性能。
