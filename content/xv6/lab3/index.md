+++
date = '2025-10-02T13:09:35+08:00'
draft = false
title = '[xv6 學習紀錄 03] Lab: page tables'
series = ["xv6 學習紀錄"]
weight = 32
+++
Lab 連結: [lab pgtbl](https://pdos.csail.mit.edu/6.S081/2022/labs/pgtbl.html)

## Speed up system calls (easy)
題目敘述:
> When each process is created, map one read-only page at `USYSCALL` (a virtual address defined in `memlayout.h`). At the start of this page, store a struct `usyscall` (also defined in `memlayout.h`), and initialize it to store the PID of the current process. For this lab, `ugetpid()` has been provided on the userspace side and will automatically use the `USYSCALL` mapping. You will receive full credit for this part of the lab if the `ugetpid` test case passes when running `pgtbltest`. 

### 目標：寫出 `ugetpid()` (優化版本的 `getpid()`)
* `getpid()`: 
    * 會需要使用到 `SYS_getpid` 這個 system call
    * 但是使用 system call 本身是耗費時間的
* `ugetpid()` 
    * 這個 function 直接不需要經由 system call 就可以達到相同的效果

### 作法
當一個 proccess 開始執行時，在 virtual memory address `USYSCALL`(定義在 `kernel/memlayout.h`) 中存放 `struct usyscall` 並且把 PID 存放於這個 struct 中。`ugetpid()` 是一個 user space 就可以執行的 function，所以可以跳過原本的 system call 的過程，以達到加速的效果。


### 原本的 `getpid()` 的流程為何
1. `user/pgtbltest.c: ugetpid_test()`
1. `getpid()`
1. `usys.S`
    ```asm
    .global getpid
    getpid:
     li a7, SYS_getpid
     ecall
     ret
    ```
1. `kernel/trampoline.S: uservec`
1. `kernel/trap.c: usertrap()`
1. `syscall.c: syscall()`
1. `sysproc.c: sys_getpid()`
1. `myproc()`
1. `kernel/trampoline.S: uservec`
    ```c
    uint64
    sys_getpid(void)
    {
      return myproc()->pid;
    }
    ```

### 優化版的 `getpid()` 的流程為何
1. `user/pgtbltest.c: ugetpid_test()`
1. `ugetpid()`
1. `user/ulib.c: ugetpid()`
    ```C
    // ulib.c

    int
    ugetpid(void)
    {
      struct usyscall *u = (struct usyscall *)USYSCALL;
      return u->pid;
    }
    ```
如同先前看到的 process address space
![process_layout2.png](process_layout2.png)
最高的 address 依序是 `MAXVA`, `TRAMPOLINE`, `TRAPFRAME`, 這一題則是要再加上 `USYSCALL`
### 程式實作
#### `usyscall` 題目幫我們定義好了
```c
// kernel/memlayout.h
struct usyscall {
  int pid;  // Process ID
};
```

之後每一個 process 都會存放一份 `usyscall`

#### `kernel/proc.h` 中
先在 `proc_pagetable()` 中加入
```C
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  struct usyscall *usyscall;   // add this !!!!!!!!!!!!!!!
};
```

準備好 data 之後，之後要在 allocate 時以及 free 時做一些處理
#### allocate 時:
* `kernel/proc.c: allocproc()`
* `kernel/proc.c: proc_pagetable()`
#### free 時:
* `kernel/proc.c: freeproc()`
* `kernel/proc.c: proc_freepagetable()`

#### `kernel/proc.c: allocproc()`
```C
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();
  p->state = USED;

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Allocate and initialize a usyscall page !!!!!!!!!!!
  if((p->usyscall = (struct usyscall *)kalloc()) == 0){ // here!
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  p->usyscall->pid = p->pid; !!!!!!!!!!!!!!

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

#### `kernel/proc.c: proc_pagetable()`:
```C
// Create a user page table for a given process, with no user memory,
// but with trampoline and trapframe pages.
pagetable_t
proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe page just below the trampoline page, for
  // trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 1);
    return 0;
  }

  // map one read-only page at USYSCALL !!!!!!!!!!!!!!!!!!!!!!!
  if(mappages(pagetable, USYSCALL, PGSIZE,
              (uint64)(p->usyscall), PTE_R | PTE_U) < 0){
    // 注意是 unmap 己經被 map 好的 TRAMPOLINE, TRAPFRAME
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  return pagetable;
}
```

#### `kernel/proc.c: freeproc()`
```C
// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;

  // free usyscall !!!!!!!!
  if (proc->usyscall)
    kfree((void *) p->usyscall);
  p->usyscall = 0;

  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

#### `kernel/proc.c: proc_freepagetable()`
```C
// Free a process's page table, and free the
// physical memory it refers to.
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, USYSCALL, 1, 0); // unmap USYSCALL !!!!!!!
  uvmfree(pagetable, sz);
}
```

