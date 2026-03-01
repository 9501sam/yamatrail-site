+++
date = '2025-09-24T12:50:00+08:00'
draft = false
title = 'xv6 學習紀錄'
series = "xv6"
+++
為了了解作業系統的運作原理，挑選了 MIT 的作業系統開放式課程 [6.1810: Operating System Engineering](https://pdos.csail.mit.edu/6.S081/2022/schedule.html)  做學習，課程中會針對一個教學用的作業系統 xv6-riscv 進行追蹤程式碼，最後我把學習的過程寫成系列文章 xv6 學習紀錄。紀錄課程的各個 lab 所學習到的東西：

| Lab 名稱 | 學習重點與實作內容 |
| :--- | :--- |
| **Xv6 and Unix utilities** | 學習 xv6 的編譯與運行在 qemu 所模擬的 RISC-V 環境中 |
| **System calls** | 學習 system call 的概念原理、程式實作 |
| **Page tables** | 學習 RISC-V 的 3-level page table 架構、memory layout、追蹤及修改 virtual memory 程式碼 |
| **Traps** | 學習 Trap 的概念、使用 GDB 追蹤 trap 的過程 |
| **Xv6 lazy page allocation** | 使用 page fault 的技術，針對 `sbrk()` 的優化 |
| **Copy-on-write fork** | 使用 page fault 的技術，針對 `fork()` 的優化 |
| **Multithreading** | 學習 context switch 的流程與 scheduler 的原理 |
| **Locks** | 了解 lock 的運作原理及使用方式 |
| **File system** | 了解 file system 從 disk, bcache, log 一直到 file descriptor level，修改 inode 以支援更大的 file size |
| **Mmap** | 透過 page fault 與 process 中加入 virtual memory area 的資訊，把 virtual memory 映射到檔案內容 |
| **Network driver** | 學習網卡的運作原理、實做一個基於 E1000 網卡的 network driver |
