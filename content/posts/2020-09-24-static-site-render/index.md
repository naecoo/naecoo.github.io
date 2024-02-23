+++
title = "一个简单的静态网站生成器"
date = "2020-09-24"
author = "naeco"
[taxonomies]
tags = ["nodejs", "blog"]
+++

### 前言
  随着云技术的发展，现在互联网上许多公司都提供了免费托管网站的服务，比如大名鼎鼎的Github Pages、Netty等，这些服务的流行也推动了静态博客站点生成器的流行。互联网早期可能很多人会用WordPress搭建博客，但现在早就是Hexo、Hugo和Vuepree等静态博客的天下了。比如本站就是采用VuePress搭建的，每次写博客只需要重新渲染打包，再加上自动化CI/CD，一键就能发布更新。很多人，包括我自己，觉得静态博客生成器很难做，其实不然，背后的技术原理其实很简单，一个基本的静态博客生成器的流程大概会有：

  - 解析所有markdown文件
  - 渲染成HTML文件
  - 生成页面链接

  所以现在，我们就根据这三个步骤，实现一个简单的静态博客生成器吧。

### 解析文章
我们可以用一个类来实现全部的逻辑，首先是类的初始化
```javascript
class StaticSiteRender {
  constructor (options = {}) {
    const defaultOptions = {
      // markdown文件目录
      sourcePath: './source',
      // 模板文件目录
      templtePath: './template',
      // 输出目录
      output: './public'
    }
    this.frontMatters = []
    this.options = Object.assign(defaultOptions, options)
    // markdown渲染器
    const md = new MarkdownIt()
    this.md = md
    // 支持markdown的frontMatter
    md.use(markdownItFrontMatter, (fm) => {
        // FIXME: 由于markdown-it插件的处理流程是在回调函数，所以只能在这里存储起来
      this.frontMatters.push(YAML.parse(fm))
    })
  }

  // ...
}
```
在类的初始化时候可以传入配置修改相关文件的路径，然后进行markdown解析器的初始化，这里我们用的是[markdown-it](https://www.npmjs.com/package/markdown-it)， 同时我们用了markdown-it的一个插件[markdown-it-front-matter](https://www.npmjs.com/package/markdown-it-front-matter)， 这个插件的作用是解析markdown文件的元信息，一般是这样的形式：

```markdown
+++
title: markdown文章
author: naeco
tags:
 - Vue
 - js
+++
### markdown
- 1
- 2
```

插件将会吧`---`之间包裹的内容解析成字符串，我们再通过YAML解析器解析成js的对象：

```javascript
{
    "title": "markdown文章",
    "author": "naeco",
    "tags": ["vue", "js"]
}
```

然后是解析markdown，这里处理比较简单，直接遍历文件夹下的所有markdown文件

```javascript
// 遍历文件夹
async parsePost() {
    this.frontMatters = []
    const postPath = path.resolve(this.options.sourcePath)
    const sources = fs.readdirSync(postPath)
    sources.forEach(async (source, index) => {
      const sourcePath = path.resolve(postPath, source)
      const content = fs.readFileSync(sourcePath, 'utf-8')
      // 解析markdown内容
      await this._renderPost(content, path.basename(source, path.extname(source)))
    })
  }
```

### 生成HTML文件

遍历得到所有markdown文件后，就可以调用markdown-it将其转换为HTML文件了

```javascript
// markdown => html
async _renderPost(content, fileName) {
    const markdown = this.md.render(content)
    const frontMatter = this.frontMatters[this.frontMatters.length - 1]
    Object.assign(frontMatter, { fileName })
    const template = fs.readFileSync(path.resolve(this.options.templtePath, './post.ejs'), 'utf-8')
    const html = ejs.render(template, {
      title: frontMatter.title || '',
      content: markdown
    })
    
    fs.writeFileSync(path.resolve(this.options.output, './post/', `${fileName}.html`), html)
  }
```

markdown-it会将markdown文件的内容转化为HTML字符串，然后我们通过模板引擎[ejs](https://www.npmjs.com/package/ejs)生成HTML文件，具体HTML模板如下：

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= title %></title>
</head>
<body>
  <main>
    <%- content %>
  </main>
</body>
</html>
```



### 生成首页HTML文件

等到所有markdown文件转换为HTML文件后，我们就可以生成博客的首页页面了，原理是一样的，通过ejs的模板文件，传递具体博文数据进去，生成html文件。

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>首页</title>
</head>
<body>
  <div>
    <%
      posts.forEach(function(post){%>
        <div>
          <h3>
            <a href="./post/<%- post.fileName %>.html">
              <%- post.title %>
            </a>
          </h3>
          <p>author: <%- post.author %></p>
          <p>creatAt: <%- post.date %></p>
          <p>
            tags:
            <%
              (post.tags || []).forEach(function(tag){%>
                <span style="margin-left: 5px;"><%- tag %></span>
              <%})
            %>
          </p>
        </div>
      <%})
    %>
  </div>
</body>
</html>
```



```javascript
 async _renderHome () {
    const template = fs.readFileSync(path.resolve(this.options.templtePath, './home.ejs'), 'utf-8')
    const html = ejs.render(template, { posts: this.frontMatters })
    fs.writeFileSync(path.resolve(this.options.output, `home.html`), html)
  }
```

这里所有的工作就已经完成了，你可以把生成的HTML文件放在托管平台上，这样就可以在互联网上访问你的博客。



### 代码

> index.js

```javascript
const path = require('path')
const fs = require('fs')
const YAML = require('YAML')
const MarkdownIt = require('markdown-it')
const markdownItFrontMatter = require('markdown-it-front-matter')
const ejs = require('ejs')

class StaticSiteRender {
  constructor (options = {}) {
    const defaultOptions = {
      // markdown文件目录
      sourcePath: './source',
      // 模板文件目录
      templtePath: './template',
      // 输出目录
      output: './public'
    }
    this.frontMatters = []
    this.options = Object.assign(defaultOptions, options)
    const md = new MarkdownIt()
    this.md = md
    md.use(markdownItFrontMatter, (fm) => {
      this.frontMatters.push(YAML.parse(fm))
    })
  }

  async parsePost() {
    this.frontMatters = []
    const postPath = path.resolve(this.options.sourcePath)
    const sources = fs.readdirSync(postPath)
    sources.forEach(async (source, index) => {
      const sourcePath = path.resolve(postPath, source)
      const content = fs.readFileSync(sourcePath, 'utf-8')
      await this._renderPost(content, path.basename(source, path.extname(source)))
    })
  }

  async _renderPost(content, fileName) {
    const markdown = this.md.render(content)
    const frontMatter = this.frontMatters[this.frontMatters.length - 1]
    Object.assign(frontMatter, { fileName })
    const template = fs.readFileSync(path.resolve(this.options.templtePath, './post.ejs'), 'utf-8')
    const html = ejs.render(template, {
      title: frontMatter.title || '',
      content: markdown
    })
    
    fs.writeFileSync(path.resolve(this.options.output, './post/', `${fileName}.html`), html)
  }
  
  async _renderHome () {
    const template = fs.readFileSync(path.resolve(this.options.templtePath, './home.ejs'), 'utf-8')
    const html = ejs.render(template, { posts: this.frontMatters })
    fs.writeFileSync(path.resolve(this.options.output, `home.html`), html)
  }

  async run () {
    await this.parsePost()
    await this._renderHome()
  }
}

const ssRender = new StaticSiteRender()
ssRender.run()
```



> home.ejs

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>首页</title>
</head>
<body>
  <div>
    <%
      posts.forEach(function(post){%>
        <div>
          <h3>
            <a href="./post/<%- post.fileName %>.html">
              <%- post.title %>
            </a>
          </h3>
          <p>author: <%- post.author %></p>
          <p>creatAt: <%- post.date %></p>
          <p>
            tags:
            <%
              (post.tags || []).forEach(function(tag){%>
                <span style="margin-left: 5px;"><%- tag %></span>
              <%})
            %>
          </p>
        </div>
      <%})
    %>
  </div>
</body>
</html>
```



> post.ejs

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= title %></title>
</head>
<body>
  <main>
    <%- content %>
  </main>
</body>
</html>
```

