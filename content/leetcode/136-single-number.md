+++
date = '2025-11-01T09:21:12+08:00'
draft = false
title = '[LeetCode 解題] 136. Single Number | XOR 與重複出現 (C++)'
series = ["LeetCode 解題"]
+++
題目連結：[136. Single Number](https://leetcode.com/problems/single-number/description/)

## 解題思路
這題是一個蠻神奇蠻剛好的一題，同時使用了 xor 與「出現兩次」這兩個元素，因為
* 同一個數字 xor 在一起會變成 0
這題就用這樣的特性解決

## 程式碼
```cpp
class Solution {
public:
    int singleNumber(vector<int>& nums) {
        int n = 0;
        for (int num : nums)
            n ^= num;
        return n;
    }
};
```

