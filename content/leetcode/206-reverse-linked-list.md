+++
date = '2025-09-23T20:31:28+08:00'
draft = false
title = '[LeetCode 解題] 206. Reverse Linked List'
series = ["LeetCode 解題"]
weight = 1
+++

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
