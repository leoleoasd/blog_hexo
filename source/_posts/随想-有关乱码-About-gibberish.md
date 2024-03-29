---
title: 随想 | 有关乱码 | About gibberish
date: 2021-11-29 10:40:12
tags:
  - 编程
categories: 教学
---

# 随想 有关乱码

首先给出几种常见『乱码』：

```
md5 378e3ce9f0c8243012cb32cedde1ad31
sha1 b4d254b6620924a05e95bf76f5dace64edcf9086
sha256 3e9818cc4bf74e65419e72f89104dca674dcb215c1487dd25fed541cd6363d72
base32 MNUGC3THNJUWC3TMOVQW43LBBI======
base64 Y2hhbmdqaWFubHVhbm1hCg==
密码哈希函数 $2y$10$P5DEiAiuSr0XT9085ioTQeIW9QrrnGEkSvdJSPmlGDnvcANlvFDwm
```

本文的重点不在于怎么判断常见乱码，但是还是简单介绍一下：

- 用`$$`分割的，可能是密码哈希函数的输出或者是`JWT`
- 后面可能有`=`，内容是字母和`+/`或者`-_`，可能是`base64`系列。如果只有大、小写字母，可能是`base32`。如果有`+/`，是传统`base64 or base32`。如果有`-_`，是urlsafe的`base64 or base32`。
- base64解码后的内容如果不是utf-8，可能是哈希函数的输出。看长度（多少bit）
- hex看长度。常见的md5和sha系列哈希的长度。

# 正视『乱码』

之前一直觉得『乱码』这个词被滥用了。

遇到一串自己看不懂的东西，就把他称为『乱码』。我一直觉得这个词不太合适，因为这串东西在别人眼里很可能是能看懂的，有意义的。但是，我本人一直也找不到一个非常恰当的称呼，因此一直没有在意。

直到某次给Golang提issue的时候，Google的工作人员分析我的程序，看到一串『乱码』。他是这么描述的：

![image-20211129110312524](/medias/image-20211129110312524.png)

『这个字符串**在我来看**是乱码，你知道是什么吗？』

确实，用『我看来是乱码』一词形容无疑更为准确。由于对密码哈希函数的不熟悉（也十分正常），看不出这串字符串是什么，但是无疑，它是有意义的，不是无意义的。

为什么今天突然想到写这个东西呢？总有一些人，认为这串东西自己看不懂就没有意义。有一次和别人一起调试一个程序的时候，他描述我的程序『输出了乱码』。在我反复多次强调『让我看看输出』之后，仍然坚持认为乱码没有意义，没有发给我看。实际上，那段程序输出的就是形如`$2y$10$P5DEi...`的哈希值。

所以，当别人的程序输出了一些我们看不懂的东西的时候，不要只描述为『输出乱码』，请**复制乱码内容**，一起询问可能能看出这是什么的人。
