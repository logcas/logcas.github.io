---
title: 每日一题——【中等】删除链表的倒数第N个节点
date: 2020-01-15 12:22:54
tags: [LeetCode,算法,链表]
---

## 题目
给定一个链表，删除链表的倒数第 n 个节点，并且返回链表的头结点。

示例：
```
给定一个链表: 1->2->3->4->5, 和 n = 2.

当删除了倒数第二个节点后，链表变为 1->2->3->5.
```
说明：

* 给定的 n 保证是有效的。

进阶：

* 你能尝试使用一趟扫描实现吗？

## 解决方法
先设置指针p让其前进n步，再设置指针deleted，让p和deleted同时前进，直到p为NULL，那么，deleted所在位置就是需要删除的位置。

单向链表删除操作需要获取前一个节点，因此在deleted前进的同时记录前一个节点pre，删除时使用。

```js
/**
 * Definition for singly-linked list.
 * function ListNode(val) {
 *     this.val = val;
 *     this.next = null;
 * }
 */
/**
 * @param {ListNode} head
 * @param {number} n
 * @return {ListNode}
 */
var removeNthFromEnd = function(head, n) {
    let tmp = new ListNode();
    tmp.next = head;
    let p = head;
    let deleted = head;
    while(n--) {
        p = p.next;
    }
    let pre = tmp;
    while(p) {
        p = p.next;
        pre = deleted;
        deleted = deleted.next;
    }
    pre.next = deleted.next;
    return tmp.next;
};
```