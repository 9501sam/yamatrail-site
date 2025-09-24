+++
date = '2025-09-24T16:13:25+08:00'
draft = false
title = '[xv6 學習紀錄 01] Lab: Xv6 and Unix utilities'
series = ["xv6 學習紀錄"]
weight = 1
+++
Lab 連結: [Lab: Xv6 and Unix utilities](https://pdos.csail.mit.edu/6.S081/2022/labs/util.html)

## Boot xv6(Easy)
這部份的詳細內容都寫在 [lab util](https://pdos.csail.mit.edu/6.S081/2022/labs/util.html) 中，會需要一個 linux 系統（windows使用者可以用虛擬機），Xv6 會跑在 linux 所架設的虛擬機上。

1. 下載原始碼
```sh
git clone git://g.csail.mit.edu/xv6-labs-2022
```
```sh
cd xv6-labs-2022
```

2. 安裝架設虛擬機的套件
我自己是用 Debian，如果你用的是 ubuntu 的話下載步驟應該也是一樣的，至於是其他系統的使用者，可以看[這裡](https://pdos.csail.mit.edu/6.S081/2022/tools.html)
```sh
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu 
```

3. compile程式碼並且讓他跑在虛擬機上
```sh
$ make qemu
...
(一大串訊息)
...
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$
```
到這裡，Xv6已經成功開機了！

4. 嘗試打個指令
```sh
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2059
xargstest.sh   2 3 93
cat            2 4 24120
echo           2 5 22944
forktest       2 6 13184
grep           2 7 27424
init           2 8 23680
kill           2 9 22904
ln             2 10 22744
ls             2 11 26312
mkdir          2 12 23040
rm             2 13 23032
sh             2 14 41856
stressfs       2 15 23904
usertests      2 16 148312
grind          2 17 38008
wc             2 18 25232
zombie         2 19 22280
console        3 20 0
```
沒意外的話，會出現以上的畫面

5. 離開虛擬機
按下```ctrl+a```放開這兩個鍵之後，再按下```x```
```sh
$ QEMU: Terminated
```
這樣就關機了

## sleep (easy)
> Implement the UNIX program sleep for xv6; your sleep should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file user/sleep.c. 

這題的大意是在說把 `sleep` 實做出來，主要是想要讓我們知道這些 program 是如何使用 system call 的

### 題目分析
觀察其他在 `user/` 中的程式如何使用 system call
* `user/echo.c`:
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"    // 等等可以先來看看這裡 include 了些什麼東西

int
main(int argc, char *argv[])
{
  int i;

  for(i = 1; i < argc; i++){
    write(1, argv[i], strlen(argv[i])); // 像是這裡
    if(i + 1 < argc){
      write(1, " ", 1);
    } else {
      write(1, "\n", 1);
    }
  }
  exit(0);
}
```

* `user/user.h`
```c
struct stat;

// system calls
int fork(void);
int exit(int) __attribute__((noreturn));
int wait(int*);
int pipe(int*);
int write(int, const void*, int);
int read(int, void*, int);
int close(int);
int kill(int);
int exec(const char*, char**);
int open(const char*, int);
int mknod(const char*, short, short);
int unlink(const char*);
int fstat(int fd, struct stat*);
int link(const char*, const char*);
int mkdir(const char*);
int chdir(const char*);
int dup(int);
int getpid(void);
char* sbrk(int);
int sleep(int);  // 我們可以使用這個呼叫 system call
int uptime(void);

// ulib.c
int stat(const char*, struct stat*);
char* strcpy(char*, const char*);
void *memmove(void*, const void*, int);
char* strchr(const char*, char c);
int strcmp(const char*, const char*);
void fprintf(int, const char*, ...);
void printf(const char*, ...);
char* gets(char*, int max);
uint strlen(const char*);
void* memset(void*, int, uint);
void* malloc(uint);
void free(void*);
int atoi(const char*); // 這是個好用的 function (把 string 轉成 int) 等等會用到
int memcmp(const void *, const void *, uint);
void *memcpy(void *, const void *, uint);
```
首先我們先知道了 program `echo` 的檔案名稱為 `user/echo.c`，可以想見等等我們的 `sleep` 會被命名為 `user/sleep.c`。

透過觀察這個檔案，我們可以了解到 `user/echo.c` 是如何使用 system call 的，他利用 `user/user.h` 所宣告的 function 使用 system call, 像是 `user/echo.c` 使用了 `write()`，等等我們的 `user/sleep.c` 也可以使用 `user/user.h` 所宣告的 `sleep()`。

至於 `sleep()` 實際上實做在哪裡？  
`sleep()` 實做在 `kernel/sysproc.c` 跳到這裡的機制在之後的 Lab2 會觀察到，這裡我們可以先看一下實做的地方就好

```c
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  argint(0, &n);
  acquire(&tickslock);
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    if(killed(myproc())){
      release(&tickslock);
      return -1;
    }
    sleep(&ticks, &tickslock);
  }
  release(&tickslock);
  return 0;
}
```

有了 `sleep()` 可以使用，接下來只剩下如何把 command-line argument (`string`) 轉換成 `int`，如同先前在 `user/user.h` 中所看見的，我們有 `atoi()` 可以使用，使用的例子在 `user/kill.c` 中有出現過。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char **argv)
{
  int i;

  if(argc < 2){
    fprintf(2, "usage: kill pid...\n");
    exit(1);
  }
  for(i=1; i<argc; i++)
    kill(atoi(argv[i]));
  exit(0);
}
```

### 程式實做
有了前面的分析，就可以拼湊出我們要的程式了：
* `user/sleep.c`:
```c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  if (argc < 2)
    fprintf(2, "Usage: sleep <ticks>\n");
  sleep(atoi(argv[0]));
  exit(0);
}
```

* `Makefile`
```makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_sleep\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
```
接著用 `./grade-lab-util` 驗證就行了
```sh
./grade-lab-util           (base)  2025-09-24 19:17:06
make: 'kernel/kernel' is up to date.
== Test sleep, no arguments == sleep, no arguments: OK (1.2s)
== Test sleep, returns == sleep, returns: OK (1.1s)
== Test sleep, makes syscall == sleep, makes syscall: OK (1.0s)
```
