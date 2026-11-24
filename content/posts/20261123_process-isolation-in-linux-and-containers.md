---
title: "Process Isolation in Linux: What a Container Really Is"
date: 2026-11-23
draft: true
description: "How Linux isolates processes, from virtual memory and UIDs to namespaces, cgroups, capabilities, seccomp, LSMs and pivot_root, and how container runtimes combine them into what we call a container. Covers what a runtime does step by step, how Kubernetes maps securityContext onto these features, why containers are weaker than VMs, and the options in between (user namespaces, gVisor, Kata). With a hands-on lab that builds a container by hand."
summary: "There is no such thing as a container in the Linux kernel. There are processes, and a handful of features that restrict what a process can see, use and do. Turn all of them on at once and you have a container. Here is what each feature does, how runtimes combine them, and where the isolation ends."
tags: ["linux", "containers", "docker", "kubernetes", "security", "namespaces", "cgroups"]
categories: ["Programming"]
---

Ask the Linux kernel to list its containers and it can't: there's no such object. Run `docker run nginx`, then `ps aux` on the host, and you'll find nginx sitting in the process list next to sshd and systemd, with an ordinary PID.

A container is a **normal process** that the kernel has been told to restrict in several independent ways: what it can **see**, what it can **use**, and what it's **allowed to do**. Each of those restrictions is a separate kernel feature that predates Docker, and each is useful on its own. This post walks through them one at a time, then shows how a runtime puts them together, how Kubernetes exposes them, and where the isolation stops.

It sits next to two earlier posts: [container storage drivers]({{< relref "20261122_container-storage-drivers-overlay2.md" >}}) covers the filesystem a container sees, and [privileged containers vs the host]({{< relref "20261113_docker-privileged-container-vs-host.md" >}}) covers what happens when you switch most of this off.

## The isolation every process already has

Before any container features, Linux already separates processes in three ways:

- **Virtual memory.** Each process gets its own address space. Address `0x7ffd1234` in one process and the same address in another point to different physical memory. A process can only read another's memory through deliberate mechanisms like shared memory or `ptrace`, and `ptrace` needs permission.
- **Users and permissions.** Every process runs as a UID and GIDs, and file permissions decide what it can open. One user's processes can't signal or trace another user's.
- **Per-process resources.** File descriptors, signal handlers, the current directory and so on belong to a process.

That's real isolation, and it's what keeps your browser from reading your SSH agent's memory. But it's coarse:

- Every process sees the **same filesystem**, the **same process list**, the **same network interfaces**, the **same hostname**.
- Nothing stops one process from using all the CPU or memory.
- **Root is all-or-nothing.** UID 0 bypasses permission checks almost entirely.

Containers are built from the features that fix each of those gaps.

## Namespaces: what a process can see

A **namespace** wraps a global system resource so that processes inside it get their own isolated instance of it.[^1] Changes inside are visible to members of the namespace and invisible to everyone else.

Linux has eight kinds:

| Namespace | Flag | Isolates | Since |
|---|---|---|---|
| Mount | `CLONE_NEWNS` | The mount table: which filesystems are mounted where | 2.4.19 |
| UTS | `CLONE_NEWUTS` | Hostname and NIS domain name | 2.6.19 |
| IPC | `CLONE_NEWIPC` | System V IPC and POSIX message queues | 2.6.19 |
| PID | `CLONE_NEWPID` | Process IDs | 2.6.24 |
| Network | `CLONE_NEWNET` | Interfaces, IPs, routes, firewall rules, ports, sockets | 2.6.29 |
| User | `CLONE_NEWUSER` | UIDs, GIDs and capabilities | 3.8 |
| Cgroup | `CLONE_NEWCGROUP` | The view of the cgroup hierarchy | 4.6 |
| Time | `CLONE_NEWTIME` | The boot-time and monotonic clocks | 5.6 |

