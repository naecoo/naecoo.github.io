+++
title = "Vue组件复用的一些问题"
date = "2019-04-18"
author = "naeco"
[taxonomies]
tags = ["vue"]
+++

### 前言
最近在做一些需求的时候，发现了Vue中组件复用令的一些问题，最终经过一番折腾才把问题解决，所以想把问题和解决思路总结一下，以供参考。
### 正文
假设我们有这样一个需求，两个tab切换控制需要显示的内容，显示的内容里面都有一个多选按钮组件，但是它们数量和内容都不一样。乍一看是个很简单的需求，我们首先把多选按钮单独写一个组件，再根据不同的tab切换传入props：
```javascript
<CheckBox :checkProps="currentProps" @save="save" />
```
类似这样的组件，根据不同的tab我们修改currentProps来显示不同的多选选项，选项勾选完成后，CheckBox组件会派发一个save的事件并返回被勾选的选项数组。代码很优雅对吧，但是问题还远远没有解决，经过测试我们发现切换不同的tabs，CheckBox返回的结果会复用。由于我们是在CheckBox的data中声明的数组，这意味着即使改变了Props，导致CheckBox重新渲染，但是data属性会被复用，所以导致了一些选项乱入的问题。那我们怎么去解决呢，为了省时间，没去深究里面的原理，所以直接换成v-if，代码如下：
```html
<CheckBox :checkProps="fruits" @save="save" v-if="isFruit" />
<CheckBox :checkProps="animal" @save="save" v-else />
```
我们用isFruit这个布尔变量去切换显示fruits多选列表还是animal列表，但是经过测试，发现这个方法也行不通。save事件返回的勾选结果还是会混淆，说明CheckBox的data还是被重用了。最后迫不得已，我们改成v-show指令：
```javascript
<CheckBox :checkProps="fruits" @save="save" v-show="isFruit" />
<CheckBox :checkProps="animal" @save="save" v-show="!isFruit" />
...
save (checkList) {
  console.log(checkList)      // ["苹果", "西瓜", "香蕉"] || ["大象", "老虎", "熊猫"] 终于不会混淆了😄😄😄
}
```
情况很意外，这样就行得通了，终于可以愉快的进行开发了。
### 原理
虽然莫名其妙地把问题解决了，但我还是很想了解导致这个问题的原因和具体的解决方法。经过一阵子的摸索和实践，终于让我发现了问题所在和解决办法。
其实问题就出在Vue的diff算法中，我们知道Vue在2.0引入了虚拟DOM的概念，使得我们可以不直接操作dom，只操作数据进行页面渲染。在虚拟DOM的后面就有diff算法，diff只会比较同一层次的节点，如果同一个层次节点类型相同的话，会重新设置该节点的属性，从而实现节点的更新。所以问题应该就出在这里，不管是动态改变props还是v-if，其实前后渲染的节点都属于同一层次，所以Vue把它们当成同一类，只是改变了节点的属性，节点内部的data会被复用。至于v-show, 我们都知道v-if和v-show的最大区别就是无论条件是否为真，v-show节点都会被渲染，控制节点显示的方式是`display:none`，所以实际上v-show控制的两个CheckBox组件都已经被渲染了，它们内部的data也是独立的，不会造成混淆。
### 解决方案
知道了问题所在的原因，那么解决这个问题就很好办了。当然，如果你想直接用v-show这个指令来控制，那这里就可以不用看，直接跳过了。其实解决方法很简单，在我们使用v-for这个指令的时候，vue会强制我们同时绑定key属性，不知道大家有没有思考过背后的原因。其实道理是一样的，在同一层次的节点中，diff算法需要这个key去区分不同的节点，所以我们只需要绑定一个key就可以解决问题了。
```html
  <CheckBox :checkProps="currentProps" @save="save" :key="isFruit ? 'fruit' : 'animal'" />
  // or
  <CheckBox :checkProps="fruits" @save="save" v-if="isFruit" key="fruit"/>
  <CheckBox :checkProps="animal" @save="save" v-else key="animal" />

```
大功告成，终于解决问题了。所以，如果需要在同一层次复用组件，需要格外注意组件内部属性复用的问题，最安全的办法就是给不同的组件都绑定key属性，让Vue能区分开它们，这样不仅仅能提高diff算法的效率，而且保证了不会发生冲突。
### 扩展
在使用列表渲染的时候，我们都会用到v-for这个指令，vue会强制要求绑定key属性，很多开发者都会传入数组的index属性。这种行为其实是很危险的，通过上述分析我们知道Vue中是通过key属性标识同一层次的相同类型节点的，如果列表数组发生改变，比如删除和排序等操作。index发生改变之后，相应的key也会发生改变，所以同一个组件可能会有不同的key值，这很可能会导致组件内部属性混淆，发生不可控行为。为了避免这种行为，最好是把key当做数据库里面的主键，传入一个唯一的不可变的标识符。