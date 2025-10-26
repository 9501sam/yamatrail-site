+++
date = '2025-10-25T13:13:07+08:00'
draft = false
title = '[xv6 學習紀錄 08-1] Lecture 10: Multiprocessors and Locks'
series = ["xv6 學習紀錄"]
weight = 81
+++

課程影片連結：
* [6.S081 Fall 2020 Lecture 10: Multiprocessors and Locks](https://www.youtube.com/watch?v=NGXu3vN7yAk)
{{< youtube NGXu3vN7yAk >}}


## Deadlock
```c
acquire(&l)
acquire(&l)
release(&l)
```

## Locks vs. Performance
* 越多 lock 雖然可能可以比較安全，但也會出現效能上的不佳

## Case study UART
* `kernel/uart.c`
```c
struct spinlock uart_tx_lock;

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

void
uartintr(void)
{
  // read and process incoming characters.
  while(1){
    int c = uartgetc();
    if(c == -1)
      break;
    consoleintr(c);
  }

  // send buffered characters.
  acquire(&uart_tx_lock);
  uartstart();
  release(&uart_tx_lock);
}
```

### Lock rules
* protect data structure 

## Lock 的實做
### broken lock
```c
void
acquire(struct spinlock *lk) // does not work!
{
	for(;;) {
		if(lk->locked == 0) {
			lk->locked = 1;
			break;
		}
	}
}
```
這是一個錯誤的實做方法，因為可能有兩個 CPU 進入到 `if (l->locked == 0)`

### 正確的 Lock 實做方法
* 使用 `amoswap`

```asm
amoswap addr, r1, r2
```

`amoswap` 具有 atomic 的性質
```text
lock addr
  tmp   <- *addr
  *addr <- r1
  r2    <- tmp
unlock
```
這裡的 `lock addr` 與 `unlock` 是硬體上保證只有一個 CPU 可以修改 `*addr`，從這裡也可以看到實做 lock 需要硬體上的支援
* `r2    <- *addr`
* `*addr <- r1`
不同的 ISA 會有不一樣的 instruction 支援

但核心概念都會是只有一個 CPU 可以針對 `*addr` 操作：
* `r2    <- *addr`
* `*addr <- r1`

情境一：成功取得 Lock (Lock 原本是 0)

    初始狀態：

        lk (在 addr) = 0 (未鎖定)

        r1 = 1 (我們想寫入的值)

        r2 = (內容不重要)

    CPU 執行 amoswap addr, r1, r2

    原子操作開始：

        (內部) tmp <- *addr (讀到 lk 的值，tmp = 0)

        *addr <- r1 (將 r1 的值 1 寫入 lk，lk 現在變成 1)

        r2 <- tmp (將 tmp 的值 0 寫入 r2，r2 現在變成 0)

    結果：

        lk (記憶體) 變成了 1 (已鎖定)。

        r2 (暫存器) 變成了 0。

    程式解讀： 程式檢查 r2 的值，發現是 0。這代表「在我們寫入 1 之前，這個 lock 的值是 0」，因此我們成功取得了 Lock。

情境二：取得 Lock 失敗 (Lock 原本是 1)

    初始狀態：

        lk (在 addr) = 1 (已被其他 CPU 鎖定)

        r1 = 1 (我們想寫入的值)

        r2 = (內容不重要)

    CPU 執行 amoswap addr, r1, r2

    原子操作開始：

        (內部) tmp <- *addr (讀到 lk 的值，tmp = 1)

        *addr <- r1 (將 r1 的值 1 寫入 lk，lk 仍然是 1)

        r2 <- tmp (將 tmp 的值 1 寫入 r2，r2 現在變成 1)

    結果：

        lk (記憶體) 仍然是 1 (保持鎖定)。

        r2 (暫存器) 變成了 1。

    程式解讀： 程式檢查 r2 的值，發現是 1。這代表「在我們嘗試寫入 1 之前，這個 lock 的值就已經是 1 了」，因此我們取得 Lock 失敗，必須繼續旋轉等待 (spin) 並重試。

#### 使用 `amoswap` 實做 `acquire()` 與 `release()`

* `kernel/spinlock.h`
```c
// Mutual exclusion lock.
struct spinlock {
  uint locked;       // Is the lock held?

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
};
```

* `kernel/spinlock.h`
```c
void
acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk))
    panic("acquire");

  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}

// Release the lock.
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  __sync_lock_release(&lk->locked);

  pop_off();
}
```
