+++
date = '2025-10-16T11:30:41+08:00'
draft = false
title = '[LeetCode 解題] 1679. Max Number of K-Sum Pairs | 考慮目前的最大與最小 (C++)'
series = ["LeetCode 解題"]
+++
題目連結：[1679. Max Number of K-Sum Pairs](https://leetcode.com/problems/max-number-of-k-sum-pairs/description/)

## 解題思路
這一題跟 11. Container With Most Water 的感覺還蠻像的，都是先把兩的 pointer 放在兩端，並且根據不同的情況，用 greedy 的方式判斷下一步要做什麼。還有這個問題是配對到了就刪除掉，所以也不會有重複的問題。

## 程式碼
```c
class Solution {
public:
    int maxOperations(vector<int>& nums, int k) {
        int left = 0;
        int right = nums.size() - 1;
        int count = 0;

        sort(nums.begin(), nums.end());

        while (left < right) {
            if (nums[left] + nums[right] == k) {
                count++;
                left++;
                right--;
            } else if (nums[left] + nums[right] < k) {
                left++;
            } else {
                right--;
            }
        }
        return count;
    }
};
```