## Print a page table (easy)
題目敘述：
> prints the contents of a page table Define a function called `vmprint()`. It should take a `pagetable_t` argument, and print that pagetable in the format described below. Insert `if(p->pid==1) vmprint(p->pagetable)` in `exec.c` just before the return argc, to print the first process's page table. You receive full credit for this part of the lab if you pass the pte printout test of make grade. 
* You can put `vmprint()` in `kernel/vm.c`.
* Use the macros at the end of the file `kernel/riscv.h`.
* The function `freewalk()` may be inspirational.
* Define the prototype for `vmprint` in `kernel/defs.h` so that you can call it from `exec.c`.
* Use `%p` in your `printf` calls to print out full 64-bit hex PTEs and addresses as shown in the example.

```asm
page table 0x0000000087f6b000
 ..0: pte 0x0000000021fd9c01 pa 0x0000000087f67000
 .. ..0: pte 0x0000000021fd9801 pa 0x0000000087f66000
 .. .. ..0: pte 0x0000000021fda01b pa 0x0000000087f68000
 .. .. ..1: pte 0x0000000021fd9417 pa 0x0000000087f65000
 .. .. ..2: pte 0x0000000021fd9007 pa 0x0000000087f64000
 .. .. ..3: pte 0x0000000021fd8c17 pa 0x0000000087f63000
 ..255: pte 0x0000000021fda801 pa 0x0000000087f6a000
 .. ..511: pte 0x0000000021fda401 pa 0x0000000087f69000
 .. .. ..509: pte 0x0000000021fdcc13 pa 0x0000000087f73000
 .. .. ..510: pte 0x0000000021fdd007 pa 0x0000000087f74000
 .. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
init: starting sh
```

#### `kernel/defs.h`
```C
void            vmprint(pagetable_t pagetable);
```

#### `vm.c`
```C
void
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable); // print level 2 pagetable

  // there are 2^9 = 512 PTEs in a page table.
  for (int i = 0; i < 512; i++) { // iterate PTEs in level 2 pagetable
    pte_t pte = pagetable[i];
    if ((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0) { 
      // find a valid PTE in level 2 pagetable
      uint64 lv1pa = PTE2PA(pte);
      pagetable_t lv1pagetable = (pagetable_t) lv1pa;
      printf(" ..%d: pte %p pa %p\n", i, pte, lv1pagetable); // print leve 1 pagetable

      for (int j = 0; j < 512; j++) { // iterate PTEs in level 1 pagetable
        pte_t pte = lv1pagetable[j];
        if ((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0) { 
          // find a valid PTE in level 1 pagetable
          uint64 lv0pa = PTE2PA(pte);
          pagetable_t lv0pagetable = (pagetable_t) lv0pa;
          printf(" .. ..%d: pte %p pa %p\n", j, pte, lv0pagetable); // print level 0 pagetable

          for (int k = 0; k < 512; k++) { // iterage PTEs in level 0 pagetable
            pte_t pte = lv0pagetable[k];
            if ((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0) {
              panic("vmptint: pagetable PTE in level 0 pagetable");
            } else if (pte & PTE_V) { // find a leaf
              uint64 pa = PTE2PA(pte);
              // pagetable_t lv0pagetable = (pagetable_t) lv1pa;
              printf(" .. .. ..%d: pte %p pa %p\n", k, pte, pa); // print leaf
            }
          }
        } else if (pte & PTE_V) {
          panic("vmptint: leaf in level 1 pagetable");
        }
      }
    } else if (pte & PTE_V) {
      panic("vmptint: leaf in level 2 pagetable");
    }
  }
}
```

> Insert `if(p->pid==1) vmprint(p->pagetable)` in `exec.c` just before the return argc, to print the first process's page table  

這裡只是希望 `pid == 1` 的 process (`sh`) 可以自動利用 `vmprint()` print 出 page tabel
#### `kernel/exec.c`
```C
int
exec(char *path, char **argv)
{
  // ... //

  if(p->pid==1)
    vmprint(p->pagetable);

  return argc; // this ends up in a0, the first argument to main(argc, argv)

  // ... //
}
```

> Explain the output of `vmprint` in terms of Fig 3-4 from the text. What does page 0 contain? What is in page 2? When running in user mode, could the process read/write the memory mapped by page 1? What does the third to last page contain? 

(TODO)

## Detect which pages have been accessed (hard)
題目敘述：
> Some garbage collectors (a form of automatic memory management) can benefit from information about which pages have been accessed (read or write). In this part of the lab, you will add a new feature to xv6 that detects and reports this information to userspace by inspecting the access bits in the RISC-V page table. The RISC-V hardware page walker marks these bits in the PTE whenever it resolves a TLB miss. 

