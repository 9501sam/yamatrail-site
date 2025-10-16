+++
date = '2025-10-15T17:47:29+08:00'
draft = false
title = '[xv6 學習紀錄 06] Lab: Copy-on-Write Fork for xv6'
series = ["xv6 學習紀錄"]
weight = 61
+++
題目敘述:
> ## The problem
>  The fork() system call in xv6 copies all of the parent process's user-space memory into the child. If the parent is large, copying can take a long time. Worse, the work is often largely wasted: fork() is commonly followed by exec() in the child, which discards the copied memory, usually without using most of it. On the other hand, if both parent and child use a copied page, and one or both writes it, the copy is truly needed. 
> ## The solution
>  Your goal in implementing copy-on-write (COW) fork() is to defer allocating and copying physical memory pages until the copies are actually needed, if ever. 
> 
>  COW fork() creates just a pagetable for the child, with PTEs for user memory pointing to the parent's physical pages. COW fork() marks all the user PTEs in both parent and child as read-only. When either process tries to write one of these COW pages, the CPU will force a page fault. The kernel page-fault handler detects this case, allocates a page of physical memory for the faulting process, copies the original page into the new page, and modifies the relevant PTE in the faulting process to refer to the new page, this time with the PTE marked writeable. When the page fault handler returns, the user process will be able to write its copy of the page. 

> Here's a reasonable plan of attack.
>    1. Modify uvmcopy() to map the parent's physical pages into the child, instead of allocating new pages. Clear PTE_W in the PTEs of both child and parent for pages that have PTE_W set.
>    1. Modify usertrap() to recognize page faults. When a write page-fault occurs on a COW page that was originally writeable, allocate a new page with kalloc(), copy the old page to the new page, and install the new page in the PTE with PTE_W set. Pages that were originally read-only (not mapped PTE_W, like pages in the text segment) should remain read-only and shared between parent and child; a process that tries to write such a page should be killed.
>    1. Ensure that each physical page is freed when the last PTE reference to it goes away -- but not before. A good way to do this is to keep, for each physical page, a "reference count" of the number of user page tables that refer to that page. Set a page's reference count to one when kalloc() allocates it. Increment a page's reference count when fork causes a child to share the page, and decrement a page's count each time any process drops the page from its page table. kfree() should only place a page back on the free list if its reference count is zero. It's OK to to keep these counts in a fixed-size array of integers. You'll have to work out a scheme for how to index the array and how to choose its size. For example, you could index the array with the page's physical address divided by 4096, and give the array a number of elements equal to highest physical address of any page placed on the free list by kinit() in kalloc.c. Feel free to modify kalloc.c (e.g., kalloc() and kfree()) to maintain the reference counts.
>    1. Modify copyout() to use the same scheme as page faults when it encounters a COW page. 

## 題目的意圖
`fork()` 時會把整個 page table 也複製給 child, 但是大多數時候，child 並不會使用這個 page table 
常常都是在 `fork()` 之後去執行 `exec()` 這時 `exec()` 又會把原本的 page table 蓋掉, 這樣太浪費 physical memory 了，

比較好的作法是：`fork()` 時先讓 parent 跟 child 都指向同樣的 pa 區域, 直到要寫入時，才真正的把 physical memory 的內容複製一份，也就是題目所謂的 Copy-on-Write

## 大致上的流程:
1. `fork()` 使用 `uvmcopy()` 把 page table 複製過去
    * 在這個 lab 中，我們希望 `fork()` 後 child 跟 parent 的 page table 都是 map 到相同的 pa

1. 現在的 `uvmcopy()` 就是把 `old` 的東西複製到 `new`

1. 根據題目的需求，我們必須要做到以下幾件事情:
    * 把 `new` 跟 `old` 的 `pte_w` 都關掉，這樣想要寫入的時候，就會進入 trap() 處理
    * `new` 跟 `old` 都要立起 `pte_c`，這樣在 trap() 中就可以認出真正的 write page fault
    * 遇到 write page fault 時，check `pte_c`
    * ref count 在 kfree() 的時候減少 1
    * ref count 在 kalloc() 的時候設為 1
    * ref count 在 uvmcopy() 的時候設增加 1: 

doing

