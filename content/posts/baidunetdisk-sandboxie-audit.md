+++
title = "沙盒里解剖百度网盘：从安装到广告到偷登录令牌"
date = 2026-08-02T18:00:00+08:00
draft = false
tags = ["Sandboxie", "隐私", "逆向", "百度网盘", "安全审计"]
categories = ["折腾笔记"]
description = "把百度网盘扔进 Sandboxie 沙盒里跑，用资源访问监控把它装机、运行、退出全过程摸了一遍。从疯狂抢文件关联、内置两套浏览器内核、VAST 视频广告模块，到最后在明文 Cookie 库里翻出 BDUSS 登录令牌和百度统计埋点——记录一次完整的“开盒”过程。"
+++

因为下载文件需要不得不装百度网盘，但又不放心让它直接跑在真实系统上，所以扔进了 Sandboxie-Plus 的沙盒里，顺手把它的资源访问监控（Resource Access Monitor）开着，从安装到运行到退出全程录了下来。这篇算是把追踪日志和沙盒目录翻了个底朝天之后的记录。

## 准备工作：为什么用 Sandboxie 而不是直接装

Sandboxie 会把目标程序对系统的一切读写重定向到一个虚拟层（`D:\Sandbox\<用户名>\<沙盒名>`），程序自己感觉不到区别，但注册表改动、文件写入全部落在沙盒目录里，不会碰真实系统。配合它自带的资源访问追踪，能拿到一份程序访问过的注册表键、文件、管道/IPC 对象、加载的 DLL 的完整清单——相当于免费的行为审计工具。日志格式是按资源路径去重后的计数汇总，不是时间线，但足够看清一个程序"都摸过什么"。

## 第一幕：装的时候就在抢地盘

安装阶段的 `CreateProcess` 记录里，启动参数写得明明白白：

```
BaiduNetdisk.exe -install "btassociation" -install "imgassociation"
  -install "createshortcut" "0" -install "rename"
  -install "regdefaultimgviewer" "imgantitamper"
```

拆开看：

- **`imgassociation` + `regdefaultimgviewer` + `imgantitamper`**：把自己注册成默认图片查看器，而且带"防篡改"参数——意味着你手动把默认看图软件改回去之后，它会自己再抢回来。日志里能看到它一次性接管了几乎所有主流图片格式和相机 RAW 格式的关联：`.jpg .jpeg .png .bmp .tiff .webp .heic .heif` 以及 `.cr2 .cr3 .nef .arw .dng .orf .pef .raf .rw2 .srw .nrw .3fr`（佳能、尼康、索尼、富士、松下、适马几乎全占了）。后面翻沙盒目录才发现它是真的准备了整套 RAW 处理引擎（`module\ImageViewer\resources\rawengine\dcpprofiles`，79MB 的 Adobe DCP 色彩配置文件）来支撑这个野心。
- **`btassociation`**：顺手把 BT 种子文件的默认打开方式也接管了。
- **`createshortcut`**：桌面、开始菜单、快速启动栏全部铺满图标，其中还包括 `Users\Public\Desktop`——**所有系统账户共享的桌面**，不只是当前用户。
- 加载了 `HNetCfg.FwPolicy2`（Windows 防火墙策略 COM 接口）和 `FirewallAPI.dll`，说明它会自己给自己加防火墙放行规则，不用你手动允许。
- `YunDetectService.exe`、`YunUtilityService.exe` 都是以 `--install`/`reg` 参数注册成**系统服务**装上的，不是普通进程。

## 第二幕：跑起来之后开始表演

装完之后的运行期日志（8000 多条）信息量更大。

**内置了完整的视频广告播放模块。** `baidunetdiskhost.exe` 的启动参数里能看到一个专门的插件宿主进程：

```
--plugin_id=3 --plugin_file_path="...\vastplayer.dll"
```

VAST（Video Ad Serving Template）是标准的视频广告插播协议。这个模块不是按需下载的，常驻在安装目录里，还专门生成播放日志：`BaiduYunGuanjia\BaiduYunKernel\VideoLog\MVastLog_*.log`。也就是说，你在网盘里预览视频时会插播广告，这在技术层面是有实锤的。

**一个网盘客户端里塞了两套独立的浏览器内核。** `module\BrowserEngine`（Electron/Chromium 套壳，647MB）和 `libcef.dll`（Chromium Embedded Framework，独立一份 176MB）——光这两样加起来 800 多 MB，界面层跑出了二十多个子进程（renderer、gpu-process、audio 服务、network 服务、node 服务……），典型的"一个云盘装成半个浏览器"。

