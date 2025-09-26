+++
date = '2025-09-27T00:37:29+08:00'
draft = false
title = "[LeetCode 解題] 17. Letter Combinations of a Phone Number | 想清楚 depth 的意義"
series = ["LeetCode 解題"]
weight = 1
+++
題目連結：[17. Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number)

我覺得這種 backtracking 或是 dfs 的題目都需要想清處 depth 的定義是什麼，想清楚就會好寫許多。
## 程式實做
```c++
class Solution {
public:
    vector<string> letterCombinations(string digits) {
        if (digits.size() == 0)
            return {};
        vector<string> phoneNumber = {
            "", "", "abc", "def", "ghi", "jkl", "mno",
            "pqrs", "tuv", "wxyz"
        };
        vector<string> combs;
        string current;
        backtracking(0, digits, current, phoneNumber, combs);

        return combs;
    }

    void backtracking(int index, string digits, string current, 
        vector<string> phoneNumber, vector<string> &combs) {
            if (index >= digits.size()) {
                combs.push_back(current);
                return;
            }

            int i = digits[index] - '0';
            for (char c : phoneNumber[i]) {
                current.push_back(c);
                backtracking(index + 1, digits, current, phoneNumber, combs);
                current.pop_back();
            }
    }
};
```
