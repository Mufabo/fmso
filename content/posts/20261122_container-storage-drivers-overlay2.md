---
title: "Container Storage Drivers: How overlay2 Turns Image Layers into a Filesystem"
date: 2026-11-22
draft: true
description: "What a container storage driver does, how overlay2 and the kernel's OverlayFS stack read-only image layers under a thin writable layer, copy-up and whiteouts, the other drivers (btrfs, zfs, vfs, fuse-overlayfs, and the removed aufs and devicemapper), containerd snapshotters and Docker 29's switch to the containerd image store. With a hands-on lab that builds an overlay mount by hand."
summary: "Fifty containers from a 1 GB image don't use 50 GB of disk. The storage driver is why: it stacks shared, read-only image layers under a thin writable layer per container. Here is how overlay2 does it, what copy-up and whiteouts are, and why your database belongs in a volume."
tags: ["linux", "containers", "docker", "containerd", "storage", "overlayfs", "kubernetes"]
categories: ["Programming"]
---

Start 50 containers from the same 1 GB image and check your disk: you haven't used 50 GB. Each container sees what looks like its own complete root filesystem, can write anywhere in it, and none of them see each other's changes. Yet on disk, the image exists exactly once.

The part of the container runtime that pulls this off is the **storage driver** (Docker's term) or **snapshotter** (containerd's term). This post explains what it does, how the default one, overlay2, works on top of the kernel's OverlayFS, what the alternatives are, and what it all means for how you build images and where you put data.

If you've read my post on [Linux storage, block devices and LVM]({{< relref "20261122_linux-storage-block-devices-filesystems-lvm.md" >}}), this sits one layer up: everything here happens *on top of* an ordinary file system like ext4 or XFS.

## What a storage driver does

A container image is a stack of **layers**. Roughly, each `RUN`, `COPY` and `ADD` instruction in a Dockerfile produces one layer, which is a tarball of the files that step added, changed or deleted:

```dockerfile
FROM debian:bookworm-slim          # layer 1: the base OS
RUN apt-get update && \
    apt-get install -y python3     # layer 2: python and its libraries
COPY app/ /app                     # layer 3: your code
```

When you run a container, the runtime has to present those layers as **one directory tree** that the container uses as `/`, plus somewhere for the container to write. Docker's documentation sums up the job: storage drivers manage the image layers and the writable container layer, and while each driver does it differently, all of them use stackable layers and copy-on-write.[^1]

So the storage driver is responsible for:

1. **Storing layers** on the host, once each, shared between every image and container that uses them.
2. **Stacking** a container's layers into a single merged view.
3. **Giving each container a writable layer** on top, private to that container.
4. **Copy-on-write (CoW)**: when a container modifies a file that came from the image, the change goes into its writable layer, and the shared image layer is left untouched.

```text
   Container A              Container B              Container C
 ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
 │ writable (A) │         │ writable (B) │         │ writable (C) │   thin, per container
 └──────┬───────┘         └──────┬───────┘         └──────┬───────┘
        └────────────────────────┼────────────────────────┘
                ┌────────────────▼─────────────────┐
                │ layer 3: /app                     │
                ├───────────────────────────────────┤
                │ layer 2: python3                  │   read-only, shared,
                ├───────────────────────────────────┤   stored once
                │ layer 1: debian base              │
                └───────────────────────────────────┘
```

When the container is deleted, its writable layer is deleted with it. That single fact explains most of the practical advice at the end of this post.

## OverlayFS: the kernel feature underneath

overlay2 is a thin piece of Docker code; the real work is done by **OverlayFS**, a union filesystem that has been in the mainline Linux kernel since 3.18. Docker's docs use the two names consistently: OverlayFS is the kernel driver, overlay2 the Docker storage driver built on it.[^2]

OverlayFS takes several directories and presents them as one:

```text
   merged/      ← the combined view (the container's "/")
     ▲
   upperdir     ← writable; all changes land here
     ▲
   lowerdir(s)  ← read-only; one or many, stacked
```

plus a `workdir`, an empty scratch directory on the same filesystem as `upperdir` that OverlayFS uses to make some operations atomic. You can build one yourself with a plain `mount` command (the lab at the end does exactly that):

```bash
mount -t overlay overlay \
  -o lowerdir=/l2:/l1,upperdir=/upper,workdir=/work \
  /merged
```

In `lowerdir`, the **leftmost directory is the top** of the stack: here `/l2` overrides `/l1`.

