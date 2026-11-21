---
title: "Turning Old Smartphones into Linux Servers"
date: 2026-11-05
draft: true
description: "Three ways to turn a retired Android phone into a small, low-power Linux server: Termux on stock Android, Android's built-in Linux Terminal, and replacing Android entirely with Nura (formerly postmarketOS). Includes how to choose a phone, step-by-step setup, networking, battery safety and what to run on it."
summary: "That phone in your drawer has an eight-core ARM CPU, several gigabytes of RAM, Wi-Fi and a built-in UPS. Here is how to turn it into a real Linux server: what works on stock Android, when to replace Android with mainline Linux, and how to deal with the parts that make phones awkward servers, starting with the battery."
tags: ["linux", "homelab", "android", "postmarketos", "nura", "termux", "self-hosting", "e-waste"]
categories: ["Programming"]
---

Most of us have at least one old phone in a drawer. A flagship from 2018 or 2019 has an eight-core 64-bit ARM CPU, 4 to 8 GB of RAM, fast flash storage, Wi-Fi, Bluetooth, a camera, a screen, and a battery that doubles as an uninterruptible power supply. It idles at a watt or two. As a computer, it is more capable than a Raspberry Pi 4, and it's free because you already own it.

Phones are also awkward servers. They are designed to sleep aggressively, kill background work, and run a locked-down vendor kernel that stopped getting updates years ago. They have no Ethernet port, a battery that was never meant to sit at 100% for years, and a USB port that has to do power, networking and storage at once.

This post covers the three realistic ways to turn an old phone into a Linux server, when to pick each one, and how to deal with the parts that make phones awkward: power management, networking and, most importantly, the battery.

Everything here is current as of November 2026: Android 16/17, Termux from F-Droid, and Nura v26.06 (the distribution that until September 2026 was called postmarketOS).

## Three ways to get Linux on a phone

| | Termux on Android | Android's Linux Terminal | Nura / postmarketOS |
| --- | --- | --- | --- |
| What it is | Linux userland running as an Android app | Debian in a VM via the Android Virtualization Framework | Android replaced by Alpine-based Linux |
| Root / unlocked bootloader | No | No | Unlocked bootloader required |
| Kernel | Android's (old vendor kernel) | A real Linux kernel in the VM | Mainline Linux (on good ports) |
| Init system / services | No real init; `termux-services` | Debian with systemd inside the VM | OpenRC or systemd |
| Containers (Docker, k3s) | No | Possible, with limits | Yes |
| Phones supported | Practically any Android 7+ phone | Recent Pixels and a few other new phones | A few hundred models, quality varies |
| Android security updates needed | Yes, and old phones don't get them | Yes | No: you get Linux distro updates |
| Best for | Trying it out, light services | Recent phones you still use | A dedicated, always-on server |

The rest of the post walks through each, but the short version is:

- **If your phone is supported well by Nura/postmarketOS, use that.** You get a normal Linux distribution with a mainline kernel, a real init system, a firewall, a package manager and security updates.
- **If it isn't, Termux works on almost anything**, as long as you accept that Android is in charge and can kill your processes.
- **The Android Linux Terminal is interesting but rarely relevant here**, because it needs recent hardware, and the point of this exercise is old hardware.

## Option 1: Termux on stock Android