---
# Lab6 Copy-on-Write Fork for xv6 紀錄
## 題目的意圖
`fork()` 時會把整個 page table 也複製給 child, 但是大多數時候，child 並不會使用這個 page table 
常常都是在 `fork()` 去執行 `exec()` 這時 `exec()` 又會把原本的 page table 蓋掉, 這樣太浪費 physical memory 了，
比較好的作法是：`fork()` 時先讓 parent 跟 child 都指向同樣的 pa , 直到要寫入時，才真正的把 physical memory 的內容複製一份

## 大致上的流程:
1. `fork()` 使用 `uvmcopy()` 把 page table 複製過去
    * 在這個 lab 中，我們希望 `fork()` 後 child 跟 parent 的 page table 都是 map 到相同的 pa

1. 現在的 `uvmcopy()` 就是把 `old` 的東西複製到 `new`

1. 根據題目的需求，我們必須要做到以下幾件事情:
    * 把 `new` 跟 `old` 的 `pte_w` 都關掉，這樣想要寫入的時候，就會進入 trap() 處理
    * `new` 跟 `old` 都要立起 `pte_c`，這樣在 trap() 中就可以認出真正的 write page fault
    * 遇到 write page fault 時，check `pte_c`
    * ref count 在 kfree() 的時候減少 1
    * ref count 在 kalloc() 的時候設為 1
    * ref count 在 uvmcopy() 的時候設增加 1: 

## 幾個重要的時間點:
### `fork()`
* `fork()` 使用 `uvmcopy()` 把 child 的 page table 建立出來，
* `fork()` 結束之後，child 跟 parent 會得到相同的 virtual memory

### `uvmcopy()`
`fork()` 使用 `uvmcopy()` 把 parent 的 page table 複製一份給 child
* 原版 `uvmcopy()`: parent 的 memory 內容真正的複製出來 (使用了 `kalloc()`)
* cow 的版本: 只有把 page table 複製給 child, prarent 與 child 實際上是指到同樣的 physical memory

### write page fault
在 `kernel/trap.c: usertrap()` (`r_scause() == 15`)
* 遇到 write page fault ，並且 `pte_c` off: 真正的 page fault
* 遇到 write page fault ，並且 `pte_c` on: 準備執行 copy on write
    * 這時候才要真正的把 page 的內容複製一份出來，因為

### `kalloc()` 與 `kfree()`
原版執行 `kfree()` 的時候就是真的把這個 page free 掉了
可是 cow 的版本中會有問題

使用 cow 了話，有可能會有很多個 page table 指向同一個 page，
而這時候**什麼時候要 free 掉這個 page** 就成為了一個很大的問題
因為其中一個 process 要 free 掉這個 page 時，這個 page 可能正同時被其他 process 使用

解決方法：使用 counter 去計算各個 page 被多少個 process 使用，如果沒有任何 process 使用，
才需要真正的 free 掉這個 page

## 破解步驟
1. 在 `uvmcopy()` 中，刪掉 child 以及 parent 的所有 `pte_w`
    * 原本的 `pte_w` 所記載的資訊去哪了
2. 之後遇到 write page-fault 時
    * 並不是真正的 page-fault，他原本是 writeable 的
    * `kalloc()` 出一個新的 page 並且把 old page 的內容複製到 new page，並且 set `pte_w`
    * 原本 readonly 的區域，child 就使用 old page 的內容就好
    * 如果 child 想要修改 readonly 的內容，一樣要把他 kill 掉
3. 要新增 reference count 的功能
4. 在 `copyout()` 中使用跟 page fault 時一樣的方法
    * 也就是並不是真正的 page fault 的意思


## 需要修改與新增的 function
* `uvmcopy()`: 
* `usertrap()`: 
* `uncopied_cow()`: 還沒有被 copy 出來的 cow page
* `cow_alloc()`:
* `copyout()`: 

## 實驗紀錄
> 1. modify `uvmcopy()` to map the parent's physical pages into the child, instead of allocating new pages. clear `pte_w` in the ptes of both child and parent for pages that have `pte_w` set. 

* 修改 `uvmcopy()`, 讓 `fork()` 使用 `uvmcopy()` 複製 parent 的 va 內容給 child 時 
讓 child 的 `page_table` map 到 parent 的 physical page

讓 `usertrap()` 有辦法分辨真正的 write page fault 與 copy-on-write
* 使用 `PTE_C`
* 原本是 `PTE_W` on 的 page:
    * `PTE_W` 變成 off
    * `PTE_C` 變成 on
