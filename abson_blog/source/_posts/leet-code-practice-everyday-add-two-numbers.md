---
title: Add Two Numbers —— leet code 每天一练
date: 2019-03-14 01:16:03
tags: 
			- leetcode
---

### 问题描述

You are given two non-empty linked lists representing two non-negative integers. The digits are stored in reverse order and each of their nodes contain a single digit. Add the two numbers and return it as a linked list.
You may assume the two numbers do not contain any leading zero, except the number 0 itself.

```
Example:
Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
Output: 7 -> 0 -> 8
Explanation: 342 + 465 = 807.
```

给出两个非空的非负的整数链表。他们所代表了数字不同位数的反序存储，并且每个节点只能存储一位数字。
现在你要将这两个数字想加，返回一个新的链表表示它们的和。
你可以假设除了数字0之外，这两个数都不会以0开头。


### 解法

```java
// typescript 
interface IListNode<T> {
  val: T;
  next: IListNode<T> | null;
}

function addTwoNumbers(l1: IListNode<number> | null, l2: IListNode<number> | null) => {
  let result: IListNode<number> = { val: 0, next: null };
  let dvi = 0;
  let last: IListNode<number> = result;
  while (!!l1 || !!l2) {

    let sum = 0;
    if (l1) {
      sum += l1.val;
    }

    if (l2) {
      sum += l2.val;
    }

    sum += dvi;
    dvi = sum / 10 >>> 0;

    const node = { val: sum % 10, next: null };

    last.next = node;
    last = last.next;

    l1 = l1 && l1.next;
    l2 = l2 && l2.next;

  }
  if (dvi > 0) {
    last && (last.next = { val: dvi, next: null });
  }

  return result.next;
};

```

**运行情况:**

```
Runtime: 124 ms, faster than 81.49% of JavaScript online submissions for Add Two Numbers.
Memory Usage: 39 MB, less than 18.38% of JavaScript online submissions for Add Two Numbers.

```

**复杂度分析：**

* 时间内负责度: O(max(m, n)). 假设 m 和 n 分别表示 l1 和 l2 的长度，上面的算法最多从重复了 max(m, n) 次。

* 空间复杂度: O(max(m, n)). 新列表的长度最大为 max(m, n) + 1.


总结：
这道题比较简单，没什么好说的....
不过反向思考，这道题是否可以解决大数想加问题，比如 2^63 * 2^63 之类的运算 ？ 举一反三，做完题多思考


