+++
date = '2026-04-29T00:01:53+08:00'
draft = false
title = 'ELSJY QM9700 安装黑苹果 macOS Monterey 记录'
tags = ["计算机", "macOS"]
+++


> 这是一篇从零开始的黑苹果安装实战记录，不作为教程文章。

之前我把我那台 MacBook Air M2 2023 8+512 的电脑卖了四千三，离开了一直熟悉的 macOS 换成了 Windows 电脑。最近突发奇想想把家里的一台小主机给安装 macOS 试试，处理器是 i7 7500U。一问 AI 才知道，这种七代低压U正好适合玩黑苹果，于是我开始了自己踩坑的历程

## 所用工具

先介绍一下我所用的工具
 - **电脑：** i9-12900H + RTX4060 的神舟游戏本（其实什么电脑都可以）
 - **U盘：** 联想 LEGION LU1 256G 固态U盘（还是推荐用速度更快的U盘，我这块写 400MB/s 左右）
 - **主机：** 7500U 主机一个
 - 网线一根

## 目标机器配置
小主机的配置

- **品牌/型号：** ELSKY QM9700/QM9600-2C
- **CPU：** Intel Core i7-7500U（Kaby Lake，第7代）
- **核显：** Intel HD Graphics 620
- **内存：** 16GB DDR4 1600MHz（三星颗粒）
- **存储：** 512GB SATA SSD（FDMST512GTCYC162）
- **有线网卡：** Realtek RTL8111 × 2（双千兆网口）
- **无线网卡：** 原装 Intel 4965AGN（不支持黑苹果），计划更换为 BCM94352 DW1550（博通芯片，黑苹果免驱）
- **BIOS：** AMI Aptio V，2019年5月9日版本，UEFI 模式
- **目标系统：** macOS Monterey 12.6
- **SMBIOS：** iMac18,1

目前这个小主机的无线网卡是 Intel 的 4965AG，macOS 没有驱动，我准备换成 BCM94352 DW1550，能直接免驱。这台小主机是工控级别的裸板设计，没有机箱，主板上有 mSATA、mini-PCIe 等接口，扩展性不错。作为台式机平台，黑苹果难度比笔记本低不少。


## 制作启动盘

一开始我用 balenaEtcher 把 Monterey 12.dmg（13.6GB 完整安装镜像）直接烧录到固态U盘。得益于固态U盘，烧录过程很顺利，一分钟不到就完成了。但问题来了：烧录完成后，Windows 完全无法识别这个U盘的分区结构。DiskGenius 看不到设备，diskpart 显示磁盘大小为 0B，Windows 磁盘管理显示"未初始化"。TransMac 虽然能看到设备，但提示 "Could not access disk/media"。之后才发现，Etcher 以 raw image 方式写入 dmg，生成的分区表是 Apple 原生格式，Windows 根本不认。更关键的是，这种方式写出来的U盘没有独立的 EFI 分区，无法往里面塞 OpenCore 引导文件。我不知道有没有其他的好办法，但我不愿意往这个方向折腾了。

折腾了好一会儿之后，我决定放弃完整镜像的方案，改用 OpenCore 在线安装。原理很简单：

1. 把U盘格式化成 FAT32
2. 放入 OpenCore EFI 引导文件
3. 放入 macOS 恢复镜像（只有 ~650MB）
4. 从U盘启动进入恢复模式，联网在线下载安装完整系统

这个方案的好处是完全绕开了分区格式兼容性的问题，FAT32 哪个操作系统都认。

用管理员权限打开 PowerShell，通过 diskpart 格式化U盘：

```
diskpart
list disk
select disk 4          # 这一块要确认是自己的盘
clean
create partition primary size=1024
format fs=fat32 quick label=OC_USB
assign letter=Z
exit
```

这里建了一个 1GB 的 FAT32 分区，一开始我只建了 512MB，结果恢复镜像 654MB 放不下，又重新来了一遍。

### 下载恢复镜像

恢复镜像需要通过 Apple 官方工具下载。先装 Python（如果没有的话）：

```
winget install Python.Python.3.12
```

然后下载 macrecovery 脚本并运行：

