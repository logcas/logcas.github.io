---
title: 每日一题——【中等】环形链表II
date: 2020-01-14 11:28:41
tags: [LeetCode,算法]
---

## 题目
给定一个链表，返回链表开始入环的第一个节点。如果链表无环，则返回null。

为了表示给定链表中的环，我们使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 pos 是 -1，则在该链表中没有环。

说明：不允许修改给定的链表。



示例 1：
```
输入：head = [3,2,0,-4], pos = 1
输出：tail connects to node index 1
解释：链表中有一个环，其尾部连接到第二个节点。
```
![image](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist.png)

示例 2：
```
输入：head = [1,2], pos = 0
输出：tail connects to node index 0
解释：链表中有一个环，其尾部连接到第一个节点。
```
![image](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test2.png)

示例 3：
```
输入：head = [1], pos = -1
输出：no cycle
解释：链表中没有环。
```
![image](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test3.png)

## 解决方法
### 方法一：借助Set
借助一个数据结构，存储节点，每遍历一个节点，先寻找Set中有没有存储，如果有，则代表有环且节点为环的入口节点。

时间复杂度：O(n)

空间复杂度：O(n)

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
 * @return {ListNode}
 */
var detectCycle = function(head) {
    const set = new Set();
    let p = head;
    while(p) {
        if (set.has(p)) {
            return p;
        }
        set.add(p);
        p = p.next;
    }
    return null;
};
```

### 方法二：快慢指针
![image](http://static-cdn.lxzmww.xyz/环形链表2.png)

假设如图所示，我们设定一个快指针，一个慢指针，快指针每次走2步，慢指针走一步。当快指针为空时，可以断定该链表不是环形链表，直接返回`null`。

当快慢指针相遇时，相遇点如图，此时有：
```
快指针路程 = 2 * 慢指针路程
```

而
```
快指针路程 = A + B + C + B
慢指针路程 = A + B
```

因此
```
A + 2B + C = 2A + 2B
         C = A
```

因此，最后令慢指针从起点出发，快慢指针每次同时前进一步，相遇时则到达环形链表入口。

时间复杂度：O(n)

空间复杂度：O(1)

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
 * @return {ListNode}
 */
var detectCycle = function(head) {
    if (!head) {
        return null;
    }
    let slow = head;
    let fast = head;
    let first = true;
    while(slow !== fast || first) {
        first = false;
        if (fast && fast.next) {
            fast = fast.next.next;
        } else {
            return null;
        }
        slow = slow.next;
    }

    slow = head;
    while(slow !== fast) {
        slow = slow.next;
        fast = fast.next;
    }

    return slow;
};
```