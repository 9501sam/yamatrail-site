+++
date = '2025-10-09T11:31:37+08:00'
draft = false
title = '[xv6 學習紀錄 04] Lab: Traps'
series = ["xv6 學習紀錄"]
weight = 43
+++

Lab 連結: [lab traps](https://pdos.csail.mit.edu/6.S081/2022/labs/traps.html)

課程影片連結：
* [6.S081 Fall 2020 Lecture 5: RISC-V Calling Convention and Stack Frames ](https://www.youtube.com/watch?v=s-Z5t_yTyTM)
* [6.S081 Fall 2020 Lecture 6: Isolation & System Call Entry/Exit](https://www.youtube.com/watch?v=T26UuauaxWA)

## Lecture 5: RISC-V Calling Convention and Stack Frames
What is a user process?
`layout asm`
`layout src`
focus reg
focus asm
si
info breakpoints
b sum_to


## RISC-V assembly (easy)
> It will be important to understand a bit of RISC-V assembly, which you were exposed to in 6.1910 (6.004). There is a file `user/call.c` in your xv6 repo. `make fs.img` compiles it and also produces a readable assembly version of the program in `user/call.asm`.
> Read the code in `call.asm` for the functions `g`, `f`, and `main`. The instruction manual for RISC-V is on the reference page. Here are some questions that you should answer (store the answers in a file `answers-traps.txt`): 

這題要我們閱讀 `user/call.c` 對應的 `call.asm` 確保我們對於 C 語言到 RISC-V 的轉換是有概念的，我們需要回答以下問題：

>  Which registers contain arguments to functions? For example, which register holds 13 in main's call to `printf`? 

arguments 是存放於 `a0`, `a1`, `a2` ... 例如
```c
printf("%d %d\n", f(8)+1, 13);
```
`13` 是第 3 個 argument, 存放於 `a2`，在 `call.asm` 中也可以看到
```asm
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13 # <-- put value 13 to a2
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7b850513          	addi	a0,a0,1976 # 7e0 <malloc+0xe6>
  30:	00000097          	auipc	ra,0x0
  34:	612080e7          	jalr	1554(ra) # 642 <printf>
```

>  Where is the call to function `f` in the assembly code for `main`? Where is the call to `g`? (Hint: the compiler may inline functions.) 

* `user/call.c`
```c
int g(int x) {
  return x+3;
}

int f(int x) {
  return g(x);
}

void main(void) {
  printf("%d %d\n", f(8)+1, 13);
  exit(0);
}
```
就以 `user/call.c` 來說，`f()` 只是呼叫 `g()`, `g()` 的作用只是回傳 `x + 3`，在 `call.asm` 中，可以看到 compiler 直接把 `f(8)+1` 的答案 `12` 算出來並且寫死丟入 `f(8)+1` 所代表的第 2 個 argument `a1` 中
```asm
void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13 # <-- call to f and call to g
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7b850513          	addi	a0,a0,1976 # 7e0 <malloc+0xe6>
  30:	00000097          	auipc	ra,0x0
  34:	612080e7          	jalr	1554(ra) # 642 <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	28e080e7          	jalr	654(ra) # 2c8 <exit>
```

>  At what address is the function `printf` located? 

根據 
```asm
  34:	612080e7          	jalr	1554(ra) # 642 <printf>
```
可得知 `printf` 的 address 在 `642` 中，`jalr 1554(ra)` 做了一下兩件事
* jump 到 `printf`
* Link: saves the address of the next instruction
```asm
void
printf(const char *fmt, ...)
{
 642:	711d                	addi	sp,sp,-96
 644:	ec06                	sd	ra,24(sp)
 646:	e822                	sd	s0,16(sp)
 648:	1000                	addi	s0,sp,32
 64a:	e40c                	sd	a1,8(s0)
 64c:	e810                	sd	a2,16(s0)
...
```
也可以在這裡的 `642` 中看到


>  What value is in the register `ra` just after the `jalr` to `printf` in `main`? 

> 

> Run the following code.
> 
> 	```c
>   unsigned int i = 0x00646c72;
> 	printf("H%x Wo%s", 57616, &i);
> 	```
>       
> 
> What is the output? Here's an ASCII table that maps bytes to characters.
> 
> The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change `57616` to a different value?
> 
> Here's a description of little- and big-endian and a more whimsical description.

> In the following code, what is going to be printed after `'y='`? (note: the answer is not a specific value.) Why does this happen?
> 
> ```c
> printf("x=%d y=%d", 3);
> ```
      



## Backtrace (moderate)


> Implement a `backtrace()` function in `kernel/printf.c`. Insert a call to this function in `sys_sleep`, and then run bttest, which calls `sys_sleep`. Your output should be a list of return addresses with this form (but the numbers will likely be different):
> 
> ```shell
>     backtrace:
>     0x0000000080002cda
>     0x0000000080002bb6
>     0x0000000080002898
> ```
>   
> 
> After `bttest` exit qemu. In a terminal window: run `addr2line -e kernel/kernel` (or `riscv64-unknown-elf-addr2line -e kernel/kernel`) and cut-and-paste the addresses from your backtrace, like this:
> 
> ```shell
>     $ addr2line -e kernel/kernel
>     0x0000000080002de2
>     0x0000000080002f4a
>     0x0000000080002bfc
>     Ctrl-D
> ```
>   
> 
> You should see something like this:
> 
> ```sh
>     kernel/sysproc.c:74
>     kernel/syscall.c:224
>     kernel/trap.c:85
> ```
我們需要先了解 xv6 的 stack 長什麼樣子才有辦法解這一題
```text
                   .
                   .
      +->          .
      |   +-----------------+   |
      |   | return address  |   |
      |   |   previous fp ------+
      |   | saved registers |
      |   | local variables |
      |   |       ...       | <-+
      |   +-----------------+   |
      |   | return address  |   |
      +------ previous fp   |   |
          | saved registers |   |
          | local variables |   |
      +-> |       ...       |   |
      |   +-----------------+   |
      |   | return address  |   |
      |   |   previous fp ------+
      |   | saved registers |
      |   | local variables |
      |   |       ...       | <-+
      |   +-----------------+   |
      |   | return address  |   |
      +------ previous fp   |   |
          | saved registers |   |
          | local variables |   |
  $fp --> |       ...       |   |
          +-----------------+   |
          | return address  |   |
          |   previous fp ------+
          | saved registers |
  $sp --> | local variables |
          +-----------------+
```
從這裡可以看到，理論上我們可以從 `sp` 開始，一次一次的利用 `fp` 往上找尋前一個 frame，就可以達成題目要的 `backtrace`


## Alarm (hard)
> In this exercise you'll add a feature to xv6 that periodically alerts a process as it uses CPU time. This might be useful for compute-bound processes that want to limit how much CPU time they chew up, or for processes that want to compute but also want to take some periodic action. More generally, you'll be implementing a primitive form of user-level interrupt/fault handlers; you could use something similar to handle page faults in the application, for example. Your solution is correct if it passes `alarmtest` and `usertests -q`.
這個 lab 要我們計算 cpu 的時間

---

# Lab4 Traps: 紀錄
[lab: Traps](https://pdos.csail.mit.edu/6.S081/2022/labs/traps.html)

## 一些跟這個 lab 有關的 register
![Table 4.2]()
![Table 3.7]()
![Figure 4.12]()

## 程式碼解析

#### `yeild()`
`yeild()` 讓 `myproc()` 的 state 變成 `RUNNABLE`
並且呼叫 `sched()`，`sched()` 可以讓其他 function 也有機會可以被執行到
`yeild()` 想要做的事情是:
"我用 CPU 用夠久了，可以換其他人使用了"
所謂的 "換其他人使用" 也就是呼叫 `sched()`，它會幫我們處理

```Clike=
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

#### `sched()` 解析
`sched()` 只會被以下幾個情境被使用到:
* `exit()`
* `yeild()`
* `sleep()`
從這裡我們可以發現，這都是在一些 "該輪到別人使用 CPU 了" 的時間點
所以可以推測 `sched()` 是用來決定下一個使用 CPU 的人是誰
不過讓我比較意外地點是，`scheduler()` 並沒有呼叫到 `sched()`，
那麼 `scheduler()` 到底扮演了什麼樣的角色呢

```C
// Switch to scheduler.  Must hold only p->lock
// and have changed proc->state. Saves and restores
// intena because intena is a property of this
// kernel thread, not this CPU. It should
// be proc->intena and proc->noff, but that would
// break in the few places where a lock is held but
// there's no process.
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock)) // 我應該要 hold 我自己的 lock
    panic("sched p->lock");
  if(mycpu()->noff != 1) // TODO: why noff should be equals to 1 ?
    panic("sched locks");
  if(p->state == RUNNING) // sched() 的 caller 會先把 myproc() 設為 RUNNABLE
    panic("sched running");
  if(intr_get()) // TODO: learn sstatus first
    panic("sched interruptible");

  intena = mycpu()->intena; // why?
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```
幾個困惑的點
* `swtch()` 實際上是如何運作的
* `mycpu()->context` 裡面到底裝著些什麼東西
* `mycpu()->context` 什麼時候會被初始化

#### `devintr()`
用來確認 interrupt 的種類
回傳值：
    * 2 -> timer interrupt
    * 1 -> 其他 device
    * 0 -> 認不出來
```Clike=
// check if it's an external interrupt or software interrupt,
// and handle it.
// returns 2 if timer interrupt,
// 1 if other device,
// 0 if not recognized.
int
devintr()
{
  uint64 scause = r_scause();

  if((scause & 0x8000000000000000L) &&
     (scause & 0xff) == 9){
    // this is a supervisor external interrupt, via PLIC.

    // irq indicates which device interrupted.
    int irq = plic_claim();

    if(irq == UART0_IRQ){
      uartintr();
    } else if(irq == VIRTIO0_IRQ){
      virtio_disk_intr();
    } else if(irq){
      printf("unexpected interrupt irq=%d\n", irq);
    }

    // the PLIC allows each device to raise at most one
    // interrupt at a time; tell the PLIC the device is
    // now allowed to interrupt again.
    if(irq)
      plic_complete(irq);

    return 1;
  } else if(scause == 0x8000000000000001L){
    // software interrupt from a machine-mode timer interrupt,
    // forwarded by timervec in kernelvec.S.

    if(cpuid() == 0){
      clockintr();
    }
    
    // acknowledge the software interrupt by clearing
    // the SSIP bit in sip.
    w_sip(r_sip() & ~2);

    return 2;
  } else {
    return 0;
  }
}
```

* `usertrap()` 跟 `kerneltrap()` 的差別在哪裡?

### Platform-Level Interrupt Controller(PLIC)

## Lab
### Backtrace
**print the saved return address in each stack frame.**
題目需求：
    1. 在 `kernel/printf.c` 中實作出 `backtrace()`
    2. 在 `sys_sleep` 呼叫 `backtrace()`

1. 把 `backtrace()` 的原型加到 `kernel/defs.h`
```Clike=
void            backtrace(void);
```

##### xv6 中的 stack:
```text
                   .
                   .
      +->          .
      |   +-----------------+   |
      |   | return address  |   |
      |   |   previous fp ------+
      |   | saved registers |
      |   | local variables |
      |   |       ...       | <-+
      |   +-----------------+   |
      |   | return address  |   |
      +------ previous fp   |   |
          | saved registers |   |
          | local variables |   |
      +-> |       ...       |   |
      |   +-----------------+   |
      |   | return address  |   |
      |   |   previous fp ------+
      |   | saved registers |
      |   | local variables |
      |   |       ...       | <-+
      |   +-----------------+   |
      |   | return address  |   |
      +------ previous fp   |   |
          | saved registers |   |
          | local variables |   |
  $fp --> |       ...       |   |
          +-----------------+   |
          | return address  |   |
          |   previous fp ------+
          | saved registers |
  $sp --> | local variables |
          +-----------------+
```


1. 把以下內容加到 `kernel/riscv.h`
```Clike=
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;

}
```
`r_fp()` 可以讓我們拿到 fram pointer `fp` 的值 
拿到 `fp` 之後，先 print 出來 `fp` 裡面到底放著什麼東西
這裡要注意一個很重要的特性(some hints 中有說明):
kernel stack 只會被塞在一個 page (4096 bytes) 中
我們可以用
```Clike=
void
backtrace(void)
{
  uint64 *cur_frame = (uint64 *)r_fp();
  printf("%p\n", *(cur_frame - 1));
}
```

```Clike=
void
backtrace(void)
{
  void *cur_frame;
  void *bot;

  cur_frame = (void *)r_fp();
  bot = (void *) PGROUNDUP((uint64)cur_frame);
  while (cur_frame < bot) {
    printf("%p\n", *((void **)cur_frame - 1));
    cur_frame = *((void **)cur_frame - 2);
  }
}
```

### Alarm
* 這裡的 tick 並不是拿來推動 cpu 的 tick，而應該是拿來當時鐘的 tick
```Clike=
// Per-process state
struct proc {
  // ...

  // these are provided to handle SYS_sigalarm
  int ticks;                   // The number of ticks between alarm calls
  int ticks_since_alarm;       // Ticks since alarm
  void (*handler)();           // Called when alarm
};
```
在 `trap.c`: `usertrap()` 中實作
```Clike=
void
usertrap(void)
{
  // ...

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2) {
    if (p->ticks_since_alarm++ > p->ticks) {
      // p->handler(); // TODO: why this error
      p->trapframe->epc = (uint64) p->handler;
    } else {
      yield();
    }
  }

  usertrapret();
}
```

* 策略: 
    * 手上有的：每一個 tick `if(which_dev == 2)` 都會成立一次
    * 題目要的：每一個 tick `if(which_dev == 2)` 都會成立一次
    * tick 的處理優先順序高於 `yield()`
    * 如果 handler 執行了，還要執行 `yield()` 嗎？
        * 我個人認為，不用，因為執行 handler 就已經跟 `yield()` 有很類似的效果了
* 雖然每一個 process 都有 `ticks`, `ticks_since_alarm`, `handler` 但是這並不代表
    每一個 process 都需要進行 alarm 的檢查
    * 判斷的標準在於 `tick == 0` 則 alarm disable
    * 對應到 `sigalarm(0, 0)` 會把 alarm disable

#### 為什麼不可以
```C
void
usertrap(void)
{
  // ...

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2) {
    if (p->ticks_since_alarm++ > p->ticks) {
      // p->handler(); // TODO: why this error
      p->trapframe->epc = (uint64) p->handler;
    } else {
      yield();
    }
  }
  usertrapret();
}
```
* `p->handler` 使用了是 (pa/va)?
* `p->handler()` 實際上會做什麼事情
* `p->trapframe->epc = (uint64) p->handler` 實際上會做什麼事情?

* 想要回答上面的問題了話，必須要搞清楚以下的幾個問題
    * 在 usertrap() 中，正在使用的 page table 是哪一個
        * 使用 kernel page table ，因為在 uservec 之後應該都是 kernel mode
    * handler 所紀錄的 address 是 va/pa ? 用的是哪一個 page table
        * 從 `user/alarmtest.c` 當中可以看到
    * 在 usertrap 中的這個當下，各個 register 的狀態如何
        * 在 `uservec`
    * `p->handler()` 實際上會做什麼事情
        * 使用 gdb trace
    * `p->trapframe->epc = (uint64) p->handler` 實際上會做什麼事情?
        * 使用 gdb trace

### `p->trapframe->epc = (uint64) p->handler` 實際上會做什麼事情?
* 當 trap 的流程**結束之後** 會繼續根據 `epc` 的內容往下執行
    * 太漂亮了!!

### `p->handler()` 實際上會做什麼事情
這個會把 program counter 跑到 handler 的地方，
the key point is: is handler a va or pa
* should be a va that using page table of user program
* in trap.c we are using kernel page table
* this is why we can not `p->handler()`

### test1/test2()/test3(): resume interrupted code
現在我缺少了什麼事情？
現在缺少了 `sigreturn()`
`handler()` 的最後面需要去呼叫 `sigreturn()` 像這樣：
```Clike=
void
periodic()
{
  count = count + 1;
  printf("alarm!\n");
  sigreturn(); // like this
}
```

問題:
當 handler 結束之後回到原本的 user program 會出錯
因為這不是正常的使用 call stack，而是在 usertrap() 中**直接**修改


所謂的一個 process 其實也就是由
1. registers 的內容
1. 一個 page table
1. 放在 memory 的 instructions
所組成的

也就是說在 `handler()` 之後我們需要還原以下一些東西:
1. registers 的內容
1. 把 `sapt` 指到屬於那個 user program 的地方

我們該怎麼知道那些 registers 的值被放到哪裡？

* trap 發生時，在 `uservec` 中會把這些 register 的值放到 `struct proc` 的 trapfram 中
* 這個 user program 的位置存放在

### 在正常情況下是如何從 kernel mode 回到 user program
應該是 userret 就會處理好
而現在的問題在於原本的 program counter(`sepc`) 在 usertrapret 被洗掉了
他被洗掉了，我們就完全找不回來的

原本的步驟為：
* `uservec`
* `userret`

題目的 hint:
* 在 sigreturn() 中需要把 user program 的 registers 給復元回去 
    * trapfram 中有做紀錄
    * 問題在於原本的 sepc 被洗掉了
    * 就像 lab pagetable 那樣新增一個 struct 去紀錄 alarm 需要紀錄的東西
        * 目前應該只需要紀錄一個 program counter 的值就好
    * 也可以把**所有的** registers 都重新備一份
        * 真的有這個必要嗎?

### test3 failed: register a0 changed
這裡來探討 `a0` 的旅程
0. 在 user program 中 `a0` 有一個值，可能是有用的，也可能是沒有在用的
1. 在 uservec 中
    * 把 `a0` 的值存到 `sscratch` 中
    * `a0` 用來存放 `TRAPFRAME`
    * 把 `a0` 的值從 `sscratch` 中拿回來
    * 也應該把 `a0` 的值放到 `p->trampoline` 中
2. 接下來在 `sigreturn()` 應該會把 a0 放回去才對

問題在於 `dummy_handler()`:
```C
//
// dummy alarm handler; after running immediately uninstall
// itself and finish signal handling
void
dummy_handler()
{
  sigalarm(0, 0);
  sigreturn();
}
```
`sigalarm(0, 0)` 會讓 alarm 關掉，關掉之後，就不會再一次的進入到 
`dummy_handler()` 中了
那麼跟 a0 有什麼關係

請注意，test3() 的 `a0` 被改掉了
* 應該是 `dummy_handler()` 有問題
* 也就是說 `sigreturn()` 沒有把 `a0` 搞定 
* 也就是說 `sys_sigreturn()` 中 `p->alarmtrame` 中的 `a0` 是錯誤的
* 也就是說 `p->trapframe` 中的 `a0` 是錯誤的
    * 我完全不這麼認為
* 在某個地方 a0 變成了 0

## TODO
- 再看一次影片
- trapmpoline.S
- using gdb

## 參考資料
[lab: Traps](https://pdos.csail.mit.edu/6.S081/2022/labs/traps.html)
[xv6 book](https://pdos.csail.mit.edu/6.S081/2022/xv6/book-riscv-rev3.pdf)
[reference page](https://pdos.csail.mit.edu/6.S081/2022/reference.html)
[RISC-V privileged instructions](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)
[10.2 自制操作系统: risc-v Supervisor寄存器
sscratch/sepc/scause/stval/senvcfg](https://blog.csdn.net/dai_xiangjun/article/details/124083732)
[MIT 6.s081 Xv6 Lab4 Traps 实验记录](https://ttzytt.com/2022/07/xv6_lab4_record/)