```
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/acidanthera/OpenCorePkg/master/Utilities/macrecovery/macrecovery.py" -OutFile "macrecovery.py"

py macrecovery.py -b Mac-FFE5EF870D7BA81A -m 00000000000000000 -os latest download
```

指令会从苹果服务器下载 Monterey 的 `BaseSystem.dmg`（约 654MB）和 `BaseSystem.chunklist`。下载完成后，在U盘根目录创建 `com.apple.recovery.boot` 文件夹，把这两个文件放进去。

### 准备 OpenCore EFI

从 GitHub 下载 OpenCore 最新 Release 包：

**https://github.com/acidanthera/OpenCorePkg/releases/latest**

下载 RELEASE 版（不要 DEBUG），解压后把 `X64/EFI` 整个文件夹复制到U盘根目录。对于 i7-7500U + HD620 + RTL8111 的配置，需要以下 kext：

| Kext | 作用 | 下载地址 |
|------|------|---------|
| Lilu | 内核补丁框架，其他 kext 的前置依赖 | github.com/acidanthera/Lilu |
| VirtualSMC | 模拟苹果 SMC 芯片 | github.com/acidanthera/VirtualSMC |
| SMCProcessor | CPU 传感器 | 包含在 VirtualSMC 中 |
| SMCSuperIO | 风扇传感器 | 包含在 VirtualSMC 中 |
| WhateverGreen | 显卡驱动补丁 | github.com/acidanthera/WhateverGreen |
| RealtekRTL8111 | 有线网卡驱动 | github.com/Mieze/RTL8111_driver_for_OS_X |

所有 kext 下载 RELEASE 版，把 `.kext` 文件夹放到 `Z:\EFI\OC\Kexts\` 目录下。接下来准备 pilst，这是整个黑苹果最核心的部分。以下是我最终使用的关键配置（基于 Sample.plist 修改）：

**DeviceProperties 显卡配置：**

```xml
<key>PciRoot(0x0)/Pci(0x2,0x0)</key>
<dict>
    <key>AAPL,ig-platform-id</key>
    <data>AAAWWQ==</data>
    <!-- 0x59160000 - HD 620 mobile -->
    <key>device-id</key>
    <data>FlkAAA==</data>
    <key>framebuffer-patch-enable</key>
    <data>AQAAAA==</data>
    <key>framebuffer-stolenmem</key>
    <data>AAAwAQ==</data>
    <key>framebuffer-fbmem</key>
    <data>AACQAA==</data>
    <!-- 三个连接器全部强制设为 HDMI 类型 -->
    <key>framebuffer-con0-enable</key>
    <data>AQAAAA==</data>
    <key>framebuffer-con0-type</key>
    <data>AAgAAA==</data>
    <key>framebuffer-con1-enable</key>
    <data>AQAAAA==</data>
    <key>framebuffer-con1-type</key>
    <data>AAgAAA==</data>
    <key>framebuffer-con2-enable</key>
    <data>AQAAAA==</data>
    <key>framebuffer-con2-type</key>
    <data>AAgAAA==</data>
</dict>
```

这里有一个重要的坑：**ig-platform-id 和 framebuffer connector 的配置**。我先后尝试了 `0x59120000`（桌面版 HD 630）和 `0x59160000`（移动版 HD 620），两个都会导致 HDMI 黑屏。最终的解决方案是使用 `0x59160000` 并把三个连接器全部强制设为 HDMI 类型（`00080000`），暴力覆盖。不管物理接口对应哪个连接器索引，总有一个能命中。

**重要选项：**

```
AppleCpuPmCfgLock = true    # 绕过 CFG Lock（BIOS 里没有关闭选项）
AppleXcpmCfgLock = true     # 同上
DisableIoMapper = true      # 禁用 VT-d 映射
```

**Misc Security：**

```
DmgLoading = Any            # 允许加载任何 DMG（恢复镜像需要）
SecureBootModel = Disabled  # 禁用安全启动模型
ScanPolicy = 0              # 扫描所有设备（否则看不到恢复分区）
Vault = Optional
```

**NVRAM boot-args：**

```
-v keepsyms=1 debug=0x100 alcid=1
```

`-v` 开启 verbose 模式，也就是跑码，方便调试。系统稳定后可以去掉。

**PlatformInfo：**

SMBIOS 选择 `iMac18,1`，这是 Kaby Lake 桌面机型。使用 GenSMBIOS 工具生成序列号、Board Serial 和 SmUUID，填入 config.plist。

**UEFI Drivers：**

只需要三个：OpenHfsPlus.efi、OpenRuntime.efi（LoadEarly 设为 true）、ResetNvramEntry.efi。

### 最终U盘文件结构

```
Z:\
├── com.apple.recovery.boot\
│   ├── BaseSystem.dmg
│   └── BaseSystem.chunklist
└── EFI\
    ├── BOOT\
    │   └── BOOTx64.efi
    └── OC\
        ├── config.plist
        ├── OpenCore.efi
        ├── Drivers\
        │   ├── OpenHfsPlus.efi
        │   ├── OpenRuntime.efi
        │   └── ResetNvramEntry.efi
        ├── Kexts\
        │   ├── Lilu.kext
        │   ├── VirtualSMC.kext
        │   ├── SMCProcessor.kext
        │   ├── SMCSuperIO.kext
        │   ├── WhateverGreen.kext
        │   └── RealtekRTL8111.kext
        └── Tools\
            └── OpenShell.efi
