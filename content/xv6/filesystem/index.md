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

## Symbolic links 相關程式碼解析
這部份著重於先了解完成 Symbolic links (moderate) 所需的程式碼內容

### `major` and `minor`
再接下來的程式碼中，會看到許多的 `major` and `minor` 可以想成
* `major`: 設備種類的編號，例如
    * `major == 1` 可能代表 disk
    * `major == 2` 可能代表 UART
* `minor`: 編號，例如在 `major == 1` 的情況之下
    * `major == 1` 可能代表第一個 disk
    * `major == 2` 可能代表第二個 disk

```c
struct devsw devsw[NDEV];
```

```c
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // (如 T_FILE, T_DIR, T_DEVICE)
  short major; // 只有在 T_DEVICE 的時候才有效
  short minor; // 只有在 T_DEVICE 的時候才有效
  short nlink;
  uint size;
  uint addrs[NDIRECT+2];
};
```

### `sys_mknod()` 與 `create()`
* `user/init.c: main()`
    * `kernel/sysfile.c`

```c
int
main(void)
{
  int pid, wpid;

  if(open("console", O_RDWR) < 0){
    // 這裡在說建立一個 major == CONSOLE, minor == 0 的 device file
    mknod("console", CONSOLE, 0);
    open("console", O_RDWR);
  }
  dup(0);  // stdout
  dup(0);  // stderr

  // ...
}
```

```c
uint64
sys_mknod(void)
{
  struct inode *ip;
  char path[MAXPATH];
  int major, minor;

  // begin_op() 與 end_op() 會保證在 logging level 上保證 atomicity
  begin_op();
  argint(1, &major); // major == CONSOLE == 1
  argint(2, &minor); // minor == 0

  // 這裡會取得 "console"
  if((argstr(0, path, MAXPATH)) < 0 ||
     // 在 path 建立一個新檔案
     // type 為 T_DEVICE
     // 指定 major, minor
     (ip = create(path, T_DEVICE, major, minor)) == 0){
    end_op();
    return -1; // 如果失敗 return -1
  }

  // create() 成功之後，會回傳一個上鎖的 inode
  // iunlockput():
  // 1. iunlock(ip): release inode->lock (sleeplock)
  // 2. iput(ip): inode->ref--
  iunlockput(ip);

  // 結束對於檔案的操作
  // 這會讓這次的操作寫入 log
  end_op();
  return 0;
}
```

```c
static struct inode*
create(char *path, short type, short major, short minor)
{
  // ip: 新建立的 inode
  // dp: directory pointer: 指向 parent 例如 /a/b/c 的 ip -> c, dp -> /a/b
  struct inode *ip, *dp;
  char name[DIRSIZ];

  // 尋找 parent, 以 create /a/b/c 為例
  // dp 會是 /a/b
  // name 會是 c
  if((dp = nameiparent(path, name)) == 0)
    return 0; // 例如 /a/b/c 但 /a/b 不存在

  // 鎖定 parent 準備修改他
  ilock(dp);

  // 檢查 dp (/a/b) 中有沒有 name (c)
  if((ip = dirlookup(dp, name, 0)) != 0){
    // /a/b 中已經有 c 了
    iunlockput(dp);
    // 準備檢查 c
    ilock(ip);
    // 檢查 type 有沒有對應到
    if(type == T_FILE && (ip->type == T_FILE || ip->type == T_DEVICE))
      return ip;
    iunlockput(ip);
    // type 不一致一樣是錯誤
    return 0;
  }

  // 如果 /a/b 中沒有 c, 使用 ialloc() 建立一個
  if((ip = ialloc(dp->dev, type)) == 0){
    iunlockput(dp);
    return 0;
  }

  // 修改這個新建立的 inode ip
  ilock(ip);
  ip->major = major;
  ip->minor = minor;
  ip->nlink = 1;
  iupdate(ip); // 把這個新的 inode 寫入 log 中

  // 如果是 T_DIR
  if(type == T_DIR){  // Create . and .. entries.
    // No ip->nlink++ for ".": avoid cyclic ref count.
    if(dirlink(ip, ".", ip->inum) < 0 || // 把 "." 指向自己 (ip->inum)
       dirlink(ip, "..", dp->inum) < 0)  // 把 ".." 指向 parent (dp->inum)
      goto fail;
  }

  // 把新的 inode 連結到 parent
  if(dirlink(dp, name, ip->inum) < 0)
    goto fail;

  // parent 的 nlink 也要增加，因為多了這個新的 file
  if(type == T_DIR){
    // now that success is guaranteed:
    dp->nlink++;  // for ".."
    iupdate(dp);
  }

  iunlockput(dp);

  return ip;

 fail:
  // something went wrong. de-allocate ip.
  ip->nlink = 0;
  iupdate(ip);
  iunlockput(ip);
  iunlockput(dp);
  return 0;
}
```

