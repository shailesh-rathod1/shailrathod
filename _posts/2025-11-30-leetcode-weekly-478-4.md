---
layout: post
title: "LeetCode Weekly Contest 478 – Q4. Minimum Operations to Equalize Subarrays (Hard)"
date: 2025-11-30 16:00:00 +0530
categories: [DSA, LeetCode]
tags: [leetcode, contest, hard, array, prefix-sum, segment-tree, math]
description: "My notes and C++ solution for LeetCode Weekly Contest 478 – Q4, using modulus check + median on a persistent segment tree."
image:
  path: /assets/img/hello-world-banner.jpg
  alt: "LeetCode Weekly Contest 478 – Q4"
  lqip: /assets/img/hello-world-banner.jpg
  no_lazy: false
---

> Problem link:  
> [LeetCode Weekly Contest 478 – Q4. Minimum Operations to Equalize Subarrays](https://leetcode.com/contest/weekly-contest-478)

---

### 🧩 Problem Summary

- Tumhe ek array `nums` aur ek integer `k` diya gaya hai.  
- Ek operation me tum kisi bhi `nums[i]` ko **exactly `+k` ya `-k`** se change kar sakte ho.  
- Har query `[l, r]` ke liye, hume minimum operations chahiye jisse  
  subarray `nums[l..r]` ke **saare elements equal** ho jaayein.  
- Agar possible hi nahi hai, to answer `-1`.  

We need: `ans[i]` for each query.

---

### 💡 Observations

#### 🔸 Feasibility Condition

- Agar `nums[i]` aur `nums[j]` ko equal banana hai using ±k moves,  
to ye tabhi possible hai jab:

    **`nums[i] % k == nums[j] % k`**

- To check subarray `[l, r]` feasible hai ya nahi:
  - Sabhi elements ka remainder (mod k) same hona chahiye.

- Agar same nahi — answer `-1`.

---

#### 🔸 Transformation for Cost

- Har number likha ja sakta hai:

   **`nums[i] = k * q[i] + rem`**

- Operation **`±k → q[i] += 1 ya q[i] -= 1.`**

- To cost = sum of absolute differences from some target `T`:

    **`cost = Σ |q[i] - T|`**

- Minimum cost tab milta hai jab **`T = median(q[l..r]).`**

---

### ⚙️ Approach

1. Pehle remainder equality check karo O(1) per query ke liye using prefix array.  
2. Then `q[i] = nums[i] / k` array banao.  
3. Build a persistent segment tree for prefix order statistics:
   - Each version stores prefix count + sum.
   - Allows:
     - median in O(log n)
     - prefix sum ≤ X in O(log n)
4. For each query:
   - If feasible → find median + compute cost:
     - cost = med * leftCnt - leftSum + rightSum - med * rightCnt

---

### ⏱️ Complexity

| Step | Time | Space |
|------|------|--------|
| Build structures | O(n log n) | O(n log n) |
| Per query | O(log n) | — |

---

### 🧾 C++ Code (Optimized Solution)
```cpp
class Solution {
public:
    struct Node {
        int left, right;
        int cnt;
        long long sum;
        Node(int l = 0, int r = 0, int c = 0, long long s = 0)
            : left(l), right(r), cnt(c), sum(s) {}
    };

    vector<Node> seg;
    vector<int> root;
    vector<long long> vals;

    int build(int l, int r) {
        int id = seg.size();
        seg.emplace_back();
        if (l == r) return id;
        int mid = (l + r) >> 1;
        seg[id].left = build(l, mid);
        seg[id].right = build(mid + 1, r);
        return id;
    }

    int update(int prev, int l, int r, int pos, long long val) {
        int id = seg.size();
        seg.push_back(seg[prev]);
        seg[id].cnt += 1;
        seg[id].sum += val;
        if (l == r) return id;
        int mid = (l + r) >> 1;
        if (pos <= mid)
            seg[id].left = update(seg[prev].left, l, mid, pos, val);
        else
            seg[id].right = update(seg[prev].right, mid + 1, r, pos, val);
        return id;
    }

    long long kth(int rRoot, int lRoot, int l, int r, int k) {
        if (l == r) return vals[l - 1];
        int leftR = seg[rRoot].left;
        int leftL = seg[lRoot].left;
        int cntLeft = seg[leftR].cnt - seg[leftL].cnt;
        int mid = (l + r) >> 1;
        if (k <= cntLeft)
            return kth(leftR, leftL, l, mid, k);
        else {
            int rightR = seg[rRoot].right;
            int rightL = seg[lRoot].right;
            return kth(rightR, rightL, mid + 1, r, k - cntLeft);
        }
    }

    pair<long long,long long> queryPrefix(int rRoot, int lRoot, int l, int r, int pos) {
        if (pos < l) return {0, 0};
        if (r <= pos) {
            long long cnt = seg[rRoot].cnt - seg[lRoot].cnt;
            long long sum = seg[rRoot].sum - seg[lRoot].sum;
            return {cnt, sum};
        }
        int mid = (l + r) >> 1;
        auto leftRes = queryPrefix(seg[rRoot].left, seg[lRoot].left, l, mid, pos);
        auto rightRes = queryPrefix(seg[rRoot].right, seg[lRoot].right, mid + 1, r, pos);
        return {leftRes.first + rightRes.first, leftRes.second + rightRes.second};
    }

    vector<long long> minOperationsQueries(vector<int>& nums, int k, vector<vector<int>>& queries) {
        int n = nums.size();

        vector<int> rem(n);
        for (int i = 0; i < n; ++i) rem[i] = nums[i] % k;

        vector<int> eq(n, 0);
        for (int i = 1; i < n; ++i)
            if (rem[i] == rem[i - 1]) eq[i] = 1;

        vector<int> prefEq(n + 1, 0);
        for (int i = 0; i < n; ++i)
            prefEq[i + 1] = prefEq[i] + eq[i];

        vector<long long> q(n);
        for (int i = 0; i < n; ++i)
            q[i] = nums[i] / (long long)k;

        vals = q;
        sort(vals.begin(), vals.end());
        vals.erase(unique(vals.begin(), vals.end()), vals.end());
        int m = vals.size();

        auto getRank = [&](long long x) {
            return int(lower_bound(vals.begin(), vals.end(), x) - vals.begin()) + 1;
        };

        seg.reserve(n * 20 + 5);
        seg.clear();
        root.assign(n + 1, 0);

        int baseRoot = build(1, m);
        root[0] = baseRoot;

        for (int i = 0; i < n; ++i) {
            int pos = getRank(q[i]);
            root[i + 1] = update(root[i], 1, m, pos, q[i]);
        }

        vector<long long> ans;
        ans.reserve(queries.size());

        for (auto &qr : queries) {
            int l = qr[0], r = qr[1];
            if (l == r) { ans.push_back(0); continue; }

            int equalPairs = prefEq[r] - prefEq[l];
            if (equalPairs != r - l) { ans.push_back(-1); continue; }

            int len = r - l + 1;
            int kMedian = (len + 1) / 2;

            int rRoot = root[r + 1], lRoot = root[l];
            long long med = kth(rRoot, lRoot, 1, m, kMedian);

            auto totalPair = queryPrefix(rRoot, lRoot, 1, m, m);
            long long totalCnt = totalPair.first, totalSum = totalPair.second;

            int medRank = getRank(med);
            auto leftPair = queryPrefix(rRoot, lRoot, 1, m, medRank);
            long long leftCnt = leftPair.first, leftSum = leftPair.second;

            long long rightCnt = totalCnt - leftCnt;
            long long rightSum = totalSum - leftSum;

            long long cost = med * leftCnt - leftSum + rightSum - med * rightCnt;
            ans.push_back(cost);
        }
        return ans;
    }
};
```
