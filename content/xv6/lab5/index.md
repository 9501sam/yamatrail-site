+++
date = '2025-10-11T11:28:40+08:00'
draft = false
title = '[xv6 學習紀錄 05] Lab: Traps'
series = ["xv6 學習紀錄"]
weight = 52
+++

* Lab 連結: [Lab: xv6 lazy page allocation](https://pdos.csail.mit.edu/6.S081/2020/labs/lazy.html)

## Eliminate allocation from sbrk() (easy)
> Your first task is to delete page allocation from the `sbrk(n)` system call implementation, which is the function `sys_sbrk()` in `sysproc.c`. The `sbrk(n)` system call grows the process's memory size by `n` bytes, and then returns the start of the newly allocated region (i.e., the old size). Your new `sbrk(n)` should just increment the process's size (`myproc()->sz`) by `n` and return the old size. It should not allocate memory -- so you should delete the call to `growproc()` (but you still need to increase the process's size!). 
* origin `sys_sbrk()`:
```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  if(growproc(n) < 0)
    return -1;
  return addr;
}
```
題目要我們改成這樣：
```c
uint64
sys_sbrk(void)
{
  uint64 addr;
  int n;

  argint(0, &n);
  addr = myproc()->sz;
  // if(growproc(n) < 0)
  //   return -1;
  myproc()->sz += n;
  return addr;
}
```

接著執行 `echo hi` 的結果：
```sh
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ echo hi
usertrap(): unexpected scause 0x000000000000000f pid=3
            sepc=0x00000000000012c4 stval=0x0000000000005008
panic: uvmunmap: not mapped
```

### 錯誤訊息解釋
* `usertrap()`:
    * 這些錯誤訊息是從 `kernel/trap.c: usertrap()` 產生出來的
* `unexpected scause 0x000000000000000f`:
    * register `scause` = 0x0f，是 page fault (TODO: 貼上來源)
* `pid = 3` 是 `sh` 的 pid
    * 在後面用 gdb 分析會知道這是 `sh`
* `sepc=0x00000000000012c4`:
    * 對應到 `sh.asm` 中的 `12ac: ld	ra,56(sp)`
* `stval=0x0000000000004008`:
    * 造成 page trap 的 virtual memory

### 使用 `gdb` 解析
```gdb
(gdb) b kernel/trap.c:70
(gdb) c
```
* `scause`
* `sepc`
* `stval`
* `struct proc p`
```gdb
(gdb) p *p
```

* `malloc()`
```gdb
(gdb) add-symbol-file user/_sh
```
`sh.c` 透過 `user/user.h` 引入 `user/umalloc.c` 的 `malloc.c`
```gdb
(gdb) b user/umalloc.c:malloc
```
問題發生在 sh -> malloc() -> morecore()

(TODO: more detailed, 說明跟 sbrk() 的關係為何)

## Lazy allocation (moderate)
## Lazytests and Usertests (moderate)


---
# Lab5 Lazy 紀錄

## 在處理 page fault 時，我們需要注意的幾件事情
* `stval`: 造成 page fault 的 virtual memory
* `sepc`: 造成 page fault 的 program counter
* instruction: 當下是正在執行什麼 instruction (?)
### registers
1. `stval`: va cause page fault
2. `scause`: the type of fault
    * Read
    * Write
    * Instruction
3. `sepc`: the va of instruction

## Eliminate allocation from `sbrk()`
這個 assignment 就只要根據題目的指示，把 `growproc()` 給註解掉，並且記得把 `myproc()->sz` 增加 `n` bytes

```Clike=
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  // if(growproc(n) < 0)
  //   return -1;
  myproc()->sz += n;
  return addr;
}
```
改好之後執行 `echo hi` 會爆出以下的錯誤
```=
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ echo hi
usertrap(): unexpected scause 0x000000000000000f pid=3
            sepc=0x00000000000012ac stval=0x0000000000004008
panic: uvmunmap: not mapped
```
##### 錯誤訊息解釋
* `usertrap()`:
    * 這些錯誤訊息是從 `kernel/trap.c: usertrap()` 印出來的
* `unexpected scause 0x000000000000000f`:
    * `usertrpa()` 處理這個 trap 時，`scause` = 0x0f 此情況並沒有對應的程式碼
* `pid = 3` 是 `echo hi` 的 pid
    * 註: `sh` 的 pid 是 2
* `sepc=0x00000000000012ac`:
    * 對應到 `sh.asm` 中的 `12ac: ld	ra,56(sp)`
    * 為什麼 `sepc` 停留在 `sh` 中而沒有進入到 `echo` ?
* `stval=0x0000000000004008`:
    * 造成 page trap 的 virtual memory (`sh` or `echo`?)

### 刪掉 `growproc()` 之後，會造成什麼樣的影響
* 也就是說 `sys_sbrk()` 沒有執行 `growproc()`
* 也就是沒有執行 `uvmalloc()`
* 也就是沒有執行 `kalloc()`
* 沒有執行 `kalloc()` 就是怎麼樣了(?)
* 最終會導致 user program (sh) 使用了他沒有分配到的記憶體位置(va)
* 我的猜想:
    * 在 sh 執行 `sw	s6,8(a0)` 時，正要 access 某個 va (`8 + a0`) 的當下
        * 硬體發現這個 PTE 的 `valid` flag 為 0 (false)
        * 於是發出了 0x0f 這個 trap (page fault)
* 所以以物理上的狀況來說 page fault 就是
> 當我要 access 某一個 va 時，我發現這個 va 在 page table 上給我的回覆為：
> 此 va 並不是使用中的狀態

## Lazy allocation
當 page fault 發生時，allocate 一個新的 page，並且回到 user space 繼續執行，修改任何的 kernel code，讓 `echo hi` 可以執行

### hints
* page fault 時 `scause` 為 13 or 15
* 從 `stval` 可以得知造成 page fault 的 va 是多少
* 請偷學 `uvmalloc()`，並呼叫 `kalloc()` 以及 `mappages()`
    * 先從偷學這裡開始
    * 當這個 va 沒有被分配時，我就手動幫他分配
    * 偷學 `uvmalloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)`
        * `pagetable`: `myproc()->pagetable`
        * `oldsz`: `myproc()->sz`
        * `newsz`: `r_stval()`
* `kernel/trap.c`: `usertrap()`
```clike=
void
usertrap(void)
{
  // ...
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scause() == 13 || r_scause() == 15){
    uint64 va = r_stval();
    lazy_alloc(va);
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
  
  // ...
}
```
* 當 `lazy_alloc()` 還沒有完成實作時，會發生什麼事情?

* `kernel/trap.c`: `lazy_alloc()`
```clike=
int
lazy_alloc(uint64 va)
{
  struct proc *p = myproc();
  pagetable_t pagetable = p->pagetable;
  uint64 new_page_va = PGROUNDDOWN(va);
  char *newmem = kalloc();

  if (newmem == 0)
    return -1;
  memset(newmem, 0, PGSIZE);
  if (mappages(pagetable, new_page_va, PGSIZE, (uint64) newmem,
        PTE_W|PTE_X|PTE_R|PTE_U) != 0) {
    kfree(newmem);
    return -1;
  }
  return 0;
}
```

* `kernel/vm.c`: 
```Clike=
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      // panic("uvmunmap: walk");
      continue;
    if((*pte & PTE_V) == 0)
      // panic("uvmunmap: not mapped");
      continue;
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```
* 之所以要變成 `continue` 是為什麼
    * `procgrow()` 與 lazy alloc 的差別
        * `growproc()` 會一次 alloc 連續的數個 page
        * lazy alloc 只會 allocate **一個** page
            * 所以說那些中間的不連續的沒有被 alloc 的 va 就會產生 `PTE_V` = 0 的問題
            
現在有了最基本的 lazy alloc, 接著來繼續完成

## Lazytests and Usertests
### hints
- [x] Handle negative sbrk() arguments.
- [x] Kill a process if it page-faults on a virtual memory address higher than any allocated with sbrk().
- [x] Handle the parent-to-child memory copy in fork() correctly.
- [ ] Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.
- [ ] Handle out-of-memory correctly: if kalloc() fails in the page fault handler, kill the current process.
- [ ] Handle faults on the invalid page below the user stack. 

### Handle negative sbrk() arguments.
對於正數，只需要把 `sz` 增加 n 就好
當參數為負數時，則要實際的釋放空間
釋放空間的方法: 就用原本的 `growproc()` 就好
```Clike=
uint64
sys_sbrk(void)
{
  int addr;
  int n;
  struct proc *p = myproc();

  addr = myproc()->sz;
  if (argint(0, &n) < 0)
    return -1;
  if (n < 0) {
    if (p->sz + n < 0)
      return -1;
    if (growproc(n) < -1)
      return -1;
  } else {
    myproc()->sz += n;
  }
  return addr;
}
```

### Kill a process if it page-faults on a virtual memory address higher than any allocated with sbrk().

澄清 lazy alloc 的意涵：
如果用 sbrk 增加記憶體空間，就增加 `sz`
真的到了 page fault 的時候再來處理 allocate 的問題就好

可是有時候遇到 page fault 並不代表這個 va 是允許存在的
以下幾種方情況並不是正常的 va 
1. `va` > `MAXVA`
1. 如果一個 page 的 `PTE_V` == 1 那麼他就一定不是 lazy alloc 因為它已經被分配了
1. `va` >= `p->sz` 因為並沒有藉由 `sbrk()` 去新增位置

解決方法:
```clike=
int
is_lazy_addr(int va)
{
  struct proc *p = myproc();

  if (va > MAXVA)
    return 0;

  pte_t *pte = walk(p->pagetable, va, 0);
  if (pte && (*pte & PTE_V))
    return 0;

  if (PGROUNDDOWN(va) > PGROUNDDOWN(p->sz))
    return 0;

  return 1;
}
```

> Handle the parent-to-child memory copy in fork() correctly.

主要在講 `kernel/proc.c: fork()` 中的呼叫的，`kernel/vm.c: uvmcopy`
原本的 `uvmcopy` 是 copy 一段連續的 memory, 可是現在使用 lazy alloc 了話，就不一定是連續的
解決方法：
`continue` 掉兩個 `panic`
```clike=
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  // ...
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      // panic("uvmcopy: pte should exist");
      continue;
    if((*pte & PTE_V) == 0)
      // panic("uvmcopy: page not present");
      continue;
    // ...
  }
  return 0;
}
```

### 問題一: `uvmunmap()` `walk()`
```shell=
$ lazytests
lazytests starting
running test lazy alloc
panic: uvmunmap: walk
```
`walk()` 出錯了
這跟前一個問題很像，都是因為 lazy alloc 只有 alloc 一個 page
而不是像原本的 `growproc` 是數個 page 的原因
解決方法：
```Clike=
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
    // ...
    
    if((pte = walk(pagetable, a, 0)) == 0)
      // panic("uvmunmap: walk");
      continue;
    // ...
}
```

> Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.

`copyin()` 與 `copyout()` 的問題
- [ ] trace code

## reference
* [6.S081 2020](https://pdos.csail.mit.edu/6.S081/2020/schedule.html)
* [Lab lazy](https://pdos.csail.mit.edu/6.S081/2020/labs/lazy.html)
* [xv6 book](https://pdos.csail.mit.edu/6.S081/2020/xv6/book-riscv-rev1.pdf)
* [video](https://youtu.be/KSYO-gTZo0A)
* [Lazy Page Allocation 实验记录](https://ttzytt.com/2022/07/xv6_lab5_record/)

