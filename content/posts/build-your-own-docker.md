---
title: "Build Your Own Docker: Containers from Scratch with Namespaces, cgroups and pivot_root"
date: 2026-11-18
draft: true
description: "Build a working container runtime from scratch: a root filesystem, chroot and pivot_root, Linux namespaces with unshare, resource limits with cgroups v2, image layers with overlayfs, networking with veth pairs, and finally a small container runtime in about 120 lines of Go."
summary: "A container is just a Linux process with a different view of the system. This post builds one by hand, first with shell commands and then as a ~120-line Go program, using the same kernel features Docker uses: namespaces, cgroups, pivot_root and overlayfs."
tags: ["docker", "containers", "linux", "namespaces", "cgroups", "go"]
---

Docker feels like a virtual machine: you run `docker run -it alpine sh` and you land in a shell with its own hostname, its own process list, its own filesystem and its own network. But there is no hypervisor, no guest kernel and no emulated hardware. A container is an ordinary Linux process that the kernel has been asked to lie to.

Three kernel features do the lying:

| Feature | Question it answers | What the container gets |
| --- | --- | --- |
| **Namespaces** | What can the process *see*? | Its own PIDs, hostname, mounts, network, IPC |
| **cgroups** | How much can the process *use*? | Limits on memory, CPU, number of processes |
| **A different root filesystem** | What files does the process *have*? | An image's files instead of the host's |

Everything else Docker does (images, registries, Dockerfiles, volumes, port mapping) is tooling around those three. In this post we build a container by hand, one feature at a time, using nothing but shell commands. Then we put it all together in a small Go program, `mydocker`, that can run `mydocker run ./rootfs /bin/sh` and drop you into an isolated, resource-limited shell.

You need a Linux machine (a throwaway VM is ideal, since we will be running things as root), the `util-linux` and `iproute2` packages (installed almost everywhere), and Go 1.20 or newer for the last part. Your kernel should use **cgroups v2**, which every current distribution does by default; check with `stat -fc %T /sys/fs/cgroup`, which should print `cgroup2fs`.

## Step 1: a root filesystem

A container image is, at its core, a directory tree: `/bin`, `/etc`, `/lib`, `/usr` and so on, with the binaries and libraries a program needs. The easiest way to get one is Alpine Linux's "mini root filesystem", a tarball of about 3 MB.[^alpine]

```sh
mkdir -p ~/mydocker/rootfs && cd ~/mydocker
curl -LO https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs
ls rootfs
# bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

(Pick the newest release from the Alpine downloads page; the version above is only an example.)

If you have Docker installed, you can also turn any image into a directory, which is a nice demonstration that an image really is just files:

```sh
docker export $(docker create alpine) | tar -x -C rootfs
```

## Step 2: chroot, the oldest trick

`chroot` changes the root directory for a process: from then on, any path starting with `/` is resolved inside the new directory.[^chroot]

```sh
$ sudo chroot rootfs /bin/sh
/ # cat /etc/os-release | head -1
NAME="Alpine Linux"
/ # ls /home
/ #
```

The shell is now running Alpine's BusyBox `sh`, it cannot see the host's `/home`, and it would happily run Alpine binaries even on an Ubuntu host. That is already the "different root filesystem" part of a container. But try this:

```sh
/ # mount -t proc proc /proc
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:03 /sbin/init
    2 root      0:00 [kthreadd]
  ...           (every process on the host)
