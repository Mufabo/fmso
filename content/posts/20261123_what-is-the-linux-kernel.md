---
title: "What Is the Linux Kernel?"
date: 2026-11-23
draft: true
description: "What the Linux kernel is and what it does: kernel space vs user space, system calls, process scheduling, memory management, drivers, filesystems and networking. Covers how Linux boots, why it's a monolithic kernel with loadable modules, how the kernel is versioned and released, what 'Linux' vs 'GNU/Linux' means, and why every container on a host shares one kernel. With a hands-on lab exploring a running kernel and building a small kernel module."
summary: "Every program on a Linux machine, from your shell to systemd to Docker, is a guest. The host is the kernel: the one program that controls the hardware and decides who gets to use it. Here is what it does, how programs talk to it, and how to look inside a running one."
tags: ["linux", "kernel", "operating-systems", "containers", "lfcs"]
categories: ["Programming"]
---

When people say "I run Linux", they usually mean Ubuntu, Fedora or Debian: a whole operating system with a shell, a package manager and a desktop. Strictly speaking, **Linux is only one program in that system: the kernel.** It's the program that starts first, runs with full control over the hardware, and stays running until you shut down. Everything else asks it for permission.

This post explains what the kernel does, how programs talk to it, how it's built and released, and how to poke at a running one. It's background for several other posts: [process isolation and containers]({{< relref "20261123_process-isolation-in-linux-and-containers.md" >}}) builds on namespaces and cgroups, and [Linux storage]({{< relref "20261122_linux-storage-block-devices-filesystems-lvm.md" >}}) covers the block layer and filesystems in more depth.

## The kernel's job

A computer has a fixed set of resources: CPU cores, RAM, disks, network cards. Dozens or hundreds of programs want to use them at the same time, and none of them should be able to break the others. The kernel's job is to share those resources out fairly and safely.

Concretely, it does six things:

| Area | What the kernel does |
|---|---|
| **Processes** | Creates and destroys processes and threads, and schedules them onto CPU cores |
| **Memory** | Gives each process its own virtual address space and maps it to physical RAM |
| **Devices** | Drives hardware through device drivers, and hides the differences behind common interfaces |
| **Filesystems** | Turns blocks on a disk into files and directories |
| **Networking** | Implements TCP/IP, routing, firewalling and sockets |
| **Security** | Enforces users, permissions, capabilities, namespaces, cgroups and security modules |

Everything that isn't on that list, such as your shell, `ls`, systemd, sshd and the C library, runs **outside** the kernel as ordinary programs.

## Kernel space and user space

The split between the kernel and everything else isn't just a convention. The CPU enforces it.

Modern CPUs have **privilege levels**. On x86 they're called rings: the kernel runs in **ring 0**, where every instruction is allowed, and programs run in **ring 3**, where instructions that touch hardware, change memory mappings or disable interrupts are forbidden. ARM has the same idea under the name exception levels (EL1 for the kernel, EL0 for programs).

Memory is split the same way. Each process's virtual address space has a user part and a kernel part. Code running in user mode that tries to read the kernel's part, or any address that isn't mapped, causes a fault, and the kernel kills the process with `SIGSEGV`.

```text
 ┌───────────────────────────────────────────────────────┐
 │  User space (ring 3)                                  │
 │   bash   nginx   systemd   python   dockerd   ...     │
 │      │       │        │                               │
 │      └── C library (glibc / musl) ──┐                 │
 ├─────────────────────────────────────┼─────────────────┤
 │  Kernel space (ring 0)              ▼ system calls    │
 │   scheduler · memory manager · VFS · network stack    │
 │   namespaces · cgroups · security modules             │
 │   device drivers (disk, NIC, GPU, USB, ...)           │
 └─────────────────────────────────────┬─────────────────┘
                                       ▼
                                   hardware
```

The result: a bug in nginx crashes nginx. A bug in the kernel can crash the whole machine. That's why kernel code is reviewed so carefully, and why "it runs in the kernel" is a serious thing to say about any feature.

## System calls: how programs ask

If a program can't touch the hardware, how does it read a file? It asks the kernel with a **system call** (syscall).[^1] A syscall is a controlled jump into kernel mode: the program puts a syscall number and its arguments into CPU registers and executes a special instruction (`syscall` on x86-64, `svc` on ARM64). The CPU switches to ring 0 and jumps to a fixed entry point in the kernel, which checks the arguments, does the work, and returns to the program in ring 3.[^2]