> Your job is to implement `pgaccess()`, a system call that reports which pages have been accessed. The system call takes three arguments. First, it takes the starting virtual address of the first user page to check. Second, it takes the number of pages to check. Finally, it takes a user address to a buffer to store the results into a bitmask (a datastructure that uses one bit per page and where the first page corresponds to the least significant bit). You will receive full credit for this part of the lab if the pgaccess test case passes when running pgtbltest. 

題目提示：
* Read `pgaccess_test()` in `user/pgtlbtest.c` to see how pgaccess is used.
* Start by implementing `sys_pgaccess()` in `kernel/sysproc.c`.
* You'll need to parse arguments using `argaddr()` and `argint()`.
* For the output bitmask, it's easier to store a temporary buffer in the kernel and copy it to the user (via `copyout()`) after filling it with the right bits.
* It's okay to set an upper limit on the number of pages that can be scanned.
    * 題目的測試是用 `unsigned int`，因此設定為 32 的 pages 就足夠了
* `walk()` in `kernel/vm.c` is very useful for finding the right PTEs.
* You'll need to define `PTE_A`, the access bit, in `kernel/riscv.h`. Consult the [RISC-V privileged architecture manual](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMFDQC-and-Priv-v1.11/riscv-privileged-20190608.pdf) to determine its value.
    * 根據 manual, acessed bit 在 `1 << 6` 的位置
    ![ptea.png](ptea.png)
    * 並且這個 bit 的賦值是硬體上幫我們做的，不需要用軟體 set
* Be sure to clear `PTE_A` after checking if it is set. Otherwise, it won't be possible to determine if the page was accessed since the last time `pgaccess()` was called (i.e., the bit will be set forever).
* `vmprint()` may come in handy to debug page tables. 

這題要我們時做出 `pgaccess()`，先觀察 `pgaccess_test()` 是如何使用 `pgaccess()` 的
```C
void
pgaccess_test()
{
  char *buf;
  unsigned int abits;
  printf("pgaccess_test starting\n");
  testname = "pgaccess_test";
  // buf 有 
  // 32 個 page == 32 * PGSIZE bytes == 32 * 2 ^ 12 bytes == 32 * 4096 bytes 大小
  buf = malloc(32 * PGSIZE);

  // buf 都還沒被 access 過，理論上這時候 abits 應該為 
  // 0b00000000000000000000000000000000
  // (32 個 0，代表 buf 的 32 個 page 都還沒被 access 過)
  if (pgaccess(buf, 32, &abits) < 0)
    err("pgaccess failed");
  buf[PGSIZE * 1] += 1; // 第 1 個 page 被 access
  buf[PGSIZE * 2] += 1; // 第 2 個 page 被 access
  buf[PGSIZE * 30] += 1; // 第 30 個 page 被 access

  // 經過上面第 1, 2, 30 個 page 被 access 之後，abits 應該為
  // 0b00100000000000000000000000000110
  // 第 1, 2, 30 bits 被設定為 1
  if (pgaccess(buf, 32, &abits) < 0)
    err("pgaccess failed");
  if (abits != ((1 << 1) | (1 << 2) | (1 << 30)))
    err("incorrect access bits set");
  free(buf);
  printf("pgaccess_test: OK\n");
}
```

這就是一個 `pgaccess` 的使用例子

### 程式實做
* `kernel/riscv.h`
```c
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // user can access
#define PTE_A (1L << 6)
```
在這裡加上 `PTE_A` 的定義，並且這個 bit 是硬體上會幫我們 set 好的，所以我們**不需要**考慮 `buf[PGSIZE * 1] += 1;` 時還要特地去更改這個 bit。

* `kernel/sysproc.c: sys_pgaccess()`
```c
int
sys_pgaccess(void)
{
  uint64 base;
  int npages;
  uint64 umask;
  uint64 mask;
  uint64 abits;
  uint64 va;
  pte_t *pte;
  struct proc *p;

  p = myproc();
  argaddr(0, &base);
  argint(1, &npages);
  argaddr(2, &umask);
  if (npages > 32)
    return -1;
  mask = 1;
  abits = 0;
  for (va = base; va < base + npages * PGSIZE; va += PGSIZE) {
    pte = walk(p->pagetable, va, 0);
    if (pte == 0)
      continue;
    if (*pte & PTE_A) {
      abits |= mask;
      *pte = *pte & (~PTE_A);
    }
    mask <<= 1;
  }
  if (copyout(p->pagetable, umask, (char *) &abits, 4) < 0)
    return -1;
  return 0;
}
```
接著就一個 page 一個 page 的檢查 `PTE_A`，如果為 1 則 unset 並且紀錄在 `abits` 中，最後在用 `copyout()` 把答案寫回給 user process 就行了。
