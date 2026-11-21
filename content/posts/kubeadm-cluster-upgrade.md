---
title: "How to Upgrade a Kubernetes Cluster with kubeadm"
date: 2026-11-07
draft: true
description: "A step-by-step guide to upgrading a kubeadm-managed Kubernetes cluster one minor version at a time: preparation, the package repository switch, control plane nodes, worker nodes, verification and recovery."
summary: "kubeadm makes Kubernetes upgrades routine, but only if you do the steps in the right order. Here is the whole procedure, from backups and version checks to the last uncordon, with an explanation of what each command actually does."
tags: ["kubernetes", "kubeadm", "upgrades", "cluster administration", "platform engineering"]
categories: ["Programming"]
---

If you built your cluster with `kubeadm`, you also upgrade it with `kubeadm`. The tool does the hard part, rewriting the control plane's static Pod manifests, renewing certificates and updating add-ons, but it leaves the packages, the draining and the ordering to you. Get the order wrong and you end up with a kubelet newer than its API server, or a cluster stuck halfway through an etcd upgrade.

This post walks through the full procedure. The examples upgrade from **v1.36 to v1.37**, the current release at the time of writing, on Debian/Ubuntu and RHEL-family nodes using the community package repositories at `pkgs.k8s.io`. The same steps work for any minor version: change the numbers.

## The rules

Three rules shape everything that follows.

**1. One minor version at a time.** You can go from 1.36.x to 1.37.x, or from 1.37.2 to 1.37.5. You cannot go from 1.35 straight to 1.37. If you are two versions behind, do two upgrades.[^1]

**2. Control plane first, then nodes.** The API server must always be at least as new as everything that talks to it. The version skew policy allows a kubelet to be up to three minor versions older than the API server, but never newer.[^2] So the order is always:

1. The first control plane node
2. The remaining control plane nodes, one at a time
3. The worker nodes, one (or a few) at a time

**3. kubeadm first, then kubelet.** On every node you upgrade the `kubeadm` package, let it do its work, and only then upgrade `kubelet` and `kubectl`.

```text
              control plane                          workers
   ┌─────────────────────────────────────┐   ┌──────────────────────┐
   │ cp-1: kubeadm upgrade apply  ──►    │   │ w-1: kubeadm upgrade │
   │ cp-2: kubeadm upgrade node   ──►    │──►│      node, kubelet   │──► done
   │ cp-3: kubeadm upgrade node          │   │ w-2: ...             │
   │ (each: drain, kubelet, uncordon)    │   │ w-3: ...             │
   └─────────────────────────────────────┘   └──────────────────────┘
```

## Before you start

### Read the release notes

Every minor release removes or changes something. Before any upgrade, read the changelog for the target version[^3] and look in particular for:

- **Removed APIs.** If a manifest or Helm chart still uses an API version that has been removed, it will fail after the upgrade. The deprecated API migration guide lists what goes away in each release.[^4]
- **Changed defaults** for the kubelet, kube-proxy or kubeadm itself.
- **Requirements for nodes**, such as the cgroup change described next.

### Check your nodes use cgroup v2

This one matters for 1.37. On Linux, the kubelet now only supports cgroup v2 by default: the `FailCgroupV1` option is `true`, so a kubelet on a cgroup v1 host refuses to start.[^1][^5] Check every node before you begin:

```bash
stat -fc %T /sys/fs/cgroup/
```

`cgroup2fs` means cgroup v2 and you're fine. `tmpfs` means cgroup v1; move that node to cgroup v2 (or a newer OS image) before upgrading it, or you'll have a node that won't come back.

### Check the cluster is healthy

`kubeadm` will refuse to upgrade a cluster with problems, but it's better to find them yourself first:

```bash
kubectl get nodes                        # all Ready, all on the same version
kubectl get pods -n kube-system          # control plane, etcd, CoreDNS, CNI all Running
sudo kubeadm certs check-expiration      # on a control plane node
```

Also confirm that swap is off on every node, or configured as the kubelet expects, and that you have enough spare capacity to drain a node without leaving Pods stuck as `Pending`.

