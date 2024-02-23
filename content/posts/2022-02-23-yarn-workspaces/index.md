+++
title = "yarn workspaces 简单介绍"
date = "2022-02-23"
author = "naeco"
[taxonomies]
tags = ["javascript"]
+++



## yarn workspace

workspace 是 [yarn](https://www.npmjs.com/package/yarn) 支持的一种特性，在 yarn 1.0.0 版本开始默认支持。该功能可以让我们在同一个代码仓库中管理多个 package，也就是 [monorepo](https://en.wikipedia.org/wiki/Monorepo) 的代码管理方式。




## monorepo
很多大型的项目都包含有多个 package，比如 vue、react、babel、rollup等，如果每一个 package 都需要单独一个仓库，整体项目的工作流程将会变得异常复杂。采用了 monorepo 的做法，将所有 package 放在同一个代码仓库中，可以统一进行构建、发布，对于调试和依赖管理都很方便。

[vue](https://github.com/vuejs/core) 的仓库就是一个很典型的 monorepo 架构，代码分成了多个 package，统一放置在 packages 目录下：

![image-20220223221506859](https://image.naeco.top/blog/1645625707203.png)



## yarn workspace 使用

使用 workspace，首先要在代码仓库根目录的 `package.json` 文件中设置 `private` 和 `workspaces` 属性：

> package.json

```json
{
   "name": "learn-yarn-workspace",
   "version": "1.0.0",
   "private": true,
   "workspaces": [
      "packages/pkg1",
      "packages/pkg2"
   ],
}
```

首先 `private` 必须设置为 `true`，代表不会被发布，然后在 `workspaces` 中声明 各个 package，此处代表 packages 目录下面的 pkg1 和 pkg2 分别为两个 package。`workspaces` 属性支持 glob 表达式，所以也可以写成：

> package.json

```json
{
  "workspaces": ["packages/*"]
}
```

这就代表 packages 的所有子目录都作为 package，纳入 workspaces。

yarn 同时还提供了一组命令，帮助提高操作效率：

1. `yarn workspace <package_name> <command>`

   在指定的 package 下运行命令

   ```shell
   # 在 foo 下安装 express
   yarn workspace foo add express
   
   # 在 foo 下移除 express
   yarn workspace foo remove express
   
   # 在 foo 下执行 scripts 中的 dev 命令
   yarn workspace foo run dev
   ```

2. `yarn workspace run <command>`

   在所有 package 运行命令，没有则报错

   ```shell
   # 所有 package 执行 scripts 中的 build 命令
   yarn workspace run build
   ```

3. `yarn workspace info [--json]`

   输出 workspace 中的依赖信息

4. `yarn <add|remove> <package> -W`

   在根目录下安装依赖

   ```shell
   # 将 eslint 作为全局依赖安装
   yarn add eslint -W
   ```

   

## yarn workspace 分析

使用 workspace 之后，yarn 会将 所有 package 的依赖统一安装在一起，提高了性能，同时节约了硬盘空间。同时，yarn 会在 `node_modules` 中生成 package 的软链接，指向 workspaces 中的最新代码文件，相对于 `yarn link` 来说，使用更加地快捷和方便。

我们以一个简单的例子来说明 yarn workspace 具体的工作机制：

![image-20220223225035020](https://image.naeco.top/blog/1645627835280.png)

> test/package.json

```json
{
  "name": "learn-yarn-workspace",
  "version": "1.0.0",
  "license": "MIT",
  "private": true,
  "workspaces": [
    "packages/*"
  ]
}
```

> test/pakcages/pkg1/package.json

```json
{
  "name": "pkg1",
  "version": "1.0.0"
}
```

> test/packages/pkg2/package.json

```json
{
  "name": "pkg2",
  "version": "1.0.0"
}
```

执行 `yarn install`后，node_modules 会生成对应模块的软链接:

![image-20220223225918484](https://image.naeco.top/blog/1645628358738.png)

所以各个模块之间可以很方便地进行引用。

那么问题来了，假设 pkg2 依赖 pkg1，但是依赖的版本和本地不一致怎么办，比方说，pkg2 依赖的是 `1.0.1` 版本，与本地的 `1.0.0` 版本不一致：

> test/packages/pkg2/package.json

```json
{
  "name": "pkg2",
  "version": "1.0.0",
  "dependencies": {
    "pkg1": "1.0.1"
  }
}
```

由于涉及到包发布的问题，这里我们改成依赖 `express` 的不同版本来说明问题:
> test/pakcages/pkg1/package.json

```json
{
  "name": "pkg1",
  "version": "1.0.0",
  "dependencies": {
    "express": "4.0.0"
  }
}
```

> test/packages/pkg2/package.json

```json
{
  "name": "pkg2",
  "version": "1.0.0",
  "dependencies": {
    "express": "3.0.0"
  }
}
```

重新执行一次 `yarn install`，我们发现项目根目录的 `node_modules`  文件下的 `express` 版本为 `3.0.0`，那么 `4.0.0` 版本的去哪里了呢？我们再打开 pkg1 的目录，发现这里的 `node_modules` 文件夹也多了一个 `express`：

![image-20220223231203310](https://image.naeco.top/blog/1645629123555.png)

很显然，yarn 为了解决对同一个依赖的冲突，会将低版本号的依赖放在根目录下的 `node_modules` ，而其他版本则会放在对应模块下面的 `node_modules` 中。同时，依赖的依赖也可能会发生版本冲突，解决方法也是一样的，我们可以看到上述的图片中，除了 `express` 之外，还有其他的依赖包，这些都是 `express` 的依赖。

我们简单总结一下规则：

1. yarn 会在根目录的 node_modules 下面为各个 package 创建软链接，指向 workspace 指定的目录。

2. 如果不同的 package 依赖同一个依赖的不同版本，为了解决冲突，低版本依赖安装在根目录，而其他依赖直接安装到对应 package 的 node_modules 下面。

3. 依赖包的依赖如果发生冲突，解决方法和 2 一致。

   


## Reference

1. [yarn workspaces docs](https://classic.yarnpkg.com/en/docs/workspaces)
2. [yarn workspace 指南](https://zhuanlan.zhihu.com/p/381794854)