**一条被加密的启动参数。** 日志里有一次 `CreateProcess` 调用，命令行不是明文，而是一长串随机字符（`StartInfo=RZuKRbG+kCH...`），完全看不出实际传了什么，算是"藏着掖着不给你审计"的一种体现。

## 第三幕：退出主程序之后，进程还在

这是最初让我起疑的地方——任务管理器里明明退出了网盘窗口，`YunDetectService.exe` 还在跑，而且是挂在 Sandboxie 自己的辅助进程（`SandboxieRpcSs.exe`、`SandboxieDcomLaunch.exe`）旁边独立存在的。翻回安装日志就能看到答案：`YunDetectService.exe reg` 这行命令的 `reg` 参数就是把自己注册进系统服务/自启动体系。它从设计上就不是主界面的子进程，退出界面壳子对它没有任何影响，得去服务管理器里单独动手。

再看它自己产生的资源访问记录，能看到两个没预料到的东西：

1. **内置 BT 下载引擎还开了个本地 Web 服务器。** 沙盒目录里有 `Transmission\public_html\index.html`——`Transmission` 是知名 BT 客户端，`public_html` 是它内置 Web 管理界面的静态资源。`kernel_btsdk.dll` 这个 BT SDK 内核模块不仅在跑下载/做种（`btConfig` 下能看到 `dht.bootstrap`、`Torrents`、`blocklists` 这些典型 BT 客户端配置），还起了本地 HTTP 服务，理论上能通过浏览器访问管理页面——普通用户完全不知道自己机器上开着这个。
2. **在监控资源管理器和系统窗口。** `WinClass` 分类下记录了大量它交互过的窗口类：`CabinetWClass`（资源管理器）、`Address Band Root`、`Breadcrumb Parent`、`ApplicationFrameWindow`、`NamespaceTreeControl`……几乎覆盖了整个 Windows 文件管理器和浏览器外壳的界面元素，说明它在持续枚举、挂钩这些窗口，这是实现"右键菜单增强""网页嗅探下载"之类功能的手段，代价是它一直盯着你桌面上开的窗口。
3. 注册表里还写了 `Image File Execution Options\baidunetdiskhost.exe` 这类映像劫持位——这个位置本来是给调试器挂钩用的，历史上也是恶意软件常滥用的进程劫持点，普通软件占用本身就不太规矩。

## 第四幕：把沙盒目录整个翻出来看看拉了多少

`D:\Sandbox\<用户>\BaiduNetdisk` 里累计 **189 个文件夹、1435 个文件，1.7GB**。一个"上传下载工具"该有的体积应该是网络传输 + 同步逻辑 + 简单界面，实际装的是：

| 模块 | 体积 | 说明 |
|---|---|---|
| `module\BrowserEngine` | 647MB | Electron 浏览器引擎 |
| `module\ImageViewer` | 386MB | 看图工具（含 RAW 引擎） |
| `libcef.dll` | 176MB | 第二套浏览器内核（CEF） |
| `module\AiEngine` | 61MB | AI 引擎，内含 OCR 模型 |
| `kernel.dll` | 52MB | 核心模块 |
| `module\VastPlayer` | 32MB | 视频广告播放器 |
| `module\SpeechSDK` | 19MB | 语音识别 SDK |

主目录里还散落着几个值得点名的文件：`minosagent.dll`（"Minos" 是百度内部的监控/统计 Agent 框架，在 `BrowserEngine` 目录下还有独立一份）、`logonbd.dll`/`logonbdext.dll`（名字带"logon"，疑似挂钩系统登录流程）、`sec_sdk.dll`（安全 SDK，通常负责设备指纹采集和反调试/反沙盒检测）、`YunOfficeAddin.dll`（会往 Office 里装加载项）、`netdisk-chrome-ext.crx`（打包好的 Chrome 扩展安装包，配合 `npYunWebDetect.dll` 检测你有没有装它的插件）。附带的许可证文件（`CEF license.txt`、`libtorrent_license.txt`、`DuiEngine license.txt`）也印证了这套拼盘的组件构成。

1.7GB 里，和"云盘存取文件"这个核心功能真正相关的部分可能不到十分之一。

## 第五幕：百度云管家——加密的行为监控探针

`BaiduYunGuanjia` 这个目录乍看不起眼，只有 4MB 左右，但性质比前面所有东西都恶劣。目录结构：

