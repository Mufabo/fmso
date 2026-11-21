---
title: "Cloud Gaming and Remote Desktop Streaming: Rent the Frames or Stream Your Own PC"
date: 2026-11-21
draft: true
description: "How cloud gaming and low-latency remote desktop streaming work, what changed in 2026 (hour caps on GeForce NOW, Xbox Cloud Gaming and Shadow), and how to self-host your own game streaming with Steam Remote Play, Sunshine and Moonlight, Apollo, Parsec or Wolf in containers."
summary: "Cloud gaming spent 2026 putting a meter on the clock: GeForce NOW, Xbox Cloud Gaming and Shadow all introduced hour caps. Meanwhile, self-hosted streaming with Sunshine, Moonlight and friends has never been better. Here is how game streaming works under the hood, what the commercial services offer now, and how to turn your own PC into a private cloud gaming server."
tags: ["cloud gaming", "game streaming", "remote desktop", "sunshine", "moonlight", "parsec", "geforce now", "xbox cloud gaming", "self-hosting", "docker", "linux"]
categories: ["Infrastructure"]
---

"Cloud gaming" and "remote desktop" sound like two different things. One is a consumer subscription that lets you play Cyberpunk on a phone; the other is what IT uses to fix your laptop. Under the hood they are the same idea: a powerful computer somewhere renders frames, compresses them into a video stream, and sends them to a thin client that sends keyboard, mouse and controller input back.

The difference is who owns the powerful computer. You can **rent the frames** from a service like GeForce NOW or Xbox Cloud Gaming, **rent a whole machine** from someone like Shadow, or **stream from a PC you already own** with tools like Sunshine and Moonlight, Steam Remote Play or Parsec.

2026 made that choice more interesting. The big cloud services all started metering playtime, while the open source self-hosting tools had one of their best years. This post covers how game streaming actually works, where the commercial services stand at the end of 2026, and how to build your own.

## How game streaming works

Every game streaming system, commercial or self-hosted, runs the same loop dozens of times per second:

1. **Render.** The host GPU renders a frame of the game.
2. **Capture.** The streaming host grabs that frame, ideally straight from GPU memory without a copy through the CPU.
3. **Encode.** The GPU's hardware video encoder (NVENC on NVIDIA, AMF/VCN on AMD, Quick Sync on Intel) compresses the frame with H.264, HEVC (H.265) or AV1.
4. **Send.** The compressed frame goes over the network, usually over UDP, because a late packet is worse than a lost one.
5. **Decode.** The client's hardware decoder turns it back into a picture.
6. **Display.** The client shows it on screen.
7. **Input.** Meanwhile the client sends your inputs back to the host, which feeds them into the game as if a local controller were plugged in.

The quality of the experience is almost entirely about **latency**: the time between pressing a button and seeing the result. Each step above adds a few milliseconds. On a good wired home network, the streaming overhead on top of the game's own latency can be in the region of 10 to 20 ms, which most people can't feel outside of competitive shooters. Over the internet to a data centre, network round-trip time is added on top, and that depends mostly on how far away the server is and how good your connection is.

A few rules of thumb fall out of this:

- **The network matters more than the GPU.** A wired Ethernet connection, or a clean 5 GHz / 6 GHz Wi-Fi link, makes far more difference than a faster graphics card.
- **Codec choice is a trade-off.** H.264 decodes on anything. HEVC gives better quality at the same bitrate. AV1 is better still, but needs a recent GPU on the host (NVIDIA RTX 40 series, AMD RX 7000, Intel Arc or newer) and an AV1 hardware decoder on the client.
- **Bitrate is cheap on a LAN and expensive over the internet.** At home you can push 50 to 150 Mbit/s for near-lossless 4K. Remotely you're limited by your upload speed.

### Game streaming vs. classic remote desktop

Classic remote desktop protocols like RDP and VNC were designed for office work: they send drawing commands or changed screen regions, and they're very efficient when most of the screen stays still. That falls apart when the whole screen changes 60 or 120 times per second, as it does in a game.

Game streaming tools take the opposite approach: they treat the screen as video, always. That's wasteful for a spreadsheet, but it gives consistent latency and frame pacing for anything that moves. It's also why tools like Parsec and Apollo have become popular for **non-gaming** work too: video editing, 3D work, and anyone who wants a remote desktop that feels local.

