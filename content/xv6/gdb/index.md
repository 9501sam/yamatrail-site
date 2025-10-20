+++
date = '2025-09-27T15:46:30+08:00'
draft = false
title = '[xv6 學習紀錄 04-2] 如何使用 gdb 追蹤 xv6 的 system call 過程'
series = ["xv6 學習紀錄"]
weight = 42
+++
本文目標：照著以下步驟就可以看到整個 system call 的過程
整個過程大致上都是照著[這個影片](https://www.youtube.com/watch?v=T26UuauaxWA)做的，但其中有幾個步驟稍稍的不同。
{{< youtube T26UuauaxWA >}}

## 2. 用 gdb-multiarch debug xv6 的方式
這裡會需要開啟 2 個終端機，  

先在其中一個終端機輸入
```sh=
make qemu-gdb #　這是被 debug 的對象
```
在另一個終端機輸入
```sh=
gdb-multiarch # 開啟 debuger 會開始針對上面的那個終端機中的程式進行除錯
```
不知道也沒關係的點：`gdb-multiarch` 是透過 `.gdbinit` 這個檔案找到 debug 的對象的
```shell=
# .gdbinit
set confirm off
set architecture riscv:rv64
target remote 127.0.0.1:26000 # 透過這個來找到 qemu
symbol-file kernel/kernel
set disassemble-next-line auto
set riscv use-compressed-breakpoints yes
```

## 動機：`$` 的出現
`make qemu` 後 xv6 開機時，會顯示的畫面
```sh=
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ 
```
xv6 第一個執行的程式是 `sh`，它最一開始會 print 出字元 `$` print 的過程需要使用 system call `write`。

> 我們的目標就在於追蹤這個 `$` 如何經由 system call `write` 所產生出來的過程。

這篇文章分為以下兩個部份
1. 以程式碼的角度解釋
1. 用 `gdb` 觀察實際運行的步驟

## 以程式碼的角度解釋
直接說結論有以下的重點：
1. `user/sh.c`: `write(2, "$ ", 2)`
1. `user/sh.asm`: `write`
### 找到對應的 C 語言程式碼
在 `user/sh.c` 中
```C=
// user/sh.c
int
getcmd(char *buf, int nbuf)
{
  write(2, "$ ", 2); // <------ where `sh` print `$` !
  memset(buf, 0, nbuf);
  gets(buf, nbuf);
  if(buf[0] == 0)
    return -1;
  return 0;
}
```
### 找到對應的組合語言
`write(2, "$ ", 2);` 這行式碼編譯出來的組合語言被放在 `user/sh.asm` 當中，對應到的組合語言片段如下:

```asm=
  write(2, "$ ", 2);
      10:	4609                	li	a2,2
      12:	00001597          	auipc	a1,0x1
      16:	2ee58593          	addi	a1,a1,750 # 1300 <malloc+0xea>
      1a:	4509                	li	a0,2
      1c:	00001097          	auipc	ra,0x1
      20:	de8080e7          	jalr	-536(ra) # e04 <write>
```

```asm=
0000000000000e04 <write>:
.global write
write:
 li a7, SYS_write
     e04:	48c1                	li	a7,16
 ecall
     e06:	00000073          	ecall
 ret
     e0a:	8082                	ret
```
組合語言解釋
* `wirte` 這個 procedure 位於 `0xe04` 中(virtual address)
* `li a7, SYS_write`
    * `li` 指的是 load immediate，把　`SYS_write`(16, 定義在 `kernel/syscall.h`) 存放到 register `a7` 當中
* `ecall` 這個指令會觸發 trap

### 觸發 trap 之後，會發生什麼事情?
觸發 trap 之後可以進入到 kernel mode，觸發 trap 有三種可能的情況
* system call  <--- `ecall` 觸發的 trap
* interrupt  
* exception  

在這個例子中，是 ```ecall```  用 system call 的方式觸發 trap，在說明 trap 之後會怎麼樣前，要先來介紹一些相關的 register:
* ```stvec```: 當 trap 發生時，RISC-V 會把 program counter 存在 ```stvec``` 中
* ```scause```: RISC-V 會把觸發 trap 的原因放到 ```scause``` 中
* ```sscratch```: 進入到 kernel space 前 ```sscratch``` 會儲存 trapframe 的位置
* ```sstatus```: Supervisor status rigister
* `satp`: Supervisor Address Translation and Protection Register
* `sepc`: Supervisor Exception Program Counter, 進入 supervisor mode 後，用來紀錄回到 user mode 時，要回到什麼 address 開始執行

當 trap 發生時，會做的事情：
1. 把 ```pc``` 複製到 ```secp```(Supervisor Exception Program Counter) 中
2. 把現在的狀態 (user or supervisor) 紀錄在 ```sstatus``` 的 SPP bit
3. 把造成 trap 的原因紀錄在 ```scause``` 
4. 把 ```stvec``` 放到 ```pc``` 中
5. 開始根據 ```pc``` 往下執行

## 4. 實際使用 `gdb` 追蹤這個過程
1. 把中斷點設在 ```ecall``` 
```gdb=
(gdb) break *0xe18 # 根據 user/sh.asm, ecall 的 address (virtual)
(gdb) continue # continue 往下執行直到 ecall (正準備執行，但還沒)
(gdb) layout asm # 可以比較方便的看到目前執行的指令
```
到了這裡，我們準備要執行 ```ecall``` 以觸發 trap 了，執行 trap 就是想要從 ```stvec``` 的那裡執行

> 這裡我用**影片中的方法無法正常執行**，影片中是用 ```(gdb) si``` 就可以往下跑到 trap 的流程，可是我在自己執行的結果卻是會直接把 trap 的流程跑完，直到 ```ecall``` 的下一個指令 ```ret``` 那裡（可能是 gdb 的版本不同），想要追蹤 trap 的流程就必須要自行設定中斷點。

2. 使用 ```si``` 往下執行之前，先 print 出 ```stvec``` 的值，並且把中斷點設到那個 address (virtual)

```gdb=
(gdb) print/x $stvec # 用 /x (hex) 的格式 print 出 stvec 的值
$1 = 0x3ffffff000
(gdb) break *0x3ffffff000 # break *stvec
(gdb) si # 執行 ecall, 真正進入 supervisor mode 並且處理由 system call 觸發的 trap
```

也可以在進入 system call 之前使用
```gdb
(gdb) break *$stvec
```
會比較方便

### ```stvec``` 中的 0x3ffffff000 指向哪一段程式碼
執行 ```ecall``` 之後
* 會跑到 `stvec` 指向的位置 0x3ffffff000 開始執行
* 也就是會跑到 `kernel/trampoline.S` 的 `<uservec>`  
* `<uservec>` 會一直執行到 `jr t0` 為止
    * 之後會跑到到 `kernel/trap.c: usertrap()` 執行

### `kernel/trap.c: usertrap()`
接下來就是 C 語言了
* 可以用 `(gdb) layout src` 來切換到 C 語言的檢視模式
```clike=
// trap.c
void
usertrap(void)
{
    /* ... */
    if(r_scause() == 8){
        syscall();
    }
    /* ... */
    usertrapret();
}
```

### `kernel/syscall.c: syscall()`
想要直接跑到 `kernel/syscall.c: syscall()` 執行了話，可以使用
```gdb=
(gdb) break syscall
(gdb) continue
```
跳到 `syscall()` 執行

在 lab syscall 中有提到這段程式碼

### ```usertrapret()```
想要直接跑到 `kernel/syscall.c: syscall()` 執行了話，可以使用
```gdb=
(gdb) break usertrapret
(gdb) continue
```
跳到 `usertrapret()` 執行

`next` 到最後一行的 ```trampoline_userret``` 時，再 `next` 一次，之後又會進入到 assembly 的部份 (`kernel/trampoline.S: <userret>`)
```gdb=
(gdb) layout asm # 以 assembly 檢視
```

### ```kernel/trapmpoline.s``` 中的 ```userret```
在 ```sret``` 那裡，也要設 break point 才能夠回到 ```user/sh.asm``` 的 ```ecall``` 下面的 ```ret```

### ```user/sh.asm``` 的 ```ret```
```gdb=
(gdb) file user/_sh
(gdb) si
(gdb) layout src # 已經執行完 write(2, "$ ", 2);
```
整個流程到此結束

## 有用的指令
### QEMU
ctrl + a c 可以 (進入/退出)到 QEMU 的 console
* `info reg`: 可以印出所有的 register
* `info mem`: 可以印出所有的 page table
### gdb
* `p/x $satp` 印出目前 pagetable 的 **physicall** address
* `x/3i 0xe06` print 出 3 個 instructions
```gdb=
(gdb) x/3i 0xe06
=> 0xe06:       li      a7,16
   0xe08:       ecall
   0xe0c:       ret
```
* `stepi`： execute one instruction
* `print $pc` print program counter
* `print/x $stvec`: 得知 trap 發生時，會跑到什麼地方開始執行
* `print/x $sepc`: `ecall` 之後，紀錄 user mode 的下一個 instruction 的位置 (紀錄 `ecall` 的位置)
* 想要在 `user/sh.c` 的 main 中設定 break point
```gdb=
(gdb) symbol-file user/_sh
(gdb) b main
```
* 遇到 `fork()` 時，gdb 預設是繼續追 parent，如果要追 child, 可以用：
```gdb=
(gdb) info inferior
(gdb) inferior 2
```

```gdb=
(gdb) info threads
(gdb) thread $pid
```

```gdb=
set follow-fork-mode child
```
## 疑問
* 從 user mode 進入到 kernel mode 之後，用來指向 page table 的 register `satp` 有什麼變化？kernel 的 page table 有固定的位置碼？
    * `ecall` 之後，扔然使用了是同一個 page table
* kernel 可不可以同時有許多 process?
* ```kernel/vm.c``` 這個檔案在整個 system call 的過程中，起到了什麼樣的作用？
* kernel mode 使用的是 physical memory 嗎？

## 參考資料
* https://pdos.csail.mit.edu/6.S081/2022/schedule.html
* https://blog.csdn.net/dai_xiangjun/article/details/123967946
* https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec06-isolation-and-system-call-entry-exit-robert/6.4-ecall-zhi-ling-zhi-hou-de-zhuang-tai
* https://blog.csdn.net/dai_xiangjun/article/details/124098461
