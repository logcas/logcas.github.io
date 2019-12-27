---
title: 每日一题——【简单】反转字符串中的单词 III.md
date: 2019-12-10 11:37:41
tags: [LeetCode,算法]
---

### 题目
给定一个字符串，你需要反转字符串中每个单词的字符顺序，同时仍保留空格和单词的初始顺序。

示例 1:
```
输入: "Let's take LeetCode contest"
输出: "s'teL ekat edoCteeL tsetnoc" 
```
注意：在字符串中，每个单词由单个空格分隔，并且字符串中不会有任何额外的空格。

### 解决方法
```js
/**
 * @param {string} s
 * @return {string}
 */
var reverseWords = function(s) {
    const words = [];
    let str = '';
    for(let i = 0;i < s.length; ++i) {
        if (s[i] === ' ') {
            if (str !== '') {
                words.push(str);
                str = '';
            }
        } else {
            str = s[i] + str;
        }
    }
    if (str !== '') {
        words.push(str);
    }
    return words.join(' ');
};
```