[Termux](https://termux.dev/) is a terminal emulator and Linux environment that runs as a normal Android app. It has its own package repository with thousands of packages built for Android: OpenSSH, Python, Node.js, Go, nginx, PostgreSQL, rsync and so on. No root is needed.

### Setting it up

Install Termux **from F-Droid or the project's GitHub releases**, not the Play Store build, which is maintained separately and lags behind. Then:

```bash
pkg update && pkg upgrade
pkg install openssh termux-services termux-api
passwd                 # set a password for SSH
whoami                 # note the generated username, e.g. u0_a123
sshd                   # starts the SSH server on port 8022
```

From another machine on the same network:

```bash
ssh -p 8022 u0_a123@192.168.1.50
```

Copy your public key into `~/.ssh/authorized_keys` and then disable password logins in `$PREFIX/etc/ssh/sshd_config`.

### Keeping it alive

This is where Termux on Android gets difficult. Android does everything it can to stop apps running in the background, and a server is nothing but background work. You need to deal with four separate mechanisms:

1. **CPU sleep.** Run `termux-wake-lock` (or tap *Acquire wakelock* in the Termux notification). Without it, the CPU sleeps when the screen is off and your server stops answering.
2. **Battery optimisation.** In Android settings, set Termux's battery usage to *Unrestricted*. Some vendors (Samsung, Xiaomi, Huawei, OnePlus) add their own extra layers of app killing; [dontkillmyapp.com](https://dontkillmyapp.com/) documents them per manufacturer.
3. **The phantom process killer.** Since Android 12, the system kills child processes of apps when there are more than 32 of them across all apps, or when one uses too much CPU. Termux's shell and everything you start from it are such child processes. The symptom is `[Process completed (signal 9) - press Enter]` with no apparent cause. Recent Android versions have a *Disable child process restrictions* toggle in Developer options; where that is missing, you can turn the monitor off over ADB (wireless debugging works, even from Termux itself after `pkg install android-tools`):

   ```bash
   adb shell "settings put global settings_enable_monitor_phantom_procs false"
   ```

   On Android 12 (not 12L) and some vendor ROMs that setting is not honoured, and you also need:

   ```bash
   adb shell "/system/bin/device_config set_sync_disabled_for_tests persistent"
   adb shell "/system/bin/device_config put activity_manager max_phantom_processes 2147483647"
   ```

4. **Reboots.** Install the **Termux:Boot** add-on (from the same source as Termux), open it once, and put a startup script in `~/.termux/boot/`:

   ```bash
   mkdir -p ~/.termux/boot
   cat > ~/.termux/boot/start-services <<'EOF'
   #!/data/data/com.termux/files/usr/bin/sh
   termux-wake-lock
   . $PREFIX/etc/profile
   sv-enable sshd
   EOF
   chmod +x ~/.termux/boot/start-services
   ```

`termux-services` gives you a small runit-based service manager: `sv-enable sshd`, `sv up nginx`, `sv status`, logs under `$PREFIX/var/log/sv/`.

### A full distribution with proot-distro

If you want Debian, Ubuntu, Alpine or Arch instead of Termux's own packages, `proot-distro` installs one in a directory and runs it with `proot`, which fakes a root filesystem by intercepting system calls:

```bash
pkg install proot-distro
proot-distro install debian
proot-distro login debian
```

You get `apt` and the whole Debian archive. But it's an illusion of a system rather than a system: there is no real root, no systemd, no namespaces or cgroups (so no Docker), and every system call pays a performance tax.

### Termux's limits

- **Ports below 1024 are off-limits** without root. Use 8080/8443 and a reverse proxy elsewhere, or port forwarding on your router.
- **No containers.** Docker and Podman need kernel features apps can't use.
- **Android owns the network.** You can't add a bridge, run WireGuard as a kernel interface (the userspace `wireguard-go` and Tailscale's userspace mode work), or control which interface is used.
- **Security updates.** Termux packages are kept up to date, but the Android underneath is whatever your phone last got from its vendor, possibly years ago. Never expose it to the internet directly.

Termux is the right choice for trying the idea out, for phones nothing else supports, and for light services on your LAN: an SSH jump host, a Syncthing node, a small web app, a cron job that runs scripts.

## Option 2: Android's built-in Linux Terminal

Starting with Android 16, Google ships a **Terminal** app (enabled in Developer options) that boots a Debian virtual machine using the **Android Virtualization Framework (AVF)**: Google's crosvm virtual machine monitor on top of the pKVM hypervisor. Unlike Termux, this is a real Linux kernel with systemd, real root inside the VM, and port forwarding from the VM to the phone that the app lets you configure.

The catch for our purposes is hardware. The Terminal app needs a device whose hypervisor allows non-protected VMs, which in practice means recent Pixels (Pixel 6 and later) and a handful of newer flagships. Snapdragon phones on stock firmware are generally not supported. If you have a retired Pixel 6 or 7, it's worth a try; for the typical drawer phone from 2018, it's not an option.

It also inherits Android's background restrictions: the VM is tied to the Terminal app, which Android can and will stop.

## Option 3: Replace Android with Nura (formerly postmarketOS)

**Nura** is a Linux distribution for phones and tablets built on Alpine Linux. Until September 2026 it was called **postmarketOS**; the project renamed itself at its first conference after an 18-month process, mostly because the old name was hard to pronounce, remember and trademark. The website moved to `nura.eco` and the old domain redirects there, but much of the tooling and documentation still uses the old name (the install tool is still `pmbootstrap`, the wiki still lives at `wiki.postmarketos.org`), so you'll see both for a while.

The project's goal since 2017 has been a ten-year lifecycle for phones. Instead of running a Linux userland on top of Android, it replaces Android entirely. On well-supported devices it runs a **mainline** Linux kernel, the same kernel source as your laptop, rather than the frozen vendor kernel the phone shipped with. That gives you everything Termux lacks:

