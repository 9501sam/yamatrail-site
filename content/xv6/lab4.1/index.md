+++
date = '2025-10-09T19:09:13+08:00'
draft = false
title = '[xv6 學習紀錄 04-1] 使用 GDB 追蹤 user program 以了解 RISC-V 的 Calling Convention 與 Stack Frames'
series = ["xv6 學習紀錄"]
weight = 41
+++

課程影片連結：
* [6.S081 Fall 2020 Lecture 5: RISC-V Calling Convention and Stack Frames ](https://www.youtube.com/watch?v=s-Z5t_yTyTM)

這篇文章目的在於整理這個影片的重點，主要有以下的重點
* 如何用 `GDB` 追蹤 user program
* RISC-V 的 Calling Convention
    * RISC-V 的 register (Caller and Callee saved)
    * RISC-V 的 Stack Frame

* `kernel/defs.h`
```c
// demos.c
void            demo1(void);
void            demo2(void);
void            demo3(void);
void            demo4(void);
void            demo5(void);
void            demo6(void);
void            demo7(void);

// asmdemo.S
int             sum_to(int);
int             sum_then_double(int);
```

* `kernel/demos.c`
```c
#include "types.h"
#include "riscv.h"
#include "defs.h"
#include "date.h"
#include "param.h"
#include "memlayout.h"
#include "spinlock.h"
#include "proc.h"
#include "sysinfo.h"

void demo1()
{
	printf("Result: %d\n", sum_to(5));
}

void demo2()
{
	printf("Tesult: %d\n", sum_ten_double(5));
}

void _demo3(char a, char b, char c, char d, char e, char f, char g, char h, char i, char j)
{
	printf("%d, %d, %d, %d, %d, %d, %d, %d, %d, %d\n", a, b, c, d, e, f, g, h, i, j);
}

void demo3()
{
	_demo3('a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j');
}

int dummymain(int argc, char *argv[])
{
	int i = 0;
	for (; i < argc; i++)
	{
		printf("Argument %d: %s\n", i, argv[i]);
	}
	return 0;
}

void demo4()
{
	char *args[] = {"foo", "bar", "baz"};
	int result = dummymain(sizeof(args)/sizeof(args[0]), args);
	if (result < 0)
		panic("Demo 4");
}

int _dummy(int n)
{
	int div = 5;
	int sum = 0;
	for (int i=0; i < n; i++)
	{
		if (i % div == 0) sum += i;
	}
	return sum;
}

void demo5()
{
	printf("Demo\n");
	int top = 50;
	int result = _dummy(top);
	printf("Result: %d\n", result);
}

void demo6()
{
	int sum = 0;
	for (int i = 1; i < 10; i++)
	{
		sum += 3 *i;
		printf("%d\n", sum_to(i));
	}
	printf("Sum: %d\n", sum);
}

struct Person {
	int id;
	int age;
	// char *name;
};

void printPerson(struct Person *p)
{
	printf("Person %d (%d)\n", p->id, p->age);
	// printf("Name:%s: %d (%d)\n", p->name, p->id, p->age);
}

void demo7()
{
	// Structs
	struct Person p;
	p.id = 1215;
	p.age = 22;
	// p.name = "Nick";
	printPerson(&p);
}
```

* `kernel/asmdemo.S`
```asm
```

* `Makefile`
```makefile
OBJS = \
  $K/entry.o \
  $K/kalloc.o \
  $K/string.o \
  ...
  $K/plic.o \
  $K/virtio_disk.o \
  $K/demos.o \
  $K/asmdemo.o
```

#### 新增 system call `demo()`
* `user/usys.pl` 加上 `entry(demo);`
* `user/user.h` 加上 `int demo(int);`
* `kernel/syscall.h` 加上 `#define SYS_demo 22`
* `kernel/syscall.c` 加上
    * `extern uint64 sys_demo(void);`
    * `[SYS_demo]    sys_demo,`

* `kernel/sysproc.c`: 實做 `sys_demo`
```c
uint64
sys_demo(void)
{
  int n;

  argint(0, &n);
  if (1 <= n && n <= 7) {
    switch(n) {
    case 1:
      demo1();
      break;
    case 2:
      demo2();
      break;
    case 3:
      demo3();
      break;
    case 4:
      demo4();
      break;
    case 5:
      demo5();
      break;
    case 6:
      demo6();
      break;
    case 7:
      demo7();
      break;
    }
  } else {
    return -1;
  }
  return 0;
}
```

* `user/demo.c`: 使用 system call `demo()`
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int n;

  if (argc != 2) {
    fprintf(2, "usage: demo <1 ~ 7>\n");
    exit(1);
  }

  n = argv[1][0] - '0';
  if (1 <= n && n <= 7) {
    demo(n);
  } else {
    fprintf(2, "usage: demo <1 ~ 7>\n");
    exit(1);
  }
  exit(0);
}
```



## 從 C language 到 Assembly, 以及用 `GDB` 追蹤
## Caller Saved 與 Callee Saved Registers
## Stack Frames
## 如果打破 Convention 會發生什麼事情
## 好用的 `GDB` 指令
### break at `user/demo.c`
```gdb
(gdb) add-symbol-file user/_demo
```
```gdb
(gdb) b user/demo.c:10
```

### 執行一行 c code
```gdb
next
```

## 附錄：`GDB` Cheat List
* `delete`: delete all the breakpoints

```gdb
p *argv@argc
```
