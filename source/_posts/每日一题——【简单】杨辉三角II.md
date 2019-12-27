---
title: 每日一题——【简单】杨辉三角II.md
date: 2019-12-10 11:35:34
tags: [LeetCode,算法]
---

### 题目
给定一个非负索引 k，其中 k ≤ 33，返回杨辉三角的第 k 行。

![image](https://upload.wikimedia.org/wikipedia/commons/0/0d/PascalTriangleAnimated2.gif)

在杨辉三角中，每个数是它左上方和右上方的数的和。

示例:
```
输入: 3
输出: [1,3,3,1]
进阶：
```
你可以优化你的算法到 O(k) 空间复杂度吗？

### 解决方法
限制空间复杂度为O(k)，说明最多申请一个长度为k的辅助数组。

从演示图也可以知道，我们必须要计算完上一轮的数值才能获得下一轮的数值。

因此，我们可以申请一个辅助数组`tmp`，计算本轮的数值，数值`res`作为返回的数组。

每次循环，`tmp`从`res`中获取上一轮的数值计算本轮数值，最后把`tmp`变成`res`。

```js
/**
 * @param {number} rowIndex
 * @return {number[]}
 */
var getRow = function(rowIndex) {
    let res = [1];
    if (rowIndex === 0) {
        return res;
    }
    for(let i = 0;i < rowIndex; ++i) {
        let tmp = [1];
        for(let j = 1; j < res.length + 1; ++j) {
            tmp[j] = (j === res.length ? 0 : res[j]) + res[j - 1];
        }
        res = tmp;
    }
    return res;
};
```