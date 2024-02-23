+++
title = "实现一个esbuild插件"
date = "2021-03-21"
author = "naeco"
[taxonomies]
tags = ["javascript", "esbuild"]
+++



​	[esbuild](https://esbuild.github.io/)是由`Go`编写的构建打包工具，对标的是`webpack`、`rollup`和`parcel`等工具，在静态语言的加持下，`esbuild`的构建速度可以是传统`js`构建工具的10-100倍，就好像跑车和自行车的区别。相对于`webpack`等工具，`esbuild`相对比较纯粹，配置也很简单，换句话说，支持的功能还不是很全面，目前还不适合用于大型的项目工程。但由于性能上的优势，`vite`和`snowpack`等`esm`构建工具都采用了esbuild作为底层支持。



### esbuild插件

​	`esbuild`之前被人所诟病的一点就是缺少插件的支持，很多功能都没办法实现，好在在`0.8.x`版本后，官方终于推出了插件的支持，目前依然是实验性的一个特性，不排除未来会对API作出改变。但这不影响我们开发插件，因为`esbuild`的插件API非常简单，即使会有变动，后续迁移的成本也不会非常高。

​	`esbuild` 插件就是一个对象，里面有`name`和`setup`两个属性，`name`是插件的名称，`setup`是一个函数，构建的时候会执行，插件的逻辑也封装在其中。以下是一个简单的`esbuild`插件示例：

```js
let envPlugin = {
  name: 'env',
  setup(build) {
     // 文件解析时触发
    // 将插件作用域限定于env文件，并为其标识命名空间"env-ns"
    build.onResolve({ filter: /^env$/ }, args => ({
      path: args.path,
      namespace: 'env-ns',
    }))

    // 加载文件时触发
    // 只有命名空间为"env-ns"的文件才会被处理
    // 将process.env对象反序列化为字符串并交由json-loader处理
    build.onLoad({ filter: /.*/, namespace: 'env-ns' }, () => ({
      contents: JSON.stringify(process.env),
      loader: 'json',
    }))
  },
}

require('esbuild').build({
  entryPoints: ['app.js'],
  bundle: true,
  outfile: 'out.js',
  // 应用插件
  plugins: [envPlugin],
}).catch(() => process.exit(1))

```

```javascript
// 应用了env插件后，构建时将会被替换成process.env对象
import { PATH } from 'env'
console.log(`PATH is ${PATH}`)
```

​	可以看到，`esbuild`插件实现还是非常简单的，只需要在`setup`函数中注册两个钩子函数，然后再添加相对应的代码逻辑即可，关于`esbuild`插件API的介绍可以查询官方的[文档](https://esbuild.github.io/plugins/)。



### esbuild-plugin-replace实现

​	先把成品放出来，[esbuild-plugin-replace](https://github.com/naecoo/esbuild-plugin-replace), 欢迎提issue和pr，顺手点个star就更好了😎。`esbuild-plugin-replace`这个插件作用是在构建时替换代码里的字符，主要用于动态更新代码的一些变量，比如版本号，构建时间，构建的`git`信息等。

​	由于代码数不多，只有62行，所以下面直接将全部代码贴上来:

```javascript

const fs = require('fs');
const MagicString = require('magic-string');

// 替换内容可以是函数或原始值，但统一封装成函数，方便处理
const toFunction = (functionOrValue) => {
  if (typeof functionOrValue === 'function') return functionOrValue;
  return () => functionOrValue;
}

const longest = (a, b) => b.length - a.length;
// 将配置中的替换选项和替换内容提取出来
const mapToFunctions = (options) => {
  const values = options.values ? Object.assign({}, options.values) : Object.assign({}, options);
  delete values.include;
  return Object.keys(values).reduce((fns, key) => {
    const functions = Object.assign({}, fns);
    functions[key] = toFunction(values[key]);
    return functions;
  }, {});
}

// 生成esbuild的filter，其实就是一个正则表达式
const generateFilter = (options) => {
  let filter = /.*/;
  if (options.include) {
    if (Object.prototype.toString.call(options.include) !== '[object RegExp]') {
      console.warn(`Options.include must be a RegExp object, but gets an '${typeof options.include}' type.`);
    } else {
      filter = options.include
    }
  }
  return filter;
}

// 核心函数，匹配代码中的字符串，用配置中的替换内容去替换
const replaceCode = (code, id, pattern, functionValues) => {
  // 这里用了magic-string这个库，方便对字符串进行处理
  const magicString = new MagicString(code);
  // 正则匹配
  while ((match = pattern.exec(code))) {
    // 获取匹配中的字符的索引
    const start = match.index;
    const end = start + match[0].length;
    // 获取要替换内容
    const replacement = String(functionValues[match[1]](id));
    // 字符串替换
    magicString.overwrite(start, end, replacement);
  }
  // 返回处理后的内容
  return magicString.toString();
}

// 插件工厂函数
exports.replace = (options = {}) => {
  // 根据include选项生成filter配置
  const filter = generateFilter(options);
  // 得到要replace的key和value对象，注意对象是函数
  const functionValues = mapToFunctions(options);
  const empty = Object.keys(functionValues).length === 0;
  // 获取对象的key，并进行排序和转义
  const keys = Object.keys(functionValues).sort(longest).map(escape);
  // 将所有key构建成一个正则表达式，用于匹配源代码
  const pattern = new RegExp(`\\b(${keys.join('|')})\\b`, 'g');
  // 返回插件
  return {
    name: 'replace',
    setup(build) {
      // 注册onLoad钩子，解析文件时将会引入
      build.onLoad({ filter }, async (args) => {
        // 首先获取源代码内容
        const source = await fs.promises.readFile(args.path, "utf8");
        // 进行replace
        const contents = empty ? source : replaceCode(source, args.path, pattern, functionValues)
        // 返回转化后代码字符串，供esbuild处理
        return { contents };
      });
    }
  };
}
module.exports = exports;
```

​	简单总结一下, `esbuild-plugin-replace`的核心逻辑就是根据用户的配置项key生成一个正则表达式，然后去匹配源代码，然后再用配置项的内容替换掉命中的字符，这里字符串操作用了[magic-string](https://www.npmjs.com/package/magic-string)这个库，非常好用，推荐一下。然后，这个插件用法也很简单：

```javascript
const { build } = require('esbuild');
const { replace } = require('esbuild-plugin-replace');

build({
  // 其他构建选项...
  plugins: [
    replace({
      '__author__': JSON.stringify('naecoo'),
      '__version__': JSON.stringify('1.0.0')
    })
  ]  
})
```

如果你的代码是这样:

```javascript
const debugInfo = {
  author: __author__,
  version: __version
}
```

构建后，将会变成：

```javascript
const debugInfo = {
  author: "naeco",
  version: "1.0.0"
}
```



### 题外话

​	esbuild的插件书写相对来说还是比较简单的，但值得注意一点的是，在构建过程中，不要过度使用插件，特别是用`js`编写的插件，因为会严重影响构建的性能，如果一定要用，请尽可能配置filter，将插件的作用域范围降至最小。同时，由于`esbuild`出的时间不算太久，很多工具和生态都不是很完善，如果要引入`esbuild`，很可能要开发人员自己手写一部分的插件，希望这篇文章可以帮助到你，也希望大家可以积极参与`esbuild`的生态，贡献更多优秀的代码。

