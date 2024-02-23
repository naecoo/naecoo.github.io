+++
title = "告别webpack，直接运行npm包"
date = "2019-09-15"
author = "naeco"
[taxonomies]
tags = ["javascript", "webpack"]
+++

### 为什么要打包？
2019年，距离es6正式发布已经过去了4年多了，es6给我们带来了许多新特性，包括全新的JavaScript模块系统（[ESM](https://flaviocopes.com/es-modules/)），它可以直接在浏览器运行。但一般我们开发项目，还是要引入Browserify和Webpack等打包工具进行打包，诚然，这些打包工具可以给项目带来很多好处、比如混淆、压缩和转译代码等等。但与此同时，也带给项目极大的复杂性，各种各样的配置文件和插件等。很多时候，我们不得不进行打包，因为[npm](https://www.npmjs.com/)的存在，我们通常会引用很多注册在`npm`上面的包，早期的`npm`包大部分都是`common.js（cjs）`风格的，这意味着浏览器不能直接运行，所以一般都要经过打包工具转换成浏览器支持的模块。
### 现状
`npm`后面推出了`module`入口的模块，即ESM风格的模块，到目前为止，已经超过70000个包提供了[ESM的版本](https://www.pika.dev/about/stats)，已经基本满足我们开发的需要了。所以为什么我们不直接跳过打包这个步骤，直接运行呢？事实上，自己写的ESM模块可以直接运行，但大部分`npm`包都会有依赖，浏览器没办法从`node_modules`导入这些依赖，所以我们依然需要对代码进行打包，转译。
### @pika/web
基于上述原因，[@pika/web](https://github.com/pikapkg/web)诞生了。`@pika/web`不是构建工具，而是依赖安装工具，它可以帮助你构建npm包，然后直接在浏览器上运行，只需要一行代码：
```bash
npx @pika/web
```
### 使用方法
`@pika/web`使用起来十分简单，我们首先需要在`package.json`的`dependencies`显式声明项目所依赖的包:
```javascript
// package.json
// ...
"dependencies": {
  "test": "^1.0.0"
},
```
然后在项目中引用这个包:
```javascript
// index.js
import { name } from 'test'
console.log(name)
```
然后执行```npx @pika/web```

![](https://user-gold-cdn.xitu.io/2019/9/13/16d293bcbc8221c3?w=322&h=57&f=png&s=5966)
pika会在项目根目录生成一个`web_modules`的文件夹，里面放着项目依赖的文件，最后一步就是将替换项目引用依赖的地址
```javascript
// before
import { name } from 'test'
// after
import { name } from './web_modules/test.js'
```
最后一步，因为我们的代码需要在浏览器执行，所以需要嵌入到`html`文件中:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Document</title>
</head>
<body>
</body>
<script type="module" src="./index.js"></script>
</html>
```
然后用浏览器打开文件，你会得到你想要的结果。

可能有的同学会有说：“我的项目引用的依赖很多，总不能一个个去替换地址吧，多麻烦啊！”，没问题，pika提供了一个`babel`的插件帮助我们替换模块引用。
### 源码解密
`@pika/web`的原理很简单，首先它会从`package.json`的`dependencies`种寻找项目依赖，收集好依赖就会逐个解析依赖，生成`rollup`的配置文件，最后用`rollup`将每一个依赖打包成模块，放到`web_modules`中。
- 收集依赖
```javascript
// ...
const pkgManifest = require(path.join(cwd, 'package.json'));
  const {namedExports, webDependencies} = pkgManifest['@pika/web'] || {
    namedExports: undefined,
    webDependencies: undefined,
  };
  const doesWhitelistExist = !!webDependencies;
  const arrayOfDeps = webDependencies || Object.keys(pkgManifest.dependencies || {});
// ...
```
- 逐个解析
```javascript
// ...
const depManifestLoc = resolveFrom(cwd, `${dep}/package.json`);
  const depManifest = require(depManifestLoc);
  let foundEntrypoint: string = depManifest.module;
  // If the package was a part of the explicit whitelist, fallback to it's main CJS entrypoint.
  if (!foundEntrypoint && isExplicit) {
    foundEntrypoint = depManifest.main || 'index.js';
  }
// ...
```
- 生成`rollup`打包配置
```javascript
// ...
const inputOptions = {
      input: depObject,
      plugins: [
        rollupPluginNodeResolve({
          mainFields: ['browser', 'module', !isStrict && 'main'].filter(Boolean),
          modulesOnly: isStrict, // Default: false
          extensions: ['.mjs', '.cjs', '.js', '.json'], // Default: [ '.mjs', '.js', '.json', '.node' ]
          // whether to prefer built-in modules (e.g. `fs`, `path`) or local ones with the same names
          preferBuiltins: false, // Default: true
        }),
        !isStrict &&
          rollupPluginCommonjs({
            extensions: ['.js', '.cjs'], // Default: [ '.js' ]
            namedExports: knownNamedExports,
          }),
        !!isBabel &&
          rollupPluginBabel({
            compact: false,
            babelrc: false,
            presets: [
              [
                babelPresetEnv,
                {
                  modules: false,
                  targets: hasBrowserlistConfig ? undefined : '>0.75%, not ie 11, not op_mini all',
                },
              ],
            ],
          }),
        !!isOptimized && rollupPluginTerser(),
      ],
      }) as any,
    };
    const outputOptions = {
      dir: destLoc,
      format: 'esm' as 'esm',
      sourcemap: sourceMap === undefined ? isOptimized : sourceMap,
      exports: 'named' as 'named',
      chunkFileNames: 'common/[name]-[hash].js',
    };
    const packageBundle = await rollup.rollup(inputOptions);
    await packageBundle.write(outputOptions);
    fs.writeFileSync(
      path.join(destLoc, 'import-map.json'),
      JSON.stringify({imports: importMap}, undefined, 2),
      {encoding: 'utf8'},
    );
// ...
```