+++
date = '2025-10-21T19:35:28+08:00'
draft = false
title = '[xv6 學習紀錄 07-1] Thread Switching'
series = ["xv6 學習紀錄"]
weight = 71
+++

課程影片連結：
* [6.S081 Fall 2020 Lecture 11: Thread Switching](https://www.youtube.com/watch?v=zRnGNndcVEA)
{{< youtube vsgrTHY5tkg >}}


## 追蹤 Context Switch 的過程
先看一下影片中的範例程式
* `user/spin.c`:
```c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char **argv)
{
  int pid;
  char c;

  pid = fork();
  if (pid == 0) {
    c = '/';
  } else {
    printf("parent pid is %d, child is %d", getpid(), pid);
    c = '\\';
  }

  for (int i = 0; ; i++) {
    if (i % 1000000 == 0)
      write(2, &c, 1);
  }

  exit(0);
}
```
在做的事情基本上是
* parent: 過了一陣子 print `\`
* child: 過了一陣子 print `/`
在這個情境之下，我們就可以觀察到 parent 與 child 的切換過程

### 開啟 gdb 並且執行 `spin`
### set breakpoint at timer interrupt
context switch 會執行的時間點在於 timer interrupt 發生的時機，也就是在 `scause == 0x8000000000000001L` 的時候會執行


* `kernel/trap.c`:
```c
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
  } else if(scause == 0x8000000000000001L){ // <- 這裡進入到 timer interrupt
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

```gdb
(gdb) b kernel/trap.c:xxx
(gdb) # 執行完 devintr() 並回到 usertrap()
(gdb) finish
```

* `kernel/trap.c`: `usertrap()`
```c
void
usertrap(void)
{
  // ...
  if(r_scause() == 8){
    // system call
    // ...
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    // ...
    setkilled(p);
  }

  if(killed(p))
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  // 在 timer interrupt 的情形之下，which_dev == 2 成立，等等會進入到 `yield()`
  // 之所以可以確保這次進來 usertrap() 追蹤到的是 timer interrupt，是因為
  // 先前設定 break point 的時候確保這次是進入到 timer interrupt
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

* what make kernel stack different is `kstack`
* `myproc()` using tp register

```gdb
print p->name
print p->pid
```
* 這裡也只能是 `spin` 了
* `pid` 會是 3 or 4


```gdb
print p->trampoline
```

```gdb
print/x p->trapframe->epc
0x62
```

* `user/spin.asm` 可以看到對應的 code 

### enter `yield()`
```
(gdb) step
```

* 這裡要避免其他 core 更改這個 process 因此需要 lock 做保護
```c
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE; // 放棄現在正在 RUNNING 的 process
  sched();             // 重新選擇一個 process 來執行
  release(&p->lock);
}
```

### step into `sched()`
```c
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```

前面有許多 sanity check，因為這裡遇到了很多的 bug，現在先不用理他

先專住在 `swtch()`
* `p->context`: 當前的 process 的 context
* `c->context`: scheduler 的 context

```gdb
p/x cpus[0].context
```
* `ra`: 等等要 return 到的位置
* `kernel.asm`: 在這裡對應到 `ra` 的 0x8001f2e
```gdb
x/i 0x80001f2e
```
在等等的 `swtch()` 結束之後，
那時的 `ra` 就會是 1f2e 然後就會回到 `scheduler()` 的這裡的 instruction

### step into `swtch()`
```gdb
tbreak swtch
c
layout asm
```

```asm
.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret
```
* why not save program counter
因為 program counter 就只是 point to swtch 並沒有太大的意義
這裡是 ra 扮演了這樣的角色
```gdb
p $ra
```
在 `sched()`

* why not save all the registers?
  call `swtch` 的 function 要自己 save caller saved registers
  也就是 (TODO:)

```gdb
p $sp
```
現在的 kstack
```gdb
stepi 28
print $sp
```
現在跳到了很開頭的 stack 的地方

```gdb
p $pc
p $ra
```

### 進入 `scheduler()`
這裡就好像夢醒一樣，回到了 `scheduler()` 執行的樣子
```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();

    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        // 現在來到了這裡，就好像是剛從 swtch() return 回來了一樣
        c->proc = 0;
      }
      release(&p->lock);
    }
  }
}
```
* 可以發現這裡能允許其他 core 執行任何的 process
* `p->lock`: 
    * turn to Runnable
    可以想像是 cpu 1 看到了 running 但是實際上是在轉換到 runnable 的過程中被看見

```gdb
tbreak proc.c:474 # c->proc = p;
```

這時候的 `ra` 是要 point to `schcd()` 這就好像是另一個 process 睡了一陣子之後，又從 `sched()` 醒過來的樣子

* process 會在 `sched()` 睡下去
* scheduler thread 在 `scheduler()` 醒過來

* : sleep 會使用 swtch??

* memory 的內容不會受到影響
