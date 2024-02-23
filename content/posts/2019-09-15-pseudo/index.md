+++
title = "伪类和伪元素"
date = "2019-09-01"
author = "naeco"
[taxonomies]
tags = ["css"]
+++

#### 前言
最近看一些前端面试题，经常有问到伪类和伪元素的作用以及两者区别，所以特意找了一些资料学习。下面是我对这一方面知识的理解和总结，可能会有遗漏或者一些出入，欢迎大家指正，相互交流～


#### 伪类

> A psuedo-class is a selector that selects elements that are in a specific state, e.g. they are the first element of their type, or they are being hovered over by the mouse pointer. They tend to act as if you had applied a class to some part of your document, often helping you cut down on excess classes in your markup, and giving you more flexible, maintainable code.

MDN上面阐述的很明确，伪类就是一个选择处于特定状态的元素的选择器，比如某一个clsss的第一个元素，某个被hover的元素等等...，我们可以理解成一个特定的CSS类，但与普通的类不一样，它只有处于DOM树无法描述的状态下才能为元素添加样式，所以将其称为伪类


#### 伪元素

> Pseudo-elements behave in a similar way, however they act as if you had added a whole new HTML element into the markup, rather than applying a class to existing elements. Pseudo-elements start with a double colon ::

伪元素和伪类很像，但是伪元素类似于增添一个新的DOM节点到DOM树中，而不是改变元素的状态。注意了，这里是类似，而不是真的增加一个节点，这也是其被称为伪元素的原因（实质上，元素被创建在文档外）。


#### 两者区别
伪类是操作文档中已有的元素，而伪元素是创建了一个文档外的元素，两者最关键的区别就是这点。此外，为了书写CSS时进行区分，一般伪类是单冒号，如`:hover`，而伪元素是双冒号`::before`。一般大部分伪类强制使用单冒号，大部分伪元素单冒号和双冒号都可以，但是为了区分，建议按照规范书写。


#### 伪类和伪元素都有哪些？

![伪类](https://user-gold-cdn.xitu.io/2019/9/1/16ceac430d44fc54?w=594&h=537&f=png&s=50228)

![伪元素](https://user-gold-cdn.xitu.io/2019/9/1/16ceac464bbf0add?w=491&h=212&f=png&s=17507)

- 伪类

| 选择器               | 作用                                                         |
| -------------------- | ------------------------------------------------------------ |
| :avtive              | 匹配被用户激活的元素（比如点击）                             |
| :blank               | 匹配空的input                                                |
| :checked             | 匹配被选中的radio或者checkbox                                |
| :disabled            | 匹配处于不可用状态的可交互元素                               |
| :empty               | 匹配没有子元素的元素                                         |
| :enabled             | 匹配处于可用状态的可交互元素                                 |
| :first-child         | 匹配在兄弟元素中处于第一的元素                               |
| :first-of-type       | 匹配在它的兄弟元素中是某个类型中的第一个的元素               |
| :focus               | 匹配获取焦点的的元素                                         |
| :focus-visible       | 匹配获取焦点且能被用户看到的元素                             |
| :hover               | 匹配用户在此悬停或者触摸的元素                               |
| :invalid             | 匹配处于不合法状态的元素，比如`<input>`正则校验不通过        |
| :lang                | 根据文档语言匹配元素                                         |
| :last-child          | 匹配在兄弟元素中处于最后的元素                               |
| :last-of-type        | 匹配在它的兄弟元素中是某个类型中的最后一个的元素             |
| :link                | 匹配没有被访问过的链接                                       |
| :is()                | 匹配符合结果的元素                                           |
| :not()               | 匹配符合结果之外的元素                                       |
| :nth-child(n)        | 匹配父元素的第n个子元素。n可以是一个数字、一个关键字或一个公式 |
| :nth-of-type(n)      | 匹配父元素的某种类型元素中的第n个子元素。n可以是一个数字、一个关键字或一个  公式 |
| :nth-last-child(n)   | 与nth-child()类似，从后往前数                                |
| :nth-last-of-type(n) | 与nth-last-of-type-child()类似，从后往前数                   |
| :only-child          | 匹配没有兄弟元素的元素                                       |
| :only-of-type        | 匹配一个元素，该元素是其兄弟元素中唯一的一个类型。           |
| :placeholder-shown   | 匹配显示默认占位符的表单元素                                 |
| :required            | 匹配内容为必填的表单元素                                     |
| :root                | 匹配根元素                                                   |
| :valid               | 匹配处于合法状态的元素                                       |
| :target              | 匹配符合当前url的锚点元素                                    |
| :visited             | 匹配被访问过的元素                                           |

- 伪元素

| 选择器         | 作用                                   |
| -------------- | -------------------------------------- |
| ::before       | 匹配在原始元素的实际内容之后出现的区域 |
| ::after        | 匹配在原始元素的实际内容之前出现的区域 |
| ::first-letter | 匹配元素的第一个字母                   |
| ::first-line   | 匹配元素第一行                         |
| ::selection    | 匹配被选中的文本或者区域               |



#### 参考资料

- [MDN, Pseudo-classes_and_pseudo-elements](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Selectors/Pseudo-classes_and_pseudo-elements)
- [AlloyTeam, 总结伪类和伪元素](http://www.alloyteam.com/2016/05/summary-of-pseudo-classes-and-pseudo-elements/)