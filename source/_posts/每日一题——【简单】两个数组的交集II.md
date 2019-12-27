---
title: 每日一题——【简单】两个数组的交集II.md
date: 2019-12-11 11:40:39
tags: [LeetCode,算法]
---

### 题目
给定两个数组，编写一个函数来计算它们的交集。

示例 1:
```
输入: nums1 = [1,2,2,1], nums2 = [2,2]
输出: [2,2]
```
示例 2:
```
输入: nums1 = [4,9,5], nums2 = [9,4,9,8,4]
输出: [4,9]
```
说明：

* 输出结果中每个元素出现的次数，应与元素在两个数组中出现的次数一致。
* 我们可以不考虑输出结果的顺序。

### 解决方法

#### 哈希法
通过构造一个哈希映射（value => count），先遍历数组1，建立映射。然后遍历数组2，当`count > 0`时则存在交集。

```js
/**
 * @param {number[]} nums1
 * @param {number[]} nums2
 * @return {number[]}
 */
var intersect = function(nums1, nums2) {
    if (nums1.length === 0 || nums2.length === 0) {
        return [];
    }
    const res = [];
    const map = new Map();
    for(let i = 0;i < nums1.length; ++i) {
        if (map.has(nums1[i])) {
            map.set(nums1[i], map.get(nums1[i]) + 1);
        } else {
            map.set(nums1[i], 1);
        }
    }
    for(let i = 0;i < nums2.length; ++i) {
        if (!map.has(nums2[i])) {
            continue;
        }
        let t = map.get(nums2[i]);
        t -= 1;
        if (t === 0) {
            map.delete(nums2[i]);
        } else {
            map.set(nums2[i], t);
        }
        res.push(nums2[i]);
    }
    return res;
};
```

时间复杂度：O(max(n,m))

空间复杂度：O(min(n,m))

#### 双指针法
当数组有序时，可以通过双指针法进行求交集。当数组无序，如果题目没有限制，可以先将数组排序，然后再使用双指针法。

分别设置`p1`、`p2`指针指向数组1、数组2，然后遍历：
1. 当`p1`元素等于`p2`元素时，添加该元素到集合中，两个指针都指向下一个。
2. 当`p1` > `p2`时，`p2`指向下一个。
3. 当`p1` < `p2`时，`p1`指向下一个。

结束条件：`p1`或`p2`超出了所在数组的最大长度。

```js
/**
 * @param {number[]} nums1
 * @param {number[]} nums2
 * @return {number[]}
 */
var intersect = function(nums1, nums2) {
    
    // 排序 + 双指针
    nums1.sort((a, b) => a - b);
    nums2.sort((a, b) => a - b);
    
    let p1 = 0;
    let p2 = 0;
    const res = [];
    
    while(p1 < nums1.length && p2 < nums2.length) {
        if (nums1[p1] === nums2[p2]) {
            res.push(nums1[p1]);
            ++p1;
            ++p2;
        } else if (nums1[p1] > nums2[p2]) {
            ++p2;
        } else {
            ++p1;
        }
    }
    
    return res;
    
};
```

时间复杂度：O(max(nlogn, mlogm))

空间复杂度：O(1)

