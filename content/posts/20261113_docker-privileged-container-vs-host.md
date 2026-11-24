---
title: "Docker: Privileged Containers vs Running Directly on the Host"
date: 2026-11-13
draft: true
description: "What docker run --privileged actually turns off, how close a privileged container is to a root process on the host, what isolation is left, and what to use instead."
summary: "A privileged container looks like a container, but security-wise it is very close to running as root on the host. Here is what --privileged removes, what it leaves in place, and how to get the access you need without it."
tags: ["docker", "containers", "security", "linux", "capabilities", "namespaces"]
categories: ["Programming"]
---

Sooner or later a container doesn't work, and someone suggests `--privileged`. It fixes the problem, so it stays. The container still *looks* like a container: it has its own filesystem, its own process list, its own hostname. It's easy to assume it is still isolated, just with a few extra permissions.

It isn't. A privileged container running as root is, for security purposes, very close to a root process on the host. The isolation that remains is mostly about giving the process a tidy private *view* of the system, not about stopping it from reaching out of that view.

This post looks at what `--privileged` actually changes, compares it with running the same program directly on the host, explains why the remaining isolation doesn't hold an attacker in, and covers what to use instead so you rarely need the flag at all.

## How a normal container is isolated

A container is not a virtual machine. It is an ordinary Linux process that shares the host's kernel, wrapped in several independent layers of restriction[^3]:

| Layer | What it does in a default container |
| --- | --- |
| **Namespaces** | Give the process its own view of mounts, process IDs, network, hostname, IPC and cgroups. |
| **Capabilities** | Root's power is split into distinct capabilities[^6]. Docker keeps a small default set (such as `CHOWN`, `NET_BIND_SERVICE`, `SETUID`, `KILL`) and drops the dangerous ones like `SYS_ADMIN` and `SYS_MODULE`[^1]. |
| **Seccomp** | A syscall filter that blocks dozens of syscalls, such as `mount`, `kexec_load`, `init_module` and `bpf`[^9]. |
| **AppArmor / SELinux** | A mandatory access control profile (`docker-default` on AppArmor) that denies things like writing to `/proc/sys` or mounting filesystems. |
| **Device cgroup** | Only a small allowlist of devices (`/dev/null`, `/dev/zero`, `/dev/urandom`, the TTY) can be opened. No disks. |
| **Masked and read-only paths** | Sensitive paths like `/proc/kcore` and `/sys/firmware` are hidden; `/proc/sys`, `/proc/sysrq-trigger` and `/sys` are read-only. |
| **cgroups (resources)** | CPU, memory and PID limits. |

One thing is *missing* from this list by default: **user namespaces**. Unless you configure `userns-remap` or rootless mode, UID 0 inside the container is UID 0 on the host[^4]. The only things standing between "root in a container" and "root on the host" are the capability, seccomp, LSM and device restrictions above.

Keep that in mind, because those are exactly what `--privileged` removes.

## What `--privileged` turns off

Docker's own documentation is blunt: the flag gives the container *all* capabilities and lifts the limitations enforced by the device cgroup controller, so the container "can then do almost everything that the host can do"[^1][^2].

Concretely, `--privileged`:

| Protection | Default container | Privileged container |
| --- | --- | --- |
| Capabilities | Small default set | **All** |
| Seccomp | Default profile | **Disabled** |
| AppArmor / SELinux | Confined | **Unconfined** |
| Devices | Small allowlist | **All host devices** appear in `/dev` |
| `/proc`, `/sys` | Masked / read-only | **Unmasked, writable** |
| Namespaces | Isolated | Still isolated (mount, PID, net, UTS, IPC, cgroup) |
| User namespace | None (root = host root) | None (root = host root) |
| Resource limits | Applied | Still applied |

You can see the difference yourself. `CapEff` is the bitmask of effective capabilities, and `Seccomp: 2` means a filter is active while `0` means none:

```bash
docker run --rm alpine grep -E 'CapEff|Seccomp:' /proc/self/status
# CapEff:  small value   <- default capability set
# Seccomp: 2             <- filtering active

docker run --rm --privileged alpine grep -E 'CapEff|Seccomp:' /proc/self/status
# CapEff:  full bitmask  <- every capability the kernel has
# Seccomp: 0             <- no filter
```

And the devices:

