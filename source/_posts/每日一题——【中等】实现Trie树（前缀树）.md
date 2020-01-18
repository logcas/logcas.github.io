---
title: 每日一题——【中等】实现Trie树（前缀树）
date: 2020-01-18 23:21:38
tags: [LeetCode,算法]
---
## 题目
实现一个 Trie (前缀树)，包含 insert, search, 和 startsWith 这三个操作。

示例:
```
Trie trie = new Trie();

trie.insert("apple");
trie.search("apple");   // 返回 true
trie.search("app");     // 返回 false
trie.startsWith("app"); // 返回 true
trie.insert("app");   
trie.search("app");     // 返回 true
```
说明:

* 你可以假设所有的输入都是由小写字母 a-z 构成的。
* 保证所有输入均为非空字符串。

链接：https://leetcode-cn.com/problems/implement-trie-prefix-tree

## 解决方法
理解了什么是前缀树的话，其实很好解决的。

这题设立了一些限制，也使得更简单：
1. 输入只有`a-z`，那么设立一个数组，根据ASCII码（以a为基准开始）随机访问即可。
2. 如果不限制输入字符，那么可能需要把数组改为链表、哈希表等结构才能满足（例如博大精深的中文）。

```js
function ListNode(ch) {
    this.ch = ch;
    this.isEnd = false;
    this.branchs = new Array(26).fill(null);
}

function getIndex(ch) {
    return ch.charCodeAt() - 'a'.charCodeAt();
}

/**
 * Initialize your data structure here.
 */
var Trie = function() {
    this.head = new ListNode();
};

/**
 * Inserts a word into the trie. 
 * @param {string} word
 * @return {void}
 */
Trie.prototype.insert = function(word) {
    let cursor = this.head;
    for(let i = 0;i < word.length; ++i) {
        const ch = word[i];
        const index = getIndex(ch);
        if (!cursor.branchs[index]) {
            cursor.branchs[index] = new ListNode(ch);
        }
        if (i === word.length - 1) {
            cursor.branchs[index].isEnd = true;
        }
        cursor = cursor.branchs[index];
    }
};

/**
 * Returns if the word is in the trie. 
 * @param {string} word
 * @return {boolean}
 */
Trie.prototype.search = function(word) {
    let cursor = this.head;
    for(let i = 0;i < word.length; ++i) {
        const ch = word[i];
        const index = getIndex(ch);
        if (cursor.branchs[index]) {
            cursor = cursor.branchs[index];
        } else {
            return false;
        }
    }
    if (cursor.isEnd) {
        return true;
    }
    return false;
};

/**
 * Returns if there is any word in the trie that starts with the given prefix. 
 * @param {string} prefix
 * @return {boolean}
 */
Trie.prototype.startsWith = function(prefix) {
    let cursor = this.head;
    for(let i = 0;i < prefix.length; ++i) {
        const ch = prefix[i];
        const index = getIndex(ch);
        if (cursor.branchs[index]) {
            cursor = cursor.branchs[index];
        } else {
            return false;
        }
    }
    return true;
};

/**
 * Your Trie object will be instantiated and called as such:
 * var obj = new Trie()
 * obj.insert(word)
 * var param_2 = obj.search(word)
 * var param_3 = obj.startsWith(prefix)
 */
```
