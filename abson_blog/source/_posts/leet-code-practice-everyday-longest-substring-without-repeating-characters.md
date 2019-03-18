---
title:  Longest Substring Without Repeating Characters —— leet code 每天一练
date: 2019-03-15 01:35:02
tags:
---

### 问题描述

Given a string, find the length of the longest substring without repeating characters.

Example 1:
```
Input: "abcabcbb"
Output: 3 
Explanation: The answer is "abc", with the length of 3. 
```

Example 2:
```
Input: "bbbbb"
Output: 1
Explanation: The answer is "b", with the length of 1.
```

Example 3:
```
Input: "pwwkew"
Output: 3
Explanation: The answer is "wke", with the length of 3. 
             Note that the answer must be a substring, "pwke" is a subsequence and not a substring.
```

问题很简单，就是找出目标字符串中，不含有重复字符的子字符串的长度！！！


### 第一种解法
滑动窗口
```javascript
function lengthOfLongestSubstring(s: string) {
  // const hashMap: { [key: string]: number } = {};
  let max = 0;
  let first = "";
  s.split('').forEach((value: string) => {
    const indxe = first.indexOf(value);
    if (indxe === -1) {
      first += value;
    } else {
      max = first.length > max ? first.length : max;
      first = first.substr(indxe + 1) + value;
    }
  });
  max = first.length > max ? first.length : max;
  return max;
}
```

**运行情况:**

```
Runtime: 88 ms, faster than 95.16% of JavaScript online submissions forLongest Substring Without Repeating Characters.
Memory Usage: 40 MB, less than 66.18% of JavaScript online submissions forLongest Substring Without Repeating Characters.
```

上面的算法利用了类似于滑动窗口的动作，获取到相同字符的位置，然后得到该子字符串。

**复杂度分析：**

* 时间复杂度: O(n + m) or O(n) = O(2n) in the worst case that each other character will be visited twice, ex: str = abcdefg.

* 空间复杂度：O(min(n, m)), 要么就是目标字符 s 的长度，要么就是子字符串 fist 的长度。


### 第二种解法
利用 hashmap 来存储字符的位置，这样在查找相同字符的 time complexity 就是 O(1) 了，以此来加快运行速度

```javascript
// typescript
function lengthOfLongestSubstring(str: string) {
    // time complexity: O(n).
    // space complexity: O(min(n, m))
    // 
    const map: Map<string, number> = new Map();
    let i = 0;
    let ans = 0;
    str.split('').forEach((key: string, index: number) => {
      if (map.has(key)) {
        i = Math.max(map.get(key)!, i);
      }
      ans = Math.max(ans, index - i + 1);
      map.set(key, index + 1);
    });
    return ans;
  }
}
```

**运行情况:**
```
Runtime: 88 ms, faster than 95.33% of JavaScript online submissions for Longest Substring Without Repeating Characters.
Memory Usage: 37.1 MB, less than 91.13% of JavaScript online submissions for Longest Substring Without Repeating Characters.
```
**复杂度分析：**

* 时间复杂度: O(n) + O(1), 遍历整个字符串的时间为 O(n), 查找时间为 O(1).

* 空间复杂度：O(min(n, m)), 要么就是目标字符 s 的长度，要么就是 map 的长度。


**变异解法**，对于字符来说，终究都是ASCI，那么我们使用 int 数组来存储每个字符出现的位置，会比使用 map 更节省内存。


### 总结：
* 使用 for 循环去判断寻找重复字符位置的速度完全比不上使用 indexof，这个 inddeof 估计其内部也是一个 hashmap.

* 在进行编写的的时候，发现一个很有趣的现象，我使用 hashmap 的时候，一开始选用的是 javascript 的对象，object = {} 来进行 keyvalue 存储，发现其内存占用跟我用第一种解法差不多，然后选用 Map 结构体，发现其内存节省了不小.

