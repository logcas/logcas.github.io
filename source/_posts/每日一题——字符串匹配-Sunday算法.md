---
title: 每日一题——字符串匹配-Sunday算法.md
date: 2019-12-04 16:35:30
tags: [LeetCode,算法]
---

## 概述

Sunday算法是Daniel M.Sunday于1990年提出的一种字符串模式匹配算法。其核心思想是：在匹配过程中，模式串并不被要求一定要按从左向右进行比较还是从右向左进行比较，它在发现不匹配时，算法能跳过尽可能多的字符以进行下一步的匹配，从而提高了匹配效率。

## 背景
Leecode上有一种实现strStr()函数的题，主要目的是对两个字符串进行匹配。

参考链接：https://leetcode-cn.com/problems/implement-strstr/submissions/

## 过程

当我第一次做这道题的时候，采用的是暴力法，依次匹配。时间复杂度上是O(m*n)，看上去虽然不是太糟糕，但暴力法做题可能失去了某种意义。于是找到了Sunday算法，也学习了一番。可以这么说，目前我觉得在效率和难易程度上，Sunday算法是绝对是效率高并且比较容易上手的算法。

为了更加清晰Sunday算法的整个过程，这里通过暴力法一步步演变。这样会更积极容易理解Sunday算法的核心思想。

### 暴力法求解

暴力法匹配的过程是通过不断移动模式串，使主串的首字符和模式串的首字符比较，如果首字符相等，则开始比较整个字符串。暴力法的效率在于，每次移动只移动一位，也就是说，存在很多重复且不必要的比较，因此浪费了效率。