```

## 安装 macOS

插上U盘，开机进 BIOS 的 Save & Exit 页面，在 Boot Override 中选择 **UEFI OS (LEGION LU1)**。成功进入 OpenCore 引导菜单后，选择 **OC_USB (external) (dmg)** 启动恢复模式。在线安装需要联网。我的小主机离路由器很远，没法直接接网线。解决方案是用笔记本的 WiFi 通过网线共享给小主机：

1. 用网线连接笔记本和小主机
2. 在笔记本上 WIN+R 打开 `ncpa.cpl`
3. 右键 WiFi 适配器 → 属性 → 共享 → 勾选"允许其他网络用户通过此计算机的 Internet 连接来连接" → 选择以太网适配器

设置完成后在恢复模式的终端中 `ping apple.com` 确认网络连通。

第一次尝试安装时遇到 "The recovery server could not be contacted" 错误。Safari 能正常上网但安装程序连不上苹果服务器。原因是系统时间差了8小时（UTC 时间不对），导致 SSL 证书验证失败。在终端中手动设置正确的 UTC 时间即可解决：

```bash
date 0428142626    # 格式：MMDDhhmmYY
```

先在 Disk Utility 中把内置 SSD 抹掉，格式选 APFS，Scheme 选 GUID Partition Map，然后返回恢复菜单，选择 Reinstall macOS Monterey。然后同意许可协议，选择目标磁盘，等待下载和安装（通过笔记本共享的网络，大约下载了 12GB）。安装过程中会重启多次。每次重启后在 OpenCore 菜单选择 **macOS Installer** 或 **Macintosh HD** 继续

## 显卡驱动调试

这是整个安装过程中最折腾的部分。安装完成后，开机跑码正常，但跑码结束后屏幕无信号，调试过程极其难受

首先在 boot-args 中添加 `-igfxvesa` 强制使用 VESA 基础显示模式。这样能看到画面但没有硬件加速，界面非常卡顿。然后尝试在 VESA 模式下进系统，运行 `system_profiler SPDisplaysDataType` 确认 GPU 被正确识别为 Intel HD Graphics 620（Device ID: 0x5916）。我先后尝试了 ig-platform-id `0x59120000`（桌面版 HD 630）和 `0x59160000`（移动版 HD 620），都会导致 HDMI 黑屏。最后解决问题，使用 `0x59160000` 并通过 framebuffer-conX-type 补丁把三个连接器全部强制设为 HDMI 类型（`0x00080000`）

原因是每个 ig-platform-id 预设了固定的连接器类型配置（LVDS、DP、HDMI 等）。如果你的物理 HDMI 接口对应的连接器索引被配置成了 DP 或 LVDS 类型，macOS 就不会在这个接口上输出信号。把三个连接器全部改成 HDMI 是一种暴力但有效的解决方案。这也是这些工控小主机的通病了。

## 最终效果

macOS Monterey 12.6 在 ELSKY QM9700 上运行流畅，显卡硬件加速正常，有线网卡即插即用。作为一台涡轮风扇小主机，日常办公和轻度开发完全够用。