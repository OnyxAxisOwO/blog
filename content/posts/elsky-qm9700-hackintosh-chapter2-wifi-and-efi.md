+++
date = '2026-05-09T02:00:00+08:00'
draft = false
title = 'ELSJY QM9700 黑苹果后续：WiFi 驱动与 OpenCore 写入硬盘'
tags = ["计算机", "macOS"]
+++

> 接上篇。系统装好了，显卡也驱动了，但无线网卡还没搞定，OpenCore 还在 U 盘上。这篇记录 BCM94352 DW1550 的 WiFi 驱动过程，以及最终把 OpenCore 写入硬盘 EFI 分区的操作。

## 更换无线网卡

原装的 Intel 4965AGN 在 macOS 下完全没有驱动支持，所以我直接换成了 BCM94352HMB（Dell DW1550）。这是一块博通芯片的 mini-PCIe 无线网卡，支持 802.11ac 双频和蓝牙 4.0，在黑苹果社区里算是经典的"免驱卡"。

之所以说"免驱"，是因为 macOS 内置了博通无线网卡的驱动框架，不需要额外安装第三方驱动就能直接识别。但事实证明，这个"免驱"在 Monterey 上是有条件的，后面会详细讲踩了多少坑。

把 Intel 4965AGN 从 mini-PCIe 插槽上拔下来，装上 DW1550，接好两根天线（主天线和副天线），物理安装部分就结束了。

## 添加 AirportBrcmFixup

虽然 BCM94352 在 macOS 中有原生驱动支持，但仍然需要 AirportBrcmFixup 这个 kext 来做补丁注入。它的作用是修正博通网卡在黑苹果环境下的各种兼容性问题，比如 country code、驱动匹配、固件加载时序等。

从 GitHub 下载 AirportBrcmFixup：

**https://github.com/acidanthera/AirportBrcmFixup/releases**

下载 RELEASE 版，把 `AirportBrcmFixup.kext` 放到 `EFI/OC/Kexts/` 目录下。然后在 config.plist 的 `Kernel → Add` 中添加声明：

```xml
<dict>
    <key>Arch</key>
    <string>x86_64</string>
    <key>BundlePath</key>
    <string>AirportBrcmFixup.kext</string>
    <key>Comment</key>
    <string>AirportBrcmFixup</string>
    <key>Enabled</key>
    <true/>
    <key>ExecutablePath</key>
    <string>Contents/MacOS/AirportBrcmFixup</string>
    <key>MaxKernel</key>
    <string></string>
    <key>MinKernel</key>
    <string></string>
    <key>PlistPath</key>
    <string>Contents/Info.plist</string>
</dict>
```

这里有一个容易忽略的点：**AirportBrcmFixup 的子插件也需要单独声明。** OpenCore 不会自动加载 kext 内部的 PlugIns，必须在 config.plist 中手动添加 `AirPortBrcmNIC_Injector.kext` 的声明：

```xml
<dict>
    <key>Arch</key>
    <string>x86_64</string>
    <key>BundlePath</key>
    <string>AirportBrcmFixup.kext/Contents/PlugIns/AirPortBrcmNIC_Injector.kext</string>
    <key>Comment</key>
    <string>AirPortBrcmNIC Injector for BCM94352</string>
    <key>Enabled</key>
    <true/>
    <key>ExecutablePath</key>
    <string></string>
    <key>MaxKernel</key>
    <string></string>
    <key>MinKernel</key>
    <string></string>
    <key>PlistPath</key>
    <string>Contents/Info.plist</string>
</dict>
```

注意 `ExecutablePath` 留空，因为这个 Injector 是纯 plist 注入型的 kext，没有可执行文件。

## 第一个坑：Block IOSkywalkFamily

一开始我在网上搜解决方案的时候，看到很多帖子说 Monterey 需要在 `Kernel → Block` 中屏蔽 `com.apple.iokit.IOSkywalkFamily`，因为 Monterey 引入了新的 IOSkywalk 网络框架来替代旧的驱动体系，会跟老博通网卡冲突。

于是我在 config.plist 中加了这个 Block 规则：