```bash
docker run --rm alpine ls /dev | wc -l              # a handful
docker run --rm --privileged alpine ls /dev | wc -l # disks, loop devices, GPUs, everything
```

So what's left is the namespaces and the resource limits. The question is whether those still count as a security boundary once everything else is gone.

## Why the remaining isolation is not a security boundary

Namespaces change what a process can *see*. Capabilities, seccomp and the LSM profile govern what it can *do* with what it sees. Remove the second group and the first group stops being a wall — it becomes a curtain.

A privileged container has `CAP_SYS_ADMIN` and can access every host device, so the host's disks are visible in `/dev` and the container has the rights to mount them. From there, the host's entire filesystem is reachable and writable; the private mount namespace no longer matters, because the data behind it is right there. The same applies to kernel interfaces that are normally masked or read-only: with `/proc` and `/sys` writable and seccomp off, a privileged process can influence the host kernel in ways a confined container cannot.

It isn't even necessary to work through the disk. Dropping the namespace flags — `--pid=host`, `--network=host`, and so on — hands over the host's process and network views directly, and a privileged container with `CAP_SYS_ADMIN` can enter other namespaces regardless. Security researchers have documented several reliable escape techniques for privileged containers (abusing the cgroup v1 `release_agent` mechanism is a well-known one), and the recurring conclusion is the same: `--privileged` provides far more access than any single escape even requires[^7][^8]. I'm deliberately not giving a working exploit here; the point for an operator is that these are understood, published techniques, not theoretical ones.

The takeaway: **do not reason about a privileged container as "a container with extra permissions."** Reason about it as a root process on the host that happens to start in a chroot-like view. If you wouldn't hand the workload root on the host, don't give it `--privileged`.

## So how is it different from just running on the host?

If a privileged container is that close to host root, is there any real difference from running the program directly? A few, but none of them is a security boundary:

- **Resource limits still apply.** cgroup CPU, memory and PID limits are enforced on a privileged container. A runaway privileged container can still be capped; a process started directly on the host is only limited if you set up cgroups yourself.
- **The default view is cleaner.** The container still starts with its own mount, PID and network namespaces, so by default it isn't *looking* at host processes and mounts. That's operational tidiness, not containment — the workload can leave that view at will.
- **Packaging and lifecycle.** You still get the image, the restart policy, the logging driver and the rest of the container workflow.

What you *don't* get is isolation from the host in the security sense. So the honest framing is: a privileged container is the container *workflow* with the container *security model* switched off. Sometimes that trade is deliberate and fine (see below). The mistake is making it by accident.

## When privileged is actually warranted

There are legitimate uses. The flag exists for them:

- **Docker-in-Docker**, where an inner daemon genuinely needs to manage devices and mounts.
- **Low-level system tools** shipped as containers: storage managers, some monitoring and hardware agents, firmware tools.
- **Kernel and device work**: anything that must load modules, reach raw block devices, or drive hardware directly.

The common thread is that the workload's whole job is to act on the host. In those cases a privileged container is reasonable *as long as you treat it with the same trust you'd give a root daemon*: control who can run it, pin the image, and keep it off the same host as untrusted workloads. The problem is almost never these cases. It's the application container that got `--privileged` to silence one permission error.

## What to use instead

Most of the time `--privileged` is reaching for a sledgehammer. The alternatives let you grant the one thing the workload needs and nothing else.

### Add only the capabilities you need

If a container fails because it lacks one capability, add that one. Start from nothing and add back:

```bash
docker run --rm \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  myapp:latest
```

`--cap-drop ALL` followed by a short `--cap-add` list is the single most useful habit for replacing `--privileged`. To find out which capability a program actually needs, run it, read the error, and add the matching capability rather than all of them[^1].

### Expose specific devices, not all of them

If the real requirement is one device, pass that device through with `--device`, which works without `--privileged`[^2]:

```bash
docker run --rm --device=/dev/snd myaudioapp:latest
```

This gives the container that device and leaves the rest of the device cgroup allowlist intact.

### Keep seccomp and AppArmor on, and loosen narrowly

If the default seccomp profile blocks a syscall the workload legitimately needs, the better fix is usually a specific capability (seccomp decisions interact with the granted capabilities), or a custom seccomp profile that allows that one syscall, rather than `seccomp=unconfined`. Same with AppArmor: write or extend a profile instead of disabling it.

### Adjust individual kernel parameters with `--sysctl`

