+++
date = '2025-10-23T14:41:29+08:00'
draft = false
title = '[xv6 學習紀錄 08] Lab: locks'
series = ["xv6 學習紀錄"]
weight = 82
+++
Lab 連結：[Lab: locks](https://pdos.csail.mit.edu/6.S081/2022/labs/lock.html)

## Memory allocator (moderate)
> Your job is to implement per-CPU freelists, and stealing when a CPU's free list is empty. You must give all of your locks names that start with "kmem". That is, you should call `initlock` for each of your locks, and pass a name that starts with "kmem". Run kalloctest to see if your implementation has reduced lock contention. To check that it can still allocate all of memory, run `usertests` `sbrkmuch`. Your output will look similar to that shown below, with much-reduced contention in total on kmem locks, although the specific numbers will differ. Make sure all tests in `usertests -q` pass. make grade should say that the `kalloctests` pass. 

### 原本的 memory allocator 如何運作

## Buffer cache (hard)
> Modify the block cache so that the number of `acquire` loop iterations for all locks in the `bcache` is close to zero when running bcachetest. Ideally the sum of the counts for all locks involved in the block cache should be zero, but it's OK if the sum is less than 500. Modify bget and `brelse` so that concurrent lookups and releases for different blocks that are in the bcache are unlikely to conflict on locks (e.g., don't all have to wait for `bcache.lock`). You must maintain the invariant that at most one copy of each block is cached. When you are done, your output should be similar to that shown below (though not identical). Make sure `usertests -q` still passes. make grade should pass all tests when you are done. 

* `bcache` 是 filesystem 才有的概念？
