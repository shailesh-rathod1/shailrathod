---
layout: post
title: "LeetCode Weekly Contest 478 – Q2. Maximum Substrings With Distinct Start (Medium)"
date: 2025-11-30 10:00:00 +0530
categories: [DSA, LeetCode]
tags: [leetcode, contest ,array, sorting, array, medium]
description: "My notes and C++ solution for LeetCode Weekly Contest 478 – Q2,  O(n) time."
image:
  path: /assets/img/hello-world-banner.jpg
  alt: "LeetCode Weekly Contest 478 – Q2"
  lqip: /assets/img/hello-world-banner.jpg
  no_lazy: false
---

> Problem link: [LeetCode Weekly Contest 478 – Q2](https://leetcode.com/contest/weekly-contest-478/problems/maximum-substrings-with-distinct-start/)

### 🧩 Problem summary

Ek string `s` di gayi hai (sirf lowercase English letters).  
Tumhe string ko **split** karna hai maximum substrings mein, aise ki:

- Har substring **non-empty** ho  
- Har substring **different character se start** ho  

Return the **maximum number of substrings** possible.

---

### 💡 Approach (Hash map, O(n))

- Har substring ek unique starting character se start honi chahiye.

- So maximum substrings tab possible hongi jab humare paas utne hi distinct characters ho.

- Isliye answer = number of unique characters in the string.

---

### ⏱️ Complexity

- **Time:** `O(n)` (single pass)
- **Space:** `O(26)` (no extra data structure)

---

### 🧾 C++ Code (with comments)

```cpp
class Solution {
public:
    int maxDistinct(string s) {
        unordered_set<char> st;
        for (char c : s)
            st.insert(c);

        return st.size(); // number of distinct characters
    }
};
```