```xml
<key>Block</key>
<array>
    <dict>
        <key>Arch</key>
        <string>x86_64</string>
        <key>Comment</key>
        <string>Block IOSkywalkFamily for Broadcom WiFi</string>
        <key>Enabled</key>
        <true/>
        <key>Identifier</key>
        <string>com.apple.iokit.IOSkywalkFamily</string>
        <key>MaxKernel</key>
        <string></string>
        <key>MinKernel</key>
        <string>21.0.0</string>
        <key>Strategy</key>
        <string>Exclude</string>
    </dict>
</array>
```

重启之后进入 verbose 模式，看到一堆报错：

```
Can't load kext com.apple.driver.AppleIRPAppender - library kext com.apple.iokit.IOSkywalkFamily not found.
Failed to resolve library dependencies.
```

系统最终还是能启动，但启动速度明显变慢了（第一次 Block 之后需要重建内核缓存），而且 WiFi 依然无法使用。菜单栏的 WiFi 图标虽然在，但开关拨不动，灰色状态。

用 `kextstat | grep -i brcm` 查看，只有 `as.lvs1974.AirportBrcmFixup` 加载了，没有看到任何 `AirPortBrcmNIC` 或 `AirPortBrcm4360` 被拉起来。说明 AirportBrcmFixup 虽然加载成功了，但它没有成功 hook 到实际的 WiFi 驱动上。

## 第二个坑：OCLP

接下来我以为是 Monterey 把老的 WiFi 驱动文件删了，于是尝试用 OpenCore Legacy Patcher（OCLP）来补回被删除的驱动。OCLP 本来是给老白苹果用的工具，用于把苹果在新系统中移除的旧驱动重新注入回系统。

但由于我的黑苹果没有有线网，也没有 WiFi（正在解决的就是这个问题），下载 OCLP 本身就是个挑战。最终我通过 iPad Air（USB-C 线连接）配合 Finder 文件传输，把从 iPad 上下载的 OCLP 安装包传到了 Mac 上。

然而 OCLP 打开之后显示 **"No patches required"**。原因是 OCLP 通过 SMBIOS 机型来判断是否需要打补丁，我的 config.plist 中设置的机型是 `iMac18,1`（2017 年 iMac），这台机器在苹果看来原生支持 Monterey，所以 OCLP 认为不需要任何补丁。

## 真正的原因

在终端中运行 `ls /System/Library/Extensions/ | grep -i 80211`，发现：

```
IO80211Family.kext
IO80211FamilyLegacy.kext
```

**两个文件都在。** 所以驱动根本没有被删除，之前的判断完全错误。Monterey 12 中 IO80211FamilyLegacy 和 IOSkywalkFamily 是共存的关系，Block 掉 IOSkywalkFamily 反而破坏了依赖链，导致一系列连锁报错。

真正的问题其实很简单：**AirportBrcmFixup 加载时序太早了。** 系统还没完成 IO80211FamilyLegacy 的初始化，AirportBrcmFixup 就尝试去 hook，结果扑了个空。

## 最终解决方案

修改 boot-args，添加 `brcmfx-delay=15000` 参数，让博通驱动延迟 15 秒再加载：

```
-v keepsyms=1 debug=0x100 alcid=1 brcmfx-driver=2 brcmfx-delay=15000
```

同时把之前添加的 `Kernel → Block` IOSkywalkFamily 规则删除（或 Enabled 设为 false），恢复为空数组。

参数说明：
- `brcmfx-driver=2`：强制使用 AirPortBrcmNIC 驱动（对应 BCM94352 芯片）
- `brcmfx-delay=15000`：延迟 15000 毫秒（15 秒）后再加载博通驱动，给系统时间完成 IO80211FamilyLegacy 的初始化

重启之后，WiFi 终于可以正常开启，扫描到周围的无线网络，连接成功。

## 最终的 Kexts 列表

```
EFI/OC/Kexts/
├── Lilu.kext
├── VirtualSMC.kext
├── SMCProcessor.kext
├── SMCSuperIO.kext
├── WhateverGreen.kext
├── RealtekRTL8111.kext
└── AirportBrcmFixup.kext
    └── Contents/PlugIns/AirPortBrcmNIC_Injector.kext
```

config.plist 中 `Kernel → Add` 的加载顺序（顺序很重要，Lilu 必须第一个）：