* 原本是 `PTE_W` off 的 page
    * `PTE_W` 維持原本的 off
    * `PTE_W` 維持原本的 off
    
    
如此一來 `usertrap()` 遇到 `r_scause() ==並且 `PTE_C` 為 on  15`，
就進入 copy-on-write 的情形處理

```clike=
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    if (*pte & PTE_W) {
      *pte &= ~PTE_W; // for copy on write: clear parent's PTE_W
      *pte |= PTE_C; // for copy on write: set PTE_C
    }
    flags = PTE_FLAGS(*pte);
    // if((mem = kalloc()) == 0)
    //   goto err;
    // memmove(mem, (char*)pa, PGSIZE);
    mem = (char *) pa;
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
    prefcnt_inc(pa);
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```
 
> 2. modify `usertrap()` to recognize page faults. when a write page-fault occurs on a cow page that was originally writeable, allocate a new page with kalloc(), copy the old page to the new page, and install the new page in the pte with `pte_w` set. pages that were originally read-only (not mapped `pte_w`, like pages in the text segment) should remain read-only and shared between parent and child; a process that tries to write such a page should be killed.  

修改 `userteap()`

* `kernel/trap.c: usertrap()`:
```clike=
// ...
  } else if((r_scause() == 15) && uncopied_cow(p->pagetable, r_stval())){
    if (cow_alloc(p->pagetable, r_stval()) < 0)
      p->killed = 1;
// ...
```

* `kernel/vm.c: uncopied_cow()`
```clike=
int
uncopied_cow(pagetable_t pagetable, uint64 va)
{
  if (va >= MAXVA)
    return 0;
  pte_t *pte;
  if ((pte = walk(pagetable, va, 0)) == 0)
    return 0;
  if ((*pte & PTE_V) == 0)
    return 0;
  if ((*pte & PTE_U) == 0)
    return 0;
  return (*pte & PTE_C);
}
```

* `kernel/vm.c: cow_alloc()`
```clike=
int
cow_alloc(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;
  uint flags;
  char *mem;

  va = pgrounddown(va);
  if (pte = walk(pagetable, va, 0) == 0)
    return -1;
  pa = pte2pa(*pte);
  flags = pte_flags(*pte);
  flags |= pte_w;
  if ((mem = kalloc()) == 0)
    return -1;
  memove(mem, (char *)pa, pgsize);
  if (mappages(pagetable, va, pgsize, (uint64)mem, flags) != 0) {
    kfree(mem);
    return -1;
  }
  return 0;
}
```
> 3. ensure that each physical page is freed when the last pte reference to it goes away -- but not before. a good way to do this is to keep, for each physical page, a "reference count" of the number of user page tables that refer to that page. set a page's reference count to one when kalloc() allocates it. increment a page's reference count when fork causes a child to share the page, and decrement a page's count each time any process drops the page from its page table. kfree() should only place a page back on the free list if its reference count is zero. it's ok to to keep these counts in a fixed-size array of integers. you'll have to work out a scheme for how to index the array and how to choose its size. for example, you could index the array with the page's physical address divided by 4096, and give the array a number of elements equal to highest physical address of any page placed on the free list by kinit() in kalloc.c. feel free to modify kalloc.c (e.g., kalloc() and kfree()) to maintain the reference counts.


因為一個已分配的 memory 可能會被多個 process 使用，我們必須要紀錄這個 page 有多少個 process 使用才行，如果使用人數為 0 時，才要真正的把這個 page 刪除掉。

* `kinit()` 時初始化
* `kalloc()` 時增加 
* `kfree()` 時增加 

複習 xv6 的 memory layout
![xv6 book figure 3.3]()
* 從 `kinit()` 可以看到，`kalloc()` 可以使用的範圍就存放在 `end` ~ `phystop`
    * `end`: 從 `kernel/kernel.ld` 可以得知他被存放在 kernel 的 `.text` 與 `.data` 之後
        * `end` 的位置不太確定，反正 linker 會幫我們弄好
        * 從圖上來看了話，`end` 是 free memory 的開始
        * free memory 也就是 `kalloc()` 可以使用的範圍
    * `phystop`: phycical address 的實際上的最大的 address = 0x88000000

從 `end` 與 `phystop`
先定義一些 macro

