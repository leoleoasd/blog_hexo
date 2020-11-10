---
title: PintOS 笔记
date: 2020-11-05 18:40:24
tags:
	- OS 
	- 操作系统
	- PintOS
---

# 安装

1. 安装依赖：`sudo apt install qemu-system-i386`

2. 用**最新的代码**： `git clone git://pintos-os.org/pintos-anon`

3. ```bash
    export GDBMACROS=$PINTOSHOME/src/misc/gdb-marcos
    export PATH=$PATH:$PINTOSHOME/src/utils
    ```

4. `cd src/utils && make`

5. 按照[这里](http://courses.mpi-sws.org/os-ss13/assignments/pintos/pintos_12.html)： 

    > comment out `#include <stropts.h>` in both `squish-pty.c` and `squish-unix.c`, and comment out lines 288-293 in `squish-pty.c`.

6. `pintos --qemu --`



​	