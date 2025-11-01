+++
date = '2025-11-01T11:48:29+08:00'
draft = false
title = '[xv6 學習紀錄 09] Lab: file system'
series = ["xv6 學習紀錄"]
weight = 94
+++

Lab 連結：[Lab: file system](https://pdos.csail.mit.edu/6.S081/2020/labs/fs.html)

## Large files (moderate)
>  Modify `bmap()` so that it implements a doubly-indirect block, in addition to direct blocks and a singly-indirect block. You'll have to have only 11 direct blocks, rather than 12, to make room for your new doubly-indirect block; you're not allowed to change the size of an on-disk inode. The first 11 elements of `ip->addrs[]` should be direct blocks; the 12th should be a singly-indirect block (just like the current one); the 13th should be your new doubly-indirect block. You are done with this exercise when `bigfile` writes 65803 blocks and usertests runs successfully: 

這題的策略會是修改 `struct dinode` 把 indirect 的部份變成兩個

## Symbolic links (moderate)
> You will implement the symlink(`char *target, char *path`) system call, which creates a new symbolic link at path that refers to file named by target. For further information, see the man page symlink. To test, add symlinktest to the Makefile and run it. Your solution is complete when the tests produce the following output (including usertests succeeding). 

這題的重點會在於新增一個 type

```c
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

目前的猜測會是修改這裡的 type 因為如果是 disk 的 `dinode`
```c
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```
disk 中的應該就只是純粹的存放資料，是沒有 type 一說的