/ # hostname
my-laptop
/ # exit
$ sudo umount rootfs/proc
```

The process sees every process on the host, shares the host's hostname, and the `proc` mount we made inside the chroot is visible (and has to be cleaned up) on the host. The filesystem view changed; nothing else did. Worse, `chroot` only affects path lookup for that one process, and a root process inside a chroot can escape it with well-known tricks. We need namespaces.

## Step 3: namespaces with unshare

A **namespace** wraps a global system resource so that processes inside it see their own isolated instance.[^namespaces] Linux has eight kinds:

| Namespace | Flag | Isolates |
| --- | --- | --- |
| Mount | `CLONE_NEWNS` | The list of mounted filesystems |
| UTS | `CLONE_NEWUTS` | Hostname and domain name |
| PID | `CLONE_NEWPID` | Process ID numbers |
| Network | `CLONE_NEWNET` | Interfaces, IPs, routes, firewall, ports |
| IPC | `CLONE_NEWIPC` | System V IPC, POSIX message queues |
| User | `CLONE_NEWUSER` | User and group IDs, capabilities |
| Cgroup | `CLONE_NEWCGROUP` | The view of the cgroup hierarchy |
| Time | `CLONE_NEWTIME` | Boot and monotonic clocks |

The `unshare` command runs a program in new namespaces.[^unshare] Let's combine it with our chroot:

```sh
sudo unshare --pid --fork --mount --uts --ipc --net chroot rootfs /bin/sh
```

The flags, one by one:

- `--pid --fork`: a new PID namespace. The first process created in it becomes PID 1, which is why `--fork` is needed: `unshare` itself stays outside, and its child is the new PID 1.
- `--mount`: a new mount namespace. `unshare` also makes all mounts private by default, so mounts made inside never propagate back to the host.
- `--uts`: our own hostname.
- `--ipc`: our own shared memory and message queues.
- `--net`: a completely empty network stack.

Now repeat the experiment:

```sh
/ # mount -t proc proc /proc
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    3 root      0:00 ps
/ # hostname container && hostname
container
/ # ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

The shell believes it is PID 1, the hostname change did not touch the host, and there is no network except a loopback device that is switched off. From a second terminal on the host, though, the shell is a perfectly normal process with a normal PID:

```sh
$ ps -ef | grep '/bin/sh' | grep -v grep
root     41237 41236  0 10:12 pts/1    00:00:00 /bin/sh
$ sudo ls -l /proc/41237/ns/
lrwxrwxrwx 1 root root 0 ... ipc -> 'ipc:[4026532611]'
lrwxrwxrwx 1 root root 0 ... mnt -> 'mnt:[4026532609]'
lrwxrwxrwx 1 root root 0 ... net -> 'net:[4026532614]'
lrwxrwxrwx 1 root root 0 ... pid -> 'pid:[4026532612]'
lrwxrwxrwx 1 root root 0 ... uts -> 'uts:[4026532610]'
...
```

Same process, two different PIDs depending on who is asking. That is the whole idea of a container in one screen. `lsns` lists all namespaces on the system, and `nsenter -t 41237 -a` lets you join them, which is exactly what `docker exec` does.

## Step 4: pivot_root instead of chroot

There is still a problem. Inside our new mount namespace, the host's root filesystem is still mounted; we just cannot reach it by path because `chroot` hides it. Real runtimes use `pivot_root` instead, which *swaps* the root mount of the mount namespace and moves the old one somewhere we can then unmount.[^pivotroot] After that, the host filesystem is not just hidden, it is gone from the container's mount table.

`pivot_root` has a few requirements: the new root must be a mount point, and neither it nor the old root may be a shared mount. A bind mount of the directory onto itself takes care of the first; the private propagation that `unshare --mount` sets up takes care of the second.

```sh
sudo unshare --pid --fork --mount --uts --ipc --net /bin/sh

# everything below runs inside the new namespaces, still with the host's root
mount --bind "$HOME/mydocker/rootfs" "$HOME/mydocker/rootfs"   # make it a mount point
cd "$HOME/mydocker/rootfs"
mkdir -p .old_root
pivot_root . .old_root      # new root is ".", the old one ends up in .old_root
cd /
mount -t proc proc /proc
umount -l /.old_root        # detach the host filesystem
rmdir /.old_root
exec /bin/sh                # replace the host's sh with Alpine's
```

