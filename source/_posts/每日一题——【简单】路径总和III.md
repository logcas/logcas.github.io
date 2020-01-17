---
title: 每日一题——【简单】路径总和III
date: 2020-01-17 11:00:49
tags: [LeetCode,算法]
---

## 题目

给定一个二叉树，它的每个结点都存放着一个整数值。

找出路径和等于给定数值的路径总数。

路径不需要从根节点开始，也不需要在叶子节点结束，但是路径方向必须是向下的（只能从父节点到子节点）。

二叉树不超过1000个节点，且节点数值范围是 [-1000000,1000000] 的整数。

示例：
```
root = [10,5,-3,3,2,null,11,3,-2,null,1], sum = 8

      10
     /  \
    5   -3
   / \    \
  3   2   11
 / \   \
3  -2   1

返回 3。和等于 8 的路径有:

1.  5 -> 3
2.  5 -> 2 -> 1
3.  -3 -> 11
```

## 解决方法


### 暴力法

双层递归，从每一个节点开始计算，存在大量重复计算。

时间复杂度：O(n^2)

```js
/**
 * Definition for a binary tree node.
 * function TreeNode(val) {
 *     this.val = val;
 *     this.left = this.right = null;
 * }
 */
/**
 * @param {TreeNode} root
 * @param {number} sum
 * @return {number}
 */
var pathSum = function(root, sum) {
    if (!root) {
        return 0;
    }
    return dfs(root, sum) + pathSum(root.left, sum) + pathSum(root.right, sum);
};

function dfs(root, target) {
    if (!root) {
        return 0;
    }
    let count = 0;
    if (root.val === target) {
        count = 1;
    }
    return count + dfs(root.left, target - root.val) + dfs(root.right, target - root.val);
}
```


### 单层递归法

设立一个数组`paths`，记录第`n`层的节点大小为`paths[n]`，当遍历到某节点时，逆序遍历`paths`寻找是否等于路径大小的累积结果。

时间复杂度：O(nlogn)，其中logn为平均路径长度。

```js
/**
 * Definition for a binary tree node.
 * function TreeNode(val) {
 *     this.val = val;
 *     this.left = this.right = null;
 * }
 */
/**
 * @param {TreeNode} root
 * @param {number} sum
 * @return {number}
 */
var pathSum = function(root, sum) {
    const paths = [];
    return helper(root, sum, paths, 0);
}

function helper(root, sum, paths, p) {
    if (!root) {
        return 0;
    }
    let res = 0;
    let cur = root.val;
    if (cur === sum) {
        res++;
    }
    for(let i = p - 1;i >= 0; --i) {
        cur += paths[i];
        if (cur === sum) {
            res++;
        }
    }
    paths[p] = root.val;
    let res1 = helper(root.left, sum, paths, p + 1);
    let res2 = helper(root.right, sum, paths, p + 1);
    return res + res1 + res2;
}
```