## The commercial services in late 2026

### The theme of the year: hour caps

The headline story of 2026 is that "unlimited" cloud gaming ended.

**GeForce NOW** started it. From 1 January 2026, almost all Performance and Ultimate subscribers are limited to 100 hours of play per month, with paid top-ups beyond that[^1]. New subscribers had been under the cap since early 2025; existing ones had a 12-month exemption that ran out at the start of 2026[^2]. Only long-time Founders members with continuous memberships are exempt[^2].

**Xbox Cloud Gaming** followed in November. Microsoft announced on 3 September 2026 that Game Pass would switch to monthly cloud allowances from November: 15 hours on Ultimate, 10 on Premium and 5 on Essential[^3][^4]. Microsoft says the change affects about 4% of Game Pass subscribers[^5]. At the same time, Xbox started selling cloud playtime packs to people without Game Pass, tested first with Xbox Insiders at introductory prices of $3.99 for 10 hours, $7.99 for 25 hours and $11.99 for 50 hours[^6].

**Shadow**, the French cloud PC provider, introduced cheaper "Lite" plans in July 2026 with a 120-hour monthly cap[^7].

The reason is simple economics. A cloud gaming session ties up a real GPU in a data centre for as long as you play. A small number of heavy users can cost far more than they pay, and the hour caps are the services admitting it.

### GeForce NOW: rent the frames

GeForce NOW doesn't sell games. You connect your own libraries from Steam, Epic, Ubisoft Connect and other stores, and NVIDIA runs the supported games on its servers. It comes in three tiers: Free (with ads), Performance at $9.99/month and Ultimate at $19.99/month[^8]. Since late 2025, Ultimate runs on Blackwell RTX 5080-class hardware with the full DLSS and frame generation feature set[^9]. Session length also grows with the tier, from one hour on Free up to eight hours on Ultimate[^10].

**Good for:** people who own games on PC storefronts but not a gaming PC, and who play less than about 100 hours a month.

### Xbox Cloud Gaming: part of Game Pass

Xbox Cloud Gaming streams games from the Game Pass catalogue rather than from your own library. After a price cut in April 2026, Game Pass Ultimate costs $22.99/month in the US[^11]. With the new 15-hour monthly allowance, cloud gaming is now clearly a side feature of Game Pass rather than a way to replace a console.

**Good for:** Game Pass subscribers who want to play a bit on a phone, tablet or TV away from the console.

### Shadow: rent the whole machine

Shadow is different from the services above: you don't get a game catalogue, you get a full Windows PC in a data centre with a dedicated GPU, and you can install anything you like on it, including games from any store, mods, emulators or work software[^12]. That flexibility costs more: Shadow's main gaming plans sit around $30 to $40 a month[^7][^13].

**Good for:** people who want a real Windows PC they can do anything with, without buying the hardware.

### Others

**Boosteroid**, a Ukrainian-founded service, competes on price with its own data centre network, though its prices also went up in 2026[^13]. **PlayStation Plus Premium** includes cloud streaming of parts of the PlayStation catalogue. And everyone still remembers **Google Stadia**, which launched in November 2019 and shut down on 18 January 2023[^14]. Google refunded hardware and game purchases[^14], but the lesson stuck: if the service goes away, so does your access.

## Self-hosted streaming: your own cloud

If you already own a capable gaming PC, you can stream from it to any other screen in the house, or anywhere with a decent internet connection. The only recurring cost is electricity, there are no hour caps, and every game you own works, including mods and emulators.

### Steam Remote Play: zero setup

If all your games are on Steam, start here. Remote Play is built into the Steam client, and the free Steam Link app acts as the client on phones, tablets, TVs and other PCs. It supports up to 4K at 60 fps with H.264 and HEVC[^15]. The trade-offs: it works best with Steam games, has fewer tuning options, and no AV1.

Valve is betting heavily on this kind of streaming. Its **Steam Frame** VR headset, launched on 14 September 2026, is explicitly "streaming-first": it ships with a dedicated 6 GHz Wi-Fi 6E adapter for the PC, and uses eye tracking for **foveated streaming**, sending the highest quality only to where you're looking[^16][^17]. That works for flat-screen games too, not just VR.

### Sunshine and Moonlight: the open source standard

The most popular self-hosted setup is a pair of open source projects:

