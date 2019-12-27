---
title: 每日一题——【简单】删除链表的倒数第N个节点.md
date: 2019-12-20 11:37:10
tags: [LeetCode,算法]
---

## 题目
给定一个链表，删除链表的倒数第 n 个节点，并且返回链表的头结点。

示例：
```
给定一个链表: 1->2->3->4->5, 和 n = 2.

当删除了倒数第二个节点后，链表变为 1->2->3->5.
```
说明：

* 给定的 n 保证是有效的。

进阶：

* 你能尝试使用一趟扫描实现吗？


## 解决方法
不管是一趟扫描还是两趟扫描，算法复杂度都为O(n)。但是为了实现一趟扫描，就需要用到双指针法了。双指针法在链表中非常常用，包括前后指针，快慢指针。

这题用的是前后指针。

1. 前指针`p0`，先在链表中向前走N步，与后指针`p1`形成距离N。
2. 同时遍历两个指针，直到前指针`p0`为空。
3. 当`p0`为空时，后指针`p1`所在的位置就是需要删除的结点的位置。

然后，需要处理的细节是：我们也需要在遍历的过程中确定一个“删除结点的前一个结点”的指针`prev`。

为了更方便处理问题，设一个空结点`newHead`，`newHead.next = head`，`prev = newHead`。这样不管删除的结点是第一个还是第几个，都是跟普通处理方式一样。最后返回`newHead.next`就可以了。

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
    let p = head;
    while(n--) {
        p = p.next;
    }
    let prev = {
        next: head
    };
    let newHead = prev;
    let q = head;
    while(p) {
        p = p.next;
        prev = q;
        q = q.next;
    }
    prev.next = q.next;
    return newHead.next;
};
```