### Back up etcd

`kubeadm` takes its own backup of etcd during the upgrade (more on that below), but you want one you control, stored off the node. For a stacked etcd (the kubeadm default), you can use the `etcdctl` binary that ships inside the etcd Pod:

```bash
NODE=$(hostname)
kubectl -n kube-system exec etcd-$NODE -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/pre-upgrade.db
```

The snapshot lands in `/var/lib/etcd` on the host, because that directory is mounted into the Pod. Copy it somewhere safe. Also copy `/etc/kubernetes/` (certificates, kubeconfigs and manifests) from each control plane node.[^6]

The upgrade doesn't touch your workloads, but back up any application state you care about too. It's cheap insurance.

## Step 1: switch the package repository

The `pkgs.k8s.io` repositories have one repository **per minor version**. A node configured for v1.36 can't see v1.37 packages, so the first job on every node is to point it at the new repository.[^7]

On Debian/Ubuntu:

```bash
sudo sed -i 's|/v1.36/|/v1.37/|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```

On RHEL, CentOS, Rocky or Fedora:

```bash
sudo sed -i 's|/v1.36/|/v1.37/|g' /etc/yum.repos.d/kubernetes.repo
```

If your nodes still use the old `apt.kubernetes.io` or `yum.kubernetes.io` repositories, those were frozen in September 2023 and don't contain any recent release. Move to `pkgs.k8s.io` first.[^1]

For a **patch** upgrade within the same minor version (1.37.1 to 1.37.3), skip this step.

## Step 2: pick the exact version

Find the latest 1.37 patch release available:

```bash
# Debian/Ubuntu
sudo apt-cache madison kubeadm

# RHEL family (DNF5; on DNF4 use --disableexcludes=kubernetes)
sudo dnf list --showduplicates kubeadm --setopt=disable_excludes=kubernetes
```

Pick the newest `1.37.x`. In the commands below, replace `x` with that patch number.

## Step 3: upgrade the first control plane node

Pick one control plane node to go first. It needs `/etc/kubernetes/admin.conf`, which every kubeadm control plane node has.

### Upgrade kubeadm

The Kubernetes packages are normally pinned so a routine `apt upgrade` doesn't change them by accident. Unpin, install, re-pin:

```bash
# Debian/Ubuntu
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.37.x-*' && \
sudo apt-mark hold kubeadm

# RHEL family
sudo dnf install -y kubeadm-'1.37.x-*' --setopt=disable_excludes=kubernetes
```

Check it:

```bash
kubeadm version
```

### Plan the upgrade

```bash
sudo kubeadm upgrade plan
```

This changes nothing. It checks that the cluster can be upgraded and prints a table: each control plane component, its current version and the version it will be upgraded to, plus the state of the component configs (kubelet, kube-proxy). If any component config needs a manual migration, `plan` tells you here. Read it before going on.

If you want to see exactly what will change in the static Pod manifests, you can also run `sudo kubeadm upgrade diff v1.37.x`, or `sudo kubeadm upgrade apply v1.37.x --dry-run`.[^8]

### Apply it

```bash
sudo kubeadm upgrade apply v1.37.x
```

When it finishes you should see:

