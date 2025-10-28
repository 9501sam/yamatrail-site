+++
date = '2025-10-27T12:28:33+08:00'
draft = false
title = '[xv6 學習紀錄 08-2] kalloc 相關程式碼解析'
series = ["xv6 學習紀錄"]
weight = 82
+++

## `kernel/kalloc.c`
### data structure `kmem` 與 `fun`
* `kmem` 負責紀錄可使用的 pages，這些 pages 用一個 linked list 做紀錄
```c
struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;
```

這個 Linked list `run` 是一個裡面就只有一個值，指向下一個可用的 page 的 address

### `kalloc()` 與 `kfree()`
```c
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

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```
* `kalloc()` 中，回傳 `freelist` 的 head，新的 head 變成 `freelist->next`
* Returns 0 if the memory cannot be allocated.
    * 之所以這會成立，是因為在最一開始最一開始的時候，`struct run *freelist;` 被初始成 `null` (0)，`kinit()` 之前的這個唯一的 `freelist` 最後會變成 linked list 的結尾，這也是 `kalloc()` 之所以把記憶體用光之後會回傳 0 的原因

```c
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

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```
* `pa` 必須是一個 page 的開頭
* `kfree()` 把這個不要的 `pa` 放到 `freelist` 的開頭

### `freelist` 的初始化過程

* `kernel/main.c`
```c
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator <- 這裡進入 kalloc.c
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode table
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}
```

* `kernel/kalloc.c: kinit()`
```c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```
1. 初始 `kmem.lock`
1. 使用 `freerange()` 把可用的 page 都放到 `freelist` 中

* `kernel/kalloc.c: freerange()`
```c
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}
```
這裡使用先前提到的 `kfree()` 一次一次的把可用 page 放到 `freelist` 的前方