Reading a file looks like this:

```text
cat /etc/hostname
  → glibc: open("/etc/hostname")      user space
  → syscall instruction               ── switch to ring 0 ──
  → kernel: permission check, VFS → ext4 → block layer → disk driver
  → return file descriptor 3          ── back to ring 3 ──
  → glibc: read(3, buf, 4096)
  → ... same again ...
  → write(1, buf, n)                  prints to your terminal
```

Linux on x86-64 has a few hundred system calls. A handful you'll meet constantly:

| Syscall | Does |
|---|---|
| `open`, `read`, `write`, `close` | File and device I/O (in Linux, almost everything is a file descriptor) |
| `fork`, `clone`, `execve`, `exit`, `wait4` | Create processes and threads, run programs, wait for them |
| `mmap`, `brk` | Get memory from the kernel |
| `socket`, `connect`, `accept`, `sendto` | Networking |
| `ioctl` | Device-specific operations, the catch-all |
| `unshare`, `setns`, `clone3` | Namespaces, which is how containers start |

Most programmers never call these directly; the C library wraps them in functions with the same names. But they are the **entire interface** between user space and the kernel. Any program, in any language, ends up making these same calls. You can watch them with `strace`, which the lab does below.

Linux has one famous rule about this interface: **don't break user space.** A program compiled against the syscall interface 20 years ago should still run on today's kernel. Kernel internals change all the time; the syscall interface almost never changes in incompatible ways.

## Processes and scheduling

A **process** is a running program: its code, its memory, its open files, and one or more **threads** of execution. For the kernel, threads and processes are both "tasks" (`struct task_struct` in the source). Threads are just tasks that share memory with each other.

A machine might have 8 cores and 500 tasks. The **scheduler** decides which task runs on which core, and for how long. It switches between them many times a second, which is why everything appears to run at once. Each switch saves the registers of the old task and loads those of the new one: a **context switch**.