```text
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.37.x". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

What `upgrade apply` actually did:[^1]

- **Checked the cluster**: the API server is reachable, all nodes are `Ready`, the control plane is healthy, and the target version is allowed by the skew policy.
- **Pulled the new images** for the API server, controller manager, scheduler and etcd (if the etcd version changes).
- **Rewrote the static Pod manifests** in `/etc/kubernetes/manifests`, one component at a time. The kubelet notices each change and restarts the component. If one fails to come up healthy, kubeadm rolls it back.
- **Migrated component configs** that need a new API version.
- **Renewed the certificates** kubeadm manages on this node. This is why a cluster that is upgraded regularly never has certificate expiry problems. Pass `--certificate-renewal=false` if you manage certificates yourself.[^9]
- **Updated CoreDNS and kube-proxy**, but only once every control plane node is on the new version. Since v1.28, kubeadm waits for the last control plane node before touching the add-ons, so on a multi-node control plane they are upgraded at the end, not now.

Notice what it **didn't** do: it didn't upgrade the kubelet on this node. The control plane is now on 1.37 while this node's kubelet is still 1.36, which the skew policy allows.

#### A note on etcd

If the new version comes with a newer etcd, the etcd Pod restarts during `upgrade apply`. The API server keeps running throughout, so requests in flight while etcd restarts will hang until it's back. To make this smoother, you can stop the API server gracefully just before starting the upgrade, so it finishes in-flight requests and closes connections first:[^1]

```bash
sudo killall -s SIGTERM kube-apiserver   # graceful shutdown
sleep 20                                 # let in-flight requests finish
sudo kubeadm upgrade apply v1.37.x
```

The kubelet restarts the API server automatically, since it's a static Pod.

### Upgrade the CNI plugin

Your network plugin (Calico, Cilium, Flannel and so on) is not managed by kubeadm. Check its documentation for whether this Kubernetes version needs a newer release, and upgrade it now if so. If it runs as a DaemonSet, you only do this once, not on every control plane node.

### Drain, upgrade the kubelet, uncordon

Now upgrade the kubelet on this node. A minor version change to the kubelet restarts every container on the node, so drain it first. Control plane nodes often run CoreDNS and other important Pods.[^10]

```bash
kubectl drain <node-name> --ignore-daemonsets
```

Then upgrade `kubelet` and `kubectl`:

```bash
# Debian/Ubuntu
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.37.x-*' kubectl='1.37.x-*' && \
sudo apt-mark hold kubelet kubectl

# RHEL family
sudo dnf install -y kubelet-'1.37.x-*' kubectl-'1.37.x-*' --setopt=disable_excludes=kubernetes
```

Restart the kubelet:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Check it came up (`systemctl status kubelet`, or `journalctl -xeu kubelet` if it didn't), then let workloads back on:

```bash
kubectl uncordon <node-name>
```

## Step 4: upgrade the other control plane nodes

Repeat the same procedure on each remaining control plane node, **one at a time**, with one difference: instead of `kubeadm upgrade apply`, run

```bash
sudo kubeadm upgrade node
```

There's no need to run `upgrade plan` or upgrade the CNI plugin again. `upgrade node` fetches the cluster configuration that the first node stored in the cluster, renews this node's certificates, rewrites its static Pod manifests and updates its kubelet configuration.[^1]

So, per node: switch the repository, upgrade `kubeadm`, `kubeadm upgrade node`, drain, upgrade `kubelet` and `kubectl`, restart the kubelet, uncordon.

Don't start one control plane node until the previous one has finished. When the last one finishes, kubeadm upgrades CoreDNS and kube-proxy.

## Step 5: upgrade the worker nodes

Worker nodes follow the same pattern, done one node at a time or in small batches, so you never drop below the capacity your workloads need.[^11]

On each worker:

```bash
# 1. Switch the package repository (Step 1), then upgrade kubeadm
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.37.x-*' && \
sudo apt-mark hold kubeadm

# 2. Update the local kubelet configuration
sudo kubeadm upgrade node
```

On a worker, `kubeadm upgrade node` is quick: there's no control plane to upgrade, so it only fetches the cluster configuration and updates the kubelet's config.

Then, from a machine with `kubectl` access:

```bash
kubectl drain <node-name> --ignore-daemonsets
```

Back on the worker:

```bash
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.37.x-*' kubectl='1.37.x-*' && \
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

And finally:

```bash
kubectl uncordon <node-name>
```

A few things that make draining go smoothly:

- **PodDisruptionBudgets** are respected by `drain`. A PDB that allows zero disruptions (for example, `minAvailable: 1` on a single-replica Deployment) blocks the drain forever. Find those before you start.
- Pods using `emptyDir` volumes need `--delete-emptydir-data`, which deletes that data.
- Bare Pods with no controller won't be recreated. `drain` refuses to delete them without `--force`.

If you have many workers, this is the part worth scripting. The loop is the same for every node.