The kernel's OverlayFS documentation describes the rules,[^3] and they're simple once you see them as "look from the top down":

### Reading

When something opens or lists a path in `merged/`, OverlayFS looks in `upperdir` first, then down through each `lowerdir`. The first match wins. For directories, the contents of every layer are merged, so `/usr/bin` shows binaries from all layers.

Reads of an unmodified file go straight to the file in its lower layer. There's no copying and essentially no overhead, and the page cache is shared too: if 50 containers run the same `python3` binary from the same layer, the kernel caches it once.

### Writing: copy-up

The lower layers are read-only, so what happens when a container modifies `/etc/nginx/nginx.conf` from the image? OverlayFS performs a **copy-up**: it copies the file into `upperdir`, then lets the write happen to the copy. From then on, the upper copy hides the lower one.

Two consequences:

- **The first write is expensive, and it copies the whole file.** OverlayFS works at file level, not block level. Appending one line to a 2 GB file from the image copies all 2 GB first. Even changing only metadata (permissions, owner, timestamps) normally triggers a full copy-up, although newer kernels have an optional `metacopy` feature that copies only metadata in that case.[^3]
- **Later writes are as fast as the underlying filesystem**, because the file now simply lives in `upperdir`.

New files are created directly in `upperdir`; nothing needs copying.

### Deleting: whiteouts and opaque directories

Deleting a file that exists in a lower layer can't remove it from there. Instead, OverlayFS creates a **whiteout** in `upperdir`: a character device with device number 0/0 and the same name. When OverlayFS sees a whiteout, it hides that name in every layer below.

Deleting a directory and re-creating it with the same name uses an **opaque directory**: a directory in `upperdir` marked with the extended attribute `trusted.overlay.opaque="y"`, which tells OverlayFS not to merge in anything from lower layers under that name.[^3]

This is exactly why the following Dockerfile doesn't make the image smaller:

```dockerfile
RUN curl -o /tmp/big.tar.gz https://example.com/big.tar.gz   # layer adds 500 MB
RUN rm /tmp/big.tar.gz                                      # layer adds a whiteout
```

The second layer contains a whiteout, so the file is invisible in the container, but the 500 MB are still in the first layer and are still downloaded by everyone who pulls the image. The fix is to download, use and delete in **one** `RUN` step, or to use a multi-stage build and copy only the result into the final image.

### Renaming directories

One oddity worth knowing: renaming a directory that comes from a lower layer is, by default, refused with `EXDEV` ("cross-device link"), the same error you get when renaming across filesystems. Well-behaved tools like `mv` handle that by falling back to copy-and-delete, so you rarely notice, but some programs that call `rename()` directly break. Newer kernels can handle it with the `redirect_dir` feature.[^3]

## How overlay2 uses OverlayFS

Docker's overlay2 driver keeps everything under `/var/lib/docker/overlay2/`.[^2] Each image layer gets a directory:

```text
/var/lib/docker/overlay2/
├── 3a36935c9df3…/          # one directory per layer
│   ├── diff/               # the layer's actual files
│   ├── link                # a short ID, e.g. 6Y5IM2XC7TSNIJZZFLJCS6I4I4
│   ├── lower               # the short IDs of the layers below it
│   ├── work/
│   └── merged/             # only for container layers while running
├── 223c28641754…/
└── l/                      # short symlinks → ../<layer-id>/diff
```

When a container starts, overlay2 creates a new layer for its writable layer and mounts an overlay with the image's `diff` directories as `lowerdir` and the container's `diff` as `upperdir`. The `l/` directory of short symlinks exists because the mount options string has a length limit of about one page (4 KiB). With dozens of layers, the full 64-character layer IDs wouldn't fit, so overlay2 uses the short names instead.

You can see the mount for any container:

```bash
docker inspect <container> --format '{{json .GraphDriver.Data}}' | jq
# {
#   "LowerDir":  "/var/lib/docker/overlay2/…-init/diff:/var/lib/docker/overlay2/…/diff:…",
#   "UpperDir":  "/var/lib/docker/overlay2/…/diff",
#   "WorkDir":   "/var/lib/docker/overlay2/…/work",
#   "MergedDir": "/var/lib/docker/overlay2/…/merged"
# }

mount | grep overlay          # the kernel's view of the same thing
```

The `-init` layer sits between the image and the container's writable layer. Docker puts files there that every container needs regardless of the image, such as placeholder `/etc/hosts`, `/etc/resolv.conf` and `/etc/hostname`, which it then bind-mounts over.

