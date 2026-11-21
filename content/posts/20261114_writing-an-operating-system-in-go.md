---
title: "Writing an Operating System in Go"
date: 2026-11-14
draft: true
description: "Can you write a kernel in Go? People have. What gets in the way, how existing projects solve it, what it costs in performance, and how to boot your first bare-metal Go program."
summary: "Go was never designed to run without an operating system underneath it, but kernels, unikernels, firmware and whole Linux userlands have been written in it anyway. What the Go runtime expects from an OS, the four ways people work around it, what Biscuit's measurements tell us, and where the upstream bare-metal proposal stands."
tags: ["go", "operating systems", "kernel", "osdev", "bare metal", "unikernel", "talos"]
categories: ["Programming"]
---

C has been the default language for operating system kernels for fifty years. Linux, the BSDs, macOS and Windows are all built on it. Rust has recently become a serious second option. Go is a much less obvious choice: it has a garbage collector, a goroutine scheduler and stacks that grow on their own, and all of that runtime machinery assumes there is an operating system underneath it.

Yet people have written kernels in Go, some of them good enough to run NGINX and Redis, and at least one Go-based operating system runs Kubernetes clusters in production. This post covers why Go is awkward for kernel work, the four main ways people get around that, what it costs in performance, and how to boot a bare-metal Go program yourself.

## Why Go is an unusual kernel language

A Go program is never just your code. Every binary carries the Go **runtime**, which provides the garbage collector, the goroutine scheduler, maps, channels, stack growth and panics. The runtime is what makes Go pleasant, and it is also the problem: it was written to sit on top of an OS and call it constantly.

The OSDev wiki sums it up: Go was never designed to be a bare-metal language, but with some tricks you can compile a kernel with it[^osdev]. Those tricks exist because of a few specific obstacles.

**The runtime needs OS services.** When the garbage collector wants more heap, it asks the OS for memory. When the scheduler needs another thread, it asks the OS to create one. When a goroutine sleeps, the runtime asks the OS for the time. In a kernel there is no OS to ask, so you have to implement these operations yourself on bare metal and hack the runtime to call your versions[^jetbrains].

**The upstream toolchain has no freestanding target.** `GOOS` can be `linux`, `windows`, `darwin`, `wasip1` and so on, but there is no `GOOS=none`. Every bare-metal Go project either patches the toolchain, swaps out runtime pieces, or uses a different compiler. (More on the upstream proposal to fix this below.)

**Allocation can't fail.** In C, a kernel's allocator returns `NULL` when memory runs out and the caller deals with it. Go allocates implicitly (a closure, an `append`, converting a value to an interface) and has no way to report failure. The Biscuit authors called out exactly this: a Go kernel cannot recover from low memory in the usual way, because Go does not expose allocation failure[^biscuit-login].

**Garbage collection needs headroom.** The collector runs best with spare heap, and in a kernel that heap is RAM no application can use. GC pauses also happen at moments you don't choose, which matters for interrupt latency.

**No inline assembly.** A kernel has to execute privileged instructions: load the interrupt table, switch page tables, read control registers, talk to I/O ports. Go has no inline assembly, so these live in separate Go assembler (`.s`) files, called through function stubs.

**Interrupt handlers are dangerous places for Go.** An interrupt can arrive while the runtime is in the middle of anything, including the garbage collector. Code that runs in a handler must not allocate, must not grow its stack and must not block. In practice handlers do as little as possible (acknowledge the device, record what happened, wake a goroutine) and the real work happens in normal Go code later.

None of these are dealbreakers. They just mean a Go kernel looks different from a C kernel.

## What the runtime actually asks the OS for

It helps to be concrete. On Linux, the Go runtime depends on a fairly small set of system services:

| The runtime needs | On Linux it uses | In your kernel you provide |
| --- | --- | --- |
| Memory for the heap and stacks | `mmap`, `munmap`, `madvise` | A physical page allocator and page tables |
| OS threads to run goroutines on | `clone` | One "thread" per CPU core, or just one to start |
| Sleeping and waking threads | `futex` | A wait/wake primitive, or spinning on a single core |
| Time | `clock_gettime` (via the vDSO) | A timer: the TSC, HPET or a platform clock |
| Somewhere to print panics | `write` to stderr | A serial port or the screen |
| Random numbers (for map hashing and `crypto/rand`) | `getrandom` | `RDRAND` or a hardware RNG |
| Asynchronous preemption, crash handling | Signals | Timer interrupts, or nothing at first |

