---
title: 从JavaScript执行机制理解作用域链、变量提升、以及闭包
date: 2020-02-06 23:11:29
tags: [JavaScript]
---

## 什么是执行上下文？
简单来说，执行上下文是一种当`JavaScript`代码执行和评估时的一种抽象的环境概念。任何`JavaScript`代码都运行在一个执行上下文中。

### 执行上下文的类型
* **全局执行上下文**
全局执行上下文是默认且基本的执行上下文。任何不在函数中的代码都是执行在全局执行上下文中。它主要有两个作用：创建一个全局对象（例如，在浏览器中就是`window`，并且把`this`指向这个全局对象。一个程序中只能有一个全局执行上下文。

* **函数执行上下文**
每当一个函数被执行或调用时，都会创建一个全新的执行上下文。一个函数可以有多个执行上下文，但只有被调用时创建。它的创建过程将在后文中提到。

* **`eval`执行上下文**
在`eval`中执行的代码也会创建一个自己的执行上下文，但这个不常用，就暂且先不说了。

## 执行栈
执行栈是一种栈结构，也就是拥有LIFO特性的数据结构，用来存放代码执行过程中所有的执行上下文。

当`JavaScript`引擎遇到代码时，会首先创建一个全局执行上下文并且推到执行栈中。当引擎执行到函数被调用，就会再创建一个函数执行上下文，然后把它推入栈中。

`JavaScript`引擎执行处于执行栈栈顶的执行上下文对应的函数。当函数执行完毕，该执行上下文就会从执行栈中弹出，然后就会继续处理执行栈中新的栈顶的执行上下文。

例如，有以下代码：
```js
let a = 'Hello World!';
function first() {
  console.log('Inside first function');
  second();
  console.log('Again inside first function');
}
function second() {
  console.log('Inside second function');
}
first();
console.log('Inside Global Execution Context');
```

执行栈的变化是这样的：
```
------------
global [top]
------------

------------
first [top]
global
------------

------------
second [top]
first 
global
------------

------------
first [top]
global
------------

------------
global [top]
------------
```

## 执行上下文的创建过程
在JavaScript代码执行之前，比如调用一个函数：在执行函数内部代码前，会先创建函数执行上下文。

执行上下文的创建有两个阶段：
1. 创建阶段
2. 执行阶段

### 创建阶段
执行上下文在创建阶段建立，主要有以下操作：
1. **词法环境**组件创建
2. **变量环境**组件创建

用伪代码表示如下：
```
ExecutionContext = {
  LexicalEnvironment = <ref. to LexicalEnvironment in memory>,
  VariableEnvironment = <ref. to VariableEnvironment in  memory>,
}
```

### 词法环境
根据ES6官方文档，词法环境是这样定义的：

> 词法环境是一种用于根据ECMAScript代码的词法嵌套结构定义标识符与变量和函数的特殊类型。一个词法环境由一个环境记录器以及外层词法环境引用（可能为NULL）组成。

简单的来说，可以把词法环境想象成一个`key-value`的结构，`key`为标识符，`value`为变量（对象或原始类型）或函数的引用。

例如，有以下的代码片段：
```js
let a = 20;
let b = 40;
function foo() {
  console.log('bar');
}
```

当整个代码片段执行完，它的词法环境就会变成这样：
```
lexicalEnvironment = {
  a: 20,
  b: 40,
  foo: <ref. to foo function>
}
```

每一个词法环境由以下三部分组成：
1. 环境记录器
2. 外层词法环境的引用
3. `this`指向

#### 环境记录器
环境记录器用于存储变量和函数的定义，比如上面的`a`、`b`、`foo`就是存储在环境记录器中。

而环境记录器又有两种类型，分别用于全局和函数执行上下文的场景：
1. 明式环境记录器，存储变量、函数和参数。
2. 对象环境记录器，用来定义出现在全局上下文中的变量和函数的关系。

简而言之，

* 在全局环境中，环境记录器是对象环境记录器。
* 在函数环境中，环境记录器是声明式环境记录器。

*另外，对于函数环境，声明式环境记录器还包含了一个传递给函数的 arguments 对象（此对象存储索引和参数的映射）和传递给函数的参数的 length。*

```js
function foo(a, b) {
  var c = a + b;
}
foo(2, 3);
// argument object
Arguments: {0: 2, 1: 3, length: 2},
```

#### 外层词法环境的引用
**外层词法环境的引用**指向外层的环境。这意味着`JavaScript`引擎可以通过它寻找不在当前词法环境的变量。对于全局执行上下文而言，它为`NULL`。对于函数执行上下文，根据函数定义时外层嵌套的词法作用域。

#### `this`指向
对于全局执行上下文，`this`指向全局对象。

对于函数执行上下文，`this`的指向取决于函数是怎么被调用的。如果函数通过对象方法的形式被调用，那么`this`指向该对象。如果通过显示指向，例如通过`call`、`bind`、`apply`显示定义，那么指向传入的`this`。否则，默认指向全局对象或者`undefined`（严格模式下）。

### 变量环境
变量环境也是词法环境的一种，组成跟上面说的词法环境一毛一样。

但是在ES6中，词法环境和变量环境的不同之处在于，`let`和`const`的变量定义在前者，`var`定义的变量存储在后者。

例如：
```js
let a = 20;
const b = 30;
var c;
function multiply(e, f) {
 var g = 20;
 return e * f * g;
}
c = multiply(20, 30);
```

此时，全局执行上下文如下：
```
GlobalExectionContext = {
  LexicalEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      a: < uninitialized >,
      b: < uninitialized >,
      multiply: < func >
    }
    outer: <null>,
    ThisBinding: <Global Object>
  },
  VariableEnvironment: {
    EnvironmentRecord: {
      Type: "Object",
      // Identifier bindings go here
      c: undefined,
    }
    outer: <null>,
    ThisBinding: <Global Object>
  }
}
```

可以看到，`a`、`b`存储在词法环境的环境记录器中，而`c`因为通过`var`去定义，所以存储在变量环境的环境记录器中。

另外，可以注意到，，`let`和`const`定义的变量并没有关联到任何值，是`< uninitialized >`，而`var`定义的变量初始值已经设为的`undefined`。

这是因为在创建阶段，引擎检查代码并且找出定于变量和函数声明，函数声明完全存储在环境中，而变量定义则不同。（`var`下为`undefined`，`let`和`const`则为未初始化），这就是我们常说的变量提升。因此，即使是`let`和`const`也是有变量提升的，但是因为它是未初始化的，这就是为什么，在执行到定义语句前获取`let`和`const`的变量，会报错，而`var`的变量未`undefined`。


### 作用域链的原理
可执行上下文中的词法环境中含有外部词法环境的引用，我们可以通过这个引用获取外部词法环境的变量、声明等，这些引用串联起来一直指向全局的词法环境，因此形成了作用域链。

### 闭包的原理
可执行上下文中的词法环境中含有外部词法环境的引用，我们可以通过这个引用获取外部词法环境的变量、声明等，因此形成了闭包。