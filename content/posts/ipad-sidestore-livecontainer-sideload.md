+++
title = "iPad 上装 LiveContainer + SideStore"
date = 2026-08-01T22:00:00+08:00
draft = false
tags = ["iOS", "iPadOS", "侧载", "SideStore", "LiveContainer", "越狱"]
categories = ["折腾笔记"]
description = "在 iPad Air M3 上从零装 LiveContainer + SideStore。真正难的不是安装，是那个把 anisette 故障伪装成「密码错误」的报错链，以及国内网络出不去导致的死循环。"
+++

手上有台 iPad Air 11 寸 M3 256G，之前拼多多百亿补贴3500买的全新，天天拿来刷抖音。最近想起来 iOS 侧载这几年生态成熟了不少，就想给它装个 LiveContainer 玩玩。

## 为什么是 LiveContainer

我这台是 M3 芯片 + iPadOS 26，传统越狱这条路是死的。palera1n 和 Dopamine 依赖的 checkm8 是 bootrom 漏洞，A12 之后就修掉了，硬件层面无解；TrollStore 依赖的 CoreTrust 漏洞在 iOS 17.0 之后也修了。所以新机器上能拿到的上限就是侧载。

LiveContainer 的思路是：它自己是一个正常安装的宿主 App，别的 IPA 塞进它的沙盒里跑，不真正装进系统。好处是**不占 App ID 配额**（免费开发者账号每 7 天只能注册 10 个 App ID），能多开，还能往客体 App 的进程里注入 dylib —— 也就是安卓那边 LSPosed 模块的等价物，也能获得相当于越狱的功能。我之前在安卓上玩 root，主要就是在 LSPosed 里给微信、B站之类挂模块。LiveContainer 的 Tweak 模块功能覆盖的正好是这一层。

## Windows 端的准备

现在 SideStore 的安装流程比以前简单，官方推荐用 iloader，配对文件那套已经自动化了。但有个前置条件文档写得不够显眼：**Windows 上必须先装 Apple 的软件来提供 usbmuxd**。我一开始直接装了 iloader，插上 iPad，报 `Failed to connect to usbmuxd: device socket io failed`。查了一下机器上根本没有任何 Apple 软件。Windows 靠通用相机驱动能认出 iPad（右下角会弹「Apple iPad 点击选择要对此设备进行的操作」），但那跟 usbmuxd 是两回事。

装 **Apple Devices**（微软商店那个）就解决了：

```
winget install --id 9NP83LWLPZ9K --source msstore
```

这里有个容易忽略的细节：**商店版的 Apple Devices 不会注册成 Windows 服务**，它靠 App 启动时拉起 `AppleMobileDeviceLauncher` 进程。所以用 iloader 的时候 **Apple Devices 得开着**，关掉就又认不到设备了。我一开始装完直接关了 App，白白多排查了一轮。不推荐装微软商店版的 iTunes，那个是沙盒化的，usbmuxd 经常出问题。要 iTunes 就从 apple.com 下 exe 版。

## 第一次登录失败：Auth error -22406

Apple Devices 装好，iloader 认到设备了，接下来登录 Apple ID。然后就撞上了第一堵墙：

```
Auth error -22406: Enter the correct password for this Apple Account.
```

字面意思是密码错误。但我明明收到了验证码，也输进去了 —— **验证码通过说明账号密码那一关已经过了**，怎么会是密码错误？

展开详细日志，关键在这一行：

```
GrandSlam error during proof login request
```

