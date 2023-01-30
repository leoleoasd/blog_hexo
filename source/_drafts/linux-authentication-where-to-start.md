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

## SUDO 是怎么工作的？ / How does sudo works?

在我学习操作系统课程的时候，我突然想到，『sudo是怎么工作的呢？』在当时，我用我贫瘠的知识做出了以下假设：

- 应该有一个用于提权的系统调用
- `sudo`程序在用户态收集用户的密码，并调用这个系统调用提升到root

但是，仔细一想便会发现，这个假设有很大的问题：

- 在输入密码的时候，一个非`root`的进程的内存里有密码应该是个非常不安全的行为
- `sudo`接受的是当前用户的密码，而不是`root`的密码，而哪些用户可以执行`sudo`是在`sudoers`配置文件里的
  - 所以，这个『用于提权的系统调用』应该需要知道哪些用户可以执行`sudo`
  - 但是不太可能是读那个配置文件？

经过一番搜索，我找到了 `setuid` 这个系统调用，发现它可以设置当前进程的uid。