That table is the whole game. Every Go OS project is, at its core, an answer to the question "who implements the right-hand column, and how does the runtime find it?" The four approaches below give four different answers, and the last two answer "Linux does".

## Four ways to do it

### 1. Replace the runtime's OS layer

The most common approach keeps the standard Go compiler and replaces the part of the runtime that talks to the OS. Your kernel provides memory, time and a console, and the rest of the runtime (GC, scheduler, channels) works unchanged. You get real Go, including most of the standard library.

This is how **Biscuit**, **eggos** and **TamaGo** work. The downside is that you are maintaining a patched runtime that has to keep up with every Go release, because the runtime's internals are not a stable API.

### 2. Use a different compiler

**gccgo** (the GCC front end for Go) and **TinyGo** (an LLVM-based Go compiler) are separate implementations with their own, smaller runtimes. TinyGo targets microcontrollers and WebAssembly and can compile Go to run on a range of bare-metal boards[^tinygo]. Its binaries are typically around a fifth of the size of standard Go binaries[^osfc2023].

Several hobby kernels use gccgo because its runtime is easier to strip down and it produces ordinary ELF objects that link with a GRUB Multiboot2 header, for example **go-dav-os**, a small x86-64 kernel that boots to a shell prompt[^davos]. The trade-off is that you are no longer using the main Go toolchain, and language features or standard library packages may lag or be missing.

### 3. Reimplement the kernel interface in user space

**gVisor**, from Google, is an application kernel that runs as an ordinary Linux process. Its core component, the Sentry, is a from-scratch Go reimplementation of the Linux system call interface, memory management, filesystems, a network stack, process management and signal handling[^gvisor-arch]. Containers run against the Sentry instead of the host kernel, so an attacker has to break both the Go kernel and the host Linux kernel to escape[^gvisor-arch].

gVisor was written in Go specifically to avoid the memory safety bugs that plague C kernels[^gvisor-readme]. It shows that "writing an OS in Go" can be a very practical idea, as long as the Go part sits on top of something else that handles the hardware.

### 4. Keep the Linux kernel, rewrite everything above it

The fourth approach flips the question. Instead of asking how to run Go without Linux, it asks how much of an operating system is actually the kernel. The answer is: less than you'd think. Booting, configuration, service supervision, networking setup, upgrades and remote management all live in user space, and Go is already very good at that kind of code.

**Talos Linux**, from Sidero Labs, is the best example. Its documentation describes it as a ground-up rewrite of user space starting from PID 1: it runs the Linux kernel, but everything downstream of that is Sidero's own Go code, shipped as a single immutable image[^talos-philosophy-13]. The kernel starts a Go program called `machined` instead of systemd, and there are no GNU utilities, no shell, no SSH and no packages[^talos-philosophy-17]. Not even BusyBox: everything on the system is there because it's needed[^talos-philosophy-16].

`machined` runs as PID 1 and does everything a traditional init system and its helpers would: boots the system, applies the machine configuration, starts services and exposes the management API[^talos-arch]. You manage a node with `talosctl`, a client for that gRPC API, rather than by logging in. Upgrades replace the whole OS image at once instead of updating individual packages[^talos-arch-view].

From a Go point of view, this sidesteps every problem in the first half of this post. `machined` is an ordinary Linux binary, built with the standard toolchain (recent Talos releases are built with Go 1.26[^talos-1143]). The garbage collector, goroutine scheduler and allocator all sit on a real kernel, so there is no patched runtime, no interrupt handler written in Go and no question of what happens when allocation fails. What you give up is the kernel itself: Talos still depends on Linux and its drivers, written in C, and currently tracks Linux 6.18[^talos-1145].

It's also not a research project. Talos is a mainstream way to run Kubernetes on bare metal and in the cloud, on a regular release cadence, and Sidero Labs has announced it is growing beyond Kubernetes with built-in hypervisor support and running containers directly on the OS, with general availability planned for December 2026[^talos-hypervisor].

If you think of an operating system as the thing you install on a machine and manage, rather than strictly the kernel, Talos is arguably the most successful operating system written in Go.

## The projects worth knowing

