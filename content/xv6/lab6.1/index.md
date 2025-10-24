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

* `user/sh.c: getcmd()`
    * `kernel/sysfile.c: sys_write()`
        * `kernel/file.c: filewrite()`: 這是屬於 `f->type == FD_DEVICE`
            * `kernel/console.c: consolewrite()`: 不太確定是如何來到這裡的
            * `kernel/uart.c: uartputc()`: 實際上的對於一些 register 做動作

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