1. Lilu.kext
2. VirtualSMC.kext
3. SMCProcessor.kext
4. SMCSuperIO.kext
5. WhateverGreen.kext
6. RealtekRTL8111.kext
7. AirportBrcmFixup.kext
8. AirportBrcmFixup.kext/Contents/PlugIns/AirPortBrcmNIC_Injector.kext

## 将 OpenCore 写入硬盘

系统和驱动全部调试完毕后，最后一步是把 OpenCore 从 U 盘写入到硬盘的 EFI 分区，这样就不需要每次都插着 U 盘启动了。

首先用 `diskutil list` 确认磁盘布局：

```
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *512.1 GB   disk0
   1:                        EFI EFI                     209.7 MB   disk0s1
   2:                 Apple_APFS Container disk1         511.9 GB   disk0s2

/dev/disk2 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *256.1 GB   disk2
   1:               Windows_FAT_32 OC_USB               1.1 GB     disk2s1
```

- `disk0` 是 512GB SATA SSD（系统盘），`disk0s1` 是硬盘的 EFI 分区
- `disk2` 是 256GB U 盘，`disk2s1` 是 OpenCore 所在的 FAT32 分区

挂载硬盘的 EFI 分区：

```bash
sudo diskutil mount disk0s1
```

确认 U 盘的 OC 文件结构正常：

```bash
ls /Volumes/OC_USB/EFI/
# 输出：BOOT  OC
```

确认硬盘 EFI 分区已挂载且为空：

```bash
ls /Volumes/EFI/
# 输出：（空）
```

复制 EFI 文件夹到硬盘：

```bash
sudo cp -R /Volumes/OC_USB/EFI /Volumes/EFI/
```

验证复制结果：

```bash
ls /Volumes/EFI/EFI/OC/
# 输出：ACPI  Kexts  Resources  config.plist  Drivers  OpenCore.efi  Tools
```

确认文件完整后，关机，拔掉 U 盘，开机。如果 BIOS 启动顺序中没有自动识别到硬盘的 OpenCore 引导项，需要进 BIOS 手动调整启动顺序，把硬盘的 UEFI 启动项设为第一位。

成功从硬盘引导进入 OpenCore 菜单，选择 macOS 正常启动，WiFi 正常工作。至此，整个黑苹果安装过程完成。

## 踩坑总结

1. **BCM94352 在 Monterey 12 上不是真正的"免驱"。** 虽然系统内置了驱动文件（IO80211FamilyLegacy.kext），但仍然需要 AirportBrcmFixup 做补丁注入，而且需要通过 `brcmfx-delay` 参数解决加载时序问题。

2. **不要在 Monterey 12 上 Block IOSkywalkFamily。** 很多过时的教程建议这么做，但 Monterey 中 IOSkywalkFamily 和 IO80211FamilyLegacy 是共存的，Block 掉 Skywalk 会破坏依赖链，导致 AppleIRPAppender 等组件报错。这个 Block 操作是 macOS Sonoma 14 及以上才需要的，因为 Sonoma 才真正移除了老的驱动文件。

3. **AirportBrcmFixup 的子插件必须在 config.plist 中单独声明。** OpenCore 不会自动递归加载 kext 内部的 PlugIns 目录，`AirPortBrcmNIC_Injector.kext` 需要手动添加到 `Kernel → Add` 列表中。

4. **没有网络的情况下传输文件是个大问题。** 我的解决方案是通过 USB 线连接 iPad，利用 Finder 的文件传输功能把需要的工具从 iPad 传到 Mac 上。安卓手机的 USB 共享网络在 macOS 上兼容性较差，iPad（Lightning 或 USB-C）的兼容性更好。

5. **写入硬盘前先确保一切稳定。** 在 U 盘引导的状态下把所有驱动、系统设置、常用软件都调试好，确认每次重启都正常之后再写入硬盘。U 盘引导期间修改 config.plist 非常方便，写入硬盘后再改就需要额外挂载 EFI 分区了。

6. **U 盘留着当救命盘。** 写入硬盘后不要格式化 U 盘上的 OC 引导文件，万一硬盘的 EFI 出问题，U 盘还能作为备用引导。
