---
title: 每日一题——【困难】合并K个有序链表
date: 2020-01-15 12:23:17
tags: [LeetCode,算法,链表]
---

## 题目
合并 k 个排序链表，返回合并后的排序链表。请分析和描述算法的复杂度。

示例:
```
输入:
[
  1->4->5,
  1->3->4,
  2->6
]
输出: 1->1->2->3->4->4->5->6
```

## 解决方法

### 直接合并
时间：O(kN)

空间：O(1)

```js
/**
 * Definition for singly-linked list.
 * function ListNode(val) {
 *     this.val = val;
 *     this.next = null;
 * }
 */
/**
 * @param {ListNode[]} lists
 * @return {ListNode}
 */
var mergeKLists = function (lists) {
    let count = lists.length;
    if (!lists.length) {
        return null;
    }
    let newHead = new ListNode();
    let cursor = newHead;
    let p = null;
    let min = Infinity;
    let index = -1;
    while (count > 1) {
        for (let i = 0; i < lists.length; ++i) {
            if (!lists[i]) {
                continue;
            }
            if (lists[i].val < min) {
                p = lists[i];
                min = lists[i].val;
                index = i;
            }
        }
        if (index === -1) {
            break;
        }
        cursor.next = p;
        cursor = cursor.next;
        lists[index] = lists[index].next;
        if (!lists[index]) {
            --count;
        }
        min = Infinity;
        p = null;
        index = -1;
    }
    let last = null;
    for(let i = 0;i < lists.length; ++i) {
        if (lists[i]) {
            last = lists[i];
            break;
        }
    }
    cursor.next = last;
    return newHead.next;
};
```

### 两两合并
时间：O(NlogK)

空间：O(1)
```js
/**
 * Definition for singly-linked list.
 * function ListNode(val) {
 *     this.val = val;
 *     this.next = null;
 * }
 */
/**
 * @param {ListNode[]} lists
 * @return {ListNode}
 */
var mergeKLists = function(lists) {
    if (lists.length === 0) {
        return null;
    }
    let _lists = [...lists];
    while(_lists.length > 1) {
        let tmp = [];
        let i = 0;
        let j = _lists.length - 1;
        while(i <= j) {
            let newList = mergeTwoList(_lists[i++], _lists[j--]);
            tmp.push(newList);
        }
        _lists = tmp;
    }
    return _lists[0];
};

function mergeTwoList(list1, list2) {
    if (!list1) {
        return list2;
    }
    if (!list2) {
        return list1;
    }
    if (list1 === list2) {
        return list1;
    }
    const newHead = new ListNode();
    let cursor = newHead;
    let p1 = list1;
    let p2 = list2;
    while(p1 && p2) {
        if (p1.val > p2.val) {
            cursor.next = p2;
            p2 = p2.next;
        } else {
            cursor.next = p1;
            p1 = p1.next;
        }
        cursor = cursor.next;
    }
    cursor.next = p1 ? p1 : p2;
    return newHead.next;
}
```