Most tasks aren't runnable most of the time. A web server waiting for a request, or a shell waiting for you to type, is **sleeping** in the kernel until something wakes it. That's what the `STAT` column in `ps` shows: `R` running or runnable, `S` sleeping, `D` uninterruptible sleep (usually waiting on disk I/O), `Z` zombie (exited, but its parent hasn't collected the exit status yet).

The default scheduler for normal tasks has been **EEVDF** since Linux 6.6, replacing the Completely Fair Scheduler (CFS) that served from 2007.[^3] It still tries to give every task a fair share of CPU time, weighted by its `nice` value and its cgroup's `cpu.weight`, but it's better at giving short, latency-sensitive tasks a quick turn. Two other recent changes worth knowing about:

- **sched_ext** (Linux 6.12) lets you load a whole scheduling policy as an eBPF program, without patching or rebooting the kernel.
- **PREEMPT_RT** (also 6.12) finally merged the real-time patches into mainline, after nearly 20 years out of tree. A real-time kernel can interrupt almost any kernel code to run a high-priority task, which industrial and audio systems need.

## Memory management

Every process thinks it has a large, private, contiguous block of memory starting near address zero. It doesn't. That's **virtual memory**: the kernel and the CPU's memory management unit (MMU) translate each virtual address to a physical one through **page tables**, one set per process, in units of 4 KiB **pages**.

That indirection buys a lot:

- **Isolation.** Process A's address `0x5000` and process B's address `0x5000` point to different physical pages. Neither can see the other's memory.
- **Lazy allocation.** When a program asks for 1 GiB with `malloc`, the kernel just records the range. Physical pages are only handed out when the program first touches them (a **page fault**). That's why a process's virtual size (`VSZ`) is usually far larger than its resident size (`RSS`).
- **Sharing.** Ten processes running `bash` share one copy of bash's code in RAM. `fork()` gives the child the parent's pages marked **copy-on-write**: they're shared until one side writes, and only then copied.
- **The page cache.** Free RAM isn't wasted. The kernel uses it to cache file contents, so reading a file the second time doesn't touch the disk. That's why `free` shows a big `buff/cache` number and little "free" memory on a healthy system.
- **Swap.** When RAM runs short, the kernel can write rarely used pages to disk and reuse the RAM.

When RAM and swap are both exhausted, the kernel has no good options left and runs the **OOM killer**, which picks a process (by a score you can see in `/proc/<pid>/oom_score`) and kills it. Inside a container, the same thing happens at the cgroup's `memory.max` limit, which is what Kubernetes reports as `OOMKilled`.

## Devices and drivers

The kernel talks to hardware through **device drivers**: code that knows how to operate one kind of controller, network card or GPU. Drivers make up by far the largest part of the kernel source.

User space mostly sees devices as files under `/dev`:

- **Block devices** (`/dev/sda`, `/dev/nvme0n1`) are read and written in blocks, and filesystems sit on top of them.
- **Character devices** (`/dev/tty`, `/dev/null`, `/dev/random`) are byte streams.
- **Network interfaces** are the exception: they have no `/dev` file and are managed through sockets and `ip`.

"Everything is a file" means a program can write to a disk, a terminal or a pipe with the same `write()` call. The kernel routes it to the right driver.

## Filesystems and the VFS

Linux supports dozens of filesystems: ext4, xfs, btrfs, vfat, NFS, overlayfs and more. Programs don't need to know which one they're using, because the kernel puts a **Virtual File System** (VFS) layer in front of them all. `open()`, `read()` and `stat()` go to the VFS, which calls the matching function in the filesystem's driver.

Some filesystems don't store anything on disk at all. They're windows into the kernel itself:

| Mount | Shows |
|---|---|
| `/proc` | One directory per process (`/proc/<pid>/`), plus kernel-wide information like `/proc/meminfo` and `/proc/cpuinfo`[^4] |
| `/sys` | Devices, drivers, and kernel subsystems as a tree, including cgroups under `/sys/fs/cgroup` |
| `/proc/sys` | Tunable kernel parameters, the files `sysctl` reads and writes[^5] |
| `/dev` | Device files, managed by the kernel's `devtmpfs` and udev |

When you run `ps`, `free` or `top`, they're mostly reading files in `/proc`.

## Networking

The full network stack lives in the kernel: Ethernet, IP, routing tables, TCP and UDP, the netfilter firewall (iptables/nftables), bridges, VLANs, veth pairs, VXLAN tunnels and network namespaces. A program opens a socket and calls `connect()`; the kernel does the handshake, retransmits lost packets, and manages buffers. This is why Kubernetes networking, Docker's bridge and most CNI plugins are really just configuration of kernel features. My posts on [networking basics]({{< relref "20261113_networking-basics-for-lfcs.md" >}}) and [cross-node pod networking]({{< relref "20261116_troubleshooting-cross-node-pod-networking.md" >}}) go into those.

**eBPF** has become the kernel's way to add logic to these paths without writing a kernel module. Small, verified programs can be attached to network hooks, syscalls and tracepoints; Cilium, Falco and most modern observability tools are built on it.

## Monolithic, but modular

There are two classic ways to design a kernel:

- A **monolithic kernel** runs everything (scheduler, memory, drivers, filesystems, network stack) in kernel space, in one address space. Calls between parts are plain function calls, so it's fast. But a bug anywhere can bring down everything.
- A **microkernel** keeps only the bare minimum in kernel space (scheduling, memory, message passing) and runs drivers and filesystems as separate user-space processes. A crashing driver can be restarted. The cost is that every interaction is a message between processes, which is slower. MINIX, QNX and seL4 work this way.

**Linux is monolithic.** Torvalds and Andrew Tanenbaum, the author of MINIX, argued about exactly this in 1992. Linux kept its design, and it's been working ever since.

To avoid one enormous kernel image with every driver ever written, Linux supports **loadable kernel modules**: pieces of kernel code (`.ko` files under `/lib/modules/$(uname -r)/`) that can be loaded and unloaded while the system runs.[^6] Plug in a USB device and udev loads the matching driver module automatically. Kubernetes setups load modules like `overlay` and `br_netfilter` explicitly. Once loaded, a module runs in kernel space with full privileges, exactly like built-in code. That's why loading one needs `CAP_SYS_MODULE`, and why containers never get that capability.

Loading a module that isn't part of the official kernel tree (an out-of-tree or proprietary driver, such as NVIDIA's older drivers) marks the kernel as **tainted**. Kernel developers generally won't debug crash reports from a tainted kernel, because they can't see the code that might have caused it.[^7]