## Step 6: verify

When every node is done:

```bash
kubectl get nodes
```

Every node should be `Ready` and show `v1.37.x` in the `VERSION` column. Then check the system Pods and the add-ons:

```bash
kubectl get pods -n kube-system
kubectl -n kube-system get deploy coredns -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n kube-system get ds kube-proxy -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl version
```

Finally, check your own workloads: are all Deployments at their desired replica counts, are any Pods crash-looping, do your smoke tests pass?

## When something goes wrong

**The upgrade stopped halfway.** `kubeadm upgrade` is idempotent. If it was interrupted (an SSH session dropped, the machine rebooted), run the same command again. It works out what's already done and finishes the rest. To repair a control plane in a bad state without changing the version, you can also rerun `sudo kubeadm upgrade apply --force` with the version the cluster is already on.[^1]

**A component won't come up.** kubeadm keeps backups under `/etc/kubernetes/tmp`:

- `kubeadm-backup-manifests-<date>-<time>` holds the old static Pod manifests. Copy them back into `/etc/kubernetes/manifests` to return a component to the previous version.
- `kubeadm-backup-etcd-<date>-<time>` holds a copy of this node's etcd data. If an etcd upgrade fails and the automatic rollback doesn't work, it can be restored into `/var/lib/etcd`. (It's empty if you use external etcd.)

These directories are never cleaned up automatically. Delete them yourself once you're confident the upgrade worked.

**The kubelet won't start after the upgrade.** Look at `journalctl -xeu kubelet`. On 1.37 the first thing to rule out is cgroup v1. Other common causes are a removed kubelet flag still set in `/etc/default/kubelet` or `/etc/sysconfig/kubelet`, or swap being turned back on.

**There is no real downgrade.** kubeadm doesn't support going back to an older minor version. If the upgrade succeeded but something in the new release breaks your workloads, your options are fixing forward or restoring from the backups you took at the start. That's why the etcd snapshot isn't optional.

## Checklist

For the impatient, the whole procedure:

1. Read the release notes; check removed APIs and node requirements (cgroup v2 for 1.37).
2. Check cluster health, and back up etcd and `/etc/kubernetes`.
3. **First control plane node:** switch repo → upgrade `kubeadm` → `kubeadm upgrade plan` → `kubeadm upgrade apply v1.37.x` → upgrade CNI → drain → upgrade `kubelet` + `kubectl` → restart kubelet → uncordon.
4. **Other control plane nodes, one at a time:** switch repo → upgrade `kubeadm` → `kubeadm upgrade node` → drain → upgrade `kubelet` + `kubectl` → restart kubelet → uncordon.
5. **Workers, a few at a time:** switch repo → upgrade `kubeadm` → `kubeadm upgrade node` → drain → upgrade `kubelet` + `kubectl` → restart kubelet → uncordon.
6. Verify nodes, system Pods, add-ons and workloads.
7. Clean up `/etc/kubernetes/tmp`.

Kubernetes ships a new minor release roughly every four months and supports each one for about fourteen months. Doing this three times a year is much easier than catching up three versions at once, so keep it routine.

## References

[^1]: [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/), Kubernetes documentation (v1.36 to v1.37).
[^2]: [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/), Kubernetes documentation.
[^3]: [Kubernetes CHANGELOG](https://git.k8s.io/kubernetes/CHANGELOG), Kubernetes on GitHub.
[^4]: [Deprecated API Migration Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/), Kubernetes documentation.
[^5]: [About cgroup v2: deprecation of cgroup v1](https://kubernetes.io/docs/concepts/architecture/cgroups/#deprecation-of-cgroup-v1), Kubernetes documentation.
[^6]: [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/), Kubernetes documentation: snapshots and restores.
[^7]: [Changing the Kubernetes package repository](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/), Kubernetes documentation.
[^8]: [kubeadm upgrade](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/), Kubernetes command reference.
[^9]: [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/), Kubernetes documentation.
[^10]: [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/), Kubernetes documentation.
[^11]: [Upgrading Linux nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/), Kubernetes documentation.