Three system calls work with them: `clone()` creates a process in new namespaces, `unshare()` moves the calling process into new ones, and `setns()` joins an existing one (that's how `docker exec` and `nsenter` get inside a running container). Every process's namespaces are visible as symlinks under `/proc/<pid>/ns/`; two processes are in the same namespace when the links point to the same inode.

What each one means for a container:

**PID.** The first process in a new PID namespace is PID 1 inside it, and it can't see or signal anything outside. The same process has a second, normal PID on the host. PID 1 is special: when it exits, the kernel kills everything else in the namespace, and it ignores signals it hasn't installed a handler for. That's why a shell script as PID 1 often doesn't stop on `docker stop`, and why tiny init processes like `tini` exist.

**Mount.** A private copy of the mount table. Combined with `pivot_root` (below), this is how a container gets the image's filesystem as `/` while the host keeps its own.

**Network.** A completely separate network stack: its own `lo`, its own routing table, its own iptables/nftables rules, its own port space. Two containers can both listen on port 80. A fresh network namespace has only a downed loopback device; the runtime or CNI plugin connects it to the outside, typically with a **veth pair** (a virtual cable with one end in the container and the other on a host bridge such as `docker0`).

**UTS.** Its own hostname, which is why `hostname` inside a container prints the container ID.

**IPC.** Its own System V shared memory, semaphores and message queues, so containers can't attach to each other's shared memory segments.

**User.** The most powerful one. It maps a range of UIDs inside to a different range outside: UID 0 in the container can be UID 100000 on the host.[^2] The process is root, with a full set of capabilities, *for resources owned by its namespace*, and an unprivileged nobody for everything else. More on this in the section on hardening.

**Cgroup.** Makes the process's own cgroup look like the root of the hierarchy, so `/proc/self/cgroup` shows `0::/` instead of revealing the host's layout.

**Time.** Lets the container see different values for the monotonic and boot-time clocks, mainly for checkpointing and migrating containers between hosts. It doesn't let a container change the wall-clock time.

Namespaces control visibility only. A process in its own PID namespace can still eat all the RAM on the machine.

## cgroups: what a process can use

**Control groups** organize processes into a hierarchy and apply resource limits and accounting to each group.[^3] Child processes inherit their parent's cgroup, so everything a container spawns stays inside its limits.

Modern systems use **cgroup v2**, a single unified tree mounted at `/sys/fs/cgroup`.[^4] Each directory is a group; files in it are the knobs:

```text
/sys/fs/cgroup/system.slice/docker-3f2a….scope/
├── cgroup.procs       # PIDs in this group
├── memory.max         # hard memory limit (OOM kill above it)
├── memory.current     # current usage
├── cpu.max            # "quota period", e.g. "50000 100000" = half a CPU
├── cpu.weight         # relative share when CPUs are contended
├── pids.max           # maximum number of processes/threads
└── io.max             # bandwidth / IOPS limits per block device
```

Every resource flag you've used maps to one of these:

| You write | Kernel setting |
|---|---|
| `docker run --memory 512m` | `memory.max` |
| `docker run --cpus 0.5` | `cpu.max = 50000 100000` |
| `docker run --pids-limit 100` | `pids.max` |
| Kubernetes `limits.memory` | `memory.max` |
| Kubernetes `limits.cpu` | `cpu.max` (CFS quota, which is what causes throttling) |
| Kubernetes `requests.cpu` | `cpu.weight` (relative share) |

When a container exceeds `memory.max`, the kernel's OOM killer kills a process **inside that cgroup**, not some random process on the host. That's the `OOMKilled` status in Kubernetes. My post on [right-sizing resource requests]({{< relref "20261119_right-sizing-kubernetes-resource-requests.md" >}}) goes into how to pick the numbers.

`pids.max` deserves a special mention: without it, a fork bomb in one container takes down the whole node.

## Capabilities: splitting up root

Traditionally, the kernel asked one question for privileged operations: is the effective UID 0? **Capabilities** break that single power into about 40 separate ones that can be granted or dropped individually.[^5] A few examples:

| Capability | Allows |
|---|---|
| `CAP_NET_BIND_SERVICE` | Binding to ports below 1024 |
| `CAP_NET_ADMIN` | Configuring interfaces, routes, firewall rules |
| `CAP_NET_RAW` | Raw sockets (ping, packet crafting) |
| `CAP_CHOWN`, `CAP_DAC_OVERRIDE` | Ignoring file ownership and permission checks |
| `CAP_SYS_PTRACE` | Tracing any process |
| `CAP_SYS_MODULE` | Loading kernel modules |
| `CAP_SYS_ADMIN` | A huge grab-bag: mount, many namespace operations, and much more. Effectively "almost root" |

Container runtimes start the process as root but with only a small allowlist of capabilities. Docker keeps 14, such as `CHOWN`, `SETUID`, `NET_BIND_SERVICE` and `KILL`, and drops everything else, including `SYS_ADMIN`, `NET_ADMIN`, `SYS_MODULE`, `SYS_PTRACE` and `SYS_TIME`.[^6] So root inside a default container can't load a kernel module, change the host's clock, or reconfigure the network, even before any other protection kicks in.

A related flag is **`no_new_privs`**. Once set on a process, nothing it executes can gain privileges, not even a setuid binary like `sudo`.[^7] It's what Kubernetes' `allowPrivilegeEscalation: false` turns on.

## seccomp: which system calls are allowed

Every interaction with the kernel goes through a system call, and the kernel's syscall interface is its largest attack surface. **seccomp** in filter mode lets a process install a small BPF program that inspects each syscall and its arguments and decides to allow it, return an error, or kill the process.[^8] The filter is inherited by children and can never be removed.

Docker's default profile is an allowlist that disables around 44 of the 300+ system calls.[^9] Among the blocked ones: loading kernel modules, `kexec_load` and `reboot`, `mount`, `pivot_root` and namespace creation via `unshare`/`setns`, setting the clock, `ptrace`, `bpf`, `userfaultfd`, and the `io_uring` calls, which have been a recurring source of kernel exploits.

Many container escapes need one of those syscalls, so a default seccomp profile quietly breaks a whole class of exploits. One thing catches people out: **Kubernetes doesn't apply it by default.** Pods run `Unconfined` unless you set `seccompProfile: { type: RuntimeDefault }` or start the kubelet with `--seccomp-default`.

## LSMs: mandatory access control

Linux Security Modules add a policy layer that even root can't override. Two are common with containers:

- **AppArmor** (Debian, Ubuntu, SUSE): path-based profiles. Docker loads a `docker-default` profile that, among other things, denies writes to sensitive parts of `/proc` and `/sys` and denies `mount`.
- **SELinux** (RHEL, Fedora): label-based. Each container gets a unique MCS label like `container_t:s0:c123,c456`, so even two containers running as the same UID can't touch each other's files, because their labels differ. Volumes need relabeling (the `:z`/`:Z` mount options) for the same reason.

LSMs are the backstop when everything else fails: if a process escapes its mount namespace, the policy still stops it from reading host files.

## pivot_root: the filesystem

The last piece is giving the process the image as its root directory. The old tool is `chroot`, but a root process can escape a chroot easily (with an open directory file descriptor and a few `chdir("..")` calls). Runtimes instead use **`pivot_root`** inside the new mount namespace: it swaps the root mount for the image's filesystem and then unmounts the old root entirely, so there's nothing left to escape to. The image filesystem itself is typically an overlay mount, as described in the [storage drivers post]({{< relref "20261122_container-storage-drivers-overlay2.md" >}}).

Runtimes also **mask** parts of `/proc` and `/sys` that leak host information or allow host changes: `/proc/kcore`, `/proc/keys`, `/proc/sched_debug` and similar are bind-mounted over with empty files, and `/proc/sys`, `/proc/irq` and others are mounted read-only.[^10]

## Putting it together: what a runtime does

High-level tools (Docker, containerd, CRI-O, Podman) delegate the actual work to a low-level **OCI runtime** such as `runc` or `crun`. The OCI runtime spec describes a container as a `config.json` listing exactly the features above: namespaces, UID mappings, cgroup limits, capabilities, seccomp filter, AppArmor/SELinux label, masked paths and the root filesystem.[^10]

When you run `docker run --memory 256m nginx`, runc does roughly this:

1. **Prepare the root filesystem**: the overlay mount of the image layers (done by the storage driver/snapshotter).
2. **Create the namespaces**: `clone()` with `CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWCGROUP`, plus `CLONE_NEWUSER` if user namespaces are configured.
3. **Create the cgroup** and write the limits (`memory.max = 268435456`), then move the process into it.
4. **Set up mounts** inside the new mount namespace: `/proc`, `/sys`, `/dev`, volumes, the `/etc/hosts` and `/etc/resolv.conf` bind mounts. Then `pivot_root` into the image and mask the sensitive `/proc` paths.
5. **Set hostname**, and let the runtime or CNI plugin wire up networking with a veth pair.
6. **Drop privileges**: reduce capabilities to the allowlist, set `no_new_privs` if requested, switch to the configured UID.
7. **Apply the LSM profile and seccomp filter**. seccomp comes last, because the setup steps themselves need syscalls the filter blocks.
8. **`execve()`** the container's entrypoint. From here on it's just nginx, running as a normal process under all those restrictions.

```text
                       Host kernel (shared)
 ┌──────────────────────────────────────────────────────────────┐
 │  nginx (host PID 48213)                                       │
 │   ├─ namespaces: pid mnt net uts ipc cgroup [user]  → sees    │
 │   ├─ cgroup: memory.max cpu.max pids.max            → uses    │
 │   ├─ capabilities: 14 of ~40, no_new_privs          → may do  │
 │   ├─ seccomp: ~44 syscalls blocked                  → may do  │
 │   ├─ AppArmor / SELinux label                       → may do  │
 │   └─ root fs: pivot_root → overlay of image layers  → sees    │
 └──────────────────────────────────────────────────────────────┘
```

None of these steps needs Docker. The lab at the end does most of them by hand with `unshare`, `chroot` and a few writes to `/sys/fs/cgroup`.

## How Kubernetes maps onto this

A **pod** is a set of containers that share some namespaces and not others. The kubelet first starts a tiny `pause` container that holds the pod's **network, IPC and UTS** namespaces; every app container joins those. That's why containers in a pod reach each other on `localhost` and share a hostname. Each container keeps its own **mount** namespace (its own image filesystem) and, by default, its own **PID** namespace, unless you set `shareProcessNamespace: true`. My post on [multi-container pods]({{< relref "20261117_multi-container-pods.md" >}}) shows patterns that rely on this.

Almost every field of `securityContext` is one of the features above:

| Pod spec field | Kernel feature |
|---|---|
| `resources.requests` / `limits` | cgroups |
| `runAsUser`, `runAsGroup`, `runAsNonRoot` | Process UID/GID |
| `capabilities.drop: ["ALL"]`, `add: [...]` | Capabilities |
| `allowPrivilegeEscalation: false` | `no_new_privs` |
| `seccompProfile.type: RuntimeDefault` | seccomp |
| `appArmorProfile`, `seLinuxOptions` | LSMs |
| `readOnlyRootFilesystem: true` | Read-only root mount |
| `hostPID`, `hostNetwork`, `hostIPC` | **Skip** that namespace and use the host's |
| `hostUsers: false` | User namespace |
| `privileged: true` | Turn most of this off |

A reasonably locked-down container looks like this:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

which is roughly what the Pod Security Standards' `restricted` profile requires. The post on [preventing privilege escalation in Kubernetes]({{< relref "20261116_preventing-privilege-escalation-in-kubernetes.md" >}}) goes through enforcing it.

## Where the isolation ends

Every container on a host talks to the **same kernel**. That's the source of everything good about containers (they start in milliseconds, cost almost nothing, and pack densely) and the source of their weakness: **the kernel is the security boundary**, and the kernel exposes hundreds of syscalls and millions of lines of code.

A virtual machine is different. Each VM has its own guest kernel, and the boundary is the hypervisor, which exposes a far smaller interface (emulated devices and a handful of hypercalls).

```text
        Containers                              VMs
 ┌──────┐ ┌──────┐ ┌──────┐         ┌──────────┐ ┌──────────┐
 │ app  │ │ app  │ │ app  │         │   app    │ │   app    │
 └──┬───┘ └──┬───┘ └──┬───┘         │  guest   │ │  guest   │
    │ syscalls (hundreds)  │         │  kernel  │ │  kernel  │
 ┌──▼────────▼────────▼───┐         └────┬─────┘ └────┬─────┘
 │    shared host kernel   │              │ hypercalls (few)
 └─────────────────────────┘         ┌────▼────────────▼─────┐
                                     │      hypervisor        │
                                     └────────────────────────┘
```

Container escapes therefore come in two flavours:

**Misconfiguration**, by far the most common in practice:

- `--privileged` / `privileged: true`: all capabilities, no seccomp, no AppArmor, every host device, writable `/sys`. From there, mounting the host's disk is one command. (See [privileged containers vs the host]({{< relref "20261113_docker-privileged-container-vs-host.md" >}}).)
- Mounting `/var/run/docker.sock` or the containerd socket: whoever can talk to it can start a privileged container.
- `CAP_SYS_ADMIN`, `hostPID`, `hostPath` mounts of `/` or `/etc`.

**Kernel and runtime bugs**, rarer but real:

- **CVE-2019-5736**: a malicious container could overwrite the host's `runc` binary through `/proc/self/exe` and gain root on the host.[^11]
- **CVE-2022-0492**: a missing check in cgroup v1's `release_agent` let a process run commands on the host; user namespaces and seccomp both mitigated it.[^12]
- **CVE-2024-21626** ("Leaky Vessels"): runc leaked a host file descriptor into the container, so a crafted working directory gave access to the host filesystem.[^13]

Each layer above exists so that a single failure isn't fatal. A bug that needs `CAP_SYS_ADMIN` doesn't work without it; one that needs `unshare` or `io_uring` is stopped by seccomp; one that gets root inside a user namespace gets only an unprivileged UID on the host.

## Hardening: the options in between

If shared-kernel isolation isn't enough, there are stronger options before you reach full VMs.

**User namespaces / rootless containers.** Root inside the container maps to an unprivileged UID on the host, so even a full escape lands as nobody. Capabilities granted inside, even `CAP_SYS_ADMIN` or `CAP_NET_ADMIN`, only apply to resources the namespace owns. Podman runs rootless by default, Docker has a rootless mode, and in Kubernetes, user namespaces for pods went **GA in v1.36**. All you need is `hostUsers: false` in the pod spec.[^14][^15] It needs a recent stack (Linux 6.3+ for idmapped mounts on the relevant filesystems, containerd 2.0+ or CRI-O 1.25+, runc 1.2+ or crun 1.9+). For most workloads it's now the single most effective switch you can flip.

**gVisor.** A user-space kernel written in Go that intercepts the container's syscalls and implements them itself, so the application never talks to the host kernel directly. Strong isolation, at the cost of syscall-heavy performance and some compatibility gaps.[^16]

**Kata Containers / Firecracker.** Each container or pod runs inside a lightweight VM with its own kernel, started fast enough to feel like a container. You get the VM boundary while keeping the container workflow and OCI images.[^17] In Kubernetes, both gVisor and Kata plug in through a `RuntimeClass`.

Roughly, from weakest to strongest boundary:

```text
plain process
 → container (namespaces + cgroups + caps + seccomp + LSM)
   → rootless / user-namespaced container
     → gVisor (user-space kernel)
       → Kata / Firecracker (micro-VM per pod)
         → full VM
```

## Hands-on lab: build a container by hand

You need root, a Linux host with cgroup v2 (any current distribution) and an Alpine root filesystem. Everything happens under `/tmp/box`.

### 1. Get a root filesystem

```bash
mkdir -p /tmp/box/rootfs && cd /tmp/box

# Either from Docker…
docker export "$(docker create alpine)" | tar -C rootfs -xf -

# …or from Alpine's minirootfs tarball (check alpinelinux.org/downloads for the current version)
# curl -L https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/x86_64/alpine-minirootfs-<version>-x86_64.tar.gz | tar -C rootfs -xzf -

ls rootfs        # bin dev etc home lib … — a complete, tiny distro
```

### 2. Create a cgroup with limits

```bash
sudo mkdir /sys/fs/cgroup/box
cat /sys/fs/cgroup/cgroup.subtree_control      # should list cpu, memory and pids

echo 50M          | sudo tee /sys/fs/cgroup/box/memory.max
echo 0            | sudo tee /sys/fs/cgroup/box/memory.swap.max
echo "50000 100000" | sudo tee /sys/fs/cgroup/box/cpu.max    # half a CPU
echo 20           | sudo tee /sys/fs/cgroup/box/pids.max
```

(On a systemd host, `systemd-run --scope -p MemoryMax=50M -p TasksMax=20 -p CPUQuota=50% …` does the same thing the polite way.)

### 3. Start the "container"

Open a root shell, put it in the cgroup, then create new namespaces and change root:

```bash
sudo -i
echo $$ > /sys/fs/cgroup/box/cgroup.procs      # this shell and its children are now limited

unshare --pid --fork --mount --uts --ipc --net --cgroup \
  chroot /tmp/box/rootfs /bin/sh
```

Inside:

```sh
mount -t proc proc /proc
hostname box
export PS1='box# '

ps                      # PID 1 is sh: you can't see the host's processes
hostname                # box
ip addr                 # only "lo", and it's DOWN
cat /proc/self/cgroup   # 0::/ — the cgroup namespace hides the real path
id                      # uid=0(root): still real root, see step 6
```

(We use `chroot` here for simplicity; a real runtime uses `pivot_root` for the reasons above.)

### 4. Hit the limits

```sh
# pids.max = 20: the shell can't start more processes
for i in $(seq 30); do sleep 300 & done
# sh: can't fork: Resource temporarily unavailable

kill $(jobs -p) 2>/dev/null

# memory.max = 50M: allocate memory until the OOM killer steps in
tail /dev/zero          # tail buffers a "line" that never ends
# Killed
```

From a second terminal on the host, watch it happen:

```bash
cat /sys/fs/cgroup/box/pids.current
cat /sys/fs/cgroup/box/memory.events     # oom_kill counter goes up
```

### 5. Look at it from the host

In the second terminal:

```bash
PID=$(pgrep -f 'chroot /tmp/box/rootfs' | head -1)
PID=$(pgrep -P "$PID" | head -1)          # the sh running inside

ps -o pid,user,cmd -p "$PID"              # a normal process with a normal host PID
sudo ls -l /proc/$PID/ns/                 # compare with:
ls -l /proc/self/ns/                      # different inode numbers = different namespaces
sudo lsns -p "$PID"                       # list its namespaces

sudo nsenter -t "$PID" -a /bin/sh         # join all its namespaces, like docker exec
```

### 6. Notice what's missing

The process is still **real root with all capabilities and no seccomp filter**. Inside the box:

```sh
grep -E 'Cap(Eff|Bnd)|Seccomp|NoNewPrivs' /proc/self/status
# CapEff: 000001ffffffffff   ← every capability
# Seccomp: 0                 ← no filter
```

Compare with a real Docker container:

```bash
docker run --rm alpine grep -E 'Cap(Eff|Bnd)|Seccomp|NoNewPrivs' /proc/self/status
# CapEff: 00000000a80425fb   ← the 14 default capabilities
# Seccomp: 2                 ← filter mode
# NoNewPrivs: 0

capsh --decode=00000000a80425fb   # from libcap: lists them by name
```

Our hand-made box isolates visibility and resources but not privilege: root in it could still `mount` the host disk, load a kernel module or change the clock. That's the gap capabilities, seccomp, LSMs and user namespaces close, and roughly what `--privileged` reopens.

### 7. A rootless box with a user namespace

As a **normal user** (no sudo):

```bash
unshare --user --map-root-user --pid --fork --mount-proc /bin/bash

id                       # uid=0(root) gid=0(root)
cat /proc/self/uid_map   # "0  1000  1": UID 0 inside = your UID outside
touch /etc/should-fail   # Permission denied: you're root only in here
```

From another terminal, `ps -o user,pid,cmd -C bash` shows that bash running as your own user. If `unshare --user` is refused on Ubuntu 24.04 or later, that's the AppArmor restriction on unprivileged user namespaces (`kernel.apparmor_restrict_unprivileged_userns`).

### 8. Inspect a real container the same way

```bash
docker run -d --name iso nginx
PID=$(docker inspect -f '{{.State.Pid}}' iso)

sudo ls -l /proc/$PID/ns/              # its namespaces
cat /proc/$PID/cgroup                  # its cgroup path on the host
grep -E 'Cap|Seccomp' /proc/$PID/status
sudo cat /proc/$PID/attr/current       # AppArmor profile / SELinux label
docker rm -f iso
```

### 9. Clean up

```bash
exit                                    # leave the box (and the root shell)
sudo rmdir /sys/fs/cgroup/box           # works once no processes are left in it
rm -rf /tmp/box
```

## Cheat sheet

```text
Container = ordinary process + namespaces + cgroups + caps + seccomp + LSM + pivot_root
Shared kernel = the security boundary (VM: hypervisor is the boundary)

SEE     namespaces:  mnt uts ipc pid net user cgroup time
                     clone() / unshare() / setns() ; /proc/<pid>/ns/ ; lsns ; nsenter
                     PID 1 inside: killed → whole namespace dies; ignores unhandled signals
USE     cgroups v2:  /sys/fs/cgroup/…/{memory.max,cpu.max,cpu.weight,pids.max,io.max}
                     limits.memory → memory.max ; limits.cpu → cpu.max ; requests.cpu → cpu.weight
DO      capabilities: ~40; Docker keeps 14; CAP_SYS_ADMIN ≈ root
        no_new_privs: allowPrivilegeEscalation: false
        seccomp:      Docker default blocks ~44 syscalls ; K8s default = Unconfined!
        LSM:          AppArmor (paths) / SELinux (labels, per-container MCS)
FS      pivot_root (not chroot) + overlay image ; masked /proc paths
Runtime runc/crun read OCI config.json: rootfs → namespaces → cgroup → mounts →
        pivot_root → caps → LSM → seccomp → execve
Pod     shares net/ipc/uts (pause container) ; own mnt/pid unless shareProcessNamespace
Danger  privileged · docker.sock · CAP_SYS_ADMIN · hostPID/hostNetwork · hostPath /
Harden  drop ALL caps · RuntimeDefault seccomp · runAsNonRoot · readOnlyRootFilesystem
        hostUsers: false (GA in K8s 1.36) · gVisor · Kata/Firecracker (RuntimeClass)
Inspect /proc/<pid>/status (CapEff, Seccomp, NoNewPrivs) · capsh --decode · lsns
```

## References

[^1]: [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html), Linux manual page.
[^2]: [user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html), Linux manual page.
[^3]: [cgroups(7)](https://man7.org/linux/man-pages/man7/cgroups.7.html), Linux manual page.
[^4]: [Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html), Linux kernel documentation.
[^5]: [capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html), Linux manual page.
[^6]: [Docker Engine security](https://docs.docker.com/engine/security/), Docker documentation.
[^7]: [No New Privileges Flag](https://docs.kernel.org/userspace-api/no_new_privs.html), Linux kernel documentation.
[^8]: [seccomp(2)](https://man7.org/linux/man-pages/man2/seccomp.2.html), Linux manual page.
[^9]: [Seccomp security profiles for Docker](https://docs.docker.com/engine/security/seccomp/), Docker documentation.
[^10]: [Linux Container Configuration](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md), OCI Runtime Specification.
[^11]: [CVE-2019-5736](https://nvd.nist.gov/vuln/detail/CVE-2019-5736), NIST National Vulnerability Database.
[^12]: [Kubernetes v1.33: User Namespaces enabled by default!](https://kubernetes.io/blog/2025/04/25/userns-enabled-by-default/), Kubernetes blog.
[^13]: [CVE-2024-21626](https://nvd.nist.gov/vuln/detail/CVE-2024-21626), NIST National Vulnerability Database.
[^14]: [Kubernetes v1.36: User Namespaces in Kubernetes are finally GA](https://kubernetes.io/blog/2026/04/23/kubernetes-v1-36-userns-ga/), Kubernetes blog.
[^15]: [User Namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/), Kubernetes documentation.
[^16]: [gVisor documentation](https://gvisor.dev/docs/), gVisor project.
[^17]: [Kata Containers](https://katacontainers.io/), Kata Containers project.
