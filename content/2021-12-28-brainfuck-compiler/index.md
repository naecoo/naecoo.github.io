+++
title = "用 JavaScript 实现一个 简单的 brainfuck 编译器"
date = "2021-12-28"
author = "naeco"
[taxonomies]
tags = ["javascript"]
+++



## 关于 brainfuck	

关于 brainfuck 语言的介绍，可以查询本文所写的 《[用 JavaScript 实现一个 简单的 brainfuck 解释器](https://naecoo.github.io/brainfuck-interpreter/)》一文，在这篇博客文章中，本人描述了 brainfuck 的模型、语法和指令集，并用 javascript 实现了一个解释器，输入 brainfuck 的源码字符，可以输出 brainfuck 程序的执行结果。

如果只是实现一个 brainfuck 解释器，未免有点过于简单了，因为 brainfuck 只有 8 种指令。作为一名有理想的程序员，我们应该有更高的追求。因此，我将实现一个简单的 brainfuck 编译器，可以将 brainfuck 语言编译为 javascript 语言，在这个过程中，我们可以学到关于编译器的一些基本原理。



## 简单的编译器知识

[编译器](https://zh.wikipedia.org/wiki/編譯器)是一门复杂的计算机知识，通常也是一种计算机程序，可以将某种编程语言的源代码转换成另一种语言，比如说将 C 语言编译成可执行程序，将 typescript 编译成 javascript 等等。不同语言的编译器的功能都不太一样，比如 C、C++ 等语言的编译器会将源代码编译成不同平台的机器语言，C#、Java 等语言则会编译成平台无关的字节码。也有一些编译器，只是将一种语言转换形式，或者转换成另外一种高级语言，比如前端常用的 [babel](https://github.com/babel/babel) ，我们将要实现的，也是这一种相对来说比较简单的编译器。

大部分的编译器都拆分成三个步骤：Parsing、Transformation 和 Code Generation

1. Parsing 就是将原始代码字符转化为更抽象的表示，比如 [AST（抽象语法树)](https://zh.wikipedia.org/wiki/抽象語法樹)
2. Transformation  就是对抽象层做各种操作，生成新的抽象层
3. Code Generation 即根据转换的抽象层，生成目标代码

### Parsing

Parsing 也可以分为两部分：[词法分析](https://zh.wikipedia.org/wiki/词法分析)和[语法分析](https://zh.wikipedia.org/wiki/%E8%AF%AD%E6%B3%95%E5%88%86%E6%9E%90)。

词法分析指的是将源代码进行转换成一个个 token 的过程，token 是源代码的最小组成单位，比如这个 js 表达式：

`var a = 1;`

词法分析后，可以拆解成 `var`、`a`、`=`、`1` 和 `;` 等 token。
而语法分析则会根据生成的 tokens 进行分析，并确定其语法结构，这种语法结构通常又被称为抽象语法树，也就是 AST。比方说上面的语句先解析成 token ，再转化成 AST 后，会是类似这样的结构：
```json
{
  "type": "Program",
  "body": [
    {
      "type": "VariableDeclaration",
      "declarations": [
        {
          "type": "VariableDeclarator",
          "id": {
            "type": "Identifier",
            "name": "a"
          },
          "init": {
            "type": "Literal",
            "value": 1,
            "raw": "1"
          }
        }
      ],
      "kind": "var"
    }
  ]
}
```



### Transformation
在进行完词法分析和语法分析后，我们得到了源代码的抽象表示层 AST，所谓 transformation，就是对 AST 进行语法的修改，甚至是转换成另外一种编程语言的 AST。

比方说上述的 js 语句，我们想把 `var` 声明改为 `const`, 于是将 `kind` 字段改成 `const` ：
```json
{
  "type": "Program",
  "body": [
    {
      "type": "VariableDeclaration",
      "declarations": [
        {
          "type": "VariableDeclarator",
          "id": {
            "type": "Identifier",
            "name": "a"
          },
          "init": {
            "type": "Literal",
            "value": 1,
            "raw": "1"
          }
        }
      ],
      "kind": "const"
    }
  ]
}
```

### Code Generation
最后一个阶段就是代码生成了，不同的编译器有不同的方式去生成目标代码，这可能会有针对性的优化方法，但对于大多数编译器，都是解析 transformation 阶段生成的 AST，遍历各个节点，然后输出目标代码字符。

当然，并不是每一个编译器都是符合上述的模型，因为不同的编译器有不同的需求，可能需要更加复杂的步骤。但我相信基于这个模型，你会对基本的编译器有一个清晰的认知。



## 代码实现
介绍一些原理性的概念后，我们就要着手去实现一个简单的编译器了，需求很简单：

1. 输入 brainfuck 的源代码，比如 `++>++`，
2. 输出对应的 javascript 目标代码

按照上述的模型，我们的编译器将会分为四部分：

1. 解析 brainfuck 源代码， 生成 tokens
2. 将 tokens 转化为 brainfuck 的 AST
3. 将 AST 转化为 javascript 的 AST
4. 生成 javascript 代码

其中， 1、2 步是 parsing、第3步是 Transformation、最后一步则是 Code Generation。



### 解析源码

我将会使用 brainfuck 源码 `>+-<[,.]` 作为示例输入进行讲解


```javascript

// token 类型，对应 brainfuck 的八个指令
const TOKENS = {
  INCREMENT: 'Increment',
  DECREASE: 'Decrease',
  MOVE_NEXT: 'MoveNext',
  MOVE_PREV: 'MovePrev',
  INPUT: 'Input',
  ONTPUT: 'Output',
  LOOP_OPEN: 'LoopOpen',
  LOOP_CLOSE: 'LoopClose'
};

// 生成 tokens 的方法，接收参数为 brainfuck 源代码
// 因为 brainfuck 的源码都是单个独立的字符，所以这一步比较简单
const tokenier = (str) => {
  const tokens = [];
  let current = 0;

  // 逐个字符进行解析
  while (current < str.length) {
    const char = str[current++];

    // 解析不同的指令符号
    switch (char) {
      case '+':
        tokens.push({
          type: TOKENS.INCREMENT,
          value: '+'
        });
        break;
      case '-':
        tokens.push({
          type: TOKENS.DECREASE,
          value: '-'
        });
        break;
      case '>':
        tokens.push({
          type: TOKENS.MOVE_NEXT,
          value: '>'
        });
        break;
      case '<':
        tokens.push({
          type: TOKENS.MOVE_PREV,
          value: '<'
        });
        break;
      case '.':
        tokens.push({
          type: TOKENS.ONTPUT,
          value: '.'
        });
        break;
      case ',':
        tokens.push({
          type: TOKENS.INPUT,
          value: ','
        });
        break;
      case '[':
        tokens.push({
          type: TOKENS.LOOP_OPEN,
          value: '['
        });
        break;
      case ']':
        tokens.push({
          type: TOKENS.LOOP_CLOSE,
          value: ']'
        });
        break;
      default:
        // 如果是其他字符，跳过即可
        break;
    }
  }

  // 返回 tokens
  return tokens;
};
```

使用示例的输入将生成如下的 tokens:

```json
[
      {
          "type": "MoveNext",
          "value": ">"
      },
      {
          "type": "Increment",
          "value": "+"
      },
      {
          "type": "Decrease",
          "value": "-"
      },
      {
          "type": "MovePrev",
          "value": "<"
      },
      {
          "type": "LoopOpen",
          "value": "["
      },
      {
          "type": "Input",
          "value": ","
      },
      {
          "type": "Output",
          "value": "."
      },
      {
          "type": "LoopClose",
          "value": "]"
      }
]
```



### 构建 AST

第二步则是根据 tokens 生成 AST，由于 brainfuck 没有标准的 AST 结构（可能有，但是我没有找到），所以我自己定义了一套简单的结构

```javascript
// 将 tokens 转化为 ast， 输入参数为tokens
const parser = (tokens) => {
  // 先把顶层的结构定义好
  const ast = {
    type: 'Program',
    body: []
  };
  // 定义一个栈，用于保存 ast 节点的引用，主要用于循环语法
  const nodeStask = [];
  // 当前ast节点的引用
  let currentNode = ast.body;

  // 定义遍历 tokens 的方法
  const walk = (token) => {
    switch (token.type) {
      case TOKENS.INCREMENT:
      case TOKENS.DECREASE:
      case TOKENS.MOVE_NEXT:
      case TOKENS.MOVE_PREV:
      case TOKENS.INPUT:
      case TOKENS.ONTPUT:
        // 除了 `[` 和 `]` 两个指令外，其他指令都是普通的 ast 节点
        // 所以直接添加新节点即可
        currentNode.push({
          type: 'ExpressionStatement',
          value: token.type
        });
        return;
    }

    // 解析 `[` 指令
    if (token.type === TOKENS.LOOP_OPEN) {
      // 定义 '[' 指令的节点
      // 简单来说就是循环节点，我们需要解析里面的指令并保存在循环节点的 body 中
      const astNode = {
        type: 'LoopStatement',
        body: []
      };
      // 更新 ast
      currentNode.push(astNode);
      // 压入栈，保持当前的 ast 节点引用，循环结束后恢复
      nodeStask.push(currentNode);
      // 更新 ast 指针
      currentNode = astNode.body;
    }

    // 解析 `]` 指令
    if (token.type === TOKENS.LOOP_CLOSE) {
      // 如果栈是空的，说明没有与之对应的 `[` 指令，语法有误
      if (nodeStask.length <= 0) {
        throw new Error('Uncaught SyntaxError: Loop syntax error');
      }

      // 弹出栈顶元素，恢复进入循环之前的指针引用
      currentNode = nodeStask.pop();
    }
  };

  // 开始遍历 tokens
  tokens.forEach(walk);
  
  // 如果栈没清空，说明 `[` 和 `]` 这两个指令输入不规范，语法有误
  if (nodeStask.length > 0) {
    throw new Error('Uncaught SyntaxError: Loop syntax error');
  }

  // 返回结果
  return ast;
};

```

基于此程序，用示例的输入，生成的 ast 会是如下：

```json
{
  "type": "Program",
      "body": [
          {
              "type": "ExpressionStatement",
              "value": "MoveNext"
          },
          {
              "type": "ExpressionStatement",
              "value": "Increment"
          },
          {
              "type": "ExpressionStatement",
              "value": "Decrease"
          },
          {
              "type": "ExpressionStatement",
              "value": "MovePrev"
          },
          {
              "type": "LoopStatement",
              "body": [
                  {
                      "type": "ExpressionStatement",
                      "value": "Input"
                  },
                  {
                      "type": "ExpressionStatement",
                      "value": "Output"
                  }
              ]
          }
      ]
}
```



### 转换 AST

因为我们的目标是生成 javascript 的代码，所以要将 brainfuck 的 ast 转换成 javascript 的 ast。那么问题来了，我们要怎么将 brainfuck 的语法翻译到 javascript 呢？其实很简单，我们可以将指令封装成 javascript 的函数， 一条指令就对应一个函数调用，而遇到循环语句呢，就用 `while` 循环替代。我们可以先构建如下的 javascript 函数，可以称之为 brainfuck 的 runtime：

```javascript
// 这个代表纸带
var t = [0];
// 这个代表当前指针指向
var p = 0;
// 获取当前指针指向格子的值
function getValue() {
  return t[p];
}
// `+` 指令
function increment() {
  t[p]++;
}
// `-` 指令
function decrease() {
  t[p]--;
}
// `>` 指令
function movePrev() {
  p--;
  if (p < 0) throw new Error('Pointer less then 0 ', p);
}
// `<` 指令
function moveNext() {
  p++;
  if (t[p] === undefined) t[p] = 0;
}
// `.` 指令
function output() {
  console.log(String.fromCharCode(t[p]));
}
// `,` 指令
function input() {
  var c = prompt('Input one Ascii character');
  if (typeof c !== 'string' || c.length <= 0) throw new Error('Invalid input: ', c);
  t[p] = c[0].charCodeAt();
}
```

下一步就是转换 ast 结构了，我们先定义遍历 ast 节点的方法，我们称之为 `traverasl`:
```javascript
// 遍历 ast 的方法（深度优先）
const traverse = (ast, visitor) => {
  // 定义遍历节点的方法
  const traverseNode = (node, parent) => {
    // 去除对应节点类型的访问器
    // 访问器可以是一个函数，或者是一个对象
    // 如果是一个对象，可以定义 enter 和 exit 函数，分别会在进入节点和退出节点时调用
    let method = visitor[node.type] || {};

    // 格式化方法
    if (typeof method === 'function') {
      method = {
        enter: method
      };
    }

    // 调用访问器的 enter 方法
    if (method.enter) method.enter(node, parent);
    
    switch (node.type) {
      case 'Program':
        // 处理子节点
        traverseArray(node.body, node);
        break;

      case 'LoopStatement':
        // 处理子节点
        traverseArray(node.body, node);
        break;

      case 'ExpressionStatement':
        // 该类型没有子节点，无需处理
        break;

      default:
        throw new Error('Unknown Ast node type: ', node.type);
    }

    // 调用访问器的 exit 方法
    if (method.exit) method.exit(node, parent);
  };

  // 定义遍历节点列表的方法
  const traverseArray = (array, parent) => {
    array.forEach((child) => {
      traverseNode(child, parent);
    });
  };

  // 开始遍历
  traverseNode(ast, null);
};
```

接下来就是转换 ast 的操作了， javascript ast 的结构参考自 [acorn](https://github.com/acornjs/acorn) 这个解析器, 但是为了简化代码，删去了一些不必要的字段。

```javascript
// 处理字符串，使其首字母变成小写
const decapitalize = (s) => (s ? `${s[0].toLowerCase()}${s.slice(1)}` : s);

// ast 转换，参数是旧的 ast，即 brainfuck 的 ast
const transformer = (ast) => {
  // 首先定义新的 ast 结构
  const newAst = {
    type: 'Program',
    body: []
  };
  
  // 在旧的 ast 上添加一个指针，对应新的 ast节点
  ast._context = newAst.body;

  // 遍历 ast
  traverse(ast, {
    // 处理 ExpressionStatement 类型的节点
    ExpressionStatement: (node, parent) => {
      // 这是普通的指令，直接往新的 ast 中添加节点即可，对应 javascript 的函数调用
      parent._context.push({
        type: 'ExpressionStatement',
        expression: {
          type: 'CallExpression',
          callee: {
            type: 'Identifier',
            name: decapitalize(node.value)
          }
        }
      });
    },

    // 处理 LoopStatement 类型的节点
    LoopStatement: (node, parent) => {
      // 构造 javascript 中 while 语句的 ast 节点
      const whileNode = {
        type: 'WhileStatement',
        // 这里 while 语句中的测试条件也被我们封装成函数了
        // 类似这样 => `while(getValue())`
        test: {
          type: 'CallExpression',
          callee: {
            type: 'Identifier',
            name: 'getValue'
          }
        },
        // while 语句中的代码
        body: {
          type: 'BlockStatement',
          body: []
        }
      };
      // 添加节点
      parent._context.push(whileNode);
      // 更新引用关系
      node._context = whileNode.body.body;
    }
  });

  return newAst;
};
```

使用示例，将输出如下的 javascript ast：

```json
{
      "type": "Program",
      "body": [
          {
              "type": "ExpressionStatement",
              "expression": {
                  "type": "CallExpression",
                  "callee": {
                      "type": "Identifier",
                      "name": "moveNext"
                  }
              }
          },
          {
              "type": "ExpressionStatement",
              "expression": {
                  "type": "CallExpression",
                  "callee": {
                      "type": "Identifier",
                      "name": "increment"
                  }
              }
          },
          {
              "type": "ExpressionStatement",
              "expression": {
                  "type": "CallExpression",
                  "callee": {
                      "type": "Identifier",
                      "name": "decrease"
                  }
              }
          },
          {
              "type": "ExpressionStatement",
              "expression": {
                  "type": "CallExpression",
                  "callee": {
                      "type": "Identifier",
                      "name": "movePrev"
                  }
              }
          },
          {
              "type": "WhileStatement",
              "test": {
                  "type": "CallExpression",
                  "callee": {
                      "type": "Identifier",
                      "name": "getValue"
                  }
              },
              "body": {
                  "type": "BlockStatement",
                  "body": [
                      {
                          "type": "ExpressionStatement",
                          "expression": {
                              "type": "CallExpression",
                              "callee": {
                                  "type": "Identifier",
                                  "name": "input"
                              }
                          }
                      },
                      {
                          "type": "ExpressionStatement",
                          "expression": {
                              "type": "CallExpression",
                              "callee": {
                                  "type": "Identifier",
                                  "name": "output"
                              }
                          }
                      }
                  ]
              }
          }
      ]
}
```



### 代码生成

最后一步便是目标代码的生成了， 代码生成相对来说是最简单，但是也是最繁琐的一步了，我们需要遍历 ast 的节点，生成对应的代码，我们可以采用递归的方法去生成：

```javascript
// 封装一个生成器类
class Generator {
  constructor() {
    // 保存代码内容
    this.output = '';
  }

  // 写入新的字符
  write(code) {
    this.output += code;
  }
  
  // Program 类型处理
  Program(node) {
    // 用一个函数进行封装
    this.write('function brainMain() { ');
    // 然后遍历子节点
    node.body.forEach((child) => {
      // 调用对应的类型方法
      this[child.type](child);
    });
    this.write(' }');
  }

  // ExpressionStatement 类型处理
  ExpressionStatement(node) {
    // 处理表达式
    this[node.expression.type](node.expression);
    this.write(';');
  }

  // BlockStatement 类型处理
  BlockStatement(node) {
    this.write(' {');

    // 处理 block
    node.body.forEach((child) => {
      this[child.type](child);
    });

    this.write('} ');
  }

  // WhileStatement 类型处理
  WhileStatement(node) {
    this.write(' while(');
    // 测试条件
    this[node.test.type](node.test);
    this.write(')');
    // 循环里面的代码
    this[node.body.type](node.body);
  }

  // CallExpression 类型处理
  CallExpression(node) {
    // 函数调用
    this[node.callee.type](node.callee);
    this.write('()');
  }

  // Identifier 类型处理
  Identifier(node) {
    this.write(node.name);
  }
}
```

采用示例输入，最终输出的代码将会是:

```javascript
function brainMain() { moveNext();increment();decrease();movePrev(); while(getValue()) {input();output();}  }
```


前面提到我们定义了 brainfuck 的 runtime，生成的代码需要依赖这个 runtime 才能执行，为了避免这个麻烦，我们生成代码的时候，同时将运行时也写入代码中:

```javascript
// runtime
const preset = `
var t = [0];
var p = 0;
function getValue() {
  return t[p];
}
function increment() {
  t[p]++;
}
function decrease() {
  t[p]--;
}
function movePrev() {
  p--;
  if (p < 0) throw new Error('Pointer less then 0 ', p);
}
function moveNext() {
  p++;
  if (t[p] === undefined) t[p] = 0;
}
function output() {
  console.log(String.fromCharCode(t[p]));
}
function input() {
  var c = prompt('Input one Ascii character');
  if (typeof c !== 'string' || c.length <= 0) throw new Error('Invalid input: ', c);
  t[p] = c[0].charCodeAt();
}
`;

const codeGenerator = (ast) => {
  // 实例化生成器
  const generator = new Generator();
  // 开始生成代码
  generator[ast.type](ast);

  // 将 runtime 也加进来
  const result = `
(function () {
// runtime
${preset}

// 生成的代码
${generator.output}

// 返回这个函数
return brainMain;
})();
	`;

  // 返回最终结果
  return {
    result,
    compileResult: generator.output
  };
};
```

### 组装

将这几个阶段组装起来，就是我们最终的成果：

```javascript
const compiler = (sourceCode) => {
  // 生成tokens
  const tokens = tokenier(sourceCode);
  // 生成 brainfuck ast
  const ast = parser(tokens);
  // 转换成 javascript ast
  const newAst = transformer(ast);
  // 生成 javascript 代码
  const code = codeGenerator(newAst);
  // 返回结果
  return {
    tokens,
    ast,
    javascriptAst: newAst,
    compiler: code.compileResult,
    code: code.result,
  }
};
```





## 总结

最后放上源码的地址 [brainfuck-compiler](https://github.com/naecoo/brainfuck-compiler/)， 如果对代码有疑问，或者觉得有问题，可以在 issue 中提出，或者直接提 PR。

同时，为了方便操作，我还做了一个 brainfuck 语言在线编译和可视化执行的网页，地址在[这里](https://naecoo.github.io/brainfuck-compiler/index.html)。



## 相关资料

-  [the-super-tiny-compiler](https://github.com/jamiebuilds/the-super-tiny-compiler)
-  [ast-explorer](https://astexplorer.net/)
-  [javascript-codegen-astring](https://github.com/davidbonnet/astring) 