## How Linux boots

The kernel is the first piece of *your* operating system to run, but not the first code the machine runs:

1. **Firmware** (UEFI, or BIOS on old machines) initializes the hardware and finds a bootloader.
2. The **bootloader** (GRUB, systemd-boot) loads the kernel image (`/boot/vmlinuz-<version>`, a compressed kernel) and an **initramfs** (`/boot/initrd.img-<version>` or `initramfs-<version>.img`) into memory, and passes a **kernel command line**.
3. The **kernel** decompresses itself, sets up memory management and the scheduler, detects hardware, and starts built-in drivers.
4. It unpacks the **initramfs**, a small temporary root filesystem holding the modules and tools needed to find the real root filesystem (disk controller drivers, LVM, LUKS decryption, RAID).[^8]
5. It mounts the **real root filesystem** and starts **PID 1**, normally systemd. From here on, user space takes over: systemd starts services, logins and your desktop.

The kernel command line (`cat /proc/cmdline`) controls a lot of this: which disk is root (`root=`), whether to boot quietly, cgroup and security settings, and debugging options.[^9]

## Versions and releases

The kernel is developed in the open at kernel.org, with Linus Torvalds managing the main ("mainline") tree.[^10] The release cycle is very regular:

1. After each release, a **two-week merge window** opens, during which new features go in.
2. Then come weekly **release candidates** (`-rc1`, `-rc2`, …), with only fixes, usually seven or eight of them.
3. The final release lands about **nine or ten weeks** after the previous one.

The version number doesn't signal anything about features. Torvalds bumps the major number when the minor gets uncomfortably large, which is how 6.19 was followed by 7.0 in 2026. As of November 2026, 7.3 is the current stable release and 7.4 is in its release-candidate phase.[^11]

After release, a kernel becomes a **stable** branch, getting backported fixes (7.3.1, 7.3.2, …) for a few months. Once a year, usually the last release of the year, one branch is picked as a **long-term support (LTS)** kernel and maintained for several years. 6.12 and 6.18 are current LTS kernels; Greg Kroah-Hartman has said he's leaning toward 7.3, not 7.4, as the 2026 LTS.[^12][^11]

Distributions mostly don't ship kernel.org kernels directly. Ubuntu, RHEL and others take a release (often an LTS), add their own patches and backports, and support it for the life of the distribution. That's why a RHEL kernel can report an old version number but still contain much newer features.

The kernel is written almost entirely in C, with some assembly for each CPU architecture. Since Linux 6.1 it has also accepted **Rust**, initially as an experiment. At the Maintainers Summit in December 2025 the developers agreed Rust was no longer experimental.[^13] So far it's used mainly for new drivers, such as the Android Binder driver and the Nova NVIDIA GPU driver, not for rewriting existing C code.

## Linux vs GNU/Linux vs a distribution

Torvalds wrote the first version of the kernel in 1991. On its own, a kernel can't do anything useful for a person: there's no shell, no `ls`, no compiler. Those came from the **GNU project**, which had been building a free Unix-like system since 1984 and had everything except a working kernel. Combine the two and you get a complete operating system, which is why the Free Software Foundation asks people to call it **GNU/Linux**.[^14]

A **distribution** adds everything else needed for a usable system: an init system, a package manager, configuration, an installer, and a security update process. So:

```text
Linux          = the kernel
GNU/Linux      = the kernel + GNU userland (bash, coreutils, glibc, gcc, ...)
Ubuntu, Fedora = a distribution: kernel + userland + systemd + packages + support
Android        = the Linux kernel + a completely different, non-GNU userland
Alpine         = the Linux kernel + musl + BusyBox (also not GNU)
```

Android is a good example: it runs the Linux kernel, but almost none of the GNU tools.

## Why the kernel matters for containers

Every container on a host shares the host's **one kernel**. A container doesn't bring its own; the `ubuntu` image contains Ubuntu's user-space files, but no kernel at all. When you run `uname -r` inside any container, you get the host's kernel version.