### Why "2"?

There used to be an `overlay` driver too. Early OverlayFS supported only a single lower directory, so the old driver faked multi-layer images with hard links, which was slow and could exhaust inodes. overlay2 uses the kernel's native multi-lower-directory support (kernel 4.0+). The legacy `overlay` driver was deprecated in Docker 18.09 and removed in 24.0.[^4]

### Requirements

overlay2 needs a supported **backing filesystem** for `/var/lib/docker`: ext4, or XFS formatted with `ftype=1` (support for `d_type`, which tells the kernel the type of each directory entry). Modern `mkfs.xfs` sets that by default, but very old XFS filesystems may lack it. `docker info` reports both:[^2]

```text
Storage Driver: overlay2
 Backing Filesystem: xfs
 Supports d_type: true
```

OverlayFS also can't use another OverlayFS as its upper layer. That's the classic Docker-in-Docker trap: the inner Docker's `/var/lib/docker` is itself on overlay, so it fails or falls back to `vfs`. Mounting a volume at the inner `/var/lib/docker` fixes it.

## The other drivers

overlay2 is the default everywhere, and Docker's docs describe it as the most widely compatible driver across currently supported distributions, needing no extra configuration.[^5] The others exist for specific situations or history:

| Driver | How it works | When to use it |
|---|---|---|
| **overlay2** | OverlayFS, file-level CoW | Default. Use it unless you have a reason not to |
| **fuse-overlayfs** | OverlayFS reimplemented in user space via FUSE | Rootless containers on kernels older than 5.11. Since 5.11, unprivileged users can mount overlay directly, so overlay2 works rootless[^5] |
| **btrfs** | Each layer is a btrfs subvolume; layers are snapshots. Block-level CoW | Only if `/var/lib/docker` is on btrfs |
| **zfs** | Each layer is a ZFS dataset; layers are clones of snapshots | Only on ZFS hosts. Strong integrity features, but uses more memory |
| **vfs** | No CoW at all: every layer is a full copy of the previous one | Testing, or filesystems nothing else supports. Slow and space-hungry |
| **aufs** | The original Docker union filesystem, never in mainline Linux | Removed in Docker 24.0[^4] |
| **devicemapper** | Thin-provisioned block devices (the LVM thin-pool machinery) | Removed in Docker 25.0[^4] |
| **windowsfilter** | Windows container layers | The only driver for Windows hosts[^5] |

The key distinction is **file-level versus block-level** copy-on-write:

- **File-level** (overlay2, aufs): copy-up copies whole files. Cheap for small files, painful for large ones. Layers are plain directories you can browse, and page cache is shared between containers.
- **Block-level** (btrfs, zfs, devicemapper): only the changed blocks are copied, which is much better for modifying large files. But layers are snapshots inside one filesystem, and page cache sharing is weaker or absent, so many containers from the same image use more RAM.

For almost every workload, overlay2's trade-offs win, especially once you follow the "put heavy writes in volumes" rule below.

To check or set the driver:

```bash
docker info --format '{{.Driver}}'
```

```json
// /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}
```

Changing the driver makes existing images and containers invisible, because they're stored in the old driver's format. `docker save` or push anything you need first.

## containerd snapshotters

Docker isn't the only runtime. containerd (used by most Kubernetes clusters) has the same concept under a different name: a **snapshotter**. A snapshot is a layer, and a snapshotter prepares the stacked, mountable view for a container.[^6]

The default is the `overlayfs` snapshotter, which works like overlay2. On a Kubernetes node you'll find the layers here:

```bash
ls /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/
# 1  2  3  …  each with fs/ (the files) and work/

sudo ctr -n k8s.io snapshots ls        # containerd's view
```

Other snapshotters include `native` (plain copies, like vfs), `btrfs`, `zfs`, `devmapper` (block devices, used by microVM runtimes like Firecracker), and **remote** snapshotters such as stargz, SOCI, nydus and overlaybd. The remote ones enable **lazy pulling**: the container starts before the image has fully downloaded, fetching file contents on demand. That can cut startup time for large images dramatically.

Podman and CRI-O use yet another library, `containers/storage`, configured in `/etc/containers/storage.conf`. Its default driver is also called `overlay`, and it's also OverlayFS underneath.

### Docker 29: containerd takes over

This is the most important recent change. **As of Docker Engine 29, new installations use the containerd image store by default**, and the classic graph drivers, overlay2 included, are deprecated. Installations that were upgraded keep using overlay2 until you switch.[^7][^8]