- **Moonlight** is the client. It started life as an open source reimplementation of NVIDIA's GameStream protocol, the technology behind the NVIDIA Shield, and runs on Windows, macOS, Linux, Android, iOS, Apple TV, the Steam Deck and more[^18].
- **Sunshine**, by the LizardByte project, is the host. It runs on the gaming PC, captures the screen, encodes with the GPU and streams to Moonlight. Unlike NVIDIA's original GameStream, it supports AMD, Intel and NVIDIA GPUs, with software encoding as a fallback[^19]. It's GPL-3.0 licensed[^18].

When NVIDIA retired GameStream in early 2023, Sunshine became the default replacement, and it has kept improving since. The May 2026 release alone brought Vulkan-based encoding and PipeWire and KWin capture on Linux, ARM64 support and signed installers on Windows, a signed macOS package with audio capture, a modernised web UI with per-client access control, and split-frame encoding on GPUs with more than one NVENC unit[^20]. Further releases followed in October, alongside Moonlight 6.2[^21]. Moonlight itself recently fixed several vulnerabilities that could let a malicious host crash the client[^22], a reminder to keep both ends updated.

### Apollo and Artemis: virtual displays

The most annoying thing about classic game streaming is resolution. If your PC is plugged into a 4K monitor and you stream to a 1080p phone in portrait mode, you either get letterboxing or end up fiddling with display settings every time.

**Apollo**, a Sunshine fork by ClassicOldSong, solves this with a built-in virtual display. When a client connects, Apollo creates a virtual monitor that matches the client's resolution, refresh rate and HDR setting, and removes it again when you disconnect[^23]. It pairs with **Artemis**, a Moonlight fork with matching client-side controls[^24]. The catch is that the virtual display relies on the SudoVDA driver and works only on Windows[^23]. The author has also said Apollo and Artemis will eventually stop being compatible with upstream Sunshine and Moonlight[^23].

The fork has itself been forked: **Helios** is a community fork of Apollo whose main goal is a native virtual display on Linux and Wayland[^25], and there are smaller scripts that replicate the same behaviour on KDE Plasma or Hyprland with a headless monitor[^26].

### Parsec: polished, but proprietary

**Parsec**, owned by Unity since 2021, is a proprietary peer-to-peer remote desktop and game streaming tool. It's known for low latency, easy NAT traversal (no port forwarding needed) and a good experience for remote co-op, where friends connect to your PC to play local multiplayer games over the internet. The free Personal plan is licensed for personal, non-commercial use only; commercial use needs a Teams or Enterprise plan[^27].

### Wolf and Games on Whales: streaming from containers

This is the option that will appeal to the Kubernetes crowd. **Games on Whales** is a project to stream games and other graphical applications running in Docker containers with hardware acceleration[^28]. Its streaming server, **Wolf**, speaks the Moonlight protocol, but instead of streaming a single desktop it starts a preconfigured container (Steam, Firefox, RetroArch and others) for each client and streams that[^29].

That means **one machine can serve several players at once**, each in their own isolated session, with games running as low-privilege containers[^30]. You can even encode on an integrated GPU while the game runs on the dedicated one[^31]. Configuration lives in a single file that controls the encoding pipelines, GPU settings and Docker or Podman details[^30]. One warning from its own documentation: Wolf's REST API must never be exposed over TCP[^32].

It's a natural fit for a homelab: a single beefy GPU server in a rack, and every TV, laptop and handheld in the house as a thin client.

## Comparison

| | Type | Cost | Your own games? | Hour cap | Open source |
| --- | --- | --- | --- | --- | --- |
| **GeForce NOW** | Cloud, rent the frames | Free / $9.99 / $19.99 per month | Yes, from supported stores | 100 h/month (paid tiers) | No |
| **Xbox Cloud Gaming** | Cloud, catalogue | Part of Game Pass ($22.99 for Ultimate) | Game Pass catalogue | 5 / 10 / 15 h/month | No |
| **Shadow** | Cloud, rent a PC | Roughly $30–40 per month | Anything you install | 120 h on Lite plans | No |
| **Steam Remote Play** | Self-hosted | Free | Steam library | None | No |
| **Sunshine + Moonlight** | Self-hosted | Free | Anything on your PC | None | Yes (GPL-3.0) |
| **Apollo + Artemis** | Self-hosted | Free | Anything on your PC | None | Yes (GPL-3.0) |
| **Parsec** | Self-hosted | Free for personal use | Anything on your PC | None | No |
| **Wolf / Games on Whales** | Self-hosted, containers | Free | Containerised apps | None | Yes |