(When run with `sudo`, `$HOME` may point at root's home; use the absolute path to your `rootfs` if so.)

Now `cat /proc/mounts` inside shows only the root filesystem and `/proc`. There is no path back to the host, not even for root.

## Step 5: limits with cgroups v2

Namespaces control what a process can see, not how much it can use. A container that allocates all of the host's memory or forks until the process table is full takes the host down with it. **Control groups** (cgroups) fix that: they organise processes into a hierarchy and let you put limits on each group.[^cgroups]

In cgroups v2 the whole interface is a filesystem mounted at `/sys/fs/cgroup`. Creating a group is `mkdir`; setting a limit is writing to a file; adding a process is writing its PID to `cgroup.procs`.

```sh
# make sure the controllers are enabled for child groups (systemd usually does this already)
cat /sys/fs/cgroup/cgroup.subtree_control
# cpuset cpu io memory hugetlb pids rdma misc

sudo mkdir /sys/fs/cgroup/mydocker
echo 100M          | sudo tee /sys/fs/cgroup/mydocker/memory.max      # hard memory limit
echo 0             | sudo tee /sys/fs/cgroup/mydocker/memory.swap.max # no swapping around it
echo 20            | sudo tee /sys/fs/cgroup/mydocker/pids.max        # at most 20 processes
echo "50000 100000"| sudo tee /sys/fs/cgroup/mydocker/cpu.max         # 50 ms per 100 ms = half a CPU
```

Start a container as in step 3 and, from the host, move its shell into the group:

```sh
echo 41237 | sudo tee /sys/fs/cgroup/mydocker/cgroup.procs
```

Every process the shell starts from now on inherits the group. Let's test the limits from inside the container:

```sh
/ # for i in $(seq 30); do sleep 300 & done
sh: can't fork: Resource temporarily unavailable
/ # kill $(jobs -p)

/ # head -c 500M /dev/zero | tail
Killed
```

The loop stops after about 18 `sleep`s, because the shell, `seq`, and the sleeps together hit the limit of 20. The second command makes `tail` buffer 500 MB looking for a newline that never comes; at 100 MB the kernel's OOM killer kills it, and only it. The host does not notice. You can read the group's statistics in `memory.current`, `memory.events` (look at `oom_kill`), `pids.current` and `cpu.stat`.

This is exactly what `docker run --memory 100m --pids-limit 20 --cpus 0.5` configures. To clean up, the group must be empty, then `sudo rmdir /sys/fs/cgroup/mydocker`.

## Step 6: image layers with overlayfs

If every container got its own full copy of the root filesystem, starting a hundred containers from the same image would mean a hundred copies. Docker avoids that with a **union filesystem**, by default overlayfs.[^overlayfs] Overlayfs stacks a read-only *lower* directory (the image) under a writable *upper* directory (the container's changes) and presents the merge of both:

```sh
cd ~/mydocker
mkdir -p layers/upper layers/work layers/merged
sudo mount -t overlay overlay \
  -o lowerdir=$PWD/rootfs,upperdir=$PWD/layers/upper,workdir=$PWD/layers/work \
  $PWD/layers/merged
```

`layers/merged` looks like a full Alpine root filesystem. Now change something in it:

```sh
$ echo hello | sudo tee layers/merged/hello.txt
$ sudo rm layers/merged/etc/motd
$ ls -l layers/upper layers/upper/etc
layers/upper:
-rw-r--r-- 1 root root    6 ... hello.txt
drwxr-xr-x 2 root root 4096 ... etc

layers/upper/etc:
c--------- 1 root root 0, 0 ... motd
$ ls rootfs/etc/motd
rootfs/etc/motd
```

The new file landed in `upper`. The deleted file is still in `rootfs` (the image is untouched), and `upper` records the deletion as a **whiteout**, a character device with device number 0/0. A file that is modified is first copied up from lower to upper in full ("copy-on-write").

Docker images take this one step further: `lowerdir` accepts a colon-separated list, so each layer of an image (one per `RUN`, `COPY` and so on in a Dockerfile) is a separate read-only directory, shared between all images and containers that use it. Point the container at `layers/merged` instead of `rootfs`, and every container gets a private, disposable writable layer for free. `docker diff` is literally a listing of the upper directory.

## Step 7: networking

Our container has `--net` isolation and therefore no network at all. Docker's default bridge network is built from three pieces: a **veth pair** (a virtual cable with one end on the host and one in the container), a **bridge** (a virtual switch, `docker0`) connecting all the host ends, and an **iptables NAT rule** so that traffic leaving the bridge gets the host's address. I've covered building that by hand in detail in [Linux Network Namespaces]({{< relref "linux-network-namespaces.md" >}}), so here is the minimal version: a single cable between host and container.

Find the PID of a running container from step 3, then on the host:

```sh
PID=$(pgrep -P "$(pgrep -n unshare)")       # the child of the newest unshare process

sudo ip link add veth-host type veth peer name veth-ctr
sudo ip link set veth-ctr netns "$PID"       # push one end into the container's namespace
sudo ip addr add 10.10.0.1/24 dev veth-host
sudo ip link set veth-host up
```

Inside the container:

```sh
/ # ip link set lo up
/ # ip addr add 10.10.0.2/24 dev veth-ctr
/ # ip link set veth-ctr up
/ # ping -c1 10.10.0.1
64 bytes from 10.10.0.1: seq=0 ttl=64 time=0.071 ms
```

Add a bridge, a default route via `10.10.0.1`, `net.ipv4.ip_forward=1` and a `MASQUERADE` rule, and you have reinvented `docker0`.

## Step 8: putting it together in Go

Now let's turn the shell session into a program. Docker's low-level runtime, `runc`, is written in Go, and Go's `syscall` package exposes everything we need.[^sysprocattr] The design follows Liz Rice's well-known "Containers From Scratch" talk:[^lizrice]

1. `mydocker run <rootfs> <cmd>` creates a cgroup and re-executes **itself** (`/proc/self/exe`) as `mydocker child <rootfs> <cmd>`, asking the kernel to start that child in new namespaces and directly inside the cgroup.
2. The child, now inside the namespaces, sets the hostname, pivots into the root filesystem, mounts `/proc` and finally `exec`s the user's command, which replaces the child and becomes PID 1 of the container.

Why the re-exec? Some setup (hostname, mounts, `pivot_root`) has to happen *inside* the new namespaces but *before* the user's command starts. In C you would `clone()` and run a function in the child. The Go runtime is multi-threaded and cannot safely do that, so the usual pattern is to start a fresh copy of the binary in the new namespaces and do the setup there. `runc` solves the same problem with a small C constructor (`nsexec`) that runs before the Go runtime starts.

```go
// mydocker: a tiny container runtime.
// Usage: sudo ./mydocker run <rootfs> <command> [args...]
package main

import (
	"errors"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strconv"
	"syscall"
)

func main() {
	if len(os.Args) < 4 {
		fmt.Fprintln(os.Stderr, "usage: mydocker run <rootfs> <command> [args...]")
		os.Exit(2)
	}
	switch os.Args[1] {
	case "run":
		run(os.Args[2], os.Args[3:])
	case "child":
		child(os.Args[2], os.Args[3:])
	default:
		fmt.Fprintf(os.Stderr, "unknown command %q\n", os.Args[1])
		os.Exit(2)
	}
}

// run is executed on the host. It prepares a cgroup and starts
// "mydocker child ..." in new namespaces, inside that cgroup.
func run(rootfs string, args []string) {
	rootfs, err := filepath.Abs(rootfs)
	must(err)

	cg, err := createCgroup("mydocker-" + strconv.Itoa(os.Getpid()))
	must(err)
	defer os.Remove(cg) // rmdir; only works once the cgroup is empty

	cgDir, err := os.Open(cg)
	must(err)
	defer cgDir.Close()

	cmd := exec.Command("/proc/self/exe", append([]string{"child", rootfs}, args...)...)
	cmd.Stdin, cmd.Stdout, cmd.Stderr = os.Stdin, os.Stdout, os.Stderr
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | // hostname
			syscall.CLONE_NEWPID | // process IDs
			syscall.CLONE_NEWNS | // mounts
			syscall.CLONE_NEWIPC | // System V IPC, message queues
			syscall.CLONE_NEWNET, // network (only a downed lo)
		// Start the child directly inside the cgroup (clone3 + CLONE_INTO_CGROUP),
		// so there is no window in which it runs unlimited.
		UseCgroupFD: true,
		CgroupFD:    int(cgDir.Fd()),
	}

	err = cmd.Run()
	var exitErr *exec.ExitError
	if errors.As(err, &exitErr) {
		os.Remove(cg)
		os.Exit(exitErr.ExitCode())
	}
	must(err)
}

// child is executed inside the new namespaces. It finishes the setup
// and then replaces itself with the user's command.
func child(rootfs string, args []string) {
	must(syscall.Sethostname([]byte("container")))

	// Make sure no mount we create propagates back to the host.
	must(syscall.Mount("", "/", "", syscall.MS_PRIVATE|syscall.MS_REC, ""))
	must(pivotRoot(rootfs))
	must(syscall.Mount("proc", "/proc", "proc", 0, ""))

	env := []string{
		"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
		"HOME=/root",
		"TERM=" + os.Getenv("TERM"),
	}
	must(os.Setenv("PATH", env[0])) // so LookPath searches the container's PATH
	path, err := exec.LookPath(args[0])
	must(err)

	// exec replaces this process: the user's command becomes PID 1.
	must(syscall.Exec(path, args, env))
}

// pivotRoot makes newRoot the root of the current mount namespace
// and detaches the old root, so the host filesystem is unreachable.
func pivotRoot(newRoot string) error {
	// pivot_root(2) requires the new root to be a mount point.
	if err := syscall.Mount(newRoot, newRoot, "", syscall.MS_BIND|syscall.MS_REC, ""); err != nil {
		return fmt.Errorf("bind mount rootfs: %w", err)
	}
	putOld := filepath.Join(newRoot, ".old_root")
	if err := os.MkdirAll(putOld, 0o700); err != nil {
		return err
	}
	if err := syscall.PivotRoot(newRoot, putOld); err != nil {
		return fmt.Errorf("pivot_root: %w", err)
	}
	if err := os.Chdir("/"); err != nil {
		return err
	}
	if err := syscall.Unmount("/.old_root", syscall.MNT_DETACH); err != nil {
		return fmt.Errorf("unmount old root: %w", err)
	}
	return os.Remove("/.old_root")
}

// createCgroup creates a cgroup v2 group with memory, swap, process and CPU limits.
func createCgroup(name string) (string, error) {
	dir := filepath.Join("/sys/fs/cgroup", name)
	if err := os.Mkdir(dir, 0o755); err != nil {
		return "", err
	}
	limits := map[string]string{
		"memory.max":      "100M",
		"memory.swap.max": "0",
		"pids.max":        "64",
		"cpu.max":         "50000 100000", // half a CPU
	}
	for file, value := range limits {
		if err := os.WriteFile(filepath.Join(dir, file), []byte(value), 0o644); err != nil {
			return "", fmt.Errorf("%s: %w", file, err)
		}
	}
	return dir, nil
}

func must(err error) {
	if err != nil {
		fmt.Fprintln(os.Stderr, "mydocker:", err)
		os.Exit(1)
	}
}
```

Build it and run it against the Alpine root filesystem from step 1:

```sh
$ go mod init mydocker && go build -o mydocker .
$ sudo ./mydocker run ~/mydocker/rootfs /bin/sh
/ # hostname
container
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    2 root      0:00 ps
/ # cat /proc/mounts
/dev/sda2 / ext4 rw,relatime 0 0
proc /proc proc rw,relatime 0 0
/ # exit
$ ls /sys/fs/cgroup | grep mydocker
$
```

`UseCgroupFD` needs Linux 5.7 or newer and Go 1.20 or newer. On older systems you can instead write the child's PID into the cgroup's `cgroup.procs` right after `cmd.Start()`, at the cost of a short window in which the child runs unlimited.

Some easy extensions, each a few lines:

- **Copy-on-write layers:** mount an overlay as in step 6 in `run` and pass the merged directory as the rootfs, so containers never modify the image.
- **Networking:** after `cmd.Start()`, create a veth pair and move one end into `cmd.Process.Pid`'s network namespace, as in step 7.
- **`/dev` and `/sys`:** mount a `tmpfs` on `/dev` with a handful of device nodes (`null`, `zero`, `random`, `urandom`, `tty`) and a read-only `sysfs` on `/sys`.
- **`mydocker exec`:** open `/proc/<pid>/ns/*` and call `setns(2)` on each, which is what `nsenter` does.

## What real Docker adds

`mydocker` is a fun toy, but it is not safe to run untrusted code in. The process inside is still root with almost every capability, and can call any system call the kernel offers. A production runtime layers several more defences on top of namespaces and cgroups:

- **Capabilities.** Root's powers are split into about forty capabilities, such as `CAP_SYS_ADMIN` or `CAP_NET_RAW`.[^capabilities] Docker drops most of them by default; our container keeps them all, so for example it can mount filesystems or load kernel modules if they are available.
- **seccomp.** A filter that blocks dangerous system calls. Docker's default profile disables around 44 of the 300+ system calls, such as `reboot`, `kexec_load` and `init_module`.[^seccomp]
- **User namespaces.** Map root inside the container to an unprivileged user outside. This is what makes rootless Docker and Podman possible, and what turns "root in the container" from dangerous into merely inconvenient.
- **LSMs.** AppArmor or SELinux profiles restrict which files and operations the container may touch, even as root.
- **Masked paths.** Parts of `/proc` and `/sys` that leak host information or allow host changes (`/proc/kcore`, `/proc/sys`, ...) are hidden or mounted read-only.
- **Images and registries.** Pulling, verifying and unpacking layered OCI images, which is a whole project of its own.

The architecture has also been split into layers. The `docker` CLI talks to `dockerd`, which delegates container lifecycle to `containerd`, which uses a shim process to call `runc`. `runc` is the piece we just rebuilt: it takes a root filesystem and a JSON config defined by the OCI Runtime Specification,[^oci] sets up namespaces, cgroups, mounts, capabilities and seccomp, and then `exec`s your process. Kubernetes talks to containerd (or CRI-O) directly and never involves Docker at all, which is why "Kubernetes removed Docker" changed so little in practice: the containers were always `runc` underneath.

## Wrapping up

A container is a process that

1. runs in its own **namespaces**, so it sees its own processes, hostname, mounts and network,
2. sits in a **cgroup**, so it cannot use more than its share,
3. has a **different root filesystem**, usually an overlay of read-only image layers plus a writable layer, installed with `pivot_root`.

That fits in about 120 lines of Go. The next time a container misbehaves, you know where to look: `/proc/<pid>/ns/` for its namespaces, `/sys/fs/cgroup/.../memory.events` for OOM kills, `/proc/<pid>/root` for its filesystem, and `nsenter` to step inside with your host's tools.

[^alpine]: [Downloads](https://alpinelinux.org/downloads/), Alpine Linux.
[^chroot]: [chroot(2) — Linux manual page](https://man7.org/linux/man-pages/man2/chroot.2.html), Michael Kerrisk, man7.org.
[^namespaces]: [namespaces(7) — Linux manual page](https://man7.org/linux/man-pages/man7/namespaces.7.html), Michael Kerrisk, man7.org.
[^unshare]: [unshare(1) — Linux manual page](https://man7.org/linux/man-pages/man1/unshare.1.html), util-linux, man7.org.
[^pivotroot]: [pivot_root(2) — Linux manual page](https://man7.org/linux/man-pages/man2/pivot_root.2.html), Michael Kerrisk, man7.org.
[^cgroups]: [Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html), The Linux Kernel documentation.
[^overlayfs]: [Overlay Filesystem](https://docs.kernel.org/filesystems/overlayfs.html), The Linux Kernel documentation.
[^sysprocattr]: [syscall.SysProcAttr](https://pkg.go.dev/syscall#SysProcAttr), The Go Programming Language.
[^lizrice]: [Containers From Scratch](https://github.com/lizrice/containers-from-scratch), Liz Rice, GitHub.
[^capabilities]: [capabilities(7) — Linux manual page](https://man7.org/linux/man-pages/man7/capabilities.7.html), Michael Kerrisk, man7.org.
[^seccomp]: [Seccomp security profiles for Docker](https://docs.docker.com/engine/security/seccomp/), Docker Docs.
[^oci]: [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec), Open Container Initiative, GitHub.