With the containerd image store, Docker hands image and layer management to containerd, and its overlayfs snapshotter replaces overlay2.[^8] The underlying mechanism (OverlayFS, copy-up, whiteouts) is the same, so everything in the first half of this post still applies. What changes in practice:

- **Where data lives.** Layers are under containerd's root (by default `/var/lib/containerd`), not `/var/lib/docker`. Docker's `data-root` setting no longer moves them, which surprises people who put `/var/lib/docker` on a separate disk.
- **Disk usage is higher.** containerd keeps images in both compressed and uncompressed form, so each layer takes more space.[^8]
- **New features.** Multi-platform images, attestations (SBOMs, provenance) and lazy pulling with remote snapshotters work natively.[^8][^9]
- **Some limits.** It isn't available with user namespace remapping (`userns-remap`).[^8]

Check which one you're on:

```bash
docker info -f '{{ .DriverStatus }}'
# [[driver-type io.containerd.snapshotter.v1]]   → containerd image store
# [[Backing Filesystem extfs] [Supports d_type true] …]   → classic overlay2
```

Switching an existing host makes your old overlay2 images invisible (they're not deleted; they come back if you switch back), so push or `docker save` what you need first.[^8]

## Practical consequences

### Put write-heavy data in volumes

This is the big one, and Docker's own documentation says it directly: use volumes for write-heavy workloads rather than the container's writable layer.[^5] Databases, message queues, caches and log files should live in a volume or bind mount, for three reasons:

1. **Performance.** Copy-up of large files is slow, and every write goes through the overlay layer. Volumes are plain directories on the host filesystem (bind-mounted into the container), bypassing the storage driver entirely.
2. **Durability.** The writable layer disappears with the container. `docker rm` on a database container without a volume deletes the database.
3. **Size.** The writable layer grows on the host's container disk with no natural limit, and is easy to forget about.

In Kubernetes, the same applies: data goes in a PersistentVolume or at least an `emptyDir`, not the container filesystem. The kubelet counts writable-layer usage towards a pod's `ephemeral-storage` and evicts pods that exceed their limit.

### Build images with layers in mind

- Clean up in the **same** `RUN` step that created the mess (`apt-get install … && rm -rf /var/lib/apt/lists/*`). Deleting in a later step only adds a whiteout.
- Use **multi-stage builds** so compilers and build caches never reach the final image.
- Order instructions from **least to most frequently changed** (base, dependencies, then your code), so the unchanged lower layers stay shared and cached.
- Don't `chmod -R` or `chown -R` large directories in a later layer: changing metadata copies every file into a new layer, doubling its size. Use `COPY --chown` / `COPY --chmod` instead.

### Inspect what a container changed

```bash
docker diff <container>       # A = added, C = changed, D = deleted
docker ps -s                  # SIZE = writable layer, (virtual) = including image
docker system df -v           # disk usage per image, container and volume
```

A container whose writable layer keeps growing is usually writing logs or temp files that should go to a volume, stdout, or a `tmpfs` mount.

### Watch inodes and the backing disk

Every layer is many small files on the host, and image-heavy hosts (CI runners especially) can run out of **inodes** before they run out of space. `df -i /var/lib/docker` (or `/var/lib/containerd`) shows it. `docker image prune -a` and `docker builder prune` reclaim space.

## Hands-on lab: an overlay mount by hand

Nothing here needs Docker. You need root and a kernel with OverlayFS (any modern distribution). Everything happens under `/tmp/ovl`.

```bash
# 1. Two "image layers", an upper (writable) layer, a workdir and a mountpoint
mkdir -p /tmp/ovl/{lower1,lower2,upper,work,merged}
cd /tmp/ovl

echo "from layer 1"        > lower1/a.txt
echo "base config"         > lower1/config.txt
mkdir lower1/cache && echo "old" > lower1/cache/data
echo "from layer 2"        > lower2/b.txt
echo "overridden config"   > lower2/config.txt     # same name as in lower1

# 2. Mount: lower2 is listed first, so it's on top of lower1
sudo mount -t overlay overlay \
  -o lowerdir=lower2:lower1,upperdir=upper,workdir=work merged

ls merged                  # a.txt  b.txt  cache  config.txt
cat merged/config.txt      # "overridden config": the top lower layer wins
ls upper                   # empty: nothing written yet

# 3. Create a new file: goes straight into upper
echo "new" | sudo tee merged/new.txt
ls upper                   # new.txt

# 4. Modify a lower file: copy-up
echo "changed" | sudo tee -a merged/a.txt
cat upper/a.txt            # full copy plus the change
cat lower1/a.txt           # untouched

# 5. Delete a lower file: whiteout
sudo rm merged/b.txt
ls -l upper/b.txt          # c--------- … 0, 0 b.txt  ← character device 0/0
ls lower2                  # b.txt is still there

# 6. Replace a lower directory: opaque directory
sudo rm -r merged/cache && sudo mkdir merged/cache
sudo getfattr -n trusted.overlay.opaque upper/cache   # trusted.overlay.opaque="y"
ls merged/cache            # empty: lower1/cache/data is hidden

# 7. See the cost of copy-up on a big file
sudo umount merged
fallocate -l 1G lower1/big.bin
sudo mount -t overlay overlay \
  -o lowerdir=lower2:lower1,upperdir=upper,workdir=work merged
time sudo sh -c 'echo x >> merged/big.bin'   # slow: copies 1 GiB first
time sudo sh -c 'echo x >> merged/big.bin'   # fast: already in upper
du -sh upper                                  # ~1 GiB

# 8. Clean up
sudo umount merged
cd / && sudo rm -rf /tmp/ovl
```

(`getfattr` is in the `attr` package if it's missing.)

If you have Docker, compare with the real thing:

```bash
docker run -d --name ovl-test nginx
docker exec ovl-test sh -c 'echo hi >> /etc/nginx/nginx.conf; rm /usr/share/nginx/html/index.html'
docker diff ovl-test
UPPER=$(docker inspect ovl-test --format '{{.GraphDriver.Data.UpperDir}}')
sudo ls -lR "$UPPER"      # the copied-up nginx.conf and the index.html whiteout
docker rm -f ovl-test
```

On a host using the containerd image store, `GraphDriver.Data` may be empty; look under `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/` and in `mount | grep overlay` instead.

## Cheat sheet

```text
Storage driver / snapshotter:  stacks read-only image layers + one writable layer per container
overlay2 = Docker driver ; OverlayFS = kernel filesystem underneath
OverlayFS dirs:               lowerdir (read-only, leftmost = top) · upperdir (writable) · workdir · merged
Read:                         top-down lookup, first match wins, directories merged, no copy
Modify lower file:            copy-up of the WHOLE file into upperdir (metadata change too, unless metacopy)
Delete lower file:            whiteout = char device 0/0 in upperdir
Replace lower dir:            opaque dir, xattr trusted.overlay.opaque="y"
Rename lower dir:             EXDEV unless redirect_dir
overlay2 on disk:             /var/lib/docker/overlay2/<id>/{diff,link,lower,work,merged} + l/
Backing FS:                   ext4, or XFS with ftype=1 ; no overlay on overlay
Other drivers:                btrfs, zfs (block-level) · vfs (no CoW) · fuse-overlayfs (old rootless)
Removed:                      aufs (v24) · overlay (v24) · devicemapper (v25)
containerd:                   snapshotters: overlayfs (default), native, btrfs, zfs, devmapper, stargz/soci/nydus
Docker 29+:                   containerd image store default on new installs ; data in /var/lib/containerd
Check:                        docker info · docker info -f '{{ .DriverStatus }}' · docker inspect …GraphDriver
Inspect changes:              docker diff · docker ps -s · docker system df -v
Rules:                        heavy writes → volumes ; clean up in the same RUN ; multi-stage builds
```

## References

[^1]: [Storage drivers](https://docs.docker.com/engine/storage/drivers/), Docker documentation.
[^2]: [OverlayFS storage driver](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/), Docker documentation.
[^3]: [Overlay Filesystem](https://docs.kernel.org/filesystems/overlayfs.html), Linux kernel documentation.
[^4]: [Deprecated Docker Engine features](https://docs.docker.com/engine/deprecated/), Docker documentation.
[^5]: [Select a storage driver](https://docs.docker.com/engine/storage/drivers/select-storage-driver/), Docker documentation.
[^6]: [Snapshotters](https://github.com/containerd/containerd/tree/main/docs/snapshotters), containerd documentation.
[^7]: [Docker Engine v29: Foundational Updates for the Future](https://www.docker.com/blog/docker-engine-version-29/), Docker blog.
[^8]: [containerd image store with Docker Engine](https://docs.docker.com/engine/storage/containerd/), Docker documentation.
[^9]: [Docker Engine version 29 release notes](https://docs.docker.com/engine/release-notes/29/), Docker documentation.