Apple 的 GrandSlam 认证是两阶段的：第一阶段验密码和二步验证，第二阶段要拿 anisette 数据做一次 proof login。挂掉的是**第二阶段**，而 Apple 在这一步返回的错误码恰好是 -22406，文案就是「密码错误」。报错在骗人。我去 iloader 仓库翻 issue，[#231](https://github.com/nab138/iloader/issues/231) 被并到 [#193](https://github.com/nab138/iloader/issues/193)，到现在都没有官方结论或修复。

## Anisette 是什么，为什么它总坏

侧载工具要拿到开发者签名权限，就得让 Apple 相信它是一台 Mac。anisette 数据就是模拟 macOS 的机器指纹 —— 这也解释了为什么登录之后，我的 Apple ID 设备列表里凭空多出来一台「MacBook Pro」。那台幽灵 Mac 就是 iloader 自己，这玩意还不能删除，删了下次登录得从头再走一遍验证码流程。anisette 服务器是社区志愿者跑的公共服务，[SideStore/anisette-servers](https://github.com/SideStore/anisette-servers) 里列了十几台。问题在于大量用户共用一个出口 IP，Apple 对这类 IP 做限流甚至封禁，所以某台服务器今天能用明天就不行，纯看运气。我把列表里的服务器全 curl 了一遍，**HTTP 接口全部健康，返回的 anisette 头一个不缺**。所以「能不能访问」这个指标毫无参考价值 —— 坏的是 v3 provisioning 握手那一步，那步是服务器代你去跟 Apple 谈，客户端这边只能看到一个模糊的失败。重置 anisette 状态之后，错误从 -22406 变成了：

```
Anisette provisioning failed: end provisioning error
invalid Trust Key (-45003)
```

这个反而是好消息 —— 它至少诚实地指出了故障在 anisette。换服务器的时候有个坑：**每换一台，必须先重置 anisette 状态**（iloader 里叫「重置 Anisette 状态」，SideStore 里叫 `Reset adi.pb`）。不重置的话旧的 Trust Key 还在，换了服务器照样报 -45003。我一开始不知道，连着换了两台都失败，差点以为是账号被风控了。最后是换到 `SideStore (.zip)` 通的。**试到第三台。**

## 第二堵墙：iPad 上又登不上了

iloader 那边登录成功，一路把 LiveContainer + SideStore 合并版推到了 iPad 上。合并版的设计是 LiveContainer 作宿主、SideStore 作为客体跑在它里面，这样只占 1 个 App ID 配额，而且 SideStore 能反过来给宿主续签。装完之后要在 iPad 的 SideStore 里再登录一次 Apple ID —— 这一步绕不过去，我一度想通过「导入证书」跳过，查了官方文档才确认：**导入证书之后仍然必须登录**，那个功能是给多设备共用一张证书用的，不是登录的替代方案。然后 iPad 上又开始报错，而且换了个花样：

```
An error occurred while provisioning: Timeout
```

```
Failed to Log In: The data couldn't be read because it isn't in the correct format.
```

第二个是 Swift 的 JSON 解码失败，意思是 anisette 服务器返回了它看不懂的东西。翻 SideStore 的 issue 才找到根因。维护者 mahee96 的说法是：这个错误发生在 anisette 服务器不可用，**或者设备本地网络限制了整个 anisette 服务器列表**的时候。[#1267](https://github.com/SideStore/SideStore/issues/1267) 底下几个中国用户说得更直白 —— 「建议挂代理或者换设置里面的 ani server，最近出网打的很严」，还有人说从电信卡切到联通卡就直接登录成功了。这里还有个更烦的问题：**iOS 同时只允许一个 VPN 隧道生效。**

SideStore 刷新和装 App 依赖 LocalDevVPN —— 它是个本地回环隧道，让设备能连自己，跟设备的安装守护进程通信。而我要访问境外的 anisette 服务器又需要梯子。两个都是 VPN，只能开一个。所以死循环成立：开梯子 → LocalDevVPN 断 → SideStore 报「未连接到 VPN」；开 LocalDevVPN → 出不去网 → anisette 超时。

**破解方法是换一台国内能直连的 anisette 服务器。** 我最后选的是 SteX：

```
https://ani.xu30.top
```

`.top` 域名，服务器在国内，直连就通，梯子彻底不用开。一次登录成功。

如果哪天这台也挂了，还有两条路：一是电脑开移动热点、梯子走 TUN 模式，iPad 连这个热点上网（这样 iPad 本身不开任何 VPN，出境流量在电脑那层代理掉）；二是把代理挪到路由层，家里有软路由的话跑个透明代理做分流，iPad 上永远只留 LocalDevVPN 常驻，后台自动刷新也能正常工作。我家有台跑 PVE 的机器，长期方案打算走第二条。



## 收尾

登录成功之后剩下的都很顺：

iPad 上先去 **设置 → 通用 → VPN与设备管理**，找到自己的 Apple 账户点信任；iPadOS 16+ 还要在 **隐私与安全性 → 开发者模式** 里打开并重启。这个开关平时是隐藏的，得先有开发者行为碰过设备才会出现 —— 不需要 Mac 上的 Xcode，你用 iloader 装了带开发证书签名的 App，条件就满足了。然后装 **LocalDevVPN**（App Store 搜得到，国区可能没有，用外区 Apple ID 下载即可），打开并连接。最后回 LiveContainer → 设置 → **「从 SideStore 导入证书」**，按钮变成「移除证书」就说明成了。这一步是把 SideStore 的签名证书交给 LC，之后它才能自己给容器里的 App 签名。

## 顺手装了 StikDebug

模拟器和 UTM 这类要跑满速得开 JIT。以前必须连电脑用 SideJITServer，现在有 **StikDebug**，初次配对之后设备端就能自己开。装完打开发现什么都点不了，顶上一行黄字 `Pairing file not found!` —— 它需要配对文件。**不用手动传**，回 iloader 的「管理配对文件」，点 **Place In All Apps**，通过 USB 直接塞进所有已侧载 App 的沙盒里，StikDebug 和以后装的 App 一次搞定。装好之后 UTM 那边就有意义了。操作顺序要注意：**先打开 UTM 让它跑着，再切到 StikDebug 给它开 JIT，最后回 UTM 启动虚拟机。** 顺序反了的话 TCG 已经以解释模式初始化，得重来。

## 一些踩坑总结

**报错文案不可信** -22406 写着「密码错误」，实际是 anisette provisioning 失败。遇到反直觉的报错（比如验证码都过了还说密码错），优先展开详细日志看调用栈。

**能访问不等于能用** 我把所有 anisette 服务器 curl 了一遍全是 200，然而故障在 provisioning 握手那一层，HTTP 探测完全测不出来。这种时候延迟排序也没意义，只能一台台实际试。

**换 anisette 服务器必须重置状态** 这是最容易漏的一步，漏了就会得出「换了也没用」的错误结论。

**两端配置独立** iloader 和 SideStore 各有各的 anisette 设置，不同步。我这边一个 `.zip` 一个 `SteX`。

**免费证书 7 天过期** 到期前开着 LocalDevVPN 在 SideStore 里刷新一次就行，数据不丢。SideStore 有后台自动刷新，但前提是 LocalDevVPN 保持连接 —— 而这又跟梯子冲突。这也是为什么值得费劲找一台国内能直连的 anisette 服务器：只有这样 LocalDevVPN 才能常驻。

**下载和安装分开做** 从源里直接装的话，SideStore 要自己下载 IPA，这时需要外网，但安装又要 LocalDevVPN，还是撞车。正确做法是在电脑上下好 IPA，用 LocalSend 传到 iPad，再从本地文件安装。这样彻底绕开 VPN 冲突，而且电脑下载快得多。

---

折腾完的感受是：这套东西的技术设计其实很清晰，难的全是外部环境，公共服务器被限流、本地网络出不去、iOS 只给一个 VPN 槽位。真正的技术门槛远低于运气门槛。至于能玩什么，我的判断是安卓那边 LSPosed 那一层的玩法基本能平移过来，Magisk 那一层（改系统、全局去广告、内存修改器）彻底没戏。对我来说够了，我 90% 的时间本来也是在给单个 App 挂模块。
