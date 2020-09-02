---
title: 在z270平台装黑苹果
date: 2017-08-18 12:01:23
tags:
  - 黑苹果
  - Hackintosh
---

## 动机

以前一直在用 win10 最近在开发的几个 web和移动 app 项目，前端是 angular4 或者基于 angular4 的 ionic3，后端是 golang 编写，每次编译的时候，我看着这个进度条，总感觉天空飘来五个字，我特么社保。

![](http://static.nicesite.vip/2018-06-12-090422.jpg)

后来换到 ubuntu desktop 16.04，感觉编译速度肉眼可见的狂暴提升，好吧既然如此，就不用再等 win10 使出洪荒之力了。

![](http://static.nicesite.vip/2018-06-12-090428.png)

再后来，换了新配置...

![](http://static.nicesite.vip/2018-06-12-090432.jpg)

## 准备工作

### 硬件设备
- 一个 Mac OS 的电脑（最好是这样，win 下麻烦些）
- 一个 8G U盘，可大不可小
- 一台 z270 平台待装机电脑

### 软件工具
- Mac OS 操作系统 或者 虚拟机
- [Clover EFI 最新版](https://sourceforge.net/projects/cloverefiboot/files/latest/download)
- [kext 驱动和 config.plist](http://static.nicesite.vip/z270-EFI.zip)

### 额外说明
* 截止到我现在这个时间，Mac OS 10.12.6 已经原生支持 Kaby Lake 平台，终于摆脱了 Sky Lake 的各种 Fake ID
* 包括 I7-7700 I5-7500 但不仅限于
* 核心显卡 HD630 也原生支持了
* AMD polaris架构虽然早已支持，但一直无法单卡启动的问题通过 [Lilu.kext](https://github.com/vit9696/Lilu) 和 [Whaterergreen.kext](https://github.com/vit9696/WhateverGreen) 解决了
* 声卡通过 [AppleALC.kext](https://github.com/vit9696/AppleALC) 驱动，升级系统就不会掉驱动了
* 额外安利几篇写的很好的 KabyLake 安装黑苹果的教程，我就看了这几篇就直接开始动手安装，性价比很高的教程，从过程到原理都有介绍到
* [Guide to installing macOS on a Kabylake Hackintosh for Sierra (10.12.5 Working)](http://hackintosher.com/guides/guide-installing-macos-kabylake-hackintosh-sierra/)
* [Updating your Hackintosh to Sierra 10.12.6](http://hackintosher.com/guides/updating-hackintosh-sierra-10-12-6/)
* [How to hackintosh AMD graphics cards in Sierra 10.12.6+](http://hackintosher.com/guides/hackintosh-amd-graphics-cards-sierra-10-12-6/)
* [Building a GTX 1080 Ti-powered Hackintosh: Installing macOS Sierra step-by-step](https://9to5mac.com/2017/04/28/building-a-gtx-1080-ti-powered-hackintosh-installing-macos-sierra-step-by-step-video/)

## 疯狂操作起来

{% youtube pugSN7REHQg %}

* 有条件的先看下这个视频教程，有个了解，歪果仁发在 youtube 上的，如果被墙了看不到就算了，直接看往下文字版吧

### 制作U盘启动器

* 在 App Store 中下载 [Mac OS Sierra](http://appstore.com/mac/macossierra)

![](http://static.nicesite.vip/2018-06-12-091034.jpg)

* 插入U盘，打开磁盘工具，格式化U盘

![](http://static.nicesite.vip/2018-06-12-091046.jpg)

* 请注意，点击主硬件设备，而不是点击某个分区，格式务必为 Mac OS 扩展（日志式），方案 GUID 分区图，抹掉，然后静静地等待 Mac OS Sierra 下载完成

```bash
sudo /Applications/Install\ macOS\ Sierra.app/Contents/Resources/createinstallmedia --applicationpath /Applications/Install\ macOS\ Sierra.app  --volume /Volumes/未命名/
```

* 将系统安装程序拷贝到U盘中，这个过程也挺慢的。拷贝完成后开始安装 Clover EFI 启动器

![](http://static.nicesite.vip/2018-06-12-091052.jpg)

* 点击更改安装位置，选择你的U盘

![](http://static.nicesite.vip/2018-06-12-091058.jpg)

* 仅安装 UEFI 开机版本
* 安装 Clover 到 EFI 系统区
* Drivers64UEFI > EmuVariableUefi-64
* Drivers64UEFI > OsxAptioFix2DrV-64
* Drivers64UEFI > PartitionDxe-64
* 把你刚才下载的 z720-EFI 解压缩，覆盖到U盘刚生成的 EFI 分区中

{% note warning %}
如果你是核心显卡用户 Intel iGPU HD530/HD630，请打开 /Volumes/EFI/EFI/CLOVER/
修改 config-iGPU.plist 为 config.plist
{% endnote %}

{% note success %}
如果你是英伟达显卡用户，请打开 /Volumes/EFI/EFI/CLOVER/
修改 config-NV.plist 为 config.plist
另外下载 [NvidiaGraphicsFixup.kext](https://sourceforge.net/projects/nvidiagraphicsfixup/) 放到 Volumes/EFI/EFI/CLOVER/kext/other/ 下
{% endnote %}

{% note danger %}
如果你是AMD显卡用户，请打开 /Volumes/EFI/EFI/CLOVER/
修改 config-AMD.plist 为 config.plist
{% endnote %}

{% note primary %}
如果你是微星主板用户下载 [OsxAptioFix2Drv-free2000.efi.zip](https://cl.ly/020C180S2R32/OsxAptioFix2Drv-free2000.efi.zip)
解压 OsxAptioFix2Drv-free2000.efi 到 /Volumes/EFI/EFI/CLOVER/drivers64UEFI/ 下
{% endnote %}

{% note info %}
如果你是技嘉或者华硕主板用户
接下来用 Clover Configurater 打开 config.plist，开始修改一些内容
左侧导航栏选择 Acpi 然后勾选 FixShutdown_0004 保存
{% endnote %}

![](http://static.nicesite.vip/2018-06-12-091107.jpg)

* 点击左侧 SMBIOS ，再点击右侧下拉按钮，请按照你的 CPU 进行选择模拟配置Mac型号

![](http://static.nicesite.vip/2018-06-12-091112.jpg)

* 保存并重启电脑，进入 BIOS 设置
* Save & Exit : Load Optimized Defaults
* M.I.T. : Advanced Memory Settings → Extreme Memory Profile(X.M.P.) : Profile1
* BIOS → Fast Boot : Disabled
* BIOS → Windows 8/10 Features : Other OS
* BIOS → LAN PXE Boot Option ROM : Disabled
* BIOS → Storage Boot Option Control : UEFI
* Peripherals → Initial Display Output : IGFX or PCIe 1 Slot
* Peripherals → Super IO Configuration → Serial Port : Disabled
* Peripherals → Network Stack Configuration → Network Stack : Disabled
* Peripherals → USB Configuration → XHCI Hand-off : Enabled
* Chipset → Vt-d : Disabled
* Chipset → Wake on LAN Enable : Disabled
* 显卡按照你的实际选择，是IGFX还是PCI1或者是PCI2
* 按 F10 保存并重启，选择你的U盘进入系统

### 开始安装系统

![](http://static.nicesite.vip/2018-06-12-091118.jpg)

* 先用磁盘工具给自己的硬盘分区，在安装就行了，这几步都没什么坑

![](http://static.nicesite.vip/2018-06-12-091122.jpg)

* 这里有点坑，登录 Apple ID 选择否，与苹果分享信息选择否，不然直接蹦系统

![](http://static.nicesite.vip/2018-06-12-091129.jpg)

![](http://static.nicesite.vip/2018-06-12-091137.jpg)

### 进入系统进行配置

* 把U盘中/Volumes/EFI/EFI/CLOVER/下所有文件复制到电脑上，拔掉U盘
* 再次安装 Clover EFI 到你的系统硬盘中，又会出现一个 EFI 分区,如果没自动出现，请自行挂载

![](http://static.nicesite.vip/2018-06-12-091157.jpg)

* 把刚才拷贝的/EFI/CLOVER/复制到EFI分区下相同目录下
* 英伟达显卡用户这时候可以去安装 [NVDIA Web Driver](http://www.nvidia.com/Download/index.aspx?lang=en-us) 了
* 好了，至此，完蛋。不，完工了

## 看看效果


![我的配置](http://static.nicesite.vip/2018-06-12-091213.jpg)

![显卡驱动](http://static.nicesite.vip/2018-06-12-091313.jpg)

![硬件状态](http://static.nicesite.vip/2018-06-12-091325.jpg)







