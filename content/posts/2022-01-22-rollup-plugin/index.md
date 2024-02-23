+++
title = "如何实现一个 Rollup 插件"
date = "2022-01-22"
author = "naeco "
[taxonomies]
tags = ["javascript"]
+++



## TLDR

直接看源码和文档

- [rollup plugin 文档](https://rollupjs.org/guide/en/#plugin-development)

- [`rollup-plugin-css2`](https://github.com/naecoo/rollup-plugin-css2)



## Rollup 介绍

[Rollup](https://rollupjs.org/guide/en/) 是面向下一代的 `javascript` 模块打包工具，相比于 [Webpack](https://github.com/webpack/webpack) 来说，Rollup 显得轻巧且灵活。过去有一种说法是库工具或者一些简单的应用才会使用 Rollup，而大型应用还是会采取 Webpack 进行构建打包。时间来到2022年，上述说法显然不再具有普遍的参考意义，越来越多的应用，甚至是大型的 App 也会采取 Rollup 来进行构建，比如 [Vite](https://github.com/vitejs/vite) 在生产环境下就放弃了 Webpack，改成 Rollup 来进行应用的构建。 对于我本人来说，一般小型的库会用 [Esbuild](https://github.com/evanw/esbuild) ，其余皆使用 Rollup。

基于这个背景下，我认为学习如何开发 Rollup 插件是非常有必要的，不管是业务还是技术上的原因，很多时候都需要在构建、编译、打包等过程中，进行定制化开发，这时候就必须要开发自己的插件了。



## Rollup 插件系统

如同 Babel 和 Webpack 等工具一样，Rollup 也有一个非常强大的插件系统，Rollup 本身提供了一个基础的构建框架，大部分功能都可以通过插件的形式实现，这也是 Rollup 的设计哲学之一。

Rollup 插件是一个对象结构，里面包含了插件的名称，`name` 字段，以及一系列钩子函数（hooks），这些钩子函数会在解析、构建、编译和打包等环节触发，帮助我们完成文件解析、代码转译等功能。

```javascript
// rollup 插件形式
const plugin = {
    // 插件名称
    name: 'some-rollup-plugin',
    
    // transform 钩子
    transofrm() {},
    
    // resolveId 钩子
    resolveId() {}
    
    ...
}
```

通常插件提供者不会直接提供插件对象本身，而是提供一个插件工厂函数，可以输入选项，生成插件对象。

```javascript
function pluginFactor(options) {
    // 解析选项
    ...
    
    // 返回插件对象
    return {
        name: 'some-rollup-plugin',
            
        ...
    }
}
    
export default pluginFactor;
```

```javascript
// rollup.config.js
import pluginFactor from 'path-to-plugin';

export default {
    input: '',
    output: {
        ...
    },
    plugins: [
        // 使用插件
        pluginFactor()
    ]
};
```

Rollup 希望插件提供者可以遵循一些插件开发的条例，这里我就不一一翻译了，感兴趣的同学直接查看[原文](https://rollupjs.org/guide/en/#conventions)吧。



### Rollup 插件 hooks

所谓 hook，中文通常叫钩子，rollup 将整体流程划分为很多种阶段，相应地触发对应的 hook 函数，比如会有 resolveId、load 、transform、moduleParsed、renderChunk、generateBundle 等等的 hooks 。

hooks 根据函数返回类型分为 sync 和 async 两种：

- sync

  hook 函数**不返回**promise

- async

  hook 函数**返回** promise


同时又有这些类型之分：


- first

  如果多个插件同时注册 hook，这个 hook 会串行执行，如果某个插件的 hook 函数返回 `null` 或者 `undefined`，那么将结束hook的执行流程，进入下一个 hook 的流程，这个 hook 剩余的函数不会执行。

- sequential

  如果多个插件同时注册 hook，这个 hook 会按照指定顺序执行。如果某个 hook 函数是 async 类型的，将会等待该函数 resolve，才会执行下一个函数

- parallel

  如果多个插件同时注册 hook，这个 hook 会按照指定顺序执行。如果某个 hook 函数是 async 类型的，**不会**等待该函数 resolve，继续执行下一个函数



rollup 的工作流程可以分为 build 和 output 两个阶段，所以 hook 也分为 [build hooks](https://rollupjs.org/guide/en/#build-hooks) 和 [output generation hooks](https://rollupjs.org/guide/en/#output-generation-hooks) 两类，我们可以简单了解一下具体的工作流程：

#### build hooks

> [rollup.js (rollupjs.org)](https://rollupjs.org/guide/en/#build-hooks)

![rollup bundle hooks](https://image.naeco.top/blog/1643203269245.png)



#### output generation hooks

> [rollup.js (rollupjs.org)](https://rollupjs.org/guide/en/#output-generation-hooks)

![rollup output generation hooks](https://image.naeco.top/blog/1643203286189.png)





rollup 的插件工作流程看上去很复杂，hook 也很多，其实并不然，对于插件开发者来说，只需要了解大概的流程即可，大部分的插件只需要用到其中两三个 hook。



### Rollup 插件上下文

rollup 在 hook 函数的[上下文](https://rollupjs.org/guide/en/#plugin-context)绑定了一些方法和属性，可以在函数内部通过`this` 访问。主要会使用到的 API 有：

-  [`this.addWatchFile`](https://rollupjs.org/guide/en/#thisaddwatchfile)

   监听模式下，动态添加文件到监听范围中

-  [`this.emitFile`](https://rollupjs.org/guide/en/#thisemitfile)

   在构建、打包的时候，输出一个新的文件，比方说将图片、字体、样式等文件输出到文件系统中

-  [`this.error`](https://rollupjs.org/guide/en/#thiserror)

   主动抛出异常，终止构建的流程

-  [`this.getModuleInfo`](https://rollupjs.org/guide/en/#thisgetmoduleinfo)

	返回模块的信息，模块包括入口文件、入口文件的依赖，以及依赖的依赖等
	
-  [`this.load`](https://rollupjs.org/guide/en/#thisload)

	加载并解析对应模块
	
-  [`this.parse`](https://rollupjs.org/guide/en/#thisload)

	调用 rollup 内部的方法，将`js` 代码解析成抽象语法树（AST）

-  [`this.resolve`](https://rollupjs.org/guide/en/#thisresolve)

	解析模块的导入，比方说可以将导入语句解析成对应文件路径，或者是网页的 URL 
	
	


## 实现一个简单的 CSS 解析插件

国际惯例，先给上源码的链接：[`rollup-plugin-css2`](https://github.com/naecoo/rollup-plugin-css2)

然后我们来简单分析一下代码：

```javascript
// rollup 插件辅助方法，下面这个方法用于生成过滤规则
import { createFilter } from '@rollup/pluginutils';
// CSS 转译器 
import transformer from '@parcel/css';
import path from 'path';

const isString = (val) => typeof val === 'string';

const isFunction = (val) => typeof val === 'function';

// 生成插件的工厂方法
const pluginGenerator = (customOptions = {}) => {
    // 合并插件选项
	const options = {
		include: ['**/*.css'],
		exclude: [],
		transformOptions: {
			minify: false,
			targets: {},
			drafts: {
				nesting: false
			}
		},
		...customOptions
	};

	// rollup 推荐每一个 transform 类型的插件都需要提供 include 和 exclude 选项，生成过滤规则
    // 主要用于限制插件作用的文件范文，避免误伤其他文件
	const filter = createFilter(options.include, options.exclude);
    // 存储 CSS 代码
	const styles = new Map();
    // 记录引入 CSS 文件的顺序
	const orders = new Set();

    // 返回插件
	return {
        // 插件名称
		name: 'css2',
		
        // transform 钩子
		async transform(code, id) {
            // 不符合过滤规则的，不处理
			if (!filter(id)) return;

            // 去除 CSS 转换器的选项
			const { minify, targets, drafts } = options.transformOptions;
			
            // 转换 CSS 代码
			const { code: transformCode } = await transformer.transform({
				code: Buffer.from(code),
				filename: id,
				minify,
				targets,
				drafts
			});
	
			const css = transformCode.toString();
            // 存储 CSS 代码
			styles.set(id, css);
            // 设置顺序
			if (!orders.has(id)) {
				orders.add(id);
			}
			
            // 返回转换后的内容，包装成一个合法的 ESM 模块
			return {
				code: `export default ${JSON.stringify(css)}`,
				map: { mappings: '' }
			};
		},
		
        // generateBundle 钩子
		generateBundle(opts) {
            // 合并 CSS 代码
			let css = '';
			orders.forEach((id) => {
				css += styles.get(id) ?? '';
			});

			const { output } = options;
			
            // 如果选项传入是一个函数，调用该函数
			if (isFunction(output)) {
				output(css, styles);
				return;
			}

			if (css.length <= 0 || !output) return;
			
            // 解析文件名称
			const name = isString(output) ? output.trim() : opts.file ?? 'bundle.js';
			const dest = path.basename(name, path.extname(name));
			if (dest) {
                // 调用 rollup 暴露给钩子函数的函数，生成静态文件
				this.emitFile({ type: 'asset', source: css, fileName: `${dest}.css` });
			}
		}
	};
};

export default pluginGenerator;
```





## 相关资料

- [Rollup](https://rollupjs.org/guide/en/)
- [Rollup Plugin](https://rollupjs.org/guide/en/#plugin-development)
- [Rollup Plugin Hooks](https://rollupjs.org/guide/en/#build-hooks)
- [Rollup Official Plugins](https://github.com/rollup/plugins)
