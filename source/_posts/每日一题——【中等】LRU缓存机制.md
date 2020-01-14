---
title: 每日一题——【中等】LRU缓存机制
date: 2020-01-14 12:26:39
tags: [LeetCode,算法]
---

## 题目
运用你所掌握的数据结构，设计和实现一个  LRU (最近最少使用) 缓存机制。它应该支持以下操作： 获取数据 get 和 写入数据 put 。

获取数据 get(key) - 如果密钥 (key) 存在于缓存中，则获取密钥的值（总是正数），否则返回 -1。
写入数据 put(key, value) - 如果密钥不存在，则写入其数据值。当缓存容量达到上限时，它应该在写入新数据之前删除最近最少使用的数据值，从而为新的数据值留出空间。

进阶:

* 你是否可以在 O(1) 时间复杂度内完成这两种操作？

示例:
```
LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回  1
cache.put(3, 3);    // 该操作会使得密钥 2 作废
cache.get(2);       // 返回 -1 (未找到)
cache.put(4, 4);    // 该操作会使得密钥 1 作废
cache.get(1);       // 返回 -1 (未找到)
cache.get(3);       // 返回  3
cache.get(4);       // 返回  4
```

## 解决方法
对于要在O(1)时间之内完成操作，那么一般都会引入哈希表。但是只有哈希表，是没办法确定要淘汰的数据是哪一个。这时候，我们一般都会想到用线性表或链表存储key，现在暂且称这个结构为`A`。

为了能在O(1)时间内取得数据的值、或者删除掉数据，我们可以通过哈希表的`key`映射到数据结构`A`的位置，对于线性表，我们可以映射到索引位置；对于链表，我们可以映射到链表的位置。但是，对于删除操作，线性表的元素删除涉及到移位，但对于链表而言，删除操作也可以做到常数时间，毕竟只是改变指向罢了。因此，通过上述分析，很容易想到用链表存储数据。

那么对于链表而言，究竟是用单链表还是双链表？从前面知道，我们可以能容易通过哈希表映射到链表的某个节点，但是对于删除操作，我们必须知道该节点的前一个节点，此时，单链表就无法满足了，因此只能使用双链表。

整个结构的图是这样的：

![image](http://static-cdn.lxzmww.xyz/LRU缓存机制.jpg)

```js
function LinkNode(key, val) {
    this.key = key;
    this.val = val;
    this.next = this.pre = null;
}

/**
 * @param {number} capacity
 */
var LRUCache = function(capacity) {
    this.map = new Map();
    this.head = this.tail = null;
    this.size = 0;
    this.capacity = capacity;
};

LRUCache.prototype.setHead = function(node) {
    if (this.head === null) {
        this.head = node;
        this.tail = this.head;
    }
    if (node === this.head) {
        return;
    }
    if (node === this.tail) {
        this.tail = node.next;
    }
    if (node.pre) {
        node.pre.next = node.next;
    }
    if (node.next) {
        node.next.pre = node.pre;
    }
    node.pre = this.head;
    this.head.next = node;
    node.next = null;
    this.head = node;
}

/** 
 * @param {number} key
 * @return {number}
 */
LRUCache.prototype.get = function(key) {
    if (this.map.has(key)) {
        const node = this.map.get(key);
        this.setHead(node);
        return node.val;
    } else {
        return -1;
    }
};

/** 
 * @param {number} key 
 * @param {number} value
 * @return {void}
 */
LRUCache.prototype.put = function(key, value) {
    if (this.map.has(key)) {
        const node = this.map.get(key);
        node.val = value;
        this.setHead(node);
    } else {
        const newNode = new LinkNode(key, value);
        if (this.size === this.capacity) {
            if (this.tail) {
                const key = this.tail.key;
                this.tail = this.tail.next;
                this.tail && (this.tail.pre = null);
                this.map.delete(key);
            }
        } else {
            ++this.size;
        }
        this.map.set(key, newNode);
        this.setHead(newNode);
    }
};

/** 
 * Your LRUCache object will be instantiated and called as such:
 * var obj = new LRUCache(capacity)
 * var param_1 = obj.get(key)
 * obj.put(key,value)
 */
```