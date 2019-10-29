---
title: Oneplus 7 Pro 刷机教程
date: 2019-06-13 13:04:28
tags: 刷机
img: /medias/WechatIMG181.jpeg
---

**不同批次的手机安装的出厂H2OS版本不同, 本流程不一定适用于所有手机.**

### 准备工作

1. 从[这里](https://developer.android.com/studio/releases/platform-tools)下载相关工具包
2. 系统设置-关于手机 点击7次系统版本号 开启开发者模式
3. 开发者模式中开启"高级重启"和"OEM解锁"
4. 长按电源键, 选择重启到引导加载器
5. 连接电脑, 在电脑上运行 `fastboot oem unlock`
6. 在手机上确认

### 氧OS

绝大部分手机出厂预装的H2os中BootLoader的版本较低, 不支持安装TWRP. 需先安装O2OS.

1. 到[这里](https://otafsg1.h2os.com/patch/amazone2/GLO/OnePlus7ProOxygen/OnePlus7ProOxygen_21.O.11_GLO_011_1906160627/OnePlus7ProOxygen_21.O.11_OTA_011_all_1906160627_b59b4dfc6d8c9a.zip)下载O2OS, 并用adb导入到手机中.
2. 在手机设置-系统更新-本地升级中, 选择此ZIP包.
3. 若成功刷入, 即可开始刷入TWRP.
4. 若刷入不成功, 则预装的H2os版本较高. 使用以下步骤安装O2OS:
    1. 将安装包通过adb导入手机中, 重启进入bootloader, 连接电脑
    2. 在电脑上从[这里](https://twrp.me/oneplus/oneplus7pro.html) 下载TWRP的两个文件(img,zip)
    3. 电脑上运行 `fastboot boot twrp-3.3.1-3-guacamole.img`
    4. 手机会重启进入TWRP. 三清后刷入氧os安装包, 刷入后再进行一次三清
    5. 重启进入系统. 若无限循环进入revocery, 则再次连接电脑, 运行fastboot指令进入TWRP后选择 Advanced - Fix Revocery Bootloop

### TWRP

1. 在电脑上从[这里](https://twrp.me/oneplus/oneplus7pro.html) 下载TWRP的两个文件(img,zip)
2. 将zip文件通过adb导入手机
3. 手机重启进入 Bootloader 连接电脑
4. 电脑上运行 `fastboot boot twrp-3.3.1-3-guacamole.img`
5. 手机会进入临时的TWRP. 在此临时的TWRP中选择刷入上面导入的 TWRP zip 包

### Magisk

1. 下载 Magisk Manager 以及 magisk zip包
2. 重启进入 Revocery , 刷入ZIP包即可.

### OTA
按照O2OS - TWRP - Magisk的顺序刷入即可.
1. 下载完整包, 并准备好TWRP安装包, Magisk安装包. 重启进入TWRP.
2. TWRP内先后刷入ROM与TWRP
3. TWRP内选择重启到另一个slot
4. TWRP内安装Magisk
5. 重启进入系统