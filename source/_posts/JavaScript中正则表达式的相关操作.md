---
title: JavaScript中正则表达式的相关操作
date: 2020-02-03 16:36:14
tags: [JavaScript, 正则表达式]
---

## String
### `replace`
参数：
* [1]: regex\string 匹配的模式\字符串
  * regex: 正则表达式，如果在全局模式`g`下，会不断地寻找匹配的文本并替换。
  * string: 字符串，只会寻找第一个匹配的文本，不会继续往下寻找。
* [2]: function\string 替换的函数\字符串
  * function: 函数，参数为(**匹配的文本**，*子匹配串1*，*子匹配串2*,..., *子匹配串n*, 源字符串的引用)，返回一个字符串表示替换匹配的文本。
  * string: 字符串，表示要代替**匹配的文本**的内容，通过一些特殊的记号表示子匹配串或其余一些位置。
    * $1、$2、$3、..、$N，表示子匹配串N
    * $& 与 regexp 相匹配的子串。
    * $` 位于匹配子串左侧的文本。
    * $' 位于匹配子串右侧的文本。

例子：
* 将一个英文句子的首字母大写
```js
const regex = /\b([a-z])(\w*)\b/g;
const str = 'hello, world, goodbye gentlemen';
console.log(str.replace(regex, (match, $1, $2) => $1.toUpperCase() + $2));
// Hello, World, Goodbye Gentlemen
```

* `trim` 方法模拟
```js
function trim1(str) {
  return str.replace(/^\s+|\s+$/g, '');
}

function trim2(str) {
  return str.replace(/^\s*(.*?)\s*$/g, '$1');
}
```

### `match`
参数：
 * [1]：regexp，正则表达式

返回：
 * 当在`g`全局环境下，返回一个数组，数组中是匹配该正则的文本，不包含子匹配串（也就是没有捕获）。
 * 当非全局模式下，返回一个类数组，第0个元素为匹配的文本，第1个元素开始为子匹配串，依次到N，其中还有`index`属性表示匹配文本开始的位置。

例子：
```js
const regexex = /aa([A-Z]+)bb/g;
const strstr = 'aaBBbb aaNADKJNFASKbb aaccbb aaSSSbb AAjkdnsakBB';
console.log(strstr.match(/aa([A-Z]+)bb/g));
console.log(strstr.match(/aa([A-Z]+)bb/));
// [ 'aaBBbb', 'aaNADKJNFASKbb', 'aaSSSbb' ]
// [
//   'aaBBbb',
//   'BB',
//   index: 0,
//   input: 'aaBBbb aaNADKJNFASKbb aaccbb aaSSSbb AAjkdnsakBB',
//   groups: undefined
// ]
```

## RegExp
### `test`
参数：一个字符串，如果字符串符合该正则，返回`true`，否则返回`false`。

### `exec`
参数：一个字符串

返回：
  * 当RegExp对象为非全局模式下，返回的跟`String.prototype.match`结果一致。
  * 当全局模式下，返回一个类数组，跟`String.protype.match`返回结果一致。但RegExp对象内部记录了`lastIndex`变量，因此，可以重复调用`exec()`，它会从上次停顿的地方继续向前匹配。当不存在匹配的文本时，返回NULL。简而言之，`RegExp.prototype.exec`在全局模式下通过不断调用可以实现非全局模式的输出。

例子：
```js
const regexex = /aa([A-Z]+)bb/g;
const strstr = 'aaBBbb aaNADKJNFASKbb aaccbb aaSSSbb AAjkdnsakBB';

let _a;
while(_a = regexex.exec(strstr)) {
  console.log(_a);
}
// [
//   'aaBBbb',
//   'BB',
//   index: 0,
//   input: 'aaBBbb aaNADKJNFASKbb aaccbb aaSSSbb AAjkdnsakBB',
//   groups: undefined
// ]
// [
//   'aaNADKJNFASKbb',
//   'NADKJNFASK',
//   index: 7,
//   input: 'aaBBbb aaNADKJNFASKbb aaccbb aaSSSbb AAjkdnsakBB',
//   groups: undefined
// ]
// [
//   'aaSSSbb',
//   'SSS',
//   index: 29,
//   input: 'aaBBbb aaNADKJNFASKbb aaccbb aaSSSbb AAjkdnsakBB',
//   groups: undefined
// ]
```
