---
title: Linux 身份认证系统 -- 从哪儿学起？
tags:
	- OS 
	- 操作系统
---

## 前言 

我一直对操作系统的内部实现非常感兴趣。最近，我在家里的内网部署了 [kanidm](https://github.com/kanidm/kanidm)，在过程中也对
Linux 的身份认证系统有了更多了解。因此，在此分享一下我的学习过程。

在本文的开头，我将讲述一小段我了解 `sudo` 是怎么工作的流程，以及一些想法。后面的内容是我对于 [Firstyear 的这篇博文](https://fy.blackhats.net.au/blog/html/2022/08/24/where_to_start_with_linux_authentication.html) 的翻译。

## Forewords

I've always been curious about the internal implementations of operating systems. I recently deployed [kanidm](https://github.com/kanidm/kanidm) in my home lab, and I've leaned a lot about how does linux authentication works during this process. I'm writing this post to share my thoughts and share my translation of [this fantastic blog from Firstyear](https://fy.blackhats.net.au/blog/html/2022/08/24/where_to_start_with_linux_authentication.html) in Chinese.

## SUDO 是怎么工作的？

在我学习操作系统课程的时候，我突然想到，『sudo是怎么工作的呢？』在当时，我用我贫瘠的知识做出了以下假设：

- 应该有一个用于提权的系统调用
- `sudo`程序在用户态收集用户的密码，并调用这个系统调用提升到root

但是，仔细一想便会发现，这个假设有很大的问题：

- `sudo`接受的是当前用户的密码，而不是`root`的密码，而哪些用户可以执行`sudo`是在`sudoers`配置文件里的
  - 所以，这个『用于提权的系统调用』应该需要知道哪些用户可以执行`sudo`
  - 但是不太可能是读那个配置文件？
- 同时，`sudo`还支持更细颗粒度的权限限制，比如要求某个用户只能以`root`执行某个特定的指令。

经过一番搜索，我找到了 `setuid` 这个系统调用，发现它可以设置当前进程的uid。进一步搜索，发现这个系统调用只能用于
『降低权限』：有 `CAP_SETUID` 能力的进程才可以调用。那么，在`sudo`这种场景下，权限的『提升』是什么时候完成的呢？

经过我的尝试，我发现在调用`sudo`输入密码之前，`sudo`这个进程的`uid`就已经是0了，说明是`sudo`这个二进制程序本身有
特殊的权限，让他能直接提升到`root`。思考到了这一步，结论就十分显然：这个特殊的权限只能是存在文件系统里的。经过一番搜索，我发现文件系统除了读写执行等权限之外，还维护了几个特殊的权限：`sudo`用的是一个叫做`setuid`的权限：

> The Unix access rights flags setuid and setgid (short for set user identity and set group identity) allow users to run an executable with the file system permissions of the executable's owner or group respectively and to change behaviour in directories.

这样，任何用户在运行 `sudo` 的时候，权限都会临时的提升到 `root`（也就是 `sudo` 这个文件的 `owner`）。 `sudo` 自然可以自己判断用户是否有权限保留 `root` 权限，或者切换到其他用户。这也是为什么如果对 `/usr/bin` 运行 `chmod -R 755` 会导致 `sudo` 没法用了的原因： `setuid` 权限被清除了。

这就是我对 Linux 身份认证系统最初的理解的来源。

## How does sudo works?

While I was learning Operating Systems at my school, it suddenly occurred to me that I don't know how `sudo` works. Based on my little knowledge of Linux, I made the following assumption:

- There should be a syscall for elevating privileges
- `sudo` collects user's password, and pass it to the syscall

However, almost immediately, I found some problems of my assumption:

- `sudo` asks for the password of the current user, not the root user; and who is allowed to `sudo` is stored in `sudoers`
  - So the syscall for elevating privileges should be able to know who can do that
  - But it is unlikely that the syscall reads the `sudoers` file
- `sudo` supports controlling which command a user is allowed to run
  - This is too complicated to be integrated into the kernel.

After some digging, I found the syscall `setuid`. It is able to set the `uid` of the current process. After some further searching, I found that this syscall is only for lowering the permissions: You have to have `CAP_SETUID` to run it. Then, in a scenario like `sudo`, when does the permission "elevation" happens?

After some trial, I discovered that when `sudo` is asking for password, the `uid` of it's process is already 0, which means that the binary file `sudo` have something unique which makes it can directly be run as `root`. At this point, the answer is quite obvious: the only reasonable answer is that the permission is stored in the file system. Again, after some searching, I found that the file system maintains some special permission other than regular `rwx`. `sudo` uses `setuid`:

> The Unix access rights flags setuid and setgid (short for set user identity and set group identity) allow users to run an executable with the file system permissions of the executable's owner or group respectively and to change behaviour in directories.

So, the permission is temporarily elevated to `root` (who is the owner of `sudo`). `sudo` itself can determin whether the user is able to maintain the `root` permission or switch to other users. That is also why `sudo` breaks if you run `chmod -R 755 /usr/bin`.

This my first understanding of linux authentication system.

