---
title: Linux 身份认证系统 -- 从哪儿学起？
tags:
  - OS 
  - Linux
categories: Records
date: 2023-03-26 11:37:49
---

## 前言 

我一直对操作系统的内部实现非常感兴趣。最近，我在家里的内网部署了 [kanidm](https://github.com/kanidm/kanidm)，在过程中也对 Linux 的身份认证系统有了更多了解。因此，在此分享一下我的学习过程。

在本文的开头，我将讲述一小段我了解 `sudo` 是怎么工作的流程，这些内容以CC-BY-NC-SA 4.0 协议开源。后面的内容是我对于 [Firstyear 的这篇博文](https://fy.blackhats.net.au/blog/html/2022/08/24/where_to_start_with_linux_authentication.html) 的翻译。

## Forewords

I've always been curious about the internal implementations of operating systems. I recently deployed [kanidm](https://github.com/kanidm/kanidm) in my home lab, and I've leaned a lot about how does linux authentication works during this process. These paragraph is licensed under CC-BY-NC-SA 4.0. I'm writing this post to share my thoughts and share my translation of [this fantastic blog from Firstyear](https://fy.blackhats.net.au/blog/html/2022/08/24/where_to_start_with_linux_authentication.html) in Chinese.

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

----

以下内容翻译自 [Firstyear 的这篇博文](https://fy.blackhats.net.au/blog/html/2022/08/24/where_to_start_with_linux_authentication.html)。**以下内容的版权属于原作者**

The following content is  translation of [this fantastic blog from Firstyear](https://fy.blackhats.net.au/blog/html/2022/08/24/where_to_start_with_linux_authentication.html). **Copyright remains to original author.**

# Linux 身份认证系统 -- 从哪儿学起？

最近有一个人问我，应该如何学习 Linux 的身份认证系统的各个组件是如何组合到一起、如何通讯的。网上并没有很多有关这个话题的资料，所以我决定来写这篇博客。

## 你...是谁？

Linux 的身份中的第一个模块就是 NSS 或者 nsswitch （注意不要和密码学库中的 NSS 混淆）。 nsswitch（name service switch）在 glibc 中提供了一个获取 uid/gid 以及名字和账户详情的方法。nsswitch可以有很多个『模组』叠加在一起，对于每个请求，第一个相应的模组的答案会被返回。

一个 `nsswitch.conf` 的例子如下：

```
passwd: compat sss
group:  compat sss
shadow: compat sss

hosts:      files mdns dns
networks:   files dns

services:   files usrfiles
protocols:  files usrfiles
rpc:        files usrfiles
ethers:     files
netmasks:   files
netgroup:   files nis
publickey:  files

bootparams: files
automount:  files nis
aliases:    files
```

这个文件使用 `服务: 模组1 模组2...`的格式。一个简单的例子是：当一个程序使用`gethostbyname`方法来进行dns查询时，它访问`host`服务，先通过`files`模组解析`/etc/hosts`，再通过`mdns`模组（也叫做 `avahi`/`bonjour`），最后调用`dns`模组解析。

`passwd`/`group`/`shadow`是有关身份的三行。最常见的情况下，你会使用`files`模组。它会查询`/etc/passwd`和`/etc/shadow`来返回响应。`compat`模组和`files`类似，只是增加了对于NIS的兼容。另一个常见的模组是`sss`，它会访问 System Services Security Daemon (SSSD)。对于我自己（这里指源博文的作者）的IDM项目，我们使用`kanidm`这个 nsswitch 模组。

你可以用`getent`命令来测试 nsswitch 是如何解析身份的，比如：

```
# getent passwd william
william:x:654401105:654401105:William:/home/william:/bin/zsh
# getent passwd 654401105
william:x:654401105:654401105:William:/home/william:/bin/zsh

# getent group william
william:x:654401105:william
# getent group 654401105
william:x:654401105:william
```

注意到，uid 和名字都可以被用于获取身份。

这些模组都是动态链接库，你可以用以下命令找到他们：

```
# ls -al /usr/lib[64]/libnss_*
```

当一个进程想要通过nsswitch来解析什么东西时，它会调用glibc，glibc会在运行时加载这些动态链接库并运行他们。这就是通常你想要给某个发行版一个新的 nsswitch 模组时需要仔细审计的原因：这些模组可能会进到**每个进程的地址空间**！这也同时有一些安全上的影响，因为每个模组，在被每个进程加载的时候，都需要访问`/etc/passwd`或者访问网络来解析身份。有些模组（比如sss）改善了这一点，我们会在这个blog的后面部分讲到。

## 证明你自己

如果nsswitch回答了『你是谁』的问题，那么PAM（pluggable authentication modules，可插拔认证模组）就是『证明你自己』。PAM是真正做出检查你的密码等信息是合法的、检查你可以登录的模块。PAM通过有不同的服务来调用不同的模块工作。大多数的 Linux 发行版都有一个包括了所有的服务定义的 `/etc/pam.d` 文件夹（和Linux上不常用的`/etc/pam.conf`有一点点语法上的区别）。我们拿ssh举个例子：当你ssh到一台机器上的时候，ssh联系PAM并告诉它：我是ssh，你能帮我验证这个身份吗？

然后，PAM会读取`/etc/pam.d/服务名称`，在我们这个例子中是`/etc/pam.d/ssh`。以下是一个从Fedora中提取的例子（Fedora和RHEL都是非常常见的发行版；每个发行版都对这些配置文件有一些微调，这也让理解它们更困难）：

```
# cat /etc/pam.d/ssh
#%PAM-1.0
auth       include      system-auth
account    include      system-auth
password   include      system-auth
session    optional     pam_keyinit.so revoke
session    required     pam_limits.so
session    include      system-auth
```

注意 "include" 分别对于 auth, account, password, session 重复了四次。它们都引用 `system-auth`, 那么让我们来看看：

```
# cat /etc/pam.d/system-auth

auth        required                                     pam_env.so
auth        required                                     pam_faildelay.so delay=2000000
auth        [default=1 ignore=ignore success=ok]         pam_usertype.so isregular
auth        [default=1 ignore=ignore success=ok]         pam_localuser.so
auth        sufficient                                   pam_unix.so nullok
auth        [default=1 ignore=ignore success=ok]         pam_usertype.so isregular
auth        sufficient                                   pam_sss.so forward_pass
auth        required                                     pam_deny.so

account     required                                     pam_unix.so
account     sufficient                                   pam_localuser.so
account     sufficient                                   pam_usertype.so issystem
account     [default=bad success=ok user_unknown=ignore] pam_sss.so
account     required                                     pam_permit.so

session     optional                                     pam_keyinit.so revoke
session     required                                     pam_limits.so
-session    optional                                     pam_systemd.so
session     [success=1 default=ignore]                   pam_succeed_if.so service in crond quiet use_uid
session     required                                     pam_unix.so
session     optional                                     pam_sss.so

password    requisite                                    pam_pwquality.so local_users_only
password    sufficient                                   pam_unix.so yescrypt shadow nullok use_authtok
password    sufficient                                   pam_sss.so use_authtok
password    required                                     pam_deny.so
```

所以，首先我们在『验证』阶段。这个阶段是PAM依次访问验证模组来检查你的用户名和密码（或者其他形式，如TOTP等）直到返回成功。我们从 `pam_env.so` 开始，它返回『通过但未结束』，于是我们继续访问 `faildelay`，以此类推。这些模组一个一个被访问，它们的结果与前面的『规则』（ required/sufficient 或者自定义）来合并到一起，得到『成功，验证结束』、『成功但是继续验证』、『失败但是继续验证』、『失败但是结束』这四种结果。在这个例子中，能真正的验证用户的是 `pam_unix.so` 和 `pam_sss.so` 。所以，如果这两个都没有返回『成功，验证结束』，`pam_deny.so` 就会最终返回一个 『失败但是结束』。这个阶段只检查你的**等录凭据（密码等）**。

第二个阶段是『账户阶段』。其实它更应该被叫做『验证』阶段：这些模块被再次访问，来检查你的账户是否有权限访问这个服务。结果以类似的形式结合到一起。

第三个阶段是『会话阶段』。每个PAM模块可以影响和设置新创建的会话：一个简单的例子是 `pam_limit.so`，它负责设置新会话的 CPU /内存/文件描述符等限制。

第四个阶段是『密码阶段』。可能有点令人疑惑：这个阶段并不是用来验证身份的，而是在你运行`passwd`命令来修改这个密码的。每个模块依次被询问：你可以修改这个用户的密码吗？如果最终失败了，你会得到一个『authentication token manipulation error』，一般只是说『这个栈中的一些模块失败了，但是我不能告诉你是哪个』。

这些模块都是动态链接库，一般可以在 `/usr/lib64/security` 找到。就像 nsswitch 一样，使用pam的应用都链接到 `libpam.so`，它会在运行时加载 `/usr/lib64/security` 中的动态链接库。鉴于`/etc/shadow`只能被`root`用户读取，同时任何需要验证密码的东西都需要来读取这个文件，这基本意味着任何pam模块实际上在任何都运行在 `root` 的地址空间中。这就是发行版仔细审计和控制哪个模块可以添加一个pam模组的原因。同时，这也意味着进程很可能需要访问网络来调用远程的身份验证服务。

## 那么，网络验证呢？

现在，我们已经覆盖了进程和守护进程如何找到用户、验证凭据的基础。现在，我们来看看SSSD，一个解析身份的守护程序的实现。

正如之前提到的，nsswitch和pam都有让动态链接库在应用程序的上下文中运行的限制，这也通常意味着，在过去，`pam_ldap.so`可能在`root`的地址空间运行，同时需要访问网络的权限以及需要解析`asn.1`g格式（一个通常用于远程代码执行的库，也可以被用作编/解码二进制结构体）。

```
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
  root: uid 0             │
│                   │
                          │
│  ┌─────────────┐  │         ┌─────────────┐
   │             │        │   │             │
│  │             │  │         │             │
   │             │        │   │             │
│  │    SSHD     │──┼────────▶│    LDAP     │
   │             │        │   │             │
│  │             │  │         │             │
   │             │        │   │             │
│  └─────────────┘  │         └─────────────┘
                          │
│                   │ Network
                          │
└ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

SSSD改变了这一点：在本地运行一个可以通过unix socket访问的守护进程。这允许了pam和nsswitch模组id仅仅提供最小化的功能，仅仅负责联系一个独立的守护进程，而大部分工作都交给守护进程完成。这有非常非常多的安全性改善，包括不需要由root进程来解析网络上的不可信的数据。

```
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐      ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
  root: uid 0                sssd: uid 123            │
│                   │      │                   │
                                                      │
│  ┌─────────────┐  │      │  ┌─────────────┐  │          ┌─────────────┐
   │             │            │             │         │   │             │
│  │             │  │      │  │             │  │          │             │
   │             │            │             │         │   │             │
│  │    SSHD     │──┼──────┼─▶│    SSSD     │──┼─────────▶│    LDAP     │
   │             │            │             │         │   │             │
│  │             │  │      │  │             │  │          │             │
   │             │            │             │         │   │             │
│  └─────────────┘  │      │  └─────────────┘  │          └─────────────┘
                                                      │
│                   │      │                   │  Network
                                                      │
└ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘      └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

另一个很大的好处是SSSD现在可以安全的缓存网络服务的响应，允许用户在离线的时候继续解析身份。这甚至包括缓存密码！

这就是SSSD能在很多主流的发行版中都占据重要地位的原因。用一个很复杂的本地守护程序来完成真正的验证工作，和能使用很多不同的验证后端的能力，这使得它被广泛部署，并且会在基于网络的验证场景上替代`pam_ldap`和`pam_krb5`。

## 巨兽之内

SSSD内部由很多个互相协作的模块组成。知道它们是如何工作的能很好的帮助debug：

```
# /etc/sssd/sssd.conf

//change the log level of communication between the pam module and the sssd daemon
[pam]
debug_level = ...

// change the log level of communication between the nsswitch module and the sssd daemon
[nss]
debug_level = ...

// change the log level of processing the operations that relate to this authentication provider domain ```
[domain/AD]
debug_level = ...
```

现在我们知道了一个新的概念：一个SSSD domain。这和Active Directory中的doamin不同。一个SSSD domain仅仅是一个『验证服务提供者』。一个SSSD的实例可以同时从多个不同的验证服务提供者解析身份。但是在主流的设置中，一般都只使用一个domain。

在大部分情况下，如果你在使用SSSD的过程中遇到问题，错误应该都在domain部分，所以这永远应该是第一个被检查的地方，

每个domain都可以不同的服务商来完成『身份』、『验证』、『访问』、『修改密码』工作。比如：

```
[domain/default]
id_provider = ldap
auth_provider = ldap
access_provider = ldap
chpass_provider = ldap
```

`id_procider`负责解析名字和`uid/gid`到身份。

`auth_provider`负责验证密码。

`access_provider`负责判断这个身份是否有权限访问这台系统。

`chpass_provider`负责更改密码。

正如你可以看到的，在这个设计中有很大的灵活性：比如，你可以使用`krb5`来验证身份，但是使用`ldap`来修改密码。

正因为这个设计，SSD可以从很多个不同的身份源来验证身份，包括samba（ad）、ldap和kerberos。这意味着在某些受限的场景下，你可能需要具体身份来源的背景知识来解决SSSD的问题。

## 常见问题

### 性能

在某些情况下，SSSD在第一次访问时会非常慢，但是在登录完成后会变快。在某些情况下，你可能会在这个时候在LDAP服务器上看到很高的负载。这是用户和用户组解析的方式产生的问题：每当你需要解析一个用户的时候，你需要解析他所在的组；当解析这些组的时候，这些组又要加载它的全部成员......我希望你能看出来这是递归的。在最差的情况下，当一个用户登录的时候，整个LDAP/AD域都被枚举，在某些情况下可能要花几分钟。

如果要避免这一点，你可以设置：

```
ignore_group_members = False
```

这样能避免组加载他们的成员。这样，所有用户组看上去都是没有成员的，不过所有用户都会展示他们所在的用户组。鉴于绝大部分程序都使用『是xx的成员』这个模式，这样设置没有什么负面作用。

### 清除缓存

SSSD在本地缓存了网络服务的响应。他附带一个缓存管理工具： sss_cache。它允许标记记录为失效，这样sssd会尽快重新从网络加载。

这样有两个问题：在某些情况下，清除缓存看起来没有作用，失效的记录被继续使用；同时，sss_cache 工具的`-E`选项并不总是会使全部记录失效。

在这样的情况下，一个通常的建议是关闭sssd，删除`/var/lib/sss/db`文件夹内的所有东西（但是不要删掉文件夹）然后重启sssd。

### 调试 Kerberos

Kerberos的难以调试是臭名昭著的。这是因为它并没有一个真正的详细/调试模式，至少不显然。为了获取到调试输出，你需要设置一个环境变量：

```
KRB5_TRACE=/dev/stderr kinit user@domain
```

这个trick在**任何**链接到kerberos的进程都有效，所以它在389-ds, sssd, 和很多很多其他工具上都有效。你可以使用这个方法来追踪哪里出了问题。

## 总结

以上就是全部内容了，我可能会持续更新这个博文！

（翻译于2023-3-26，如果原文有更新，欢迎在评论区叫我更新翻译）