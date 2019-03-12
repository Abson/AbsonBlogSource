---
title: Two Sum —— leet code 每天一练
date: 2019-03-12 15:05:01
tags: 
			- leetcode
---

### 问题描述：

Given an array of integers, return indices of the two numbers such that they add up to a specific target.
You may assume that each input would have exactly one solution, and you may not use the sameelement twice.

**Example:**
```
Given nums = [2, 7, 11, 15], target = 9,

Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1].
```

----

### 第一种解法，使用 Recursion：

```typescript
// typescript
var twoSum = function(nums: number[], target: number) {
  const result: number[] = [];
  for (let i = 0; i < nums.length; ++i) {
      const diff = target - nums[i];
      for (let j = nums.length; j > i; --j) {
        if (diff === nums[j]) {
          result.push(i);
          result.push(j);
        }
      }
  }
  return result;
};
```

**运行情况:**
```
Runtime: 328 ms, faster than 5.07% of JavaScript online submissions for Two Sum.
Memory Usage: 34.8 MB, less than 44.49% of JavaScript online submissions for Two Sum.
```

复杂度分析：

* 时间复杂度为：O(n^2). 对于每个元素而言，我们通过循环遍历剩余得的数组来查找它的补数，这花费了 O(n) 的时间复杂度，因此，这个时间复杂度为 O(n^2).

* 空间复杂度：O(1).

----

### 第二种解法，使用 HashTable

```typescript
// typescript
var twoSum = function(nums: number[], target: number) {
  const map: {[key: number]: number} = {};
  for (let i = 0; i < nums.length; ++i) {
    if (map[target - nums[i]] !== undefined) {
      return [map[target - nums[i]], i];
    }
    map[nums[i]] = i;
  }
};

```

**运行情况:**
```
Runtime: 60 ms, faster than 91.05% of JavaScript online submissions for Two Sum.
Memory Usage: 34.4 MB, less than 83.16% of JavaScript online submissions for Two Sum.
```


复杂度分析：

* 时间复杂度为：O(n). 我们只遍历了一次数组包含的 n 个元素. 每次查找哈希表所花费得时间为 O(1).

* 空间复杂度：O(n). 额外空间得大小需要依赖于元素的存储数量，最大存储量为 n 个元素，最小为 1.

----

额外提供 C++ 解法：


```cpp
C++
class Solution
{
public:
  static std::vector<int> twoSum(std::vector<int> &nums, int target)
  {
    auto table = new std::map<int, int>();
    for (int i = 0; i < nums.size(); ++i)
    {
      auto iterator = (*table).find(target - nums[i]);
      if (iterator != table->end())
      {
        return {(*table)[target - nums[i]], i};
      }

      (*table)[nums[i]] = i;
    }
    return {};
  }
};
```
**运行情况:**
```
Runtime: 16 ms, faster than 58.40% of C++ online submissions for Two Sum.
Memory Usage: 10 MB, less than 56.09% of C++ online submissions for Two Sum.
```

计算型果然还是 C++ 比较快，看上去比 JS 快了 6~7 倍...