- a real init system (OpenRC by default, systemd as an option), so services start at boot and get restarted;
- Alpine's `apk` package manager and repositories, plus Nura's own packages;
- a firewall (nftables) enabled out of the box;
- root, namespaces and cgroups, so Docker, Podman and k3s work;
- regular security updates, with a stable release every six months (June and December, a month after Alpine's) or the rolling `edge` branch.

People run all sorts of things on these: Home Assistant, Jellyfin in a k3s cluster on a OnePlus 6T, and an offline Wi-Fi hotspot with a captive portal on a Pixel 3a XL that streamed 720p video to four devices at around 5% CPU and roughly 2.4 W.

### Choosing a phone

Device support is the deciding factor, and it varies enormously. The wiki's [devices page](https://wiki.postmarketos.org/wiki/Devices) lists every port with a feature matrix, sorted into categories:

- **main**: the best supported. Currently empty; the project removed the devices that used to be here and plans to readmit devices against a stricter standard.
- **community**: maintained ports that get stable release images. **Pick one of these if you can.**
- **testing**: may boot and work well enough for a server, but with no guarantees.
- **downstream**: runs on the old Android vendor kernel. Avoid for a server: you lose most of the benefits.

For a server you don't care about calls, audio, the camera or the modem. What matters is: boots reliably, **Wi-Fi works**, **USB networking** works, the **battery and charging** drivers work, and ideally **USB OTG** works for an Ethernet adapter or external disk.

Device support changes with every release, so check the [official device list](https://wiki.postmarketos.org/wiki/Devices) and your phone's own wiki page rather than relying on a list in a blog post. The device page tells you which category the port is in, which features work, the device's codename (which you'll need for `pmbootstrap`), and any device-specific installation steps.

The most important prerequisite: **the bootloader must be unlockable**. Many carrier-sold phones (US Verizon and AT&T models especially) and most US Samsung models can't be unlocked. Unlocking also wipes the phone, so back up anything you still want.

### Installing

There are two routes: download a prebuilt image from the website for your device and flash it, or build your own with **`pmbootstrap`**, the project's build and install tool. For a server, `pmbootstrap` is worth it because you can choose a console-only system with no graphical interface at all.

`pmbootstrap` runs on any Linux machine (x86_64 or aarch64) and is available from PyPI and many distribution repositories:

```bash
pipx install pmbootstrap      # or your distribution's package
pmbootstrap init
```

`init` asks a series of questions. The important answers for a server:

- **Channel**: `v26.06` (stable) or `edge` (rolling, newer packages, occasionally broken).
- **Vendor / device**: e.g. `oneplus` / `enchilada`.
- **User interface**: `console`. No graphical shell, minimal RAM use.
- **Extra packages**: e.g. `openssh-server,nftables,htop`.
- **systemd**: say no for a console install unless you specifically want it; OpenRC is the default.

Then unlock the bootloader (usually `fastboot flashing unlock`; check your device's wiki page, since some phones need a vendor tool or a second-stage bootloader like `lk2nd`), reboot into fastboot mode, and:

```bash
pmbootstrap install                 # add --fde for full-disk encryption
pmbootstrap flasher flash_rootfs
pmbootstrap flasher flash_kernel
fastboot reboot
```

On first boot the phone exposes a USB network interface. Plug it into your computer and SSH in over USB:

```bash
ssh user@172.16.42.1
```

(`user` is whatever username you chose during `init`.)

### Making it a server

**Wi-Fi.** NetworkManager handles networking:

```bash
sudo nmcli device wifi connect "MyNetwork" password "secret"
nmcli connection show
```

The connection is saved and reconnects at boot. Give the phone a fixed address with a DHCP reservation on your router so you can always find it.

**SSH.** Put your key in `~/.ssh/authorized_keys`, then set `PasswordAuthentication no` in `/etc/ssh/sshd_config` and make sure the service starts at boot:

```bash
sudo rc-update add sshd default
sudo rc-service sshd restart
```

**Services.** With OpenRC, anything you install is enabled and started the same way:

```bash
sudo apk add nginx
sudo rc-update add nginx default
sudo rc-service nginx start
rc-status                 # what is running
```

**Firewall.** The firewall is on by default and only allows a small set of things, notably SSH. This is the most common "it's running but I can't reach it" problem. Open ports by dropping a file into `/etc/nftables.d/`:

```bash
sudo tee /etc/nftables.d/50_web.nft <<'EOF'
table inet filter {
    chain input {
        tcp dport { 80, 443 } accept comment "web server"
    }
}
EOF
sudo rc-service nftables restart
```

**Containers.** Docker (`apk add docker`, `rc-update add docker default`), Podman and k3s all work on mainline ports. Remember you're on `aarch64` and Alpine (musl): use multi-arch images, and expect the occasional Java or glibc-only binary to need extra work.

**Keep the screen off.** With the console UI, the screen shows a text console. Blank it and save some power by turning the backlight off (the path varies by device):

```bash
ls /sys/class/backlight/
echo 0 | sudo tee /sys/class/backlight/*/brightness
```

**Updates.** `sudo apk upgrade -a` regularly, and plan for the twice-yearly release upgrades (or stay on `edge`).

### What about other distributions?

Nura has by far the widest device support, but if you prefer something else:

- **Mobian** (Debian) and **Pocketblue** (Fedora Atomic) support some of the same mainline devices, notably the OnePlus 6/6T and Pixel 3a.
- **Droidian** and **Ubuntu Touch** run a Linux userland on top of the Android vendor kernel through Halium. They support more phones, but you get the old kernel, which matters a lot less on a phone you carry than on a server.

## Networking: Wi-Fi, USB, Ethernet

Phones have no Ethernet port, and that is the second-biggest compromise after the battery.

- **Wi-Fi** is simplest and fine for most home services. It adds a few milliseconds of latency and occasional drops. Put the phone close to the access point and disable Wi-Fi power saving if it causes stalls (`nmcli connection modify "MyNetwork" wifi.powersave 2` on Nura).
- **USB Ethernet adapter** gives you a wired connection, if the phone supports USB OTG on mainline. The problem: most phones can't act as a USB host and charge at the same time. A USB-C hub with power delivery pass-through *sometimes* fixes this; whether it does depends on the phone, the hub and the kernel drivers. Test before you rely on it.
- **USB networking to another computer** (the `172.16.42.1` interface) is always there on Nura, which makes it a good management fallback when Wi-Fi breaks.

For access from outside your home, don't forward ports to the phone. Use a VPN: **WireGuard** (kernel module on Nura, userspace on Termux) or **Tailscale**, which works in both.

## The battery: the part to take seriously

Lithium-ion batteries degrade fastest when they are kept **full**, **hot**, or both. A phone used as a server is plugged in permanently, so it sits at 100% charge, often in a warm cupboard, for years. The result can be a swollen battery. In the worst case, an old, damaged battery can fail and catch fire. This is not hypothetical: it is the main reason to treat a phone server differently from a Raspberry Pi.

Some ways to reduce the risk, from easiest to most involved:

1. **Inspect the battery before you start.** If the phone's back or screen is bulging, or the case doesn't sit flat, the battery is already swollen. Don't use it; replace the battery or recycle the phone.
2. **Keep it cool and in the open.** On a shelf with airflow, not in a closed drawer, not in direct sunlight, and not on something flammable. A metal shelf or ceramic tile underneath is cheap insurance.
3. **Limit the charge level.**
   - On stock Android, many phones now have a built-in charging limit (often 80%) in battery settings. Use it. Rooted phones can use tools like ACC to set custom thresholds.
   - On Nura, check whether your charger driver exposes a threshold in sysfs:

     ```bash
     ls /sys/class/power_supply/
     cat /sys/class/power_supply/*/capacity
     ls /sys/class/power_supply/*/ | grep -E 'charge_control|constant_charge'
     ```

     If you see `charge_control_end_threshold` or `constant_charge_voltage_max`, you can lower it (and persist the setting with a boot script). If not, one person in early 2026 went further and changed the device tree on their postmarketOS phone to cap the maximum charge voltage at 3.8 V instead of 4.4 V, holding the battery at roughly 40–50%.
   - Without any of that, use a **smart plug** controlled by Home Assistant or a cron job on another machine: switch the charger on at, say, 40% and off at 80%. The phone reports its battery level; the plug does the rest.
4. **Remove the battery and power the phone directly.** For a phone that will never move again, this is the safest option, but it's real electronics work: phones generally won't boot from USB alone with the battery removed, so people feed a regulated supply into the battery connector or charge controller. Only do this if you know what you're doing.

## What to run on it

A phone has plenty of CPU for anything that isn't heavy on storage or I/O. Good fits:

- **DNS-level ad blocking**: Pi-hole or AdGuard Home.
- **Home automation**: Home Assistant (container), Zigbee2MQTT with a USB stick (if OTG works), MQTT.
- **Sync and backup targets**: Syncthing, a restic/rsync target on an SD card or USB disk.
- **Small web services**: a static site, a wiki, Uptime Kuma, a Gitea/Forgejo instance for personal repos.
- **Media**: Jellyfin for direct play. Transcoding uses the CPU, which phones won't love for long.
- **A VPN node**: WireGuard or a Tailscale exit node/subnet router.
- **Cameras and sensors**: the phone already has a camera, microphone, accelerometer and light sensor. On Android, an IP-camera app turns it into a security camera; on Nura, camera support depends heavily on the device.
- **Learning**: a three-phone k3s cluster is a much quieter homelab than a rack.

Poor fits: anything that needs lots of storage (internal flash is small and not replaceable), anything I/O-heavy, and anything that has to be reachable from the internet on Termux.

## Security checklist

- **Never expose a stock Android phone directly to the internet.** Old phones run old kernels with known vulnerabilities. A VPN keeps the phone on your private network.
- **SSH keys only**, password authentication off.
- **Keep it updated**: `pkg upgrade` in Termux, `apk upgrade -a` on Nura.
- **Remove the SIM card** unless you need mobile data. On Nura you can also leave the modem service disabled entirely.
- **Consider full-disk encryption** (`pmbootstrap install --fde`) if the data on it matters and someone could walk off with the phone. The trade-off: you have to type the passphrase at boot, over USB or on the screen, so unattended reboots stop working.

## Bottom line

- **An old phone is a capable, nearly silent, very low-power ARM server** with a built-in UPS. For many home services it's plenty.
- **Termux** works on almost any Android phone without root, but Android stays in charge: you'll fight the phantom process killer and battery optimisation, you can't run containers, and the OS underneath doesn't get security updates.
- **Android's Linux Terminal** gives you a real Debian VM, but only on recent hardware.
- **Nura (formerly postmarketOS)** turns a supported phone into a proper Linux server with a mainline kernel, OpenRC or systemd, a firewall, containers and regular updates. Check the device list first and pick a community device with working Wi-Fi and charging.
- **Take the battery seriously.** Inspect it, keep it cool, limit the charge level, and never leave a swollen battery plugged in.

## Sources

- [postmarketOS v26.06 release](https://postmarketos.org/blog/2026/06/21/v26.06-release/), postmarketOS blog.
- [postmarketOS announce rebrand to new, shorter name](https://www.omglinux.com/postmarketos-new-name/), OMG! Linux, on the rename to Nura.
- [postmarketOS is now Nura after nearly a decade of bringing Linux to phones](https://www.notebookcheck.net/postmarketOS-is-now-Nura-after-nearly-a-decade-of-bringing-Linux-to-phones.1410027.0.html), Notebookcheck.
- [Releases](https://docs.postmarketos.org/pmaports/main/releases.html), postmarketOS documentation: release schedule and support periods.
- [Devices](https://wiki.postmarketos.org/wiki/Devices), postmarketOS wiki: device list, categories and feature matrix.
- [pmbootstrap usage](https://docs.postmarketos.org/pmbootstrap/usage.html), pmbootstrap documentation.
- [Android Virtualization Framework guide](https://cdn.jsdelivr.net/gh/ferrumclaudepilgrim/claude-code-android@main/docs/avf-guide.md), on the Terminal app's device requirements on Android 17.
- [Andronix on Android 12 and beyond](https://docs.andronix.app/android-12/andronix-on-android-12-and-beyond), on the phantom process killer and how to disable it.
- [proot-distro](https://github.com/termux/proot-distro), Termux.
- [Limiting Battery Risk On Repurposed Smartphones With PostmarketOS](https://hackaday.com/2026/02/01/limiting-battery-risk-on-repurposed-smartphones-with-postmarketos), Hackaday.
- [From Smartphone To A Home Server](https://hackaday.com/blog/page/564/?m=false), Hackaday, on Hardware Haven's Home Assistant phone server.
- [I turned an old phone into a Linux homelab](https://dev.to/kamaaaal/i-turned-an-old-phone-into-a-linux-homelab-54cf), DEV Community: k3s and Jellyfin on a OnePlus 6T.
- [Portable web server on a Pixel 3a XL](https://lemmy.world/comment/25920669), Lemmy: power and CPU figures for an offline media node.
- [Linux mobile resources](https://linmob.net/resources/), LINMOB.net: overview of mainline-capable phones and distributions.
