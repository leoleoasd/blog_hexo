---
title: Mac上Docker的一些坑
date: 2019-06-30 16:37:19
tags: 
    - MAC
    - Docker
categories: Records
top: true
---

在搭建我的个人开发环境的过程中, 对于PHP开发我选择了Docker这样的方案. 这种方案相比valet最大的好处就在于自由: 可以自由定制自己所需的nginx配置, php配置, 在安装php插件时也会方便一些.
但是, 在搭建完毕运行之后, 碰到了许多坑.
我最先搭建的是WHMCS测试站. 搭建完毕之后发现无论打开什么页面, 最短的加载时间也在5s左右. 对于搭建在本地的whmcs来讲, 这自然是十分不正常的, 百度了一下如何解决之后, 我决定开启php-fpm的slowlog来看看是哪里出了问题.
在docker中开启slowlog之后, 超时时间设置为了1s, 但是当页面加载时间为5s时, log文件内仍无任何内容.
Google了一番, 发现php-fpm使用SYS_PTRACE这一个系统调用来统计程序运行的时间, 而默认情况下docker容器内并没有此权限, 只要在docker-compose.yml内加入 `privileged: true` 即可解决问题.
这就是第一个坑: `PHP-fpm在无任何错误提示的时候不能产出slowlog`

折腾了半天终于能看到slowlog了, 内容却让我很疑惑: 
```log
[30-Jun-2019 15:33:20]  [pool www] pid 10
script_filename = /data/www/whmcs//admin/login.php
[0x00007f063aa211b0] Composer\Autoload\includeFile() /data/www/whmcs/vendor/composer/ClassLoader.php:322
[0x00007f063aa21120] loadClass() unknown:0
[0x00007f063aa210c0] spl_autoload_call() unknown:0
[0x00007ffdbaff9320] ???() /data/www/whmcs/loghandler.php:44
```
占据主要时间的前三个函数都是composer的函数, 貌似没有任何解决方案. 在Google一番之后, 发现这里的慢主要在于 OS X 下的 Docker 磁盘性能过低, 导致读取php文件速度过慢, 时间变长. 测试了各种解决方案之后, [docker-sync](//docker-sync.io) 解决了这个问题, 将整个网页的加载时间由5s缩短为300ms.

更新: docker-sync有各种玄学bug, 还是虚拟机为妙.