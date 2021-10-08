---
title: PintOS 从编译到 Kernel Panic：在 CLion 里愉快的 Debug
date: 2020-11-08 18:40:24
tags:
	- MacOS
	- 环境配置
	- 操作系统
	- PintOS
---

注：流程支持M1的mac、Intel的Mac以及Ubuntu。在两种Mac上进行过测试，**没有在Ubuntu**上测试过，不过大体应该差不多。

1. MacOS：`brew install qemu x86_64-elf-gcc x86_64-elf-binutils`

    Ubuntu: `apt install qemu gcc`

2. **最新的代码**在： `git clone git://pintos-os.org/pintos-anon`，但是建议使用我的修改版本： `git clone https://github.com/leoleoasd/pintos_configuration.git`。

3. 修改`~/.bashrc`（如果`echo $0`输出了`-bash`）或者`~/.zshrc`（如果`echo $0`输出了`-zsh`），增加以下内容到文件末尾：
    ```bash
    export GDBMACROS=$PINTOSHOME/src/misc/gdb-marcos
    export PATH=$PATH:$PINTOSHOME/src/utils
    ```

4. `cd src/utils && make`

5. `cd src/threads && make && cd build && pintos --qemu --`

6.  在`src/threads`新建一个 bash 脚本，写入以下内容：

    ```bash
    #!/usr/bin/env bash
    make
    cd build || exit
    killall qemu-system-i386 # 杀掉上一次打开的QEMU。Clion不能在停止Debug的时候关闭QEMU。
    
    CMD="run alarm-signle" # 你要运行的指令。
    nohup bash -c "DISPLAY=window ../../utils/pintos --qemu --gdb -- $CMD > pintos.log" &
    echo "Done!"
    ```

7.  用 clion 打开项目。**如果之前打开过项目，删除`.idea`文件夹后再打开**。点击右上角的 `Add configurations`, 选择 `GDB Remote Debug`, 填入以下信息：

    ’target remote‘ args:  tcp:localhost:1234

    Symbol file: 选择你 `src/threads/build/kernel.o`

    Path mappings:

    ​	remote: `../../threads`

    ​	local:     `你的项目的完整路径/src/threads`

    Before launch: 添加一个`Run external tool`, 选择上面新建的 bash 脚本

8.  `Cmd+D` or `Ctrl+D`即可调试。

​	

