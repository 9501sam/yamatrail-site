+++
date = '2025-10-16T13:14:24+08:00'
draft = false
title = '[LeetCode 解題] 1004. Max Consecutive Ones III | two ponters 表達 length == 0 的方式 (C++)'
series = ["LeetCode 解題"]
+++
題目連結：[1004. Max Consecutive Ones III](https://leetcode.com/problems/max-consecutive-ones-iii/description/)

## 解題思路
這題使用兩個 pointer `left` 與 `right` 來紀錄當前的 array 大小，像是：
```text
0 1 2 3 4 5 6 7 8 9
    ↑       ↑
    l       r
```
這個時候 `left == 2`, `right == 6` 長度的計算為
```c
length = right - left + 1 
```
也就是 6 - 2 + 1 == 5

### 如何表達 `length == 0`
這樣的設計會碰到的下一個問題是
1. 假設 `right == 6`
1. 如何表達 `length == 0` 的情況??

會像是下面這個樣子：
```text
0 1 2 3 4 5 6 7 8 9
            ↑ ↑
            r l
```

雖然是 `left` 不過卻在 `right` 的右邊，但這樣做的好處就在於我們計算長度的公式：
```c
length = right - left + 1 
```
依然成立，(length = 6 - 7 + 1 = 0)，這樣這個公式就不必為了長度為 0 的狀況煩惱

### 最左邊的 Edge Case
這裡要思考的是初始狀態下，應該要把 `left` 指向哪裡?
* 以 `k == 0` 的情況之下

* `nums[0] == 0` 時，似乎應該要 `left == 1`
```text
index: 0 1 2 3 4 5 6 7 8 9
nums : 0 x x x x x x x x x
       ↑ ↑
       r l
```

* `nums[0] == 1` 時，似乎應該要 `left == 0`
```text
index: 0 1 2 3 4 5 6 7 8 9
nums : 0 x x x x x x x x x
       ↑ 
       r 
       l 
```
這裡在程式碼中的 `while()` 迴圈可以得到解決，不用太擔心

### 最右邊的 Edge Case
如果 `left` 超過了右邊的邊界 `n - 1` 時，應該要怎麼辦？
* 這裡仔細思考的話，就讓他超過，變成 `n` 各個情況依然成立，只是判斷的時候要稍微注意一下


## 程式碼
```cpp
class Solution {
public:
    int longestOnes(vector<int>& nums, int k) {
        int n = nums.size();
        int right;
        int left;
        int zeroCount = 0;
        int ret = 0;

        left = 0;
        for (right = 0; right < nums.size(); right++) {
            if (nums[right] == 0) {
                zeroCount++;
                if (zeroCount > k) {
                    while (left < n) {
                        if (nums[left] == 0)
                            zeroCount--;
                        left++;
                        if (zeroCount <= k)
                            break;
                    }
                }
            }
            ret = max(ret, right - left + 1);
        }
        return ret;
    }
};
```

## 結論
只要對於 `left` 不要壓上「不能超越 `right`」的物理意義，可以發現演算法上有比較好的一般性，只是在確認的時候要稍微過一下。
