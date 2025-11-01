+++
date = '2025-11-01T09:04:58+08:00'
draft = false
title = '[LeetCode 解題] 746. Min Cost Climbing Stairs | 用前兩天的資訊推測出今天 | (C++)'
series = ["LeetCode 解題"]
+++
題目連結：[746. Min Cost Climbing Stairs](https://leetcode.com/problems/min-cost-climbing-stairs/description/)

## 解題思路
這題利用 DP 的方式，「有了前兩天的資訊可以推測出今天」依此類推的得知最後的答案。

## 程式碼
```c
class Solution {
public:
    int minCostClimbingStairs(vector<int>& cost) {
        int n = cost.size();
        vector<int> dp(n, 0);
        dp[0] = cost[0];
        dp[1] = cost[1];
        for (int i = 2; i < n; i++)
            dp[i] = min(dp[i - 1], dp[i - 2]) + cost[i];
        return min(dp[n - 1], dp[n - 2]);
    }
};
```
