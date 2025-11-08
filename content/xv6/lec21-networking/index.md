+++
date = '2025-11-06T12:45:24+08:00'
draft = false
title = '[xv6 學習紀錄 11-1] Networking'
series = ["xv6 學習紀錄"]
weight = 111
+++
課程連結：[6.S081 Fall 2020 Lecture 21: Networking](https://www.youtube.com/watch?v=Fcjychg4Tvk)
{{< youtube Fcjychg4Tvk >}}

## Protocal nesting
* header 的 type 決定了之後的內容要如何解讀

## The OS network stack

## `tcpdump` 的解析

## Queue 的使用
影片中有提及不管是在那一個層級的 stack 當中，queue 的使用都是很常見的

可能會有一個 buffer allocator `MBUF`

## E1000 NIC
這在 lab: network driver 中會運用的一個 NIC, 影片中有提及跟現代的 NIC 比較起來
* 現在的 NIC 當中有做一些 check

## Livelock Problem
最後的結論是用 Polling 來解決 