### `writei()`
這裡是真的修改 inode 中的資料的地方，像是 large file 那一題實際上最後也是會呼叫到 `writei()` 之後 `writei()` 會繼續呼叫 `bmap()` 到這裡來

* `kernel/sysfile.c: sys_write()`
    * `kernel/file.c: filewrite()`
        * `kernel/fs.c: writei()`

```c
// Write data to inode. 把資料寫入 inode
// Caller must hold ip->lock.
// If user_src==1, then src is a user virtual address;
// otherwise, src is a kernel address.
// Returns the number of bytes successfully written.
// If the return value is less than the requested n,
// there was an error of some kind.

// ip: 要寫入的 inode
// 如果 user_src == 1 src 是一個 user virtual address
// 如果 user_src == 0 src 是一個 kernel address
// return 寫入的 bytes 數量
// write n bytes
int
writei(struct inode *ip, int user_src, uint64 src, uint off, uint n)
{
  // tot: 總共寫入了多少個 bytes
  // m: 這次的迴圈要寫入多少 bytes
  uint tot, m;
  struct buf *bp;

  // check off (offset)
  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > MAXFILE*BSIZE)
    return -1;

  for(tot=0; tot<n; tot+=m, off+=m, src+=m){
    uint addr = bmap(ip, off/BSIZE);
    if(addr == 0)
      break;
    bp = bread(ip->dev, addr);
    m = min(n - tot, BSIZE - off%BSIZE);
    if(either_copyin(bp->data + (off % BSIZE), user_src, src, m) == -1) {
      brelse(bp);
      break;
    }
    log_write(bp);
    brelse(bp);
  }

  if(off > ip->size)
    ip->size = off;

  // write the i-node back to disk even if the size didn't change
  // because the loop above might have called bmap() and added a new
  // block to ip->addrs[].
  iupdate(ip);

  return tot;
}
```

### `open()` 解析
* `kernel/fcntl.h` (File Control)
```c
#define O_RDONLY  0x000 // read only
#define O_WRONLY  0x001 // write only
#define O_RDWR    0x002 // read and write
#define O_CREATE  0x200 // create, 如果檔案不存在就建立
#define O_TRUNC   0x400 // truncate, 如果檔案存在就清空
```

* `kernel/sysfile.c`
```c
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  // open 的 flag 放在第 1 個 argument 中，flag 的定義放在 fcntl.h
  argint(1, &omode); 
  if((n = argstr(0, path, MAXPATH)) < 0)
    return -1;

  // begin_op() 與 end_op() 會保證在 logging level 上保證 atomicity
  begin_op();

  // O_CREATE: 如果檔案不存在，就建立一個檔案
  if(omode & O_CREATE){
    // create() 會直接回傳一個已經 ilock(ip) 的 inode, 所以不必再上鎖
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    // 如果這個檔案不存在並且沒有 O_CREATE, 是錯誤
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    // namei() 回傳的 ip 還沒上鎖，所以要再 ilock(ip)
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      // 如果是 T_DIR 並且 not read only
      iunlockput(ip);
      end_op();
      return -1;
    }
  }

  // 到這裡，我們已經拿到了一個上鎖的 struct inode *ip

  // 如果是 T_DEVICE, 要先檢查 ip->major 是否合法
  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op();
    return -1;
  }

  // 分配 file descriptor 與 file structure

  // filealloc(): find an empty struct file from ftable
  // fdalloc(): find a file descriptor from myproc()->ofile[NOFILE]
  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op();
    return -1;
  }

  // 現在拿到了 file `f` and file descriptor `fd`

  if(ip->type == T_DEVICE){
    // 如果是 T_DEVICE 就把這些資訊放入 f 中
    f->type = FD_DEVICE;
    f->major = ip->major;
  } else {
    // 如果是 FD_INODE, 代表普通檔案或是 dir
    f->type = FD_INODE;
    // 把 offset 設定為 
    f->off = 0;
  }
  f->ip = ip; // 這個 file 代表的是 inode ip
  // f->readable and f->writeable 由當初的 flag 決定
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);

  // 如果是 O_TRUNC, 清空檔案
  if((omode & O_TRUNC) && ip->type == T_FILE){
    itrunc(ip);
  }

  // 對 inode `ip` 做完了動作，解鎖
  // 這裡是用 iunlock() 而不是 iunlockput()
  // iunlockput() 會順帶減少 f->ref, 但這裡不用
  // 這裡只是 unlock, 而不是釋放引用
  iunlock(ip);
  end_op();

  return fd;
}
```

### `filealloc()` 與 `fdalloc()`
* `kernel/file.h`
```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE, FD_DEVICE } type;
  int ref; // reference count
  char readable;
  char writable;
  struct pipe *pipe; // FD_PIPE
  struct inode *ip;  // FD_INODE and FD_DEVICE
  uint off;          // FD_INODE
  short major;       // FD_DEVICE
};
```
