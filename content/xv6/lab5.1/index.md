+++
date = '2025-10-11T11:20:50+08:00'
draft = false
title = '[xv6 學習紀錄 05-1] Page Fault'
series = ["xv6 學習紀錄"]
weight = 51
+++

課程影片連結：
* [6.S081 Fall 2020 Lecture 8: Page Faults](https://www.youtube.com/watch?v=KSYO-gTZo0A)
{{< youtube KSYO-gTZo0A >}}

## Page Fault
回顧 virtual memory 的特性
* isolation
* level of indirection
在原先 xv6 的設計中 pagetable 的設計是 static
現在在 page fault 的時候做一些動作，可以設計為 dynamic 的 mapping
### Information neede
1. 造成 page fault 的 virtual memory
    * `stval` register
1. The type of page fault
    * `scause`: R/W/Instruction
1. Virtual address of instruction that cause page fault
    * 這是為了在處理完 page fault 之後，可以回到原本的地方繼續執行
    * `trampframe->epc`
### Allocation: `sbrk()`
* 原先的方式：eager allocatoin
* 可變更為：Lazy Allocation
## 1. Lazy Allocation
先把 `sys_sbrk()` 變更為：
```c
p->sz = p->sz + n;
```
1. allocate one page
1. zero the page
1. map the page
1. restart instruction

??: what if no more memory

### `sys_sbrk()`
```c
p->sz = p->sz + n;
```

### `usertrap()`
```c
if (scause == 15)
```

### `uvmunmap()`
continue;

### Zero fill on demand
多個 page map 到同一個 zero page 並且用 read only

page fault: someone whant to write
1. make a new page with write
1. restart instruction

less pa useage

## 2. Copy-on-Write fork
跟前面的 zero fill on demand 有一點像
這裡需要用一個 counter 計算每一個 physical page 正在被多少個 process map 到

## 3. Demand Paging
### If out of memory
1. evict a page to file system
1. use the page just free
1. restart instruction

### 如何選擇要被 evict 的 page
* least recently used
* not dirty page??

## 4. Memory-Mapped Files
`mmap()` map va to a file discriptor
### multiple process???
### file walking

## Summary
page tables + traps/pagefault = 
* powerful
* elegant
virtual feature

