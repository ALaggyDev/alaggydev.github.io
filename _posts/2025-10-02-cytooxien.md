---
title: How Minecraft servers can track you across accounts and IPs using resource packs
description: How we uncovered a device fingerprinting exploit on cytooxien.net and more
date: 2025-10-02 00:00:00 +0800
categories: []
---

## Introduction

A while ago, [NikOverflow](https://github.com/NikOverflow) was playing on a popular German Minecraft server named Cytooxien, and discovered some strange error messages in the console.

![A bunch of resource pack loading errors](/assets/img/cytooxien/pack_error.png){: w="600"}

Strange, huh? So we investigated further. Never would we thought that we were about to go down a rabbit hole of weird Minecraft exploits, tricks and shenanigans. We were about to discover exploits that were hidden from general Minecraft community for **over one and a half years**! Using nothing more than server resource packs, we discovered that:
1. servers can quietly track players across different accounts & IPs
2. a design oversight in a popular hacked client led to catastrophic result
3. and a few more clever tricks along the way

We call this exploit ***TrackPack***. TrackPack has massive implications for anyone who uses multiple Minecraft accounts. It means that even if you switch accounts or IPs, the server can still recognize you via your device's resource pack cache. Servers can use this to detect ban evasion, alternative accounts etc.

In this post, I will delve into the story of how we discovered the exploit, how it works, what else did we uncovered and more!

P.S. NikOverflow is also making a video about this, so stay tuned. Coming soon!

*We also wrote a Proof of Concept (PoC) Paper plugin to demonstrate the exploit in detail. Check it out in [here (TrackPack)](https://github.com/ALaggyDev/TrackPack).*

## Discovery

### Initial discovery

NikOverflow noticed that when a client joins cytooxien.net, the client would produce **around 28** of these error messages in the console:

![A bunch of resource pack loading errors](/assets/img/cytooxien/pack_error.png)

This is *really weird*, huh? The client was trying to download resource packs from `http://127.0.0.1:0`, for about **28 times**!?

We knew that *something was up*, and so I investigated furhter.

### Resource pack mystery

I checked the folder where server resource packs are cached. Luckily for me, the Minecraft client already maintained a log at `.minecraft/downloads/log.json`.

![Raw log](/assets/img/cytooxien/raw_log.png)

A bunch of resource pack download requests!

`http://127.0.0.1:15000/default/img/steve.png`, `https://resource.cytooxien.de/generate/{number}`, `http://127.0.0.1:0`, same hash and uuid, hmm... what was going on here!?

And this http endpoint - `https://resource.cytooxien.de/generate/{number}` - is *superrrr weird*. No matter what you put into `{number}`, it returns a zip file whose `pack.mcmeta` contains that exact string. Seriously, try it yourself.

```
{"pack":{"pack_format":22,"supported_formats":[22,1000],"description":"69420"}}
```
_the `pack.mcmeta` file inside the zip file from `https://resource.cytooxien.de/generate/69420`_

### Device fingerprinting

With a bit of thinking, I realized what was going on: the server was performing **device fingerprinting** on its players using the client's resource pack cache!

> [Device fingerprinting](https://en.wikipedia.org/wiki/Device_fingerprint) is a technique for uniquely identifying a device by collecting and analyzing various hardware and software characteristics. They are commonly used in websites to track users across different websites or IPs. Common techniques on the web involves cookies, user agents, screen resolution, IP addresses etc.
{: .prompt-info }

Essentially, the server is probing to see which resource packs are already cached locally and uses that to identify the device (or instance) the player is using.

The packs from `https://resource.cytooxien.de/generate/1` to `https://resource.cytooxien.de/generate/28` (currently 28) is used as *detection* resource packs. `http://127.0.0.1:0` is a url that always fails when requesting from it.

The flow of the exploit looks like this:
1. First, the server sends a series of bad resource pack requests with hash and uuid (packs 1–28) using the failing URL `http://127.0.0.1:0`.

2. If the server detects none of the packs are cached on the client, then:
- generate a "fingerprint" for the user (consisting of 6 numbers from 1 to 28 typically), and sends real resource pack requests with valid urls. The client will then downloads and caches them.
3. or else, use the information of which resource packs are cached to deduce the fingerprint.

By doing this, every player are attached an unique identifiable fingerprint. Because the resource pack cache is shared across different accounts, the exploit can be used to track players across different accounts or IPs. Truly mind-blowing.

Here's a simplified version of `.minecraft/downloads/log.json` for more clarity:
```json
// This is a simplified version of .minecraft/downloads/log.json.
// I have annotated certain events to make it clearer what the server is doing.

// ---------- First time joining the server ----------

// These 3 packs are irrelevant, for now
{"time":"...","hash":"4160bc...","file":{"name":"...","size":6930893},"id":"e36da0...","url":"https://fsn1.your-objectstorage.com/cxn-assets/production/resources/4160bc093503a1b7284955f3a471055f3f635803"}
{"time":"...","hash":"dad91b...","error":"download_failed","id":"0761e8...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"36c373...","error":"download_failed","id":"673f36...","url":"http://127.0.0.1:15000/default/img/steve.png"}

// The server sends bad pack requests. All pack requests fail because the packs are not cached on the client.
{"time":"...","hash":"00f8c2...","error":"download_failed","id":"6904fd...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"ecb87b...","error":"download_failed","id":"1ee63a...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"8f1f93...","error":"download_failed","id":"b4f462...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"8a280b...","error":"download_failed","id":"f6511e...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"094942...","error":"download_failed","id":"3dc90e...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"edd1b3...","error":"download_failed","id":"d4dbad...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"c395be...","error":"download_failed","id":"f21c86...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"bb0ecb...","error":"download_failed","id":"3bceb1...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"886e5a...","error":"download_failed","id":"462e33...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"59c04e...","error":"download_failed","id":"e9cd17...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"6205f8...","error":"download_failed","id":"ad6291...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"604eb8...","error":"download_failed","id":"11e2e7...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"bc06b6...","error":"download_failed","id":"52bd28...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"49944d...","error":"download_failed","id":"b3ffc6...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"c01585...","error":"download_failed","id":"df57d1...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"efdbf2...","error":"download_failed","id":"847629...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"70ff63...","error":"download_failed","id":"12b105...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"92a99a...","error":"download_failed","id":"979bb6...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"8d118f...","error":"download_failed","id":"610f8a...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"85d027...","error":"download_failed","id":"8bddb9...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"60ffe7...","error":"download_failed","id":"5ca653...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"9b1bc3...","error":"download_failed","id":"6a5186...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"44c064...","error":"download_failed","id":"28a765...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"abfe4b...","error":"download_failed","id":"8a7cc4...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"239e87...","error":"download_failed","id":"ce3af8...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"26d231...","error":"download_failed","id":"e300b5...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"67e2d5...","error":"download_failed","id":"402e52...","url":"http://127.0.0.1:0"}

// Since I am a "new player", the server "mark" me with the tags of [17, 26, 3, 27, 24, 11]
// The server deliberately sends out real pack requests, and the client caches them. A "fingerprint" belonging to the client is created.
{"time":"...","hash":"094942...","file":{"name":"...","size":222},"id":"3dc90e...","url":"https://resource.cytooxien.de/generate/17"}
{"time":"...","hash":"dad91b...","error":"download_failed","id":"65ef60...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"c01585...","file":{"name":"...","size":222},"id":"df57d1...","url":"https://resource.cytooxien.de/generate/26"}
{"time":"...","hash":"70ff63...","file":{"name":"...","size":221},"id":"12b105...","url":"https://resource.cytooxien.de/generate/3"}
{"time":"...","hash":"c395be...","file":{"name":"...","size":222},"id":"f21c86...","url":"https://resource.cytooxien.de/generate/27"}
{"time":"...","hash":"886e5a...","file":{"name":"...","size":222},"id":"462e33...","url":"https://resource.cytooxien.de/generate/24"}
{"time":"...","hash":"92a99a...","file":{"name":"...","size":222},"id":"979bb6...","url":"https://resource.cytooxien.de/generate/11"}


// ---------- Second time joining the server ----------

// These 3 packs are irrelevant, for now
{"time":"...","hash":"4160bc...","file":{"name":"...","size":6930893},"id":"e36da0...","url":"https://fsn1.your-objectstorage.com/cxn-assets/production/resources/4160bc093503a1b7284955f3a471055f3f635803"}
{"time":"...","hash":"dad91b...","error":"download_failed","id":"65ef60...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"36c373...","error":"download_failed","id":"6bb69d...","url":"http://127.0.0.1:15000/default/img/steve.png"}

// Some pack requests now succeed (because the client already cached them), while some pack requests still fail.
// Thus, the server can use these info to deduce the unique "fingerprint" belonging to the client.
{"time":"...","hash":"00f8c2...","error":"download_failed","id":"6904fd...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"ecb87b...","error":"download_failed","id":"1ee63a...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"8f1f93...","error":"download_failed","id":"b4f462...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"8a280b...","error":"download_failed","id":"f6511e...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"094942...","file":{"name":"...","size":222},"id":"3dc90e...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"edd1b3...","error":"download_failed","id":"d4dbad...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"c395be...","file":{"name":"...","size":222},"id":"f21c86...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"bb0ecb...","error":"download_failed","id":"3bceb1...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"886e5a...","file":{"name":"...","size":222},"id":"462e33...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"59c04e...","error":"download_failed","id":"e9cd17...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"6205f8...","error":"download_failed","id":"ad6291...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"604eb8...","error":"download_failed","id":"11e2e7...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"bc06b6...","error":"download_failed","id":"52bd28...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"49944d...","error":"download_failed","id":"b3ffc6...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"c01585...","file":{"name":"...","size":222},"id":"df57d1...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"efdbf2...","error":"download_failed","id":"847629...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"70ff63...","file":{"name":"...","size":221},"id":"12b105...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"92a99a...","file":{"name":"...","size":222},"id":"979bb6...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"8d118f...","error":"download_failed","id":"610f8a...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"85d027...","error":"download_failed","id":"8bddb9...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"60ffe7...","error":"download_failed","id":"5ca653...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"9b1bc3...","error":"download_failed","id":"6a5186...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"44c064...","error":"download_failed","id":"28a765...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"abfe4b...","error":"download_failed","id":"8a7cc4...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"239e87...","error":"download_failed","id":"ce3af8...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"26d231...","error":"download_failed","id":"e300b5...","url":"http://127.0.0.1:0"}
{"time":"...","hash":"67e2d5...","error":"download_failed","id":"402e52...","url":"http://127.0.0.1:0"}

// Notice that the server doesn't "mark" me anymore because the client has already cached some resource packs
```

### Toast notification

I quickly built a simple Proof of Concept plugin and tried the exploit myself. But actually, it doesn't fully work yet! Turns out, when the client fails to download a resource pack, Minecraft actually shows a toast notification in the top-right corner.

![Failed to download message](/assets/img/cytooxien/failed_to_download_msg.png)

This is bad because any player can notice that *something's up*. So how can we bypass this?


### Resource pack shenanigans

We had a suspicion of how they did it, but to verify it I needed to look inside of Cytooxien's main resource pack.

I tried to open Cytooxien's resource pack, but hmm:

![Windows Explorer can't unzip](/assets/img/cytooxien/unzip_explorer.png){: w="800" }
_Windows Explorer thinks there's nothing in this zip file_

![7zip can't unzip](/assets/img/cytooxien/unzip_7zip.png){: w="400" }
_7zip can't open this zip file_

![Linux can't unzip](/assets/img/cytooxien/unzip_linux.png){: w="800" }
_the `file` command also thinks there's nothing in this zip file_

Well, the zip file seems to be *intentionally broken* (persumably by [PackSquash](https://github.com/ComunidadAylas/PackSquash)?). This may stop some script kiddies or naive coders, but I am no ordinary script kiddies.

By finding the Minecraft code that unzips resource packs, I quickly wrote some code to unzip the resource pack the same way Minecraft does.

*I am not actually going to post my code here, because it is unethical to steal resource packs without permission, right?*

So anyway, I got the unzipped Cytooxien's main resource pack:

![Unzipped pack](/assets/img/cytooxien/unzipped_pack.png)

I opened the language file, and surely enough, I found what I had expected:

![Translation key overwrite](/assets/img/cytooxien/lang_overwrite.png)

They overwrote the toast messages with empty strings! And yes, the resource pack included language files for **all 134 languages** in Minecraft!

The background image of the toast was treated the same way, really. They overwrote the toast background to an transparent png image.

![A transparent toast background image](/assets/img/cytooxien/transparent_toast.png)

Technically the toast message is still there, but *you just can't see it*.

And with that, the main part of the exploit is done now! There's some extra nuanced things that the server is doing, but it is not that relevant right now.

> Furthermore, by checking the [Exif metadata](https://en.wikipedia.org/wiki/Exif) of `toast/system.png`, we knew that the exploit was created on `2024:01:24 21:54:47+01:00` (on GIMP, Linux). This exploit stayed under the radar for over 1.5 years!
{: .prompt-tip }

## Reaching out to the creator

After discovering this exploit, we wanted to talk to the creator of this exploit. It took us *a lot of tries*, including opening a ticket on the discord server, DM-ing multiple admins, but finally we got in touch with the creator of this exploit. Unfortunately, he wanted to remain anonymous and was quite unhappy that we choosed to publish this exploit.

We did know some technical details about this exploit through him, but unfortunately we can't disclose everything he told us.

## There's more???

There's still one thing unexplained though - **the resource pack request with the strange url `http://127.0.0.1:15000/default/img/steve.png`**. When asked, the creator refused to answer about this. Clearly, there's more we don't know yet, and we had to figure it out.

> Before you read the next part though, try figuring out the true purpose of this url yourself. It's a fun opportunity to test your research skills.
> <br>
> Scroll down when you're ready.
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> <br>
> The answer:
{: .prompt-info }

It took me an embarrassing amount of time to figure out the true purpose behind this. Turns out, if you search `"15000:/default"` on Google, a hacked client named [**LiquidBounce**](https://liquidbounce.net/) pops up.

![LiquidBounce](/assets/img/cytooxien/liquidbounce.png)

Apparently, when LiquidBounce is running, it exposes [an http server on port 15000 to serve theme-related files](https://github.com/CCBlueX/LiquidBounce/blob/c8a09554bc1a77fecb0a2983b7831fb686fa7120/src/main/kotlin/net/ccbluex/liquidbounce/integration/interop/ClientInteropServer.kt#L47). 🙄 The `steve.png` in question is located at [`LiquidBounce/src-theme/public/img/steve.png`](https://github.com/CCBlueX/LiquidBounce/blob/nextgen/src-theme/public/img/steve.png).

So by checking whether the download succeeds, the server can detect if a player is using LiquidBounce or not. For over a year, LiquidBounce users are being secretly tracked. Truly fascinating and terrifying.

_P.S. I have skipped over some minor things the server is also doing (e.g. detecting resource pack spoofers, resource pack cache overflow, checksum etc)._

## Finale

### How can players avoid being tracked?

If you are doing some *shady stuff* and don't want to be tracked by this exploit, here are some steps you can take:

- **Use separate Minecraft instances**: For different accounts, use seperate instances to prevent cache sharing. (e.g. PolyMC)
- **Clear your resource pack cache**: Regularly delete the contents of `.minecraft/downloads` to remove cached packs.

### For LiquidBounce users

Update your client, maybe? *Or use a different client lol*

### Conclusion

This exploit has massive implications to both players and server owners. It brings device fingerprinting, a technology mostly seen in browsers and phones, to the Minecraft landscape. It gives servers a powerful way to detect ban evasion or alternative accounts. I hope this post helps raise awareness of the security of resource pack handling and this new meta of device fingerprinting in Minecraft.

We also submitted a private bug report about this technique to the Mojang bug tracker, but so far we received no responses yet.

If you want to see a Proof of Concept of this technique, check out our [TrackPack PoC plugin](https://github.com/ALaggyDev/TrackPack).

Thanks for reading!

Our Github profiles:
- [Laggy](https://github.com/ALaggyDev)
- [NikOverflow](https://github.com/NikOverflow)
