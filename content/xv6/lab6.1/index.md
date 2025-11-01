+++
date = '2025-10-16T15:52:53+08:00'
draft = false
title = '[xv6 學習紀錄 06-1] Interrupts'
series = ["xv6 學習紀錄"]
weight = 61
+++

課程影片連結：
* [6.S081 Fall 2020 Lecture 9: Interrupts](https://www.youtube.com/watch?v=zRnGNndcVEA)
{{< youtube zRnGNndcVEA >}}

## Terms
* PLIC
* UART

## Interrupts 的意義

跟 `syscall`, `traps` 很像
## Interrupts 從哪裡來?
一些 CPU 的 fig

programming device, 

memory mapped I/O

UART store/load to registers

## Case Study: `$ ls` 的出現

* `kerenl/main.c`
    * `kernel/console.c: consoleinit()`
        * `kernel/uart.c: uartinit()`
    * `kernel/plic.c: plicinit()`
    * `kernel/plic.c: plicinithart()`
    * `kernel/proc.c: scheduler()`

```c
void
main()
{
  if(cpuid() == 0){
    consoleinit(); // <--
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
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

### UART
* `kernel/console.c: consoleinit()`
```c
void
consoleinit(void)
{
  initlock(&cons.lock, "cons");

  uartinit(); // <--

  // connect read and write system calls
  // to consoleread and consolewrite.
  devsw[CONSOLE].read = consoleread;
  devsw[CONSOLE].write = consolewrite;
}
```
* `kernel/uart.c: uartinit()`
```c
void
uartinit(void)
{
  // disable interrupts.
  WriteReg(IER, 0x00);

  // special mode to set baud rate.
  WriteReg(LCR, LCR_BAUD_LATCH);

  // LSB for baud rate of 38.4K.
  WriteReg(0, 0x03);

  // MSB for baud rate of 38.4K.
  WriteReg(1, 0x00);

  // leave set-baud mode,
  // and set word length to 8 bits, no parity.
  WriteReg(LCR, LCR_EIGHT_BITS);

  // reset and enable FIFOs.
  WriteReg(FCR, FCR_FIFO_ENABLE | FCR_FIFO_CLEAR);

  // enable transmit and receive interrupts.
  WriteReg(IER, IER_TX_ENABLE | IER_RX_ENABLE);

  initlock(&uart_tx_lock, "uart");
}
```
在這裡可以設定一些 UART 的參數，像是 baud rate 等等

### PLIC (Platform-Level Interrupt Controller)
UART 的資訊先傳送到 PLIC，PLIC 再傳送到 OS，這裡則是在做 PLIC 的設定

* `kernel/plic.c: plicinit()`: 設定有哪些 interrupt 是開啟的
```c
void
plicinit(void)
{
  // set desired IRQ priorities non-zero (otherwise disabled).
  *(uint32*)(PLIC + UART0_IRQ*4) = 1;
  *(uint32*)(PLIC + VIRTIO0_IRQ*4) = 1;
}
```
目前開啟的有來自 UART 的 interrupt

* `kernel/plic.c: plicinithart()`: 這個 kernel 可以接受哪些 interrupt
```c
void
plicinithart(void)
{
  int hart = cpuid();
  
  // set enable bits for this hart's S-mode
  // for the uart and virtio disk.
  *(uint32*)PLIC_SENABLE(hart) = (1 << UART0_IRQ) | (1 << VIRTIO0_IRQ);

  // set this hart's S-mode priority threshold to 0.
  *(uint32*)PLIC_SPRIORITY(hart) = 0;
}
```
### scheduler
這裡真的進入 scheduler，會開始第一個 user process `user/init.c`，之後的詳情要看 context switch 的

* `kernel/proc.c: scheduler()`
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
        c->proc = 0;
      }
      release(&p->lock);
    }
  }
}
```

* `kernel/riscv.h`: 在進入到 `scheduler()` 的時候都會 `intr_on()`
```c
// enable device interrupts
static inline void
intr_on()
{
  w_sstatus(r_sstatus() | SSTATUS_SIE);
}
```

## `$` 的出現
* `user/init.c`
* `user/sh.c: getcmd()`
    * `kernel/sysfile.c: sys_write()`
        * `kernel/file.c: filewrite()`: 這是屬於 `f->type == FD_DEVICE`
            * `kernel/console.c: consolewrite()`: 不太確定是如何來到這裡的
            * `kernel/uart.c: uartputc()`: 實際上的對於一些 register 做動作

* `user/sh.c: getcmd()`
```c
int
getcmd(char *buf, int nbuf)
{
  write(2, "$ ", 2);
  memset(buf, 0, nbuf);
  gets(buf, nbuf);
  if(buf[0] == 0) // EOF
    return -1;
  return 0;
}
```

* `kernel/sysfile.c: sys_write()`
```c
uint64
sys_write(void)
{
  struct file *f;
  int n;
  uint64 p;
  
  argaddr(1, &p);
  argint(2, &n);
  if(argfd(0, 0, &f) < 0)
    return -1;

  return filewrite(f, p, n);
}
```
* `kernel/file.c: filewrite()`: 這是屬於 `f->type == FD_DEVICE`
```c
// Write to file f.
// addr is a user virtual address.
int
filewrite(struct file *f, uint64 addr, int n)
{
  int r, ret = 0;

  if(f->writable == 0)
    return -1;

  if(f->type == FD_PIPE){
    ret = pipewrite(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){ // <-------------------
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].write)
      return -1;
    ret = devsw[f->major].write(1, addr, n); // <--------------------
  } else if(f->type == FD_INODE){
    // write a few blocks at a time to avoid exceeding
    // the maximum log transaction size, including
    // i-node, indirect block, allocation blocks,
    // and 2 blocks of slop for non-aligned writes.
    // this really belongs lower down, since writei()
    // might be writing a device like the console.
    int max = ((MAXOPBLOCKS-1-1-2) / 2) * BSIZE;
    int i = 0;
    while(i < n){
      int n1 = n - i;
      if(n1 > max)
        n1 = max;

      begin_op();
      ilock(f->ip);
      if ((r = writei(f->ip, 1, addr + i, f->off, n1)) > 0)
        f->off += r;
      iunlock(f->ip);
      end_op();

      if(r != n1){
        // error from writei
        break;
      }
      i += r;
    }
    ret = (i == n ? n : -1);
  } else {
    panic("filewrite");
  }

  return ret;
}
```

* `kernel/console.c: consolewrite()`: (來到這裡的機制？)
```c
//
// user write()s to the console go here.
//
int
consolewrite(int user_src, uint64 src, int n)
{
  int i;

  for(i = 0; i < n; i++){
    char c;
    if(either_copyin(&c, user_src, src+i, 1) == -1)
      break;
    uartputc(c); <<----------------
  }

  return i;
}
```

* `kernel/uart.c: uartputc()`: 實際上的對於一些 register 做動作
```c
// the transmit output buffer.
struct spinlock uart_tx_lock;
#define UART_TX_BUF_SIZE 32
char uart_tx_buf[UART_TX_BUF_SIZE];
uint64 uart_tx_w; // write next to uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE]
uint64 uart_tx_r; // read next from uart_tx_buf[uart_tx_r % UART_TX_BUF_SIZE]

// add a character to the output buffer and tell the
// UART to start sending if it isn't already.
// blocks if the output buffer is full.
// because it may block, it can't be called
// from interrupts; it's only suitable for use
// by write().
void
uartputc(int c)
{
  acquire(&uart_tx_lock);

  if(panicked){
    for(;;)
      ;
  }
  while(uart_tx_w == uart_tx_r + UART_TX_BUF_SIZE){
    // buffer is full.
    // wait for uartstart() to open up space in the buffer.
    sleep(&uart_tx_r, &uart_tx_lock);
  }
  uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE] = c;
  uart_tx_w += 1;
  uartstart();
  release(&uart_tx_lock);
}
```
這裡的 buffer 有分 consumer/producer

* `kernel/uart.c: uartstart()`
```c
// if the UART is idle, and a character is waiting
// in the transmit buffer, send it.
// caller must hold uart_tx_lock.
// called from both the top- and bottom-half.
void
uartstart()
{
  while(1){
    if(uart_tx_w == uart_tx_r){
      // transmit buffer is empty.
      return;
    }
    
    if((ReadReg(LSR) & LSR_TX_IDLE) == 0){
      // the UART transmit holding register is full,
      // so we cannot give it another byte.
      // it will interrupt when it's ready for a new byte.
      return;
    }
    
    int c = uart_tx_buf[uart_tx_r % UART_TX_BUF_SIZE];
    uart_tx_r += 1;
    
    // maybe uartputc() is waiting for space in the buffer.
    wakeup(&uart_tx_r);
    
    WriteReg(THR, c); <<- 把要寫進去的東西放到 transmission register
  }
}
```
(54:43)

## Interrupt (HW)

### user press `l` key
1. if `SIE` bit set
1. clear `SIE` bit

1. sepc <- pc
1. save current mode
1. mode <- supervisor mode
1. PC <- `stvec` (points to trampoline page)
    * enter `usertrap()`

### in `usertrap()`
* `kernel/trap.c: usertrap()`: `which_dev = devintr()`
    `kernel/plic.c: plicinit()`
    `kernel/uart.c: uartgetc()`: 這不知道怎麼過來的

* `uartintr()`
    * `uartstart()`
* `consoleintr()`

? keyboard/console -> uart

### Interrupts and concurrency
### Producer/consumer
* `uart_tw_w`
* `uart_tw_r`
* `lock`

fileread()
`consoleread()`
`consoleintr()`
`wakeup()`

### Polling
* CPU spins until device has data
    * waste cpu cycles if device slow
    * but if fast, save entry/exit
=> dynamically switch between polling/interrupts

