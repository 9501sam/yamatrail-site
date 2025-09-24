+++
date = '2025-09-23T20:31:28+08:00'
draft = false
title = '[LeetCode 解題] 206. Reverse Linked List | 注意 Edge Case (C++)'
series = ["LeetCode 解題"]
weight = 1
+++
題目連結：[206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)

## 解題思路
這題還蠻直接的，把 Linked List 給掃過一遍，並且一個一個的把指標顛倒過來。

剛開始的時候我用了這個寫法：
```c++
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        if (!head) 
            return head;
        ListNode *slow = head;
        ListNode *fast = head->next;
        while (fast != nullptr) {
            ListNode *nextNode = fast->next;
            fast->next = slow;
            slow = fast;
            fast = nextNode;
        }
        slow->next = nullptr;
        return slow;
    }
};
```
但這個寫法是錯誤的，因為 `slow` 從 `head` 開始的話，最一開始的 `head` 沒有指向 `nullptr`，所以需要把 `slow` 從 `nullptr` 開始會比較好。

我學到的是面對到 Edge Case 時，要特別小心注意才行。

## 程式碼
```c++
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        ListNode *slow = nullptr;
        ListNode *fast = head;
        while (fast != nullptr) {
            ListNode *nextNode = fast->next;
            fast->next = slow;
            slow = fast;
            fast = nextNode;
        }
        return slow;
    }
};
```
