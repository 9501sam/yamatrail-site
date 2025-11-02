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

這題的策略會是實做出一個 doubly-indirect block
* 不可以更改 on-disk inode `dinode` 的大小，所以這裡要犧牲一個 direct block 來作為 doubly-indirect block 的用途
* `ip->addrs[0] ~ ip->addr[10]` 做為原本的 direct block 的用途
* 第 12 個的 `ip->addr[11]` 做為原本的 singly-indirect block (原本的 `ip->addr[12]`) 的用途
* 第 13 個的 `ip->addr[12]` 是新增的，這是 doubly-indirect block

> * Make sure you understand `bmap()`. Write out a diagram of the relationships between `ip->addrs[]`, the indirect block, the doubly-indirect block and the singly-indirect blocks it points to, and data blocks. Make sure you understand why adding a doubly-indirect block increases the maximum file size by `256*256` blocks (really -1, since you have to decrease the number of direct blocks by one). 

這個 hint 要我們先理解原版的 `bmap()`

這在 [File System 相關程式碼解析](../filesystem/) 有分析過了
* `kernel/fs.c: bmap()`
```c
// Inode content
//
// The content (data) associated with each inode is stored
// in blocks on the disk. The first NDIRECT block numbers
// are listed in ip->addrs[].  The next NINDIRECT blocks are
// listed in block ip->addrs[NDIRECT].

// Return the disk block address of the nth block in inode ip.
// If there is no such block, bmap allocates one.
// returns 0 if out of disk space.
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  // direct 的情形
  if(bn < NDIRECT){
    // 直接拿第 bn 個 address
    // 例如 bn == 0 就拿 ip->addrs[0]
    // 一直到第 12 個的 bn == 11 就拿取 ip->addrs[11]
    // 現在 addr 拿到了 block address
    if((addr = ip->addrs[bn]) == 0){
      // 如果 addr 不存在 (addr == 0) balloc 一個新的 block 給 ip->addrs[bn]
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      // 原本沒有內容的 ip->addrs[bn] 現在有了一個新的 block
      ip->addrs[bn] = addr;
    }
    // 拿到了 addr 之後就回傳，這也是 bmap() 最重要的任務
    // 得知一個 inode 中的第 bn 個 block 的 block address
    return addr;
  }

  // 如果不是 direct mapping，例如想知道第 bn == 12 個 block
  // 實際上這是 indirect mapping 中的第 (12 - NDIRECT) == 0 個 block
  bn -= NDIRECT;

  // bn 照理來說不應該超過 256
  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    // 先取得圖片中 "indirect" 的那一個 address
    // 如果為 0 則一樣 balloc() 一個新的 block 給他
    if((addr = ip->addrs[NDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT] = addr;
    }
    // 往下尋找 bmap() 需要回傳的 address
    // 要從 addr = ip->addrs[NDIRECT] 的這個 block 中尋找
    // 使用 bread() 讀取這個 block 的資料
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    // 還是一樣的套路，沒有資料則 balloc 一個新的 block
    // 現在的 addr 就會是 bmap() 要回傳的 address 了
    if((addr = a[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }

  // 如果不是 direct or indirect, panic()
  panic("bmap: out of range");
}
```

> * Think about how you'll index the doubly-indirect block, and the indirect blocks it points to, with the logical block number.

> * If you change the definition of `NDIRECT`, you'll probably have to change the declaration of `addrs[]` in `struct inode` in `file.h`. Make sure that `struct inode` and `struct dinode` have the same number of elements in their `addrs[]` arrays.


> * If you change the definition of `NDIRECT`, make sure to create a new `fs.img`, since `mkfs` uses `NDIRECT` to build the file system.

更改完 `NDIRECT` 之後要重新 `make` 一次

> * If your file system gets into a bad state, perhaps by crashing, delete `fs.img` (do this from Unix, not xv6). make will build a new clean file system image for you.

如果系統 crash 掉，先 delete `fs.img` 再重新 make 出一個新的

> * Don't forget to `brelse()` each block that you `bread()`.

實做 `bmap()` 要注意的點

> * You should allocate indirect blocks and doubly-indirect blocks only as needed, like the original `bmap()`.

有需要的時候再使用 `balloc()` 出新的 block，這個套路在原版的 `bmap()` 就已經看過了

> * Make sure `itrunc` frees all blocks of a file, including double-indirect blocks.