## Setting up self-hosted streaming well

Whatever host you choose, the same handful of things decide whether it feels local or laggy:

- **Wire the host.** The gaming PC should always be on Ethernet. Ideally the client is too; otherwise use 5 GHz or 6 GHz Wi-Fi close to the access point.
- **Use hardware encoding.** Make sure the host is using NVENC, AMF/VCN or Quick Sync, not software encoding. Software encoding eats CPU and adds latency.
- **Pick the best codec both sides support.** AV1 if host and client can do it in hardware, otherwise HEVC, with H.264 as the fallback.
- **Match resolution and frame rate to the client.** Streaming 4K to a 7-inch handheld wastes bandwidth and encoder time. Apollo's virtual display, or a dedicated headless display on Linux, makes this automatic.
- **Don't open ports for remote access.** Rather than port-forwarding Sunshine to the internet, put host and client on the same WireGuard-based VPN (Tailscale, Headscale, plain WireGuard). You get encryption and authentication for free, and only devices you control can reach the host.
- **Keep both ends updated.** Sunshine and Moonlight have both shipped security fixes this year[^20][^22].
- **Enable Wake-on-LAN.** Moonlight can wake a sleeping host, so the PC doesn't need to run all day.

## Which one should you pick?

- **You don't own a gaming PC and play casually:** GeForce NOW, as long as you stay under 100 hours a month.
- **You already pay for Game Pass Ultimate:** use Xbox Cloud Gaming for the odd session away from the console; 15 hours a month won't replace it.
- **You need a full Windows machine, not just games:** Shadow.
- **You own a gaming PC and play mostly Steam games at home:** Steam Remote Play.
- **You own a gaming PC and want the best quality and flexibility:** Sunshine and Moonlight.
- **You stream mostly to phones, tablets or handhelds from Windows:** Apollo and Artemis, for the automatic virtual display.
- **You want friends to join your local co-op games remotely:** Parsec.
- **You run a homelab and want several people on one GPU server:** Wolf and Games on Whales.

## Closing thoughts

Cloud gaming in 2026 has settled into what it probably always had to be: a good way to play occasionally without the hardware, priced by the hour like any other cloud resource. That's a reasonable product, but it's not the "end of the gaming PC" that Stadia promised in 2019.

The more interesting development is on the other side. Your own PC is now a perfectly good cloud gaming server, the software to run it is free and open source, and it keeps getting better: virtual displays, AV1, container-based multi-user hosting, and even dedicated hardware like the Steam Frame built around streaming from a PC you own. If you have a gaming PC sitting idle while you're on the couch, in bed or away from home, it's worth an afternoon to set it up.

## References