That's what makes containers cheap: starting one is just starting a process with some namespaces and cgroups applied, which takes milliseconds. It's also what makes them weaker than VMs: **the kernel is the security boundary**, and a kernel bug reachable through a syscall can let a process escape its container. A VM runs its own guest kernel, and the boundary is the much smaller interface of the hypervisor. The [process isolation post]({{< relref "20261123_process-isolation-in-linux-and-containers.md" >}}) covers this in detail, along with gVisor, which puts a second kernel written in Go between the container and the host.

It also explains a few practical things:

- A container can't use a kernel feature the host kernel doesn't have. An image that "needs kernel 6.x" means the **host** needs it.
- Kernel modules are loaded on the host, never in a container.
- Linux containers need a Linux kernel, so Docker Desktop on macOS and Windows runs a small Linux VM in the background.

If you'd like to see what it takes to write a kernel from scratch, [writing an operating system in Go]({{< relref "20261114_writing-an-operating-system-in-go.md" >}}) goes down that path.

## Hands-on lab: look inside a running kernel

Everything below works on any modern distribution. Steps 1 to 6 need no special privileges; step 7 needs root and a test machine or VM.

### 1. Which kernel is running?

```bash
uname -r              # e.g. 6.8.0-49-generic: version plus distro build suffix
uname -a              # plus architecture and build date
cat /proc/version     # including the compiler used to build it
cat /proc/cmdline     # the kernel command line from the bootloader
ls -lh /boot          # vmlinuz-* (kernel images) and initrd/initramfs-* files
```

### 2. Watch system calls

```bash
sudo apt install strace     # or: dnf install strace

strace -c ls /              # summary: which syscalls, how many, how long
strace -e trace=openat,read,write cat /etc/hostname
strace -f -e trace=process bash -c 'ls > /dev/null'   # fork/clone/execve/wait4
```

The last one shows how bash runs a command: `clone()` to create a child, `execve()` in the child to replace it with `ls`, and `wait4()` in the parent to wait for it.

Count how much time a program spends in kernel mode versus user mode:

```bash
time dd if=/dev/zero of=/dev/null bs=1M count=20000
# "user" = time in your code, "sys" = time in the kernel on your behalf
```

### 3. Explore /proc

```bash
cat /proc/meminfo | head        # what free reads
cat /proc/loadavg               # what uptime reads
ls /proc/$$/                    # your shell, as a directory
cat /proc/$$/status | grep -E 'State|Threads|VmRSS|VmSize'
ls -l /proc/$$/fd               # open file descriptors: 0, 1, 2 point to your terminal
cat /proc/$$/maps | head        # its virtual memory layout
ps -eo pid,stat,comm | head -20 # STAT column: R, S, D, Z ...
```

Kernel threads show up too. They have no user-space program, so `ps` shows their names in brackets:

```bash
ps -ef | grep '\[' | head       # [kthreadd], [ksoftirqd/0], [kworker/...] ...
```

### 4. Read kernel messages and tunables

```bash
sudo dmesg | head -30           # the kernel's own log since boot
sudo dmesg -w                   # follow it; plug in a USB stick and watch
journalctl -k -b                # the same through systemd

sysctl -a 2>/dev/null | wc -l   # how many tunables there are
sysctl kernel.hostname vm.swappiness net.ipv4.ip_forward
cat /proc/sys/net/ipv4/ip_forward   # the same value, as a file
```

### 5. Look at modules

```bash
lsmod | head                    # loaded modules, sizes, and who uses them
modinfo overlay                 # description, license, file, parameters
ls /lib/modules/$(uname -r)/kernel/
cat /proc/sys/kernel/tainted    # 0 means not tainted
```

### 6. Compare with a container

```bash
uname -r
docker run --rm alpine uname -r      # same version: there's only one kernel
docker run --rm alpine cat /proc/version
```

### 7. Build and load your own kernel module

This puts your own code into kernel space. Do it in a VM, not on a machine you care about. On systems with Secure Boot enabled, unsigned modules are rejected, so either disable Secure Boot in the VM or sign the module.

Install the build tools and headers for your running kernel:

```bash
sudo apt install build-essential linux-headers-$(uname -r)
# Fedora/RHEL: sudo dnf install gcc make kernel-devel-$(uname -r)
mkdir ~/hello-mod && cd ~/hello-mod
```

`hello.c`:

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/sched.h>

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Hello from kernel space");