```
BaiduYunGuanjia\
├── AppData\
│   ├── at_monhavior      ← 疑似 "行为监控" 数据库
│   ├── at_stat            ← 统计数据库
│   ├── at_trche            ← 追踪缓存数据库
│   └── MXLog_*.txt        ← 4MB+ 埋点日志
└── BaiduYunKernel\VideoLog\ ← 广告播放记录（前面提到的 MVastLog）
```

直接拿十六进制看这几个数据库文件的文件头：

```
at_monhavior:  da 63 d7 84 62 27 77 de ...
at_stat:       28 19 54 ff bc 2c 8d c6 ...
at_trche:      a6 02 73 34 5a cc ad 08 ...
```

从文件名后缀（`-shm`/`-wal`）能看出底层是 SQLite（WAL 模式），但内容完全不是标准 SQLite 的明文格式（应以 `SQLite format 3\0` 开头），而是被加密/混淆过——普通用户和第三方安全工具都读不出里面记的是什么行为数据。`MXLog`/`XLog` 那两份日志同样是编码乱码。

名字本身也说明了记录的类别：`at_monhavior`（大概率是 monitor behavior 的缩写变体）、`at_stat`（统计）、`MXLog`/`XLog`（埋点上报，"M" 很可能对应前面出现的 `minosagent.dll`）。这基本可以定性为一整套独立于主程序、专门做用户行为埋点和广告播放追踪的探针，而且**特意加密防止被审计**。

## 第六幕：翻到明文 Cookie 库，找到了登录令牌

这是整个过程里分量最重的发现。内嵌浏览器的 `Network\Cookies` 是一份 **未加密** 的 SQLite 数据库，直接用 grep 提取可打印字符串就能读出内容：

**账号登录令牌，明文存放：**

```
.baidu.com         BDUSS=Z2MUdLZlRwSmpqMUJrMDh-STVVT3ptSEF5NXNXfk5xREJj...
.pan.baidu.com      STOKEN=c912fdc3d11455582b16bd0456b5dfdf30923e1065d51b3a5d483794e...
.pcsdata.baidu.com   BDUSS=...
```

`BDUSS` 是百度全家桶通用的登录凭证，拿到它基本等于拿到了免密登录账号的钥匙。讽刺的是，前面 `BaiduYunGuanjia` 里那些**行为监控数据**被认真加密保护了起来，**你的账号令牌反而是明文躺在文件里**的。

**百度统计埋点，连搜索路径都记录：**

```
.pan.baidu.com   Hm_lvt_e961721efc295d08f83faba5dbe3a12b = 1785660151,1785660682/aipan/search/
.hm.baidu.com    HMACCOUNT = FC6C618B672A97FC
```

`Hm_lvt_` 是标准的"百度统计"追踪 Cookie，`hm.baidu.com` 就是它的域名。这条记录里能直接读出**具体的访问时间戳和页面路径**（`aipan/search`，网盘搜索页）——说明内嵌浏览器组件确实在跑百度统计脚本，把用户在网盘里的操作路径上报出去，不是猜测，是有真实数据摆在那的。

**AB 测试分桶：**

```
.miao.baidu.com   ab_bid / ab_jid = 01c3eb83529b143a9b27426e3603f18869d3
```

"miao" 是百度内部的灰度发布/AB 测试系统，给设备分配了一个测试分组 ID，用来推送不同版本的功能——用户在不知情的情况下被当成了实验样本。

## 总结

把安装、运行、退出、目录体积、加密行为库、明文 Cookie 六个维度拼起来，能看出一条完整的链路：

1. **装的时候**：抢文件关联（还带防篡改锁定）、抢防火墙规则、桌面图标铺满所有用户、注册系统服务保证自己活得比主程序久。
2. **跑起来之后**：塞两套浏览器内核、内置视频广告模块、内置 BT 下载引擎带本地 Web 服务、用加密参数掩盖部分启动行为。
3. **后台常驻**：退出界面也杀不掉的检测服务，持续监控资源管理器和系统窗口。
4. **数据层面**：一整套加密的用户行为埋点系统（百度云管家），加上明文存储的账号登录令牌和百度统计追踪 Cookie。

对"账号安全"这一条，如果你也长期挂着百度网盘客户端，建议去百度账号安全中心查一下登录设备列表，必要时改密码把本地缓存的 BDUSS 令牌顶掉。至于其它的抢地盘行为，能用 Sandboxie 隔离终归是省心一些——至少不用担心它把手伸到真实系统的注册表和文件关联里。
