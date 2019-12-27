---
title: 每日一题——【专题】N数之和的解决方法.md
date: 2019-12-06 11:19:36
tags: [LeetCode,算法]
---

## 前言
我们经常会看到类似于两数之和、三数之和、四数之和的题目，这些都是比较经典的算法题，难度不大。其中，两数之和属于简单题，三数之和属于中等难度题（偏简单的中等）；而四数之和则为三数之和的变形，区别不大，只要掌握三数之和的做法，那么四数只是多了层循环而已。

这些题的共同之处在于它们都跟双指针的方案有关，所以这里归纳一下N数之和的解决方法，看看有哪些相同的方法。

## 两数之和
两数之和属于简单题，但是这一题有两种变形：
1. 输入为有序数组
2. 输入为无序数组，并且求索引位置

先来看看第一种：
### 输入：有序数组
对于有序数组，例如：
```
[1,3,4,6,7,9]
```

只要设置两个指针，指针`left = 0`，指针`right = nums.length - 1`。

假设，`result = nums[left] + nums[right]`，即双指针指向的元素和为`result`。

只要`result === target`，就找到了答案。

当`result > target`时，因为数组有序，肯定是`right`指针的元素过大了，因此移动`right`指针，`right--`。

当`result < target`时，肯定是因为`left`指针的元素太小了，因此`left++`。

结束循环时机：`left >= right`。

这样，每个元素最多只经过一次遍历，所以时间复杂度为O(n)。

```js
/**
 * @param {number[]} numbers
 * @param {number} target
 * @return {number[]}
 */
var twoSum = function(numbers, target) {
    let i = 0;
    let j = numbers.length - 1;
    while(i < j) {
        const sum = numbers[i] + numbers[j];
        if (sum === target) {
            return [i + 1, j + 1];
        } else if (sum > target) {
            --j;
        } else {
            ++i;
        }
    }
};
```

### 输入：无序数组
通常，输入的是无序数组的话，都不会让你求元素是什么组合，而是求**元素所在的索引**。因为这样，你才无法通过排序辅助，因为一排序跟输入有序数组就一样了，这个题也没有什么意义了。

可以想一下， 如果通过暴力法，两次循环，时间复杂度为O(n^2)，空间复杂度为O(1)。

因为输入无序数组，因此双指针法貌似没有效果了。

这题的突破口是通过空间换时间，我们设置一个辅助的字典结构。每次遍历时，我们都保存一下当前元素以及它的索引位置。

```
map[元素值] = 索引
```

通过字典的辅助，我们可以在每次遍历中，求插值`delta = target - nums[i]`，说明如果要与`nums[i]`构成等于`target`的值，则需要`delta`。通过字典我们可以很快地找出是否有元素等于`delta`，即`map.has(delta)`。如果有，就很容易得出`[i, map.get[delta]]`的索引结果。

通过一个辅助结构，空间复杂度为O(n)，时间复杂度依然为O(n)。

```js
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
var twoSum = function(nums, target) {

    const map = new Map();
    for(let i = 0;i < nums.length; ++i) {
        const t = target - nums[i];
        if (map.has(t)) {
            const j = map.get(t);
            return i < j ? [i, j] : [j, i];
        }
        map.set(nums[i], i);
    }
    
};
```

## 三数之和
LeetCode上的三数之和的题是这样的：
```
给定一个包含 n 个整数的数组 nums，判断 nums 中是否存在三个元素 a，b，c ，使得 a + b + c = 0 ？找出所有满足条件且不重复的三元组。

注意：答案中不可以包含重复的三元组。

例如, 给定数组 nums = [-1, 0, 1, 2, -1, -4]，

满足要求的三元组集合为：
[
  [-1, 0, 1],
  [-1, -1, 2]
]
```

输入的是一个无序数组，需要返回一个元素组合数组。

细想一下，三数之和实际上是**两数之和+固定一位数**。并且，题目求的是元素而不是元素的索引，那么说明我们可以通过排序+双指针解决两数之和的问题，因此，只要我们固定一位数，然会就变成了解两数之和的问题了。

但是需要注意的是，这里需要进行去重。因为我们是通过排序解决的，那么排序后重复的元素都会相邻，这样，我们可以在移动的时候判断下一个元素与当前元素是否相等，如果相等，就跳过。

```js
/**
 * @param {number[]} nums
 * @return {number[][]}
 */
var threeSum = function(nums) {
    if (nums.length < 3) {
        return [];
    }
    nums.sort((a, b) => a - b);
    const arr = [];
    let i = 0;
    while(i < nums.length) {
        if (nums[i] > 0) {
            break;
        }
        let left = i + 1;
        let right = nums.length - 1;
        while(left < right) {
            let target = nums[i] + nums[left] + nums[right];
            if (target === 0) {
                arr.push([nums[i], nums[left], nums[right]]);
                while(left < nums.length - 1 && nums[left] === nums[left + 1]) {
                    ++left;
                }
                while(right > 0 && nums[right] === nums[right - 1]) {
                    --right;
                }
                ++left;
                --right;
            } else if (target > 0) {
                --right;
            } else {
                ++left;
            }
        }
        while(i < nums.length - 1 && nums[i] === nums[i + 1]) {
            ++i;
        }
        ++i;
    }
    return arr;
};
```

时间复杂度：排序O(nlogn) + 双指针法*循环O(n * n)，因此总体的时间复杂度为O(n^2)。

空间复杂度：O(1)

## 四数之和
从上面可以看到：

三数之和 = 两数之和 + 固定一位

那么，四数之和相当于：

四数之和 = 两数之和 + 固定两位

固定一位需要一层循环，那么固定两位则需要两层循环。

```js
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[][]}
 */
var fourSum = function(nums, target) {
    if (nums.length < 4) {
        return [];
    }
    const arr = [];
    nums.sort((a, b) => a - b);
    let i = 0;
    while(i < nums.length) {
        let j = i + 1;
        while(j < nums.length) {
            let left = j + 1;
            let right = nums.length - 1;
            while(left < right) {
                const result = nums[i] + nums[j] + nums[left] + nums[right];
                if (target === result) {
                    arr.push([nums[i], nums[j], nums[left], nums[right]]);
                    while(left < nums.length - 1 && nums[left] === nums[left + 1]) {
                        left++;
                    }
                    while(right > 0 && nums[right] === nums[right - 1]) {
                        right--;
                    }
                    ++left;
                    --right;
                } else if (result > target) {
                    --right;
                } else {
                    ++left;
                }
            }
            while(j < nums.length - 1 && nums[j] === nums[j + 1]) {
                ++j;
            }
            ++j;
        }
        while(i < nums.length - 1 && nums[i] === nums[i + 1]) {
            ++i;
        }
        ++i;
    }
    return arr;
};
```

时间复杂度：O(n^3)

## 总结
可以看到，双指针法在解决有序数组中的一些问题有很高效率的帮助，因此在遇到类似的题目时，可以往双指针的方向去考虑。