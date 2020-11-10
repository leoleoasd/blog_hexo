---
title: PintOS 从编译到 Kernel Panic：在 CLion 里愉快的 Debug
date: 2020-11-08 18:40:24
tags:
	- MacOS
	- 环境配置
	- 操作系统
	- PintOS
---

1. MacOS：`brew install qemu i386-elf-gcc`

    Ubuntu: `apt install qemu gcc`

2. 用**最新的代码**： `git clone git://pintos-os.org/pintos-anon`

    或者下载我的修改版本： `git clone https://github.com/leoleoasd/pintos_configuration.git`，并跳过第 4，5 ，10步。

3. ```bash
    export GDBMACROS=$PINTOSHOME/src/misc/gdb-marcos
    export PATH=$PATH:$PINTOSHOME/src/utils
    ```

4. 修改 `src/Make.config`, 把`` ifneq (0, $(shell expr `uname -m` : '$(X86)')) `` 的 if 块替换为以下内容：

    ```bash
    ifeq (0, $(shell expr `uname` : 'Darwin'))
      ifneq (0, $(shell expr `uname -m` : '$(X86)'))
        CC = gcc
        LD = ld
        OBJCOPY = objcopy
      else
        ifneq (0, $(shell expr `uname -m` : '$(X86_64)'))
          CC = gcc -m32
          LD = ld -melf_i386
          OBJCOPY = objcopy
        else
          CC = i386-elf-gcc
          LD = i386-elf-ld
          OBJCOPY = i386-elf-objcopy
        endif
      endif
    else
      CC = x86_64-elf-gcc -m32
      LD = x86_64-elf-ld -m elf_i386
      OBJCOPY = x86_64-elf-objcopy
    endif
    ```

5. 按照[这里](http://courses.mpi-sws.org/os-ss13/assignments/pintos/pintos_12.html)： 

    > comment out `#include <stropts.h>` in both `squish-pty.c` and `squish-unix.c`, and comment out lines 288-293 in `squish-pty.c`.

6. `cd src/utils && make`

7. `cd src/threads && make && cd build && pintos --qemu --`

8. 在`src/threads`新建一个 bash 脚本，写入以下内容：

    ```bash
    #!/usr/bin/env bash
    make
    cd build || exit
    killall qemu-system-i386
    
    CMD="run alarm-signle"
    nohup bash -c "DISPLAY=window ../../utils/pintos --qemu --gdb -- $CMD > pintos.log" &
    echo "Done!"
    ```

9. 在项目根目录（`src`外面）新建一个`CMakeLists.txt`。**用于使用 clion 提供的代码提示功能， 不能用于编译代码。** 内容从 [这里](https://github.com/leoleoasd/pintos/blob/master/CMakeLists.txt) 复制。

10. 用 clion 打开项目。**如果之前打开过项目，删除`.idea`文件夹后再打开**。点击右上角的 `Add configurations`, 选择 `GDB Remote Debug`, 填入以下信息：

    ’target remote‘ args:  tcp:localhost:1234

    Symbol file: 选择你 `src/threads/build/kernel.o`

    Path mappings:

    ​	remote: `../../threads`

    ​	local:     `你的项目的完整路径/src/threads`

    Before launch: 添加一个`Run external tool`, 选择上面新建的 bash 脚本

11. `Cmd+D`即可调试。

​	

