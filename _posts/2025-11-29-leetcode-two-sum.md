---
layout: post
title: "LeetCode – 1. Two Sum (Easy)"
date: 2025-11-29 20:00:00 +0530
categories: [DSA, LeetCode]
tags: [leetcode, array, hash-map, easy]
description: "My notes and C++ solution for LeetCode Two Sum, using a hash map for O(n) time."
image:
  path: /assets/img/hello-world-banner.jpg
  alt: "LeetCode Two Sum"
  lqip: /assets/img/hello-world-banner.jpg
  no_lazy: false
---

> Problem link: [LeetCode – Two Sum](https://leetcode.com/problems/two-sum/)

### 🧩 Problem summary

Given an array of integers `nums` and an integer `target`, return *indices* of the two numbers such that they add up to `target`.

- Exactly one solution exists.
- You may not use the same element twice.
- You can return the answer in any order.

---

### 💡 Approach (Hash map, O(n))

Idea:

- We scan the array once.
- For each number `nums[i]`, we check if `target - nums[i]` was seen before.
- Use `unordered_map<value, index>` to store previous numbers.

Why it works:

- If `nums[i] + nums[j] = target`, and we are at `i`, we only need to know if `target - nums[i]` appeared before at some `j`.
- Hash map lookup is **O(1)** on average.

---

### ⏱️ Complexity

- **Time:** `O(n)` – one pass
- **Space:** `O(n)` – in worst case we store all elements in the map

---

### 🧾 C++ Code (with comments)

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        // value -> index
        unordered_map<int, int> mp;

        for (int i = 0; i < (int)nums.size(); i++) {
            int x = nums[i];
            int need = target - x;

            // if 'need' already seen, we found the pair
            if (mp.count(need)) {
                return { mp[need], i };
            }

            // otherwise store current value and its index
            mp[x] = i;
        }

        // problem guarantees at least one solution, so we should never reach here
        return {};
    }
};
```
