---
title: 每日一题——【简单】数组拆分 I.md
date: 2019-12-05 11:23:57
tags: [LeetCode,算法]
---

### 题目
给定长度为 2n 的数组, 你的任务是将这些数分成 n 对, 例如 (a1, b1), (a2, b2), ..., (an, bn) ，使得从1 到 n 的 min(ai, bi) 总和最大。

示例 1:
```
输入: [1,4,3,2]

输出: 4
解释: n 等于 2, 最大总和为 4 = min(1, 2) + min(3, 4).
```
提示:
```
n 是正整数,范围在 [1, 10000].
数组中的元素范围在 [-10000, 10000].
```


### 解决方法
这题做的时候是出现在LeetCode双指针专题的，但是实际上跟双指针又没什么关系。

求 min(a, b) 的合的最大值，那么a、b这两个数越接近，那么和就越大。

因此可以很容易想到排序之后的数组，元素相互之间的差就最小，然后就是求偶数位置的和。

```js
/**
 * @param {number[]} nums
 * @return {number}
 */
var arrayPairSum = function(nums) {
    const _nums = [...nums];
    _nums.sort((a, b) => a - b);
    return _nums.reduce((pre, cur, index) => {
        return pre + (index % 2 === 0 ? cur : 0);
    }, 0);
};
```