```clike=
#define pa2refindex(pa) ((pgrounddown(pa) - kernbase) / pgsize)
uint prefcnt[(phystop - kernbase) / pgsize];
```

* `kfree()`
* `kalloc()`
    * 把最後回傳的那個 page 對應到的 count 的值設為 1
    * 請注意，增加 count 是在 fork() 做的事情
* `init`

* `kernel/kalloc.c`:
```clike=
// Physical memory allocator, for user processes,
// kernel stacks, page-table pages,
// and pipe buffers. Allocates whole 4096-byte pages.

#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "spinlock.h"
#include "riscv.h"
#include "defs.h"

void freerange(void *pa_start, void *pa_end);

extern char end[]; // first address after kernel.
                   // defined by kernel.ld.

#define NUM_PREF_COUNT (PHYSTOP - KERNBASE) / PGSIZE
#define PA2REFIDX(pa) ((PGROUNDDOWN(pa) - KERNBASE) / PGSIZE)
#define PA2CNT(pa) prefcnt.count[PA2REFIDX((uint64) pa)]

struct {
  struct spinlock lock;
  uint count[NUM_PREF_COUNT];
} prefcnt;

struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;

void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&prefcnt.lock, "prefcnt");
  freerange(end, (void*)PHYSTOP);
}

void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE) {
    acquire(&prefcnt.lock);
    PA2CNT(p) = 1;
    release(&prefcnt.lock);
    kfree(p);
  }
}

// Free the page of physical memory pointed at by pa,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&prefcnt.lock);
  if (--PA2CNT(pa) <= 0) {
    // Fill with junk to catch dangling refs.
    memset(pa, 1, PGSIZE);

    r = (struct run*)pa;
    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  }

  release(&prefcnt.lock);
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r) {
    memset((char*)r, 5, PGSIZE); // fill with junk
    acquire(&prefcnt.lock);
    PA2CNT(r) = 1;
    release(&prefcnt.lock);
  }
  return (void*)r;
}

void
prefcnt_inc(uint64 pa)
{
  acquire(&prefcnt.lock);
  PA2CNT(pa)++;
  release(&prefcnt.lock);
}
```

> 4. modify copyout() to use the same scheme as page faults when it encounters a cow page. 
 
copy 出去的時候，不要指到跟 kernel 一樣的 pa, （要自己 copy 一份給 user mode 使用）
* `kernel/vm.c: copyout()`:
```clike=
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if (uncopied_cow(pagetable, va0)) {
      if (cow_alloc(pagetable, va0) == -1)
        return -1;
    }
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

## debug
* `freelist` 並沒有被正確的初始化
    * 解決方法: `pa2cnt(p) = 1` in freerange

* `panic: mappages: remap`
    * 不應該出現 remap
在
```
2  0x0000000080000e8e in cow_alloc (pagetable=0x87f73000, va=16384)
    at kernel/vm.c:487
```
出現了 `panic: mappages: remap` 的錯誤

照裡來說不應該出現 remap 的情況
`cow_alloc()` 由 `usertrap()` 遇到 write page fault 時呼叫
在 `cow_alloc()` 的情況下應該要

在 `cow_alloc()` 中的 `uvmunmap()` 必須要 `do_free` 這樣

> 先用執行 `cowtests` 來發現錯誤會比較容易
* 現在 `pipe() failed` 來看看是為什麼
    * 使用 gdb 得知為 `sys_pipe()` 中的 `copyout()` 發生了錯誤
    * 而且遇到的
[] 使用紙筆畫出 `filetest()` 的流程
[] 強烈建議好好檢查 `cowalloc()` in `copyout()` 
[] 不可 write 的 page 就維持不可以 write

## reference
[6.1810](https://pdos.csail.mit.edu/6.S081/2022/schedule.html)
[lab cow](https://pdos.csail.mit.edu/6.S081/2022/labs/cow.html)
[mit 6.s081 xv6 lab6 cow 实验记录](https://ttzytt.com/2022/07/xv6_lab6_record/)
https://five-embeddev.com/riscv-isa-manual/latest/supervisor.html#sec:scause
[11分鐘讀懂 linker scripts](https://blog.louie.lu/2016/11/06/10%E5%88%86%E9%90%98%E8%AE%80%E6%87%82-linker-scripts/)

