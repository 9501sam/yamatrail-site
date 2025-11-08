+++
date = '2025-11-06T13:40:57+08:00'
draft = false
title = '[xv6 學習紀錄 11-2] xv6 book ch5: Interrupts and device drivers'
series = ["xv6 學習紀錄"]
weight = 110
+++
這一篇筆記主要是為了 lab: network driver 中，要我們先讀一下 xv6 book ch5: Interrupts and device drivers，這篇文章是我自己的筆記

## Interrupts and device drivers
* Driver 需要了解 device 的 interface，可能很複雜或是文件寫得很差
* Device 通常可以製造一些 interrupt, 在 xv6 中，這在 `devintr()` 中處理
* `kernel/trap.c: devintr()`: 像是這裡回傳是否為 device interrupt
```c
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
    }
#ifdef LAB_NET
    else if(irq == E1000_IRQ){
      e1000_intr();
    }
#endif
    else if(irq){
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
* 許多 device driver 分為兩個部份：
    * process's kernel thread
    * interrupt time

接著分別使用 Console input/output 來看 driver 如何運作
## 5.1 Code: Console input
* console driver 寫在 `kernel/console.c`, 可以處理 keyboard 的 input, backspace and control-u
* 當我們使用鍵盤輸入時，qemu 會幫我們模擬成 UART 訊息
* 這裡模擬的 UART 種類是 16550 chip
    * 在這裡的 qemu 環境中，它代表的是 keyboard and display
* UART 透過 memory-mapped control register 跟軟體溝通
* 在 xv6 中 `UART0` 在 `0x10000000L` 的位置
    * `kernel/memlayout.h`
    ```c
    // qemu puts UART registers here in physical memory.
    #define UART0 0x10000000L
    ```
* 在開頭 `UART0` 之後的 offset 定義於 `kernel/uart.c`
*  `kernel/console.c: consoleinit()`: 這裡定義了 UART 的初始化過程
```c
void
consoleinit(void)
{
  initlock(&cons.lock, "cons");

  uartinit();

  // connect read and write system calls
  // to consoleread and consolewrite.
  devsw[CONSOLE].read = consoleread;
  devsw[CONSOLE].write = consolewrite;
}
```
* `consoleread()`

## 5.2 Code: Console output
* `uartputc()`
* `uartstart()`
* `uart_tx_buf`

## 5.3 Concurrency in drivers
需要使用 lock 做控制

## 5.4 Timer interrupts
* CLINT register