這在 [File System 相關程式碼解析](../filesystem/) 有分析過原版的，等一下再加入 doubly-indirect 的部份
```c
// Truncate inode (discard contents).
// Caller must hold ip->lock.
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  // 刪除 12 個 direct
  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  // 刪除 indirect 的部份
  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    // 刪除 indirect block 中 0 ~ 511 的 data block
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

> * `usertests` takes longer to run than in previous labs because for this lab `FSSIZE` is larger and big files are larger. 

先前的 `usertests` 就已經蠻慢的了，這個 big file 的 lab 又會更慢一點，畢竟 `FSSIZE` 又更大

### 程式實做

#### data structure
* `kernel/fs.h`
```c
#define BSIZE 1024  // block size

#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint)) // 512
#define MAXFILE (NDIRECT + NINDIRECT)
```
現在 `NDIRECT` 改為 11，把多出來的做為 doubly-indirect 用
* `kernel/fs.h`
```c
#define BSIZE 1024  // block size

#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint)) // 512
#define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT * NINDIRECT)
```

連帶 `struct inode` 與 `struct dinode` 要做更動
* `kernel/file.h`
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
  uint addrs[NDIRECT+2];
};
```

* `kernel/fs.h`
```c
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // Data block addresses
};
```

#### `bmap()`
```c
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[bn] = addr;
    }
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT] = addr;
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }
  // 現在 bn 代表的是 doubly-indirect 中的第 bn 個
  bn -= NINDIRECT;

  // 在這時候先釐清一下
  // 在 ip->addrs[] 的層級中，要拿的是 ip->addrs[NDIRECT + 1]
  // 在這一個 doubly-indirect block 中總共有 512 個 address
  // 這 512 個 address 指向 512 個 indirect block
  // 第 0 個 address 指向的第 0 個 indirect block 負責 bn == 0 ~ 511
  // 第 1 個 address 指向的第 1 個 indirect block 負責 bn == 512 ~ (512 * 2 - 1)
  // 第 n 個 address 指向的第 n 個 indirect block 負責 ((n - 1) * NINDIRECT) <= bn && bn < n * NINDIRECT
  // doubly-indirect 中的第 bn 個 block
  // 是由 doubly-indirect block 中
  // 第 (bn / NINDIRECT) 個 address 指向的第 (bn / NINDIRECT) 個 indirect block 負責
  // 在這個 indrect block 中，
  // 又是其中的第 (bn % NINDIRECT) 個 address

  // 總之
  // 1. ip->addrs[NDIRECT + 1] 得知 doubly-indirect block
  // 2. doubly-indirect block 中的第 (bn / NINDIRECT) 個 address 可得知 indirect block
  // 3. indirect block 中的第 (bn % NINDIRECT) 個 address 是 bmap() 要回傳的 address
  if (bn < NINDIRECT * NINDIRECT) {
    if ((addr = ip->addrs[NDIRECT + 1]) == 0) {
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT + 1] = addr;
    }
    // 此時 addr 拿到的是 doubly-indirect block
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    // 現在要拿第 (bn / NINDIRECT) 個 address，
    // 可得知 indirect block 的 address
    if((addr = a[bn / NINDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn / NINDIRECT] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    // 現在這個 addr 是 indirect block 的 address
    // 要拿取第 (bn % NINDIRECT) 個 address
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if ((addr = a[bn % NINDIRECT]) == 0) {
      addr = balloc(ip->dev);
      if(addr){
        a[bn % NINDIRECT] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

#### `itrunc()`
```c
// Truncate inode (discard contents).
// Caller must hold ip->lock.
void
itrunc(struct inode *ip)
{
  int i, j, k;
  struct buf *bp;
  uint *a;
  // for doubly-indirect block
  struct buf *bp0, *bp1;
  uint *a0, *a1;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  // 處理 doubly-indirect block 的部份
  if (ip->addrs[NDIRECT + 1]) {
    bp1 = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a1 = (uint*)bp1->data;
    // 分別檢查這之中的 512 個 indirect block
    for(j = 0; j < NINDIRECT; j++){
      if(a1[j]) {
        bp0 = bread(ip->dev, a1[j]);
        a0 = (uint *) bp0->data;
        for (k = 0; k < NINDIRECT; k++) {
          if (a0[k])
            bfree(ip->dev, a0[k]);
        }
        brelse(bp0);
        bfree(ip->dev, a1[j]);
        a1[j] = 0;
      }
    }
    brelse(bp1);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

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