[^1]: [GeForce Now's 100-hour monthly game streaming limit kicks in for (almost) everyone in 2026](https://9to5google.com/2025/12/23/nvidia-geforce-now-100-hour-limit-2026/), 9to5Google.
[^2]: [Nvidia imposes 100-hour monthly cap to GeForce Now users](https://respawn.outlookindia.com/amp/story/gaming/gaming-news/nvidia-imposes-100-hour-monthly-cap-to-geforce-now-users), Outlook Respawn.
[^3]: [Upcoming Changes to Xbox Cloud Gaming](https://news.xbox.com/en-us/2026/09/03/xbox-cloud-gaming-changes/), Xbox Wire, 3 September 2026.
[^4]: [Xbox to cap cloud gaming hours for Game Pass subscribers](https://www.gamedeveloper.com/cloud/xbox-to-cap-cloud-gaming-hours-for-game-pass-subscribers), Game Developer.
[^5]: [Microsoft's recent decisions paint a clear picture: Xbox Cloud Gaming is no longer the future the company once promised](https://windowscentral.com/gaming/xbox/microsoft-clearly-doesnt-see-much-of-a-future-for-xbox-cloud-gaming), Windows Central.
[^6]: [Xbox Reveals Prices for Cloud Gaming Packs as Game Pass Subscribers Face Monthly Limits](https://beebom.com/xbox-cloud-gaming-pack-prices-revealed/amp/), Beebom; pack prices from [Xbox Cloud Gaming hour limits](https://tech-insider.org/?p=26361), Tech Insider.
[^7]: [How to set up Shadow PC cloud gaming](https://tech-insider.org/how-to-set-up-shadow-pc-cloud-gaming-2026/), Tech Insider: Shadow Lite plans and pricing.
[^8]: [NVIDIA GeForce Now at GDC 2026](https://www.windowscentral.com/hardware/nvidia/nvidia-geforce-now-new-gdc-2026), Windows Central.
[^9]: [GeForce NOW 2025 year in review](https://blogs.nvidia.com/?p=88570), NVIDIA blog.
[^10]: [GeForce Now tiers 2026](https://tech-insider.org/geforce-now-tiers-2026/), Tech Insider.
[^11]: [Xbox Game Pass Ultimate with cloud gaming gets a price cut](https://9to5google.com/2026/04/21/xbox-game-pass-ultimate-price-cut/), 9to5Google.
[^12]: [Shadow PC](https://shadow.tech/), Shadow.
[^13]: [GeForce NOW vs Boosteroid vs Shadow PC](https://tech-insider.org/geforce-now-vs-boosteroid-vs-shadow-pc-2026/), Tech Insider.
[^14]: [Stadia Announcement FAQ](https://support.google.com/stadia/answer/9810862?hl=en), Google; launch date from [Google confirms Stadia is shutting down](https://www.nme.com/news/google-confirms-stadia-is-shutting-down-3319876), NME.
[^15]: [Steam Link and Steam Remote Play](https://tech-insider.org/sv/?p=1921), Tech Insider (Swedish), citing Valve's Remote Play page.
[^16]: [Valve Steam Frame: release date, price, specs](https://vr.org/steam-frame), VR.org.
[^17]: [Valve's Steam Frame VR headset is finally official](https://www.engadget.com/ar-vr/valves-steam-frame-vr-headset-is-finally-official-and-its-coming-in-2026-181909387.html), Engadget.
[^18]: [Moonlight game streaming setup](https://tech-insider.org/moonlight-game-streaming-setup-2026/), Tech Insider; and [Sunshine documentation](https://www.mintlify.com/LizardByte/Sunshine).
[^19]: [LizardByte/Sunshine](https://github.com/LizardByte/Sunshine), GitHub.
[^20]: [Sunshine game streaming tool adds Vulkan encoding plus XDG PipeWire and KWin direct screencast capture](https://www.gamingonlinux.com/2026/05/sunshine-game-streaming-tool-adds-vulkan-encoding-plus-xdg-pipewire-and-kwin-direct-screencast-capture/), GamingOnLinux.
[^21]: [Sunshine and Moonlight game streaming setup](https://tech-insider.org/sunshine-moonlight-game-streaming-setup-2026/), Tech Insider.
[^22]: [Moonlight on Flathub](https://flathub.org/en/apps/com.moonlight_stream.Moonlight), Flathub: release notes.
[^23]: [Apollo](https://github.com/ClassicOldSong/Apollo), ClassicOldSong on GitHub.
[^24]: [Artemis Android](https://github.com/fsesma/artemis), GitHub mirror of the Artemis client.
[^25]: [Helios](https://github.com/unjordi/Helios), GitHub.
[^26]: [sunshine-virtual-display](https://github.com/ImStillBlue/sunshine-virtual-display) and [sunshine-hyprland-virtual-display](https://github.com/jhonsnake/sunshine-hyprland-virtual-display), GitHub.
[^27]: [Is Parsec free?](https://www.lilachbullock.com/is-parsec-free/), Lilach Bullock, summarising Parsec's pricing page (October 2026).
[^28]: [Games on Whales overview](https://games-on-whales.github.io/gow/overview.html), Games on Whales.
[^29]: [Wolf: Apps](https://games-on-whales.github.io/wolf/stable/apps/index.html), Games on Whales.
[^30]: [Wolf documentation](https://games-on-whales.github.io/wolf/stable), Games on Whales.
[^31]: [Wolf README](https://github.com/games-on-whales/wolf), games-on-whales on GitHub.
[^32]: [Wolf](https://www.positioniseverything.net/tools/wolf/), Position Is Everything, summarising the Wolf documentation.
