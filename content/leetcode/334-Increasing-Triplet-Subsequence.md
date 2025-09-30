+++
date = '2025-09-30T11:13:47+08:00'
draft = false
title = '[LeetCode 解題] 334. Increasing Triplet Subsequence | 挑選較好的順序 (C++)'
series = ["LeetCode 解題"]
+++
題目連結：[334. Increasing Triplet Subsequence](https://leetcode.com/problems/increasing-triplet-subsequence/description/)

## 解題思路
這題我剛開始是朝著先找出 `i` 與 `k` 之後再檢查 `j` 是否存在，想了幾遍都覺得有些瑕疵，後來發現依照 `i`, `j`, `k` 的順序尋找會比較合理一些，可以用很 greedy 的方式去尋找。  

如果再來一次的話，我應該會先列出以下兩種可能：
1. 先找出最大最小的 `i` 與 `k`, 再找出中間的 `j`
1. 依照順序尋找 `i`, `j`, `k`  

評估之後挑選較有可能的順序，並且保持另一個方案的可能性。

## 程式碼
```c++
class Solution {
public:
    bool increasingTriplet(vector<int>& nums) {
        if (nums.size() < 3)
            return false;
        int i = 0;
        int j = -1;
        for (int l = 1; l < nums.size(); l++) {
            if (nums[l] < nums[i]) {
                i = l;
            } else if (j > 0 && nums[l] > nums[j]) {
                return true;
            } else if (nums[l] > nums[i]) {
                j = l;
            }
        }
        return false;
    }
};
```