![image](http://img.lxzmww.xyz/%E6%9A%B4%E5%8A%9B%E5%8C%B9%E9%85%8D%E5%AD%97%E7%AC%A6%E4%B8%B2.png)

此时，暴力法的代码是这样的：
```js
/**
 * @param {string} haystack
 * @param {string} needle
 * @return {number}
 */

function compare(full, cp) {
  for(let i = 0;i < cp.length; ++i) {
    if (full[i] !== cp[i]) {
      return false;
    }
  }
  return true;
}

var strStr = function(haystack, needle) {
    if (needle === '') {
      return 0;
    }
    for(let i = 0;i <= haystack.length - needle.length; ++i) {
      if (haystack[i] === needle[0]) {
        if (compare(haystack.slice(i), needle)) {
          return i;
        }
      }
    }
    return -1;
};
```

暴力法没有利用到其他的匹配信息，但Sunday算法利用了。现在我们一步一步的把上面的代码改写成Sunday算法。

### Sunday 算法

#### 匹配机制

Sunday算法匹配机制非常容易理解：

1. 目标字符串 `String`

2. 模式串 `Pattern`

3. 当前查询索引 `idx` （初始为 00）

4. 待匹配字符串 `str_cut`: `String [ idx : idx + len(Pattern) ]`

每次匹配都会从 目标字符串中 提取 待匹配字符串与 模式串 进行匹配：

1. 若匹配，则返回当前 `idx`

2. 不匹配，则查看 待匹配字符串 的后一位字符 `c`：

3. 若`c`存在于`Pattern`中，则 `idx = idx + 偏移表[c]`

4. 否则，`idx = idx + len(pattern)`

Repeat Loop 直到 `idx + len(pattern) > len(String)`

#### 偏移表

如果`c`存在于`Pattern`中，那么需要偏移多少才是合理呢？移动1位，相当于暴力法，太慢。移动太多了，会漏掉中间的字符串。所以，移动的偏移量取决于字符(如果有相等，取最右)在`Pattern`中向右边的距离+1。

例如 aab：
1. a 的偏移位就是 `1 + 1 = 2`
2. b 的偏移位就是 `0 + 1 = 1`

所以，Sunday 算法的开始需要为模式串建立一个偏移表：
```js
/**
 * @param {string} haystack
 * @param {string} needle
 * @return {number}
 */

function compare(full, cp) {
  for(let i = 0;i < cp.length; ++i) {
    if (full[i] !== cp[i]) {
      return false;
    }
  }
  return true;
}

var strStr = function(haystack, needle) {
    if (needle === '') {
      return 0;
    }
    
    // 建立偏移表
    const dic = new Map();
    for(let i = 0;i < needle.length; ++i) {
      dic.set(needle[i], needle.length - i);
    }
    
    for(let i = 0;i <= haystack.length - needle.length; ++i) {
      if (haystack[i] === needle[0]) {
        if (compare(haystack.slice(i), needle)) {
          return i;
        }
      }
    }
    return -1;
};
```

#### 匹配过程
这里从一个示例出发：

![image](http://img.lxzmww.xyz/Sunday0.jpg)

显然可以看到，匹配失败，此时面临一个抉择：移动X位。根据Sunday算法，我们直到需要看下一位，也就是主串中的字符`d`。可以看到，`d`是不存在于`abc`中的，也就是说，本次是移动`len('abc') + 1 = 4`位。

![image](http://img.lxzmww.xyz/Sunday1.jpg)

现在来到了这个地方，当然也是匹配失败。所以找到字符`b`，存在于模式串`abc`中，根据之前建立的偏移量表，可以很容易得出移动的位数是`2`。

**也就是说，每次移动，下一位的字符都会与模式串中的最右匹配的字符位置对应：**

![image](http://img.lxzmww.xyz/Sunday2.jpg)

这次也不匹配，差一点点。然后找到字符`z`，发现不在`abc`中，因此移动的位移为`len('abc') + 1 = 4`。

![image](http://img.lxzmww.xyz/Sunday3.jpg)

然后，我们就顺利地找到了匹配的位置。这就是Sunday算法的过程。

最终代码如下：
```js
/**
 * @param {string} haystack
 * @param {string} needle
 * @return {number}
 */

function compare(full, cp) {
  for(let i = 0;i < cp.length; ++i) {
    if (full[i] !== cp[i]) {
      return false;
    }
  }
  return true;
}

var strStr = function(haystack, needle) {
    if (needle === '') {
      return 0;
    }
    const dic = new Map();
    for(let i = 0;i < needle.length; ++i) {
      dic.set(needle[i], needle.length - i);
    }
    for(let i = 0;i <= haystack.length - needle.length;) {
      if (haystack[i] === needle[0]) {
        if (compare(haystack.slice(i), needle)) {
          return i;
        } else {
          const k = i + needle.length;
          if (k < haystack.length) {
            let plus = needle.length + 1;
            if (dic.has(haystack[k])) {
              plus = dic.get(haystack[k]);
            }
            i += plus;
          } else break;
        }
      } else {
        ++i;
      }
    }
    return -1;
};
```

#### 复杂度分析
##### 空间复杂度
空间上，只比暴力法增加了一个偏移量表。但这个表存储的方式非常多，可以用数组和字典。如果用数组，在纯字母的匹配下，也就48个字符，是常数。此时的空间复杂度为O(1)。其他情况，需要看匹配串的字符种类数量。

##### 时间复杂度
时间上比较好分析。

我们取两种极端的情况：
1. 每次偏移都是`len(pattern) + 1`，也就是下一个字符永远都不是模式串中的字符。

![image](http://img.lxzmww.xyz/Sunday4.jpg)

这时候时间复杂度最低，为O(m/n)。

2. 每次偏移只偏移一位
![image](http://img.lxzmww.xyz/Sunday5.jpg)

因为只偏移一位，因此Sunday算法就退化为暴力法，这时候时间复杂度为O(m*n)。

## 参考链接
https://leetcode-cn.com/problems/implement-strstr/solution/python3-sundayjie-fa-9996-by-tes/

https://www.cnblogs.com/sunsky303/p/11693792.html