static int __init hello_init(void)
{
    pr_info("hello: loaded by %s (pid %d)\n", current->comm, current->pid);
    return 0;
}

static void __exit hello_exit(void)
{
    pr_info("hello: unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);
```

`Makefile` (the indented lines must start with a tab):

```make
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

Build, load, check, unload:

```bash
make
modinfo hello.ko
sudo insmod hello.ko
sudo dmesg | tail -2            # hello: loaded by insmod (pid 12345)
lsmod | grep hello
sudo rmmod hello
sudo dmesg | tail -1            # hello: unloaded
```

There's no `printf` here: kernel code can't use the C library, because the C library is a user-space program that calls into the kernel. `pr_info` writes to the kernel log instead, and `current` is the kernel's pointer to the task that's running right now. A bug in this function, like dereferencing a null pointer, wouldn't crash a program. It would produce a kernel "oops" and possibly take the machine down.

## Cheat sheet

```text
Kernel      = the one program in ring 0; everything else is user space (ring 3)
Interface   = system calls (open read write mmap clone execve socket ioctl ...)
              "don't break user space": syscall ABI stays compatible
Jobs        processes/scheduling · memory · drivers · filesystems (VFS) · network · security
Scheduler   EEVDF (since 6.6, replaced CFS) · sched_ext (BPF, 6.12) · PREEMPT_RT (6.12)
Memory      virtual memory, 4 KiB pages, page tables · lazy allocation · copy-on-write
            page cache uses free RAM · swap · OOM killer (/proc/<pid>/oom_score)
Design      monolithic + loadable modules (.ko in /lib/modules/$(uname -r))
            lsmod · modinfo · modprobe · rmmod · /proc/sys/kernel/tainted
Boot        firmware → bootloader → vmlinuz + initramfs → kernel → mount root → PID 1
            /proc/cmdline = kernel command line
Windows     /proc (processes, meminfo) · /sys (devices, cgroups) · /proc/sys (sysctl)
            dmesg / journalctl -k = kernel log
Releases    2-week merge window → weekly -rc → release every ~9-10 weeks
            stable point releases · yearly LTS (6.12, 6.18, likely 7.3)
Languages   C + assembly · Rust (no longer experimental since Dec 2025, mostly drivers)
Names       Linux = kernel · GNU/Linux = kernel + GNU tools · distro = everything
Containers  share the host kernel → fast, but the kernel is the security boundary
Inspect     uname -r · strace -c · time (user vs sys) · ps STAT · lsns · sysctl -a
```

## References

[^1]: [syscalls(2)](https://man7.org/linux/man-pages/man2/syscalls.2.html), Linux manual page.
[^2]: [syscall(2)](https://man7.org/linux/man-pages/man2/syscall.2.html), Linux manual page (calling conventions per architecture).
[^3]: [EEVDF Scheduler](https://docs.kernel.org/scheduler/sched-eevdf.html), Linux kernel documentation.
[^4]: [proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html), Linux manual page.
[^5]: [Documentation for /proc/sys/kernel/](https://docs.kernel.org/admin-guide/sysctl/kernel.html), Linux kernel documentation.
[^6]: [modprobe(8)](https://man7.org/linux/man-pages/man8/modprobe.8.html), Linux manual page.
[^7]: [Tainted kernels](https://docs.kernel.org/admin-guide/tainted-kernels.html), Linux kernel documentation.
[^8]: [Using the initial RAM disk (initrd)](https://docs.kernel.org/admin-guide/initrd.html), Linux kernel documentation.
[^9]: [The kernel's command-line parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html), Linux kernel documentation.
[^10]: [How the development process works](https://docs.kernel.org/process/2.Process.html), Linux kernel documentation.
[^11]: [Linux 7.3 Will More Than Likely Be This Year's LTS Kernel](https://phoronix.com/news/Linux-7.3-Likely-LTS), Phoronix, 4 November 2026.
[^12]: [Releases](https://kernel.org/releases.html), The Linux Kernel Archives.
[^13]: [Rust Goes Mainstream in the Linux Kernel](https://thenewstack.io/rust-goes-mainstream-in-the-linux-kernel/), The New Stack, 10 December 2025.
[^14]: [Linux and the GNU System](https://www.gnu.org/gnu/linux-and-gnu.html), GNU Project.