| Project | What it is | Approach | State |
| --- | --- | --- | --- |
| [Biscuit](https://pdos.csail.mit.edu/projects/biscuit.html) | Monolithic POSIX-subset kernel for x86-64, from MIT | Modified Go runtime | Research project (OSDI 2018); the reference point for performance |
| [gopher-os](https://github.com/achilleasa/gopher-os) | Experimental x86-64 kernel aiming at Linux syscall compatibility | Standard Go with custom boot code | Proof of concept, inactive since 2018 |
| [eggos](https://github.com/icexin/eggos) | Unikernel that turns a normal Go program into a bootable x86 image | Modified runtime layer, a little C | Pinned to Go 1.16; last commit 2023 |
| [TamaGo](https://github.com/usbarmory/tamago) | Framework for bare-metal Go on AMD64, ARM, ARM64 and RISC-V | Small patch to the Go distribution plus driver packages | Actively maintained, used in shipping products |
| [TinyGo](https://tinygo.org/) | Alternative Go compiler for embedded systems | LLVM backend, own runtime | Active; focused on microcontrollers rather than kernels |
| [gVisor](https://gvisor.dev/) | User-space kernel implementing the Linux syscall surface | Runs on top of Linux | Production use in container sandboxing |
| [Talos Linux](https://www.talos.dev/) | Immutable, API-managed OS for Kubernetes | Linux kernel, Go user space from PID 1 | Production use; actively developed (v1.14) |

A few notes on the ones you'll come across most:

**gopher-os** set out to prove that Go is a suitable tool for low-level code running at ring 0, with the long-term goal of a POSIX-compliant, Linux-compatible kernel[^gopheros]. It got as far as interrupt handling, virtual memory and a good part of an ACPI interpreter. Its README is honest that it was never usable: boot the ISO and it eventually panics with `Kmain returned`[^gopheros]. Its commit history is still a good read if you want to see how the boot code and interrupt gates are wired up.

**eggos** is the most fun to try. It runs goroutines, channels and the garbage collector on bare x86, ships a TCP/UDP network stack, a VFS, a JavaScript interpreter, an NES emulator and simple HTTP and SSH servers[^eggos-readme]. Its `egg` tool builds a normal Go program into a bootable ELF unikernel[^eggos-pkg]. The catch is that it only works with Go 1.16.x; newer versions break it[^eggos-faq].

**TamaGo** is the one to use if you actually want to ship something on bare metal. It's maintained by the team behind the USB armory and powers real firmware: GoKey (an OpenPGP and U2F smartcard), GoTEE (a trusted execution environment), ArmoredWitness and an encrypted USB drive[^tamago-ecosystem], plus go-boot, a UEFI boot manager[^tamago-wiki]. Its authors credit Biscuit with proving that the Go runtime could be adapted for bare metal, but deliberately kept TamaGo's scope narrower: run one Go application on the hardware, with a less invasive change to Go[^tamago-faq].

## What it costs: the Biscuit numbers

Biscuit is the only serious attempt to measure what Go costs in a kernel, which makes it the best evidence either way.

It's a real kernel. Biscuit implements enough of POSIX (virtual memory, `mmap`, TCP/IP sockets, a logging file system, `poll`) to run significant applications[^biscuit-osdi], with drivers for AHCI SATA disks and Intel 10 Gb network cards[^biscuit-login]. The implementation is nearly 28,000 lines of Go, 1,546 lines of assembly and no C at all[^biscuit-paper]. It also makes liberal use of the high-level parts of Go: closures, channels, maps, interfaces and garbage-collected allocation[^biscuit-paper].

The results:

- **Garbage collection cost up to 3% of CPU** on their benchmarks[^biscuit-paper].
- **All Go language overheads together** (mainly GC and the checks for stack growth on every function call) took up to 13% of kernel CPU time on kernel-heavy benchmarks such as NGINX and Redis[^biscuit-osdi].
- **Head to head with C**, on nearly identical system call, page fault and context switch paths written in both languages, the Go versions were 5% to 15% slower[^cutler-abstract].
- **Against Linux**, NGINX and Redis throughput on Biscuit was within 10% of Linux[^biscuit-login]. The authors caution that this comparison says little about language choice, since the two kernels differ in so many other ways[^biscuit-paper].

The most interesting result is how Biscuit dealt with the out-of-memory problem. Since Go can't report allocation failure, Biscuit uses static analysis of its own source code to work out the maximum heap memory each system call can need, and reserves that much before starting the call[^biscuit-paper]. Because the wait for memory happens before any locks are taken, the scheme can't deadlock, which the authors note is a problem Linux has struggled with[^biscuit-login]. Go's simple, analysable syntax is what made this practical.

The authors deliberately don't declare a winner between C and Go[^biscuit-paper]. A fair reading: Go costs you a single-digit to low-double-digit percentage of kernel CPU, and buys you memory safety, a garbage collector, and a much more pleasant language for concurrent code. For a research kernel, a unikernel or security-critical firmware, that's often a good trade. For a general-purpose kernel where every cycle matters, it's harder to justify, and that's exactly the trade Talos avoids by leaving the kernel in C and using Go everywhere else.

## What booting a Go kernel looks like

Whatever bare-metal project you pick, the boot sequence on x86-64 follows the same shape:

1. **A bootloader loads the kernel.** Most hobby projects use GRUB with a Multiboot or Multiboot2 header, so they never have to write a boot sector. UEFI is the modern alternative, and TamaGo can run under UEFI or as a KVM guest such as QEMU microvm[^tamago-faq].
2. **An assembly stub takes over.** It sets up an initial stack, enables long mode and paging if the bootloader hasn't, and loads a basic descriptor and interrupt table. gopher-os's commit log describes this nicely: the stub loads a blank 256-entry interrupt table, and each entry points to a small assembly gate that saves registers and calls a handler registered from Go[^gopheros-idt].
3. **The Go runtime initialises.** Your replacement OS layer hands the runtime a region of memory for its heap, a clock and a console. From here, `make`, `new`, goroutines and maps work.
4. **`main` runs.** From this point it's ordinary Go, with `unsafe` for touching hardware.

(In Talos, steps 1 to 3 are Linux's job. The Go part starts when the kernel executes `/sbin/init`, which is `machined`.)

Talking to hardware from Go is mostly a matter of `unsafe.Pointer` and fixed addresses. The classic first step, writing to the VGA text buffer at physical address `0xB8000`, looks like this:

```go
package vga

import "unsafe"

const (
	width  = 80
	height = 25
	// Light grey text on a black background.
	attr = 0x07
)

// The text-mode framebuffer: 80x25 cells, each a character byte
// followed by a colour byte.
var buf = (*[width * height]uint16)(unsafe.Pointer(uintptr(0xB8000)))

// Print writes s starting at the given row, ignoring anything
// past the end of the line.
//
//go:nosplit
func Print(row int, s string) {
	for i := 0; i < len(s) && i < width; i++ {
		buf[row*width+i] = uint16(attr)<<8 | uint16(s[i])
	}
}
```

The `//go:nosplit` directive tells the compiler not to insert the stack-growth check at the start of the function, which is what you want for code that may run before the runtime is fully set up or inside an interrupt handler. It only makes sense for small functions that don't allocate.

On a normal Linux system this code would crash immediately: `0xB8000` isn't mapped into your process. In a kernel running with identity-mapped low memory, it prints text on the screen.

## Try it: bare-metal Go in QEMU with TamaGo

The quickest way to see a Go program run without an OS is TamaGo's example application in an emulator. TamaGo consists of a modified Go distribution that adds the `tamago` value for `GOOS`, plus Go packages for processor, SoC and board support[^tamago-wiki].

You build the TamaGo toolchain once (instructions are in the [tamago-go](https://github.com/usbarmory/tamago-go) repository), point a `TAMAGO` variable at its `go` binary, and then:

```bash
git clone https://github.com/usbarmory/tamago-example
cd tamago-example
make qemu
```

That boots the example unikernel in QEMU[^tamago-wiki]. Building your own program uses the familiar `go build` with a different `GOOS` and some linker flags that tell the linker where in memory the program will live. For example, for the RISC-V SiFive board that QEMU emulates:

```bash
GOOS=tamago GOARCH=riscv64 ${TAMAGO} build -ldflags "-T 0x80010000 -R 0x1000" main.go
```

That command comes from the TamaGo documentation[^tamago-pkg]. The program you build is standard Go: it imports the board package for its target, and from there `fmt`, goroutines, `crypto` and the rest behave as you'd expect.

Once you're comfortable with that, eggos (with a Go 1.16 toolchain installed alongside your normal one) is a good second step for x86, and reading Biscuit's source is the best way to see what a full Go kernel with processes and a file system looks like. If you're more interested in the "Go everywhere except the kernel" model, the [Talos source](https://github.com/siderolabs/talos) is a large, well-organised Go codebase, and its `machined` package is a good place to see what a modern init system looks like when it's written as a Go service.

## The upstream proposal: GOOS=none

Bare-metal Go has been proposed upstream several times. In 2019 a proposal asked for `GOOS=none` on amd64 so Go programs could run without an OS, mainly as virtual machine images[^proposal-35956]. In 2020 the TamaGo authors proposed bare-metal ARM support[^proposal-37503], and the Go team's response at the time was that it seemed beyond the scope of the core team[^proposal-37503-scope]. Both proposals were closed.

The current attempt is [issue #73608](https://github.com/golang/go/issues/73608), opened in May 2025 by Andrea Barisani of the TamaGo project. It proposes a new `GOOS=none` target in which the runtime, instead of making system calls, calls a small set of functions that a board support package or the application provides[^proposal-73608]. Printing a character, for instance, becomes a function you define and link into the runtime:

```go
// printk emits a single 8-bit character to standard output
//
//go:linkname printk runtime.printk
func printk(c byte)
```

That snippet is from the proposal itself[^hn-proposal]. The idea is that this narrow interface is the only thing the runtime needs, so the rest of the bare-metal support (drivers, boards) can live entirely outside the Go repository. The open questions are the same ones that sank earlier attempts: freezing a stable runtime API that bare-metal code can depend on, and keeping it working as the compiler and runtime change[^besthub]. At the most recent snapshot of the issue I could find, it was still open; check the issue for its current status before relying on it.

If it's accepted, writing a Go kernel would no longer require a patched toolchain. You would build with the standard `go` command and supply the functions yourself. That would be the biggest change to Go OS development since Biscuit.

## Should you write your OS in Go?

**It's a good fit if:**

- You're learning how operating systems work and already know Go. Bugs are far easier to find without memory corruption hiding them.
- You're building a **unikernel** or firmware that runs a single application. This is TamaGo's niche and it works well.
- You care more about **memory safety** than the last 10% of performance: trusted execution environments, security tokens, sandboxes.
- You're doing **OS research** and want to try ideas quickly. Biscuit's authors found that Go's high-level features subjectively made programming easier[^cutler-abstract], and a few percent of CPU matters less in a prototype than how fast you can change it.
- You're building an **appliance or infrastructure OS** where the interesting work is configuration, supervision and management rather than the kernel. Keep Linux and write the rest in Go, as Talos does.

**It's a poor fit if:**

- You need hard real-time guarantees. Garbage collection pauses are short but not under your control.
- You're targeting tiny devices with little RAM, unless you use TinyGo.
- You want a large, general-purpose kernel that people will run everywhere. Rust has a much larger bare-metal ecosystem now, and C still has every driver ever written.
- You don't want to maintain a patched runtime through every Go release, at least until `GOOS=none` lands.

## Bottom line

- **Yes, you can write an OS in Go.** Biscuit runs NGINX and Redis within about 10% of Linux's throughput, and TamaGo runs shipping firmware.
- **The Go runtime is both the obstacle and the prize.** You have to supply memory, time, threads and a console to it, and in return you get garbage collection, goroutines and most of the standard library at ring 0.
- **The cost is measurable and moderate**: up to 3% of CPU for GC, up to around 13% for all of Go's runtime features combined, and 5–15% slower than C on identical kernel paths.
- **The most successful Go operating systems don't replace the kernel.** Talos keeps Linux and rewrites all of user space in Go, from PID 1 up. gVisor reimplements the Linux interface in Go on top of a real kernel. Both get Go's safety where it matters most without fighting the runtime.
- **For bare metal, TamaGo is the practical choice today.**
- **Watch issue #73608.** An upstream `GOOS=none` target would remove the biggest practical barrier.

[^osdev]: [Go](https://wiki.osdev.org/Go), OSDev Wiki.
[^jetbrains]: [OS in Go? Why Not?](https://blog.jetbrains.com/go/2023/05/16/os-in-go-why-not/), JetBrains GoLand Blog, May 2023.
[^biscuit-login]: Cody Cutler, M. Frans Kaashoek, Robert T. Morris, [The Benefits and Costs of Writing a POSIX Kernel in a High-Level Language](https://pdos.csail.mit.edu/papers/biscuit:login19.pdf), USENIX ;login:, Spring 2019.
[^tinygo]: [Short introduction to TinyGo](https://www.tinygo.org/getting-started/overview/), TinyGo documentation.
[^osfc2023]: [Tamago and TinyGo: working with firmware and kernel written in Go!](https://talks.osfc.io/open-source-firmware-conference-2023/talk/PUMRQ9/), Open Source Firmware Conference 2023.
[^davos]: [go-dav-os](https://github.com/dmarro89/go-dav-os), a hobby x86-64 kernel in Go using gccgo and GRUB Multiboot2.
[^gvisor-arch]: [Introduction to gVisor security](https://gvisor.dev/docs/architecture_guide/intro/), gVisor documentation.
[^gvisor-readme]: [gVisor README](https://github.com/google/gvisor), "Why Go?" section.
[^talos-philosophy-13]: [Learn More: Philosophy](https://www.talos.dev/v1.3/learn-more/_print/), Talos Linux documentation.
[^talos-philosophy-17]: [Learn More: Philosophy and Components](https://talos.dev/v1.7/learn-more/_print), Talos Linux documentation.
[^talos-philosophy-16]: [Philosophy](https://docs.siderolabs.com/talos/v1.6/learn-more/philosophy), Sidero Labs documentation.
[^talos-arch]: [How to Understand Talos Linux Architecture](https://oneuptime.com/blog/post/2026-03-03-understand-talos-linux-architecture/markdown), OneUptime, March 2026.
[^talos-arch-view]: [How to Understand Talos Linux Architecture (web version)](https://oneuptime.com/blog/post/2026-03-03-understand-talos-linux-architecture/view), OneUptime, March 2026.
[^talos-1143]: [Talos v1.14.3 release notes](https://github.com/siderolabs/talos/releases/tag/v1.14.3), October 2026.
[^talos-1145]: [Talos 1.14.5 release notes](https://newreleases.io/project/github/siderolabs/talos/release/v1.14.5), October 2026.
[^talos-hypervisor]: [Talos Linux adds native hypervisor and edge container support](https://www.prnewswire.com/news-releases/talos-linux-adds-native-hypervisor-and-edge-container-support--see-it-live-at-taloscon-2026-302877040.html), Sidero Labs press release, 2026.
[^gopheros]: [gopher-os](https://github.com/achilleasa/gopher-os) by Achilleas Anagnostopoulos, README.
[^eggos-readme]: [eggos](https://github.com/icexin/eggos), README.
[^eggos-pkg]: [eggos package documentation](https://pkg.go.dev/github.com/icexin/eggos), pkg.go.dev.
[^eggos-faq]: [eggos FAQ](https://github.com/icexin/eggos/blob/main/FAQ.md), on Go version requirements.
[^tamago-ecosystem]: [TamaGo README: applications using TamaGo](https://github.com/usbarmory/tamago).
[^tamago-wiki]: [TamaGo wiki](https://github.com/usbarmory/tamago/wiki).
[^tamago-faq]: [TamaGo Frequently Asked Questions](https://github.com/usbarmory/tamago/wiki/Frequently-Asked-Questions-(FAQ)).
[^biscuit-osdi]: Cody Cutler, M. Frans Kaashoek, Robert T. Morris, [The benefits and costs of writing a POSIX kernel in a high-level language](https://www.usenix.org/conference/osdi18/presentation/cutler), OSDI 2018.
[^biscuit-paper]: [Full OSDI 2018 paper (PDF)](https://pdos.csail.mit.edu/6.S081/2021/readings/biscuit.pdf), MIT PDOS.
[^cutler-abstract]: [Biscuit PhD thesis, Cody Cutler (PDF)](https://pdos.csail.mit.edu/papers/biscuit:thesis.pdf), MIT, abstract.
[^gopheros-idt]: [gopher-os commit: Load IDT and define gate error-code-aware handlers](https://github.com/achilleasa/gopher-os/commit/5a2efb2bd358c614744e6f65c93aea066238b2be), June 2017.
[^tamago-pkg]: [TamaGo package documentation](https://pkg.go.dev/github.com/usbarmory/tamago), pkg.go.dev.
[^proposal-35956]: [proposal: support GOOS=none GOARCH=amd64 for running Go programs without an OS (#35956)](https://github.com/golang/go/issues/35956), Go issue tracker.
[^proposal-37503]: [proposal: all: add bare metal ARM support (#37503)](https://github.com/golang/go/issues/37503), Go issue tracker.
[^proposal-37503-scope]: Discussion on [#37503](https://golang.org/issue/37503), Go issue tracker.
[^proposal-73608]: [proposal: all: add bare metal support (#73608)](https://github.com/golang/go/issues/73608), Go issue tracker, opened May 2025.
[^hn-proposal]: [Proposal: Add bare metal support to Go](https://news.ycombinator.com/item?id=43919862), Hacker News discussion quoting the proposal.
[^besthub]: [Can Go Run Without an OS? Exploring the New Bare-Metal Support Proposal](https://www.besthub.dev/articles/can-go-run-without-an-os-exploring-the-new-bare-metal-support-proposal-001080a01e3f).
