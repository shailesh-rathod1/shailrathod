---
layout: post
title: "LeetCode Weekly Contest 478 – Q1. Count Elements With at Least K Greater Values (Medium)"
date: 2025-11-30 10:00:00 +0530
categories: [DSA, LeetCode]
tags: [leetcode, contest ,array, sorting, array, medium]
description: "My notes and C++ solution for LeetCode Weekly Contest 478 – Q1,  O(n log n) time."
image:
  path: /assets/img/hello-world-banner.jpg
  alt: "LeetCode Weekly Contest 478 – Q1"
  lqip: /assets/img/hello-world-banner.jpg
  no_lazy: false
---

> Problem link: [LeetCode Weekly Contest 478 – Q1](https://leetcode.com/contest/weekly-contest-478/problems/count-elements-with-at-least-k-greater-values/)

### 🧩 Problem summary

- Tumhe ek integer array `nums` aur ek integer `k` diya gaya hai.  
- Ek element **qualified** tab bola jaata hai agar usse **strictly greater** kam se kam `k` elements exist karte ho array mein.  
- Return karo total qualified elements ki count.

---

### 💡 Approach (Hash map, O(n))

- Pehle array ko sort karo — taaki sab elements increasing order mein aa jaayein.

- Jab hum kisi element pe hain, to uske right side ke elements hamesha strictly greater honge.

- Agar uske right side ke elements ≥ k hain, to wo element (aur uske duplicates) qualified hain.

- Bas frequency ke hisaab se count kar lo.

---

### ⏱️ Complexity

- **Time:** `O(n log n)` (due to sorting)
- **Space:** `O(1)` (no extra data structure)

---

### 🧾 C++ Code (with comments)

```cpp
class Solution {
public:
    int countElements(vector<int>& nums, int k) {
        sort(nums.begin(), nums.end());
        int n = nums.size();
        int ans = 0;

        int i = 0;
        while (i < n) {
            int j = i;
            while (j < n && nums[j] == nums[i]) j++;

            int greater = n - j; // strictly greater elements count
            if (greater >= k)
                ans += (j - i); // add all duplicates of this value

            i = j;
        }

        return ans;
    }
};
```
