+++
date = '2025-12-03T20:16:29+08:00'
draft = false
title = 'Ch02: Building the 5.x Linux Kernel from Source - Part 1'
+++

# Step 1 – obtaining a Linux kernel source tree
```sh
wget --https-only -O ~/Downloads/linux-5.4.1.tar.xz https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.4.1.tar.xz
```

# Step 2 – extracting the kernel source tree
```sh
cd ~/Downloads ; ls -lh linux-5.4.1.tar.xz
```

```sh
tar xf ~/Downloads/linux-5.4.1.tar.xz
```

```sh
mkdir -p ~/kernels
tar xf ~/Downloads/linux-5.4.1.tar.xz --directory=${HOME}/kernels/
```

設定環境變數是一個 good practice，可加入 `~/.bashrc` 中:
```sh
export LLKD_KSRC=${HOME}/kernels/linux-5.4
```

* 現在已經不像以前一定會把 source code 放到 `usr/src` 了
## A brief tour of the kernel source tree
```sh
user@ubuntu:~/kernels/linux-5.4$ ls
COPYING        Kconfig      README  crypto   init    mm       security  virt
CREDITS        LICENSES     arch    drivers  ipc     net      sound
Documentation  MAINTAINERS  block   fs       kernel  samples  tools
Kbuild         Makefile     certs   include  lib     scripts  usr
```

* 使用 `du -m .` 可以看到檔案大小為 `1011M` 幾乎是 1 GB 了

```sh
user@ubuntu:~/kernels/linux-5.4$ head Makefile
# SPDX-License-Identifier: GPL-2.0
VERSION = 5
PATCHLEVEL = 4
SUBLEVEL = 1
EXTRAVERSION =
NAME = Kleptomaniac Octopus

# *DOCUMENTATION*
# To see a list of typical targets execute "make help"
# More info can be located in ./README
```
* 這裡可以知道目前的 source code 版本為 5.4.1

### 最上層的檔案與資料夾
* `README`: 告知 documents 的所在位置
* `COPYING`: licence 的所在位置
* `MAINTAINERS`

#### Major subsystem directories
* `Makefile`: 這是 top-level 的 `Makefile` 使用 `kbuild` 與 kernel modules 使用這個 `Makefile` (至少在初始化階段)
* `kernel`: Core kernel subsystem
    * process/thread life cycle
    * CPU scheduling
    * locking
    * cgroups,
    * timers
    * interrupts
    * signaling
    * modules
    * tracing
* `mm`: memory management
* `fs`: virtual file system
* `block`: block I/O, file system 的更底層
* `net`: network
* `ipc`: inter-processes communication
* `sound`: Advanced Linux Sound Architecture (ALSA)
* `virt`: virtualization code like KVM

#### Infrastructure/misc
* `arch`: architecture 相關，像是 x86-64, risc-v, arm 等等
* `crypto`: 加密相關
* `include`: arch-independent kernel headers, 但也有例外，像是 `arch/<cpu>/include/...`
* `init`: 初始化的過程 `init/main.c:start_kernel()`, `start_kernel()`
* `lib`: 注意 kernel 並不支援 shared library
* `scripts`: 一些有用的 scripts
* `security`: Linux Security Module (LSM)
* `tools`: mostly userspace applications

```sh
user@ubuntu:~/kernels/linux-5.4$ cd ${LLKD_KSRC} ; ls arch/
Kconfig  arm    csky     ia64        mips   openrisc  riscv  sparc      x86
alpha    arm64  h8300    m68k        nds32  parisc    s390   um         xtensa
arc      c6x    hexagon  microblaze  nios2  powerpc   sh     unicore32
```
kernel 所支援的 ISA

# Step 3 – configuring the Linux kernel
# Customizing the kernel menu – adding our own menu item
