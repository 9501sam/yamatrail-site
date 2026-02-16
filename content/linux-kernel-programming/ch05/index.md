+++
date = '2025-12-25T10:07:49+08:00'
draft = false
title = 'Ch05: Writing Your First Kernel Module - LKMs Part 2'
weight = 5
+++

# Cross-compiling a kernel module
在 ch4 時，我們 compile 了一個 kernel module, 這次則是要 cross-compile 一個 kernel module

## Attempt 1 – setting the "special" environment variables
先試著設定 `ARCH` 與 `CROSS_COMPILE`
```sh
cd ~/Linux-Kernel-Programming/ch5/cross
```

```sh
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
```

![](206.png)

但這個嘗試失敗了，這是因為在這個例子中，是因為 `Makefile` 會去找目前這台電腦的 Kernel source，因此要來對 `Makefile` 做下列的修改

```makefile
ifeq ($(ARCH),arm)
  # *UPDATE* 'KDIR' below to point to the ARM Linux kernel source tree on your box
  KDIR ?= /home/user/rpi_work/kernel_rpi/linux
else ifeq ($(ARCH),arm64)
  # *UPDATE* 'KDIR' below to point to the ARM64 (Aarch64) Linux kernel source
  # tree on your box
  KDIR ?= /home/user/rpi_work/kernel_rpi/linux
else ifeq ($(ARCH),powerpc)
  # *UPDATE* 'KDIR' below to point to the PPC64 Linux kernel source tree on your box
  KDIR ?= ~/kernel/linux-4.9.1
else
  # 'KDIR' is the Linux 'kernel headers' package on your host system; this is
  # usually an x86_64, but could be anything, really (f.e. building directly
  # on a Raspberry Pi implies that it's the host)
  KDIR ?= /lib/modules/$(shell uname -r)/build
endif
```

這裡主要是把 kernel source 的路徑改為正確的 `/home/user/rpi_work/kernel_rpi/linux`

## Attempt 2 – pointing the Makefile to the correct kernel source tree for the target

修正路徑之後，再一次嘗試 cross compile

```sh
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
```

![](209.png)

這是因為現在的 `rpi_work/kernel_rpi` 還是一個 "virgin" state, 它還沒有獲得一個 `.config` 的設定檔

```sh
cd ~/rpi_work/kernel_rpi/linux
```

* 書上用的是 `bcmrpi_defconfig` 但我用的是 64-bit 的 raspberry pi 4 所以要用 `bcm2711_defconfig`
```sh
make ARCH=arm64 bcm2711_defconfig
```
![](210.png)

```sh
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- oldconfig
```
![](210-old.png)

到了這裡，已經產生出一個 `.config` 了

```sh
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- prepare
```
![](210-prepare.png)

```sh
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- 
```
![](210-make.png)
(這個會跑一陣子)
這個指令會產生出以下的檔案
* `arch/arm64/boot/Image`: 未壓縮的 Kernel Image
* `arch/arm64/boot/dts/broadcom/*.dtb`: Device Tree Blobs
* `arch/arm64/boot/dts/overlays/*.dtbo`: Device Tree Overlays

這裡的流程跟 ch03 在 corss compile pi 的 kernel 時有一點類似

## Attempt 3 – cross-compiling our kernel module
現在 kernel 已經 build 好了，現在就可以再重新的 build 一次
```sh
cd /home/user/Linux-Kernel-Programming/ch5/cross
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

現在 `make` 時，就會搭配上個步驟產生出的 `.config` 以及上個步驟 `make` 出來的東西產生出我們需要的 `.ko` file

![](211.png)
現在這個 `./helloworld_lkm.ko` 出現了！

接著把檔案複製到 pi 中
```sh
scp ./helloworld_lkm.ko user@192.168.100.104:/home/user
```

```sh
sudo insmod ./helloworld_lkm.ko
```
![](212.png)

這是因為目前 pi 上跑的 kernel version 跟這個 moduel 所對應的 kernel version 不一樣
```sh
cat /proc/version
modinfo ./helloworld_lkm.ko
```
![](213.png)

這讓我們學到了 module 只被允許被 insert 到它所對應的 kernel 版本，在這裡來回想一下 module 的 kernel version 跟我們現在 pi 上的 kernel version 是怎麼來的

1. module 的 kernel version:
    1. `Makefile` 中指定 kernel source 路徑為 `~/rpi_work/kernel_rpi/linux`
    1. 在 `~/rpi_work/kernel_rpi/linux` 裡我們針對這個 kernel version 做 config 以及 compile
    1. 回到 `/home/user/Linux-Kernel-Programming/ch5/cross` 來 `make` 出 `.ko` 時，它所搭配的 kernel version 就被前兩個步驟決定好了
1. pi 上的 kernel version
    * 這是單純使用 imager 工具所提供的 Pi OS 版本

## Attempt 4 – cross-compiling our kernel module
為了解決 Attempt 3 所留下的問題，有兩種解決方式，要麼 module 去搭配現在 pi 上運行的 kernel version，要麼 kernel 去搭配現在要使用的 module 的版本，以現在的環境來說，後者比較方便一些

這裡基本上只需要用 ch03 的方式去更改 pi 中的 kernel version 就可以了

```sh
cat /proc/version
modinfo ./helloworld_lkm.ko
sudo insmod ./helloworld_lkm.ko
dmesg | tail -n 5
sudo rmmod helloworld_lkm 2>/dev/null
dmesg | tail -n 5
```
![216.png](216.png)
到了這裡，我們已經可以 cross compile 一個 module 到手上的 pi 上了!

# Gathering minimal system information
這裡要開始蒐集一些系統上的資訊並且作為 log 輸出，書上提供範例 `ch5/min_sysinfo`
```sh
cd ch5/min_sysinfo
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

這裡我遇到了這個問題
```sh
user@raspberrypi:~ $ sudo insmod ./min_sysinfo.ko
insmod: ERROR: could not insert module ./min_sysinfo.ko: Unknown symbol in module
```
解決方式為在 `Makefile` 中加入
```makefile
EXTRA_CFLAGS   += -DDEBUG -fno-stack-protector
```

產生出 `min_sysinfo.ko` 之後，複製到 pi 上，並且在 pi 上
```sh
sudo insmod ./min_sysinfo.ko
dmesg
```
![219.png](219.png)

除了用 cross compile 的方式跑在 arm64 上，這個 `min_sysinfo.c` 也可以為了 x86 編譯
![219x86.png](219x86.png)

這個範例主要顯現了同一個 `.c` file 可以經由編譯程序上的調整達到 portability

## Being a bit more security-aware

# Emulating "library-like" features for kernel modules
實際上不是 Library，但是有一些技巧可以做到 Library-like 的事情
## Performing library emulation via multiple source files
## Understanding function and variable scope in a kernel module
## Understanding module stacking
### Trying out module stacking