A frequent reason people reach for privileged mode is needing to set a `sysctl`. You can usually set the specific one:

```bash
docker run --rm --sysctl net.core.somaxconn=1024 myapp:latest
```

### Run the whole daemon with user namespaces

For a defence that applies to every container, remap container UIDs to unprivileged host UIDs. With `userns-remap`, container UID 0 maps to an ordinary unprivileged UID on the host, so "root in the container" is no longer root on the host[^4]. Enable it in `/etc/docker/daemon.json`:

```json
{
  "userns-remap": "default"
}
```

Two things to know. First, it is **not** a cure for `--privileged`: Docker's documentation lists privileged mode as incompatible with user namespaces unless you also pass `--userns=host`, which disables the remapping for that container[^4]. So `--privileged` and real UID isolation are mutually exclusive — another sign of how much the flag gives away. Second, remapping changes file ownership semantics for bind mounts, so test it against your workloads before rolling it out.

### Or move the daemon off root entirely: rootless mode

Rootless mode runs both the daemon and the containers as a non-root user, inside a user namespace[^4]. Unlike `userns-remap`, where the daemon still runs as root, rootless shrinks the blast radius of a daemon or runtime compromise because there is no host root in the picture to begin with. It comes with trade-offs — networking goes through a userspace stack, binding ports below 1024 needs extra setup, and some storage drivers have kernel requirements — but for many workloads it's a strong default. It is still not a reason to then turn around and run privileged containers inside it.

## A decision checklist

Before adding `--privileged`, work down this list and stop at the first thing that works:

1. **Does the container need a specific device?** Use `--device`.
2. **Is it missing one capability?** `--cap-drop ALL` plus a targeted `--cap-add`.
3. **Is one syscall blocked by seccomp?** A custom seccomp profile, or the capability that covers it.
4. **Is it one kernel parameter?** `--sysctl`.
5. **Does it genuinely need broad host access, like Docker-in-Docker or a storage tool?** Then privileged may be justified — treat the container as a root host process: restrict who can launch it, pin the image, and isolate the host it runs on.
6. **Want defence in depth across all containers?** Turn on `userns-remap`, or run the daemon rootless. Neither mixes with `--privileged`.

## Bottom line

- **A privileged container is not a hardened container.** `--privileged` gives all capabilities, disables seccomp and AppArmor, and exposes every host device, leaving only namespaces and resource limits[^1][^2].
- **The leftover isolation is a view, not a boundary.** With full capabilities and host devices available, reaching the host filesystem or other namespaces is a documented, reliable step, not a hypothetical one[^7][^8].
- **Treat it like host root.** The difference from running directly on the host is workflow and resource limits, not security.
- **You rarely need it.** `--device`, `--cap-add`, `--sysctl` and custom seccomp profiles cover almost every real requirement with a fraction of the exposure.
- **For a baseline, isolate UIDs.** `userns-remap` or rootless mode keeps container root from being host root — but neither works together with `--privileged`, which tells you something about the flag[^4].

## References

[^1]: [docker run reference](https://docs.docker.com/reference/cli/docker/container/run/), Docker documentation: the `--privileged` flag gives all capabilities and lifts device cgroup limitations; `--cap-add` and `--cap-drop`.
[^2]: [Runtime privilege and Linux capabilities](https://docs.docker.com/engine/containers/run/#runtime-privilege-and-linux-capabilities), Docker documentation: `--privileged`, `--device` and device access.
[^3]: [Docker security](https://docs.docker.com/engine/security/), Docker documentation: namespaces, cgroups, capabilities and the default protections around a container.
[^4]: [Isolate containers with a user namespace](https://docs.docker.com/engine/security/userns-remap/), Docker documentation: UID remapping, known limitations, and the incompatibility between `--privileged` and user namespaces without `--userns=host`. See also [Rootless mode](https://docs.docker.com/engine/security/rootless/).
[^6]: [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html), Linux man pages: the capability model that splits root's privileges.
[^7]: [Understanding Docker container escapes](https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/), Trail of Bits: how a privileged container can reach the host, and why privileged mode grants far more than an escape requires.
[^8]: [Docker --privileged](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/docker-privileged), HackTricks: a catalogue of the protections privileged mode disables.
[^9]: [Seccomp security profiles for Docker](https://docs.docker.com/engine/security/seccomp/), Docker documentation: the default seccomp profile and the syscalls it blocks.
