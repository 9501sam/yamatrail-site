+++
date = '2025-10-16T11:08:09+08:00'
draft = false
title = '[LeetCode 解題] 345 Reverse Vowels of a String | 用個 helper function 會方便許多 (C++)'
series = ["LeetCode 解題"]
+++

題目連結：[345. Reverse Vowels of a String](https://leetcode.com/problems/reverse-vowels-of-a-string/description/)

## 解題思路
這題就很直覺得掃過兩遍，第一遍收集 Vowels，第二變把 Vowels 用相反的順序放回去，真有什麼好說的東西的話應該是寫個 helper function `isVowels()` 會方便一些。

## 程式碼
```cpp
class Solution {
private:
    bool isVowels(char c) {
        if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u')
            return true;
        if (c == 'A' || c == 'E' || c == 'I' || c == 'O' || c == 'U')
            return true;
        return false;
    }
public:
    string reverseVowels(string s) {
        int n = s.size();
        queue<char> q;
        for (int i = 0; i < n; i++)
            if (isVowels(s[i]))
                q.push(s[i]);
        for (int i = n - 1; i >= 0; i--)
            if (isVowels(s[i])) {
                s[i] = q.front();
                q.pop();
            }
        return s;
    }
};
```
