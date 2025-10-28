+++
date = '2025-10-23T14:41:29+08:00'
draft = false
title = '[xv6 學習紀錄 08] Lab: locks'
series = ["xv6 學習紀錄"]
weight = 84
+++
Lab 連結：[Lab: locks](https://pdos.csail.mit.edu/6.S081/2022/labs/lock.html)

## Memory allocator (moderate)
> Your job is to implement per-CPU freelists, and stealing when a CPU's free list is empty. You must give all of your locks names that start with "kmem". That is, you should call `initlock` for each of your locks, and pass a name that starts with "kmem". Run kalloctest to see if your implementation has reduced lock contention. To check that it can still allocate all of memory, run `usertests` `sbrkmuch`. Your output will look similar to that shown below, with much-reduced contention in total on kmem locks, although the specific numbers will differ. Make sure all tests in `usertests -q` pass. make grade should say that the `kalloctests` pass. 

### 做出 per-CPU freelists
> You can use the constant `NCPU` from `kernel/param.h` 

把原先的 `kmem` 變成一個 array，題目說大小可以直接用 `NCPU` 的 8 個 CPU
```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];
```

> Let `freerange` give all free memory to the CPU running `freerange`. 

這裡的意思也就是 CPU 0 在初始的過程中會拿到所有的 memory，這裡要想到的問題是 CPU 1 ~ CPU 7 在初始過後沒有拿到任何的 memory 
這在之後 steal 的機制會說明
`freerange`
```c
void
kinit()
{
  for (int i = 0; i < NCPU; i++)
    initlock(&kmem[i].lock, "kmem");

  freerange(end, (void*)PHYSTOP);
}
```

* `kfree()` 與 `kalloc()` 變成了都只針對
```c
void
kfree(void *pa)
{
  int id;
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  // 這裡變成都是針對當前 CPU 的 freelist 進行操作
  push_off();
  id = cpuid();
  pop_off();
  acquire(&kmem[id].lock);
  r->next = kmem[id].freelist;
  kmem[id].freelist = r;
  release(&kmem[id].lock);
}

void *
kalloc(void)
{
  int id;
  struct run *r;

  // 這裡變成都是針對當前 CPU 的 freelist 進行操作
  push_off();
  id = cpuid();
  pop_off();
  acquire(&kmem[id].lock);
  r = kmem[id].freelist;
  if(r)
    kmem[id].freelist = r->next;
  release(&kmem[id].lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

### steal 的機制
如同先前所說，在 `freerange()` 的初始之後，CPU 1 會拿到所有的 memory，試想現在 CPU 1 想要 `kalloc()` 一個 page，雖然 CPU 0 還有很多 memory 可以用，但是 CPU 1 還是會拿不到任何的 memory，這時候我們可以
1. steal CPU 0 的所有 page
1. steal CPU 0 的一個 page

這兩個方法都是可行的想法，但是沒有幫助到最一開始的目的，這裡我用的是，`kalloc()` 發現沒有 memory 可以用的時候
* steal CPU 0 的 page 數量的一半

```c
void *
kalloc(void)
{
  int id;
  int victim;
  struct run *r;
  struct run *slow;
  struct run *prevslow = 0;
  struct run *fast;

  push_off();
  id = cpuid();
  pop_off();
  acquire(&kmem[id].lock);
  r = kmem[id].freelist;
  if (r) {
    kmem[id].freelist = r->next;
    release(&kmem[id].lock);
    memset((char*)r, 5, PGSIZE); // fill with junk
    return (void*)r;
  } 
  release(&kmem[id].lock);

  for (victim = 0; victim < NCPU; victim++) {
    if (victim == id)
      continue;
    acquire(&kmem[victim].lock);
    if (kmem[victim].freelist) {
      fast = slow = kmem[victim].freelist;
      while (fast && fast->next) {
        prevslow = slow;
        slow = slow->next;
        fast = fast->next->next;
      }
      if (!prevslow) {
        r = kmem[victim].freelist;
        kmem[victim].freelist = (struct run *) 0;
      } else {
        r = slow;
        prevslow->next = (struct run *) 0;
      }

      acquire(&kmem[id].lock);
      kmem[id].freelist = r->next;
      release(&kmem[id].lock);
      release(&kmem[victim].lock);

      memset((char*)r, 5, PGSIZE); // fill with junk
      return (void*)r;
    }
    release(&kmem[victim].lock);
  }
  return (void *) 0;
}
```
實做過程中間覺得 `acquire()` 跟 `release()` 的時機真的需要想清楚，尤其是不同的分支都要顧慮到，不然還蠻容易漏掉的。

## Buffer cache (hard)
> If multiple processes use the file system intensively, they will likely contend for `bcache.lock`, which protects the disk block cache in `kernel/bio.c`. `bcachetest` creates several processes that repeatedly read different files in order to generate contention on `bcache.lock`; its output looks like this (before you complete this lab): 

這裡的情況是說假設有很多的 process 在使用 file system 這時候的 `bcache.lock` (用來保護 disk block 的 lock) 會被激烈的被搶奪，`bcachetest` 製造了這個 `bcache.lock` 的 contention 的情形

```sh
$ bcachetest
start test0
test0 results:
--- lock kmem/bcache stats
lock: kmem: #test-and-set 0 #acquire() 33035
lock: bcache: #test-and-set 16142 #acquire() 65978
--- top 5 contended locks:
lock: virtio_disk: #test-and-set 162870 #acquire() 1188
lock: proc: #test-and-set 51936 #acquire() 73732
lock: bcache: #test-and-set 16142 #acquire() 65978
lock: uart: #test-and-set 7505 #acquire() 117
lock: proc: #test-and-set 6937 #acquire() 73420
tot= 16142
test0: FAIL
start test1
test1 OK
```
這裡要注意的是 `bcache.lock` 的 contention 過高

> Modify the block cache so that the number of `acquire` loop iterations for all locks in the `bcache` is close to zero when running bcachetest. Ideally the sum of the counts for all locks involved in the block cache should be zero, but it's OK if the sum is less than 500. Modify bget and `brelse` so that concurrent lookups and releases for different blocks that are in the bcache are unlikely to conflict on locks (e.g., don't all have to wait for `bcache.lock`). You must maintain the invariant that at most one copy of each block is cached. When you are done, your output should be similar to that shown below (though not identical). Make sure `usertests -q` still passes. make grade should pass all tests when you are done. 

題目要我們更改

我猜根據 lock 多，可以獲得較多 concurrency 的原則，應該是可以透過增加 lock 來減少 contention 的數量

> * Read the description of the block cache in the xv6 book (Section 8.1-8.3). 
