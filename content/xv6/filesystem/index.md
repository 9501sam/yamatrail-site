+++
date = '2025-10-31T17:28:19+08:00'
title = '[xv6 學習紀錄 09-1] File System 相關程式碼解析'
series = ["xv6 學習紀錄"]
weight = 91
+++
```c
// Init fs
void
fsinit(int dev) {
  readsb(dev, &sb);
  if(sb.magic != FSMAGIC)
    panic("invalid file system");
  initlog(dev, &sb);
}

// Zero a block.
static void
bzero(int dev, int bno)
{
  struct buf *bp;

  bp = bread(dev, bno);
  memset(bp->data, 0, BSIZE);
  log_write(bp);
  brelse(bp);
}
```
## `balloc()` 與 `bfree()`
```c
// Blocks.

// Allocate a zeroed disk block.
// returns 0 if out of disk space.
static uint
balloc(uint dev)
{
  int b, bi, m;
  struct buf *bp;

  bp = 0;

  // 掃過 super block 
  for(b = 0; b < sb.size; b += BPB){
    bp = bread(dev, BBLOCK(b, sb));
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);
      if((bp->data[bi/8] & m) == 0){  // Is block free?
        bp->data[bi/8] |= m;  // Mark block in use.
        log_write(bp);
        brelse(bp);
        bzero(dev, b + bi);
        return b + bi;
      }
    }
    brelse(bp);
  }
  printf("balloc: out of blocks\n");
  return 0;
}
```
`balloc()` 是 block allocate 的意思

```c
// Free a disk block.
static void
bfree(int dev, uint b)
{
  struct buf *bp;
  int bi, m;

  bp = bread(dev, BBLOCK(b, sb));
  bi = b % BPB;
  m = 1 << (bi % 8);
  if((bp->data[bi/8] & m) == 0)
    panic("freeing free block");
  bp->data[bi/8] &= ~m;
  log_write(bp);
  brelse(bp);
}
```

### 一個 file 在 disk 中的樣子
![dinode.png](dinode.png)
在這張圖片中，每一個 "data" 都是一個 `BSIZE` (兩個 block)，即大小為，1024 bytes

也可以推論出在 xv6 中，一個檔案的最大的 size 為 12 + 256 = 268 blocks

* `kernel/fs.h`
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

```c
#define NINDIRECT (BSIZE / sizeof(uint))
```
* `NINDIRECT` (TODO)

知道一個檔案在 disk 中長什麼樣子之後，下一個要思考的問題會是如何跟 memory 中的資訊做連結

#### 在 memory 中的 file
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
  uint addrs[NDIRECT+1]; // 12 個 direct 與 1 個 indirect
};
```

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

### `itrunc()`
`itrunc()` 負責把 `inode` 中的資料刪除，包含 direct and indirect
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
