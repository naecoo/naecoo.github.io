+++
title = "用 JavaScript 实现一个 简单的 brainfuck 解释器"
date = "2021-12-01"
author = "naeco"
[taxonomies]
tags = ["javascript"]
+++



## brainfuck 介绍

[Brainfuck](https://en.wikipedia.org/wiki/Brainfuck) 是一种极小化的计算机语言，它是由 [Urban Müller](https://en.wikipedia.org/wiki/Brainfuck) 在 1993 年创建的。由于 fuck 在英语中是脏话，这种语言 有时被称为 brainf*ck 或 brainf**k，甚至被简称为 BF。这是一种图灵完备的编程语言，但它不像 C++ 那么多元化，因为仅仅只有 8 个指令（操作符）。这种语言基于一个简单的机器模型，除了指令，这个机器还包括：一个以字节为单位、被初始化为零的的数组（我们称之为纸带），一个指向该数组的指针，以及用于输入输出的两个字节流。

下面是八个指令的描述，其中每一个指令由一个字符所标识：

| 字符 | 含义                                                         |
| ---- | ------------------------------------------------------------ |
| >    | 指针加一，即纸带向右移动一个格子                             |
| <    | 指针减一，即纸带向左移动一个格子                             |
| +    | 指针指向的字节的值加一                                       |
| -    | 指针指向的字节的值减一                                       |
| .    | 输出指针指向的格子内容（ASCⅡ码）                             |
| ,    | 输入内容到指针指向的单元（ASCⅡ码）                           |
| [    | 如果指针指向的单元值为零，向后跳转到对应的 ] 指令的下一个指令 |
| ]    | 如果指针指向的单元值不为零，向前跳转到对应的 [ 指令的下一个指令处 |



### 程序示例

```javascript
'+++>++'
// [3, 2]
```

这个程序将在纸带第一格的值改变为 3，第二格的值改变为 2

```javascript
'++++++[>++++++<-]>.'
```

这个程序稍微有点复杂，首先将第一格内容增加到 6，然后碰到一个 “[” 字符，因为当前格子的值不为 0，所以进入循环。首先是跳到第二个格子，连续加到 6，此时第二个格子的值为 6，然后跳转回第一个格子，然后减一，此时第一个格子的值为 5。依然不等于 0，所以继续循环，知道第一个格子的值为 0，跳出循环。跳出循环后，又跳到第二个格子，然后输出当前格子的值，因为第二个格子的值为 36，所以输出 “$” 符号（ASCⅡ码）。



## 解释器实现

brainfuck 由于只有 8 个操作符，所以解释器实现也比较的简单，大概不到 100 行的代码。所以这里不再划分模块了，直接上代码：

```javascript
const readlineSync = require('readline-sync');

// 设定纸带格子数上限
const max_tape_length = 5000;
// 设定 brainfuck 程序执行步骤的上限，防止出现死循环等情况
const max_step = Math.pow(2, 16);

// brainfuck 解释器
function brainfuckInterpreter(source = '') {
  // 检查语法
  checkSyntax(source);

  // 初始化源代码索引
  let current = 0;
  // 初始化纸带当前位置
  let pointer = 0;
  // 初始化纸带
  const tape = [0];
  // 初始化一个栈，用于循环等语句
  const loopStack = [];
  let step = 1;

  // 开始执行 brainfuck 代码
  while (current < source.length) {
    // 记录 brainfuck 程序执行次数，如果超出阈值，直接抛出异常
    step++;
    if (step > max_step) {
      throw new Error('Program execution is too long, may fall into an infinite loop, please check the source code');
    }
	
    // 取出源码的字符
    const char = source[current];

    // 根据不同的操作符，执行相应的逻辑
    switch (char) {
      case '>':
        // 纸带向右移动一步
        pointer++;
        // 如果超出了上限，直接抛出异常
        if (pointer > max_tape_length) {
          throw new Error('Tape\'s length exceed limit.');
        }
        // 给纸带上的格子设定初始值
        if (tape[pointer] === void 0) {
          tape[pointer] = 0;
        }
        break;

      case '<':
        // 纸带向左移动一步
        pointer--;
        // 同样地，检查是否超出限制
        if (pointer < 0) {
          throw new Error('Tape\'s length exceed limit.');
        }
        break;

      case '+':
        // 纸带当前格子的值加一
        tape[pointer]++;
        break;

      case '-':
        // 纸带当前格子的值减一
        tape[pointer]--;
        break;

      case '.':
        // 将纸带当前值转化为 ASCⅡ码 码打印出来
        console.log('output:: ', String.fromCharCode(tape[pointer]));
        break;

      case ',':
        // 从终端读取第一个字符，存储字符对应的 ASCⅡ码 码值
        const input = readlineSync.question('input:: ');
        if (typeof input === 'string') {
          tape[pointer] = input[0].codePointAt();
        }
        break;

      case '[':
        // 添加进栈顶
        // 这里为什么使用栈这种数据结构呢，因为可能会存在多重嵌套的循环结构，使用栈能够确保总是可以取到当前循环的 “[” 字符
        loopStack.push(current);

        // 检查纸带当前格子的值，如果等于0，不执行循环
        if (tape[pointer] === 0) {
          // 这里需要找到与之对应“]”字符，跳过内部嵌套的循环语句
          // TODO: 可以在预处理阶段生成“[”到“]”的映射信息，就可以避免重复执行，提高性能
          let leftCount = 1;
          while (current < source.length) {
            const op = source[current];
            if (op === "[") {
              leftCount++;
            } else if (op === "]") {
              leftCount--;
              // 找到元素
              if (leftCount === 0) {
                break;
              }
            }
            current++;
          }
          continue;
        }
        break;

      case ']':
        // 弹出栈顶元素，即本次循环的 “[” 元素位置
        const loopStartIndex = loopStack.pop();
        // 检查纸带当前格子的值，如果不等于0，继续循环，直接跳回到 “[” 元素的位置即可，否则跳到下一个元素
        if (tape[pointer] !== 0) {
          current = loopStartIndex;
          continue;
        }
        break;
    }
	
    // 跳到下一个字符
    current++;
  }

  // 返回最终的纸带
  return tape;
}

function checkSyntax(source = "") {
  // 合法字符
  const set = new Set([">", "<", "+", "-", ".", ",", "[", "]"]);
  // 记录 “[” 的数量
  let leftCount = 0;
  for (let i = 0; i < source.length; i++) {
    const char = source[i];
    // 非法字符检测
    if (!set.has(char)) {
      throw new Error(`Invalid char: ${char} at index ${i}`);
    }
    if (char === "[") {
      leftCount++;
    } else if (char === "]") {
      // 出现“]”，但是没有对应的“]”
      if (leftCount <= 0) {
        throw new Error(`Invalid Loop syntax at index ${i}`);
      }
      leftCount--;
    }
  }
  // 保证每一个“[”都有与之对应的“]”
  if (leftCount !== 0) {
    throw new Error("Invalid Loop syntax");
  }
}

```



### 使用方法

``` javascript
const tape = brainfuckInterpreter('+++>++>+');
console.log(tape);
// [3, 2, 1]
```

### brainfuck 编译器

实现一个解释器相对来说还是比较简单了，同时我还实现了一个简单的编译器，详情请看[博文](https://naecoo.github.io/brainfuck-compiler/)



## 项目地址

[brainfuck-interpreter](https://github.com/naecoo/brainfuck-interpreter)



## 相关资料

- [Brainfuck - Wikipedia](https://en.wikipedia.org/wiki/Brainfuck)
- [Visual brainfuck ](https://sites.google.com/site/visualbf/)
