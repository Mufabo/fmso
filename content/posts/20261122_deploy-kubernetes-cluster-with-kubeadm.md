---
title: "Deploy a Kubernetes Cluster with kubeadm"
date: 2026-11-22
draft: true
description: "A step-by-step guide to building a three-node Kubernetes v1.37 cluster with kubeadm: node preparation, containerd and the cgroup driver, installing kubeadm, kubelet and kubectl from pkgs.k8s.io, kubeadm init with a config file, a Pod network, joining workers, smoke tests, and what to do when it goes wrong."
summary: "kubeadm turns a handful of Linux machines into a conformant Kubernetes cluster in about half an hour. Here is every step, from swap and kernel modules to the first Pod, with an explanation of what each command actually does."
tags: ["kubernetes", "kubeadm", "containerd", "cluster administration", "cka", "platform engineering"]
categories: ["Programming"]
---

Managed Kubernetes hides the control plane from you. [Kubernetes the Hard Way](/posts/kubernetes-the-hard-way/) shows you every piece of it, one certificate at a time. **kubeadm** sits in between: it's the official tool for bootstrapping a real, conformant cluster on machines you control, and it does the tedious parts (certificates, kubeconfigs, static Pod manifests, bootstrap tokens) while leaving the infrastructure choices to you.

It's also what the CKA exam expects you to know, and what many on-premises and edge clusters run on.

This post builds a cluster with **one control plane node and two workers**, running **Kubernetes v1.37** (the current release at the time of writing) on Ubuntu or Debian, with notes for RHEL-family systems. The steps are the same for any recent minor version: change the numbers.

## What kubeadm does, and doesn't do

kubeadm has a deliberately narrow job: take machines that already have a container runtime and the kubelet, and turn them into a minimum viable cluster that passes the conformance tests.[^1]

| kubeadm does | You do |
|---|---|
| Generates the cluster CA and all component certificates | Provision the machines and the network between them |
| Writes kubeconfig files for admin, controller manager, scheduler, kubelet | Install and configure a container runtime |
| Writes static Pod manifests for etcd, API server, controller manager, scheduler | Install `kubeadm`, `kubelet` and `kubectl` packages |
| Creates bootstrap tokens so nodes can join | Install a Pod network (CNI) plugin |
| Installs CoreDNS and kube-proxy | Load balancer for an HA control plane, storage, ingress, monitoring |
| Upgrades all of the above ([kubeadm upgrade](/posts/kubeadm-cluster-upgrade/)) | Upgrade the OS, the runtime and the CNI plugin |

The end result looks like this:

```text
            ┌────────────────────── cp1 (control plane) ──────────────────────┐
            │  kubelet ──► static Pods from /etc/kubernetes/manifests/         │
            │              ├─ etcd                                             │
            │              ├─ kube-apiserver  :6443  ◄────────────┐            │
            │              ├─ kube-controller-manager             │            │
            │              └─ kube-scheduler                      │            │
            │  DaemonSets: kube-proxy, CNI agent                  │            │
            │  Deployment: CoreDNS                                │            │
            └─────────────────────────────────────────────────────┼────────────┘
                                                                  │ TLS
              ┌─────────── w1 (worker) ──────────┐  ┌─────────── w2 (worker) ──────────┐
              │ containerd · kubelet             │  │ containerd · kubelet             │
              │ kube-proxy · CNI agent           │  │ kube-proxy · CNI agent           │
              │ your Pods                        │  │ your Pods                        │
              └──────────────────────────────────┘  └──────────────────────────────────┘
```

Note the chicken-and-egg trick at the heart of kubeadm: the control plane components themselves run as containers, started by the kubelet from files in `/etc/kubernetes/manifests/`. The kubelet doesn't need an API server to run *static Pods*, so it can start the API server.

## The lab

| Host | Role | IP | Minimum size |
|---|---|---|---|
| `cp1` | control plane | 10.0.0.10 | 2 vCPU, 2 GiB RAM (4 GiB is more comfortable) |
| `w1` | worker | 10.0.0.11 | 2 vCPU, 2 GiB RAM |
| `w2` | worker | 10.0.0.12 | 2 vCPU, 2 GiB RAM |

Three address ranges matter, and **they must not overlap**:

- **Node network**: `10.0.0.0/24`, where the machines live.
- **Pod network**: `10.244.0.0/16`, where Pods get their IPs. Chosen by you, handed to the CNI plugin.
- **Service network**: `10.96.0.0/12`, kubeadm's default for ClusterIP Services.

If your node network already uses one of those, pick different ranges now. Changing the Pod CIDR after the fact is painful (more on that in [IPAM in Kubernetes](/posts/ipam-in-kubernetes/)).

Any VMs work: local ones with Vagrant, Multipass or Proxmox, or cloud instances. Use an LTS distribution, such as Ubuntu 24.04/26.04 or Debian 12/13.

## Step 1: prepare every node

Everything in this step runs on **all three machines**.

### Check the basics

kubeadm's requirements are modest:[^2] 2 GiB of RAM per machine, 2 CPUs on control plane nodes, full network connectivity between the machines, and a **unique hostname, MAC address and `product_uuid`** on every node. The last one bites people who clone VMs from a template:

```bash
hostnamectl                               # hostname
ip link                                   # MAC addresses
sudo cat /sys/class/dmi/id/product_uuid   # must differ between nodes
```

Give each machine its name, and make the names resolvable. In a lab, `/etc/hosts` is enough:

```bash
sudo hostnamectl set-hostname cp1     # w1, w2 on the others

cat <<EOF | sudo tee -a /etc/hosts
10.0.0.10 cp1 k8s-api.lab.internal
10.0.0.11 w1
10.0.0.12 w2
EOF
```

`k8s-api.lab.internal` is the name we'll use as the cluster's API endpoint. It points at `cp1` for now. Why bother will become clear in [Step 4](#step-4-initialise-the-control-plane).

### Open the ports

If there's a firewall between nodes (a cloud security group, `ufw`, `firewalld`), these need to be open:[^3]

| Node | Port | Protocol | Used by |
|---|---|---|---|
| Control plane | 6443 | TCP | Kubernetes API server (everyone) |
| Control plane | 2379–2380 | TCP | etcd client and peer API |
| Control plane | 10250 | TCP | kubelet API |
| Control plane | 10257 | TCP | kube-controller-manager (localhost) |
| Control plane | 10259 | TCP | kube-scheduler (localhost) |
| Worker | 10250 | TCP | kubelet API |
| Worker | 10256 | TCP | kube-proxy health check |
| Worker | 30000–32767 | TCP, UDP | NodePort Services |

Plus whatever your CNI plugin needs. Flannel, used below, sends its VXLAN traffic over **UDP 8472**. Once the API server is up you can check reachability from a worker with `nc -zv k8s-api.lab.internal 6443`.

### Turn off swap

By default the kubelet refuses to start if it finds swap enabled.[^2] Kubernetes does support swap these days, but it has to be configured deliberately (`failSwapOn: false` plus a `swapBehavior` in the kubelet configuration). For a first cluster, turn it off:

```bash
sudo swapoff -a
sudo sed -i '/\sswap\s/ s/^/#/' /etc/fstab      # keep it off after reboots
```

Some distributions also enable swap through a systemd unit or zram (`swapon --show` tells you). Disable those too, e.g. `sudo systemctl mask dev-zram0.swap` or remove the `zram-generator` package.

### Kernel modules and sysctls

Pod-to-Pod traffic leaving a node has to be routed, and the Linux kernel doesn't forward IPv4 packets between interfaces by default.[^4] Many CNI plugins also rely on bridged traffic being visible to iptables, which needs the `br_netfilter` module. And containerd's image layers use `overlay`.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

Verify:

```bash
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables
```

Both sysctls should print `1`. kubeadm's preflight checks fail if `ip_forward` isn't set.

### RHEL family only: SELinux

On RHEL, Rocky, Alma or Fedora, the Kubernetes docs still recommend putting SELinux into permissive mode, so containers can read the host file system that Pod networks need:[^2]

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

You can keep it enforcing if you're prepared to write the policy yourself, but kubeadm doesn't test that configuration.

## Step 2: install the container runtime

The kubelet doesn't run containers itself. It talks to a **container runtime** over the Container Runtime Interface (CRI). Docker Engine on its own no longer qualifies (dockershim was removed in v1.24, and Docker now needs the `cri-dockerd` adapter), so the usual choices are **containerd** and **CRI-O**.[^4] We'll use containerd.

On all nodes:

```bash
# Debian/Ubuntu
sudo apt-get update
sudo apt-get install -y containerd

# RHEL family: containerd.io from Docker's repository is the common source
# sudo dnf install -y containerd.io
```

### The cgroup driver: the one setting that matters

Linux **cgroups** are how resource limits for Pods and containers are enforced. Something has to manage the cgroup tree, and there are two "drivers" for doing it: `cgroupfs` (write to the cgroup file system directly) and `systemd` (ask systemd to do it).

Two rules from the Kubernetes docs:[^4]

1. **The kubelet and the container runtime must use the same driver.**
2. On a system that boots with systemd, and especially with **cgroup v2** (the default on every current distribution), use the **`systemd`** driver. Two separate managers of one cgroup tree is a recipe for instability under resource pressure.

kubeadm already configures the kubelet for `systemd`. containerd's default configuration doesn't, so generate the default config and flip the switch:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep -n 'SystemdCgroup' /etc/containerd/config.toml     # should show "= true"
sudo systemctl restart containerd
sudo systemctl enable containerd
```

The `sed` works for both containerd 1.7 and 2.x, but the setting lives in a differently named section. In 1.7 it's under `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]`, in 2.x under `[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]`.[^4] Also check that `cri` isn't listed in `disabled_plugins` at the top of the file; some distribution packages ship it disabled.

Get a cgroup driver mismatch wrong and the symptom is maddening: the cluster comes up, then control plane Pods restart every few minutes for no obvious reason. Check this first if you see that.

### crictl

`crictl` is a CLI for talking to any CRI runtime directly, handy when the API server isn't up yet. It comes with the `cri-tools` package that the Kubernetes repository installs in the next step. Point it at containerd so it stops guessing:

```bash
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
EOF
```

## Step 3: install kubeadm, kubelet and kubectl

Three packages, on **all nodes**:

- **`kubeadm`**: the bootstrapping tool.
- **`kubelet`**: the node agent that starts Pods. Runs on every node, including the control plane.
- **`kubectl`**: the client. Strictly only needed where you'll run commands from, but it's convenient everywhere.

They come from the community repositories at `pkgs.k8s.io`, which have **one repository per minor version**. The old `apt.kubernetes.io` and `yum.kubernetes.io` repositories were frozen in September 2023 and shut down in March 2024, so tutorials that use them no longer work.[^5]

### Debian/Ubuntu

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

### RHEL, Rocky, Alma, Fedora

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.37/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.37/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# DNF5 (Fedora 41+, RHEL 10+); on DNF4 use --disableexcludes=kubernetes
sudo dnf install -y kubelet kubeadm kubectl --setopt=disable_excludes=kubernetes
sudo systemctl enable --now kubelet
```

The `apt-mark hold` and the `exclude=` line do the same job: they stop a routine `apt upgrade` or `dnf update` from bumping Kubernetes underneath you. Kubernetes upgrades follow their own procedure, in a specific order.

### "Why is the kubelet crash-looping?"

Run `systemctl status kubelet` now and you'll see it restarting every few seconds. **That's expected.** The kubelet is waiting for a configuration file (`/var/lib/kubelet/config.yaml`) that `kubeadm init` or `kubeadm join` will write. Until then, systemd keeps restarting it.[^2]

## Step 4: initialise the control plane

From here on, commands run on **`cp1`** unless stated otherwise.

### Write a kubeadm configuration file

You can pass everything as flags (`kubeadm init --pod-network-cidr=10.244.0.0/16 ...`), but a configuration file is easier to review, keep in Git and reuse. kubeadm's current config API is `kubeadm.k8s.io/v1beta4`:[^6]

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.0.10        # this node's IP on the node network
  bindPort: 6443
nodeRegistration:
  name: cp1
  criSocket: unix:///run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
clusterName: lab
controlPlaneEndpoint: k8s-api.lab.internal:6443
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
  dnsDomain: cluster.local
apiServer:
  certSANs:
    - k8s-api.lab.internal
    - 10.0.0.10
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

A few choices worth explaining:

- **`kubernetesVersion`** is left out, so kubeadm deploys its own version. Set it (`v1.37.x`) if you want to be explicit.
- **`controlPlaneEndpoint`** is the address every component and every kubeconfig will use to reach the API server. It's baked into certificates and kubeconfigs, and you can't easily change it later. Using a DNS name instead of `cp1`'s IP means that if you later add two more control plane nodes behind a load balancer, you just repoint the name. **Always set it, even for a single control plane.**
- **`podSubnet`** must match what the CNI plugin expects. Flannel's default is `10.244.0.0/16`.
- **`cgroupDriver: systemd`** is already kubeadm's default; it's written out here to make the match with containerd visible.

`kubeadm config print init-defaults` prints every field with its default if you want to see what else is there.

### Preflight and image pull

Pull the control plane images first. It isn't required, but it separates "my network can't reach the registry" from "my cluster won't start":

```bash
sudo kubeadm config images list --config kubeadm-config.yaml
sudo kubeadm config images pull --config kubeadm-config.yaml
```

If kubeadm warns that containerd's **sandbox ("pause") image** differs from the one it expects, set `sandbox_image` (containerd 1.7) or `sandbox` (containerd 2.x) in `/etc/containerd/config.toml` to the `registry.k8s.io/pause` version from that list, and restart containerd.[^4]

### Run kubeadm init

```bash
sudo kubeadm init --config kubeadm-config.yaml
```

This takes a minute or two. kubeadm works through a series of **phases**,[^7] each of which you can also run on its own with `kubeadm init phase <name>`:

| Phase | What it does |
|---|---|
| `preflight` | Checks swap, ports, kernel settings, runtime, required binaries. Fails fast. |
| `certs` | Creates the cluster CA, etcd CA, front-proxy CA, and certificates for every component, in `/etc/kubernetes/pki/` |
| `kubeconfig` | Writes `admin.conf`, `super-admin.conf`, `controller-manager.conf`, `scheduler.conf` and `kubelet.conf` in `/etc/kubernetes/` |
| `kubelet-start` | Writes `/var/lib/kubelet/config.yaml` and (re)starts the kubelet, which stops crash-looping |
| `control-plane`, `etcd` | Writes static Pod manifests to `/etc/kubernetes/manifests/`; the kubelet starts them |
| `wait-control-plane` | Waits until the API server answers health checks |
| `upload-config` | Stores the kubeadm and kubelet configuration in ConfigMaps, so joining nodes and future upgrades can read them |
| `mark-control-plane` | Labels and taints the node so ordinary Pods don't schedule there |
| `bootstrap-token` | Creates a join token and the RBAC rules that let new nodes request certificates |
| `addon` | Installs CoreDNS and kube-proxy |

At the end you get:

```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
...
You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join k8s-api.lab.internal:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:1a2b3c... \
        --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-api.lab.internal:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:1a2b3c...
```

**Copy the worker `join` command somewhere.** You can regenerate it, but it's convenient to have.

### Set up kubectl

Do what it says:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

`admin.conf` gives you a user in the `kubeadm:cluster-admins` group, bound to `cluster-admin`. Since v1.29 kubeadm also writes a `super-admin.conf` whose user is in `system:masters`, which bypasses RBAC entirely. Keep that one off your laptop; it's a break-glass credential.[^8] (More on why in [Authentication in Kubernetes](/posts/authentication-in-kubernetes/).)

Now look:

```bash
$ kubectl get nodes
NAME   STATUS     ROLES           AGE   VERSION
cp1    NotReady   control-plane   1m    v1.37.x

$ kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
coredns-xxxxxxxxxx-aaaaa      0/1     Pending   0          1m
coredns-xxxxxxxxxx-bbbbb      0/1     Pending   0          1m
etcd-cp1                      1/1     Running   0          1m
kube-apiserver-cp1            1/1     Running   0          1m
kube-controller-manager-cp1   1/1     Running   0          1m
kube-proxy-xxxxx              1/1     Running   0          1m
kube-scheduler-cp1            1/1     Running   0          1m
```

`NotReady` and `Pending` CoreDNS are **correct at this point**. The node reports `NotReady` because there's no Pod network yet, and CoreDNS can't start without one. That's the next step.

## Step 5: install a Pod network

Kubernetes defines what Pod networking must look like (every Pod gets its own IP, and every Pod can reach every other Pod without NAT) but leaves the implementation to a **CNI plugin**.[^9] kubeadm deliberately doesn't install one.

The common choices:

| Plugin | Good for | NetworkPolicy |
|---|---|---|
| **Flannel** | Simplest possible overlay; labs and learning | No |
| **Calico** | BGP or overlay routing, mature policy engine, widely used on-prem | Yes |
| **Cilium** | eBPF data plane, can replace kube-proxy, deep observability with Hubble | Yes |

For a first cluster, Flannel has the fewest moving parts. Install it once, from `cp1`:[^10]

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Flannel runs a DaemonSet that gives each node a slice of `10.244.0.0/16` (a `/24` per node, carved out by the controller manager from `podSubnet`) and tunnels traffic between nodes over VXLAN.

Wait a minute and check again:

```bash
$ kubectl get pods -n kube-flannel
NAME                    READY   STATUS    RESTARTS   AGE
kube-flannel-ds-xxxxx   1/1     Running   0          45s

$ kubectl get nodes
NAME   STATUS   ROLES           AGE   VERSION
cp1    Ready    control-plane   4m    v1.37.x
```

CoreDNS should now be `Running` too.

If you'll want NetworkPolicies later (and in anything beyond a lab, you will), use Calico or Cilium instead, following their kubeadm quick-start guides. Make sure the Pod CIDR you give them matches `podSubnet`. Switching CNI plugins on a running cluster is possible but disruptive, so choose before you deploy workloads.

## Step 6: join the workers

On **`w1` and `w2`**, run the worker join command from the `kubeadm init` output, as root:

```bash
sudo kubeadm join k8s-api.lab.internal:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:1a2b3c...
```

Two pieces of information make the join secure in both directions:[^11]

- **`--token`** is a bootstrap token. It proves to the cluster that the node is allowed to join. It's stored as a Secret in `kube-system` and expires after **24 hours** by default.
- **`--discovery-token-ca-cert-hash`** is the SHA-256 hash of the cluster CA's public key. It proves to the node that it's talking to the right cluster, not an impostor answering on that address.

With both checked, the kubelet on the new node submits a certificate signing request, gets a client certificate signed by the cluster CA, and registers itself. From then on it rotates that certificate automatically.

### Lost the join command, or the token expired?

Generate a new one on `cp1`:

```bash
kubeadm token create --print-join-command
```

`kubeadm token list` shows existing tokens and their expiry.

### Check

Back on `cp1`:

```bash
$ kubectl get nodes -o wide
NAME   STATUS   ROLES           AGE   VERSION   INTERNAL-IP   OS-IMAGE             CONTAINER-RUNTIME
cp1    Ready    control-plane   9m    v1.37.x   10.0.0.10     Ubuntu 24.04.3 LTS   containerd://...
w1     Ready    <none>          1m    v1.37.x   10.0.0.11     Ubuntu 24.04.3 LTS   containerd://...
w2     Ready    <none>          1m    v1.37.x   10.0.0.12     Ubuntu 24.04.3 LTS   containerd://...
```

The `<none>` role is cosmetic. If it bothers you:

```bash
kubectl label node w1 node-role.kubernetes.io/worker=
kubectl label node w2 node-role.kubernetes.io/worker=
```

If a node uses the wrong IP in `INTERNAL-IP` (common on VMs with a NAT interface plus a private one, like Vagrant), the kubelet picked the interface with the default route. Fix it by setting `KUBELET_EXTRA_ARGS=--node-ip=10.0.0.11` in `/etc/default/kubelet` (Debian/Ubuntu) or `/etc/sysconfig/kubelet` (RHEL), then `sudo systemctl restart kubelet`.

## Step 7: smoke test

A cluster that shows `Ready` can still be broken. These four checks cover scheduling, Pod networking across nodes, Services and DNS.

**Scheduling and cross-node networking.** Run a Deployment with enough replicas to land on both workers:

```bash
kubectl create deployment web --image=nginx --replicas=4
kubectl rollout status deployment web
kubectl get pods -o wide        # Pods on w1 and w2, IPs from 10.244.x.0/24
```

**Services.** Expose it and reach it through a NodePort from outside the cluster:

```bash
kubectl expose deployment web --port=80 --type=NodePort
NODEPORT=$(kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}')
curl -s http://10.0.0.11:$NODEPORT | grep -i '<title>'
curl -s http://10.0.0.12:$NODEPORT | grep -i '<title>'
```

Both should print `<title>Welcome to nginx!</title>`, whichever node the Pods are actually on. That proves kube-proxy and the overlay work.

**DNS.** Resolve the Service by name from inside a Pod:

```bash
kubectl run dnstest --rm -it --restart=Never --image=busybox:1.36 -- \
  nslookup web.default.svc.cluster.local
```

**Clean up:**

```bash
kubectl delete svc,deployment web
```

If the curl works from one node but not the other, or DNS times out on some nodes, the overlay is broken between nodes. Check UDP 8472 in your firewall first, then see [Troubleshooting Cross-Node Pod Networking](/posts/troubleshooting-cross-node-pod-networking/).

## Optional: run kubectl from your laptop

Copy the kubeconfig off the control plane:

```bash
scp user@10.0.0.10:~/.kube/config ~/.kube/lab-config
export KUBECONFIG=~/.kube/lab-config
kubectl get nodes
```

Your laptop has to resolve `k8s-api.lab.internal`, so add it to your local `/etc/hosts` too. This is also where the extra `certSANs` matter: the API server's certificate must contain whatever name or IP you connect to.

## Optional: allow workloads on the control plane

kubeadm taints control plane nodes with `node-role.kubernetes.io/control-plane:NoSchedule`, so regular Pods stay off them. For a single-node lab cluster, remove it:

```bash
kubectl taint nodes cp1 node-role.kubernetes.io/control-plane:NoSchedule-
```

Don't do this on a cluster you care about. A runaway workload starving etcd of memory takes the whole cluster with it.

## Going highly available

One control plane node is a single point of failure: if `cp1` dies, the workloads keep running, but nothing can be scheduled, scaled or healed until it's back. kubeadm supports two HA topologies:[^12]

- **Stacked etcd**: each control plane node also runs an etcd member. Three control plane nodes means a three-member etcd cluster that survives one failure. Simpler, and the default.
- **External etcd**: etcd runs on its own machines. More hardware, but losing a control plane node doesn't also cost an etcd member.

The setup differs from what we did in only three ways:

1. Put a **load balancer** (HAProxy plus keepalived, kube-vip, or a cloud load balancer) in front of port 6443 on all control plane nodes, and point `controlPlaneEndpoint` at it. This is why we used a DNS name above.
2. Initialise the first node with **`--upload-certs`**. This encrypts the shared CA and service account keys and stores them in a Secret, so the other control plane nodes can fetch them instead of you copying `/etc/kubernetes/pki` around by hand.
3. Join the other control plane nodes with **`--control-plane --certificate-key <key>`**. The key is printed by `kubeadm init`; the uploaded certificates are deleted after two hours, so regenerate them if needed with `sudo kubeadm init phase upload-certs --upload-certs`.

## Day 2: what to know next

**Certificates expire after one year.** kubeadm issues component certificates valid for a year (the CA for ten). Upgrading the cluster renews them automatically, which is one reason to upgrade regularly. Otherwise:[^13]

```bash
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all       # then restart the control plane static Pods
```

**Back up etcd.** Everything the cluster knows lives in etcd. Take snapshots with `etcdctl snapshot save`, and copy `/etc/kubernetes/pki` too.

**Upgrade one minor version at a time.** kubeadm handles the control plane, but the order matters: see my post on [upgrading a cluster with kubeadm](/posts/kubeadm-cluster-upgrade/).

**What's still missing.** A bare kubeadm cluster has no persistent storage, no ingress or Gateway controller, no metrics-server (so `kubectl top` and the HorizontalPodAutoscaler won't work), no monitoring and no log collection. Each is an add-on you choose. Related posts: [CKA storage](/posts/cka-storage/), [Kubernetes Gateway API](/posts/kubernetes-gateway-api/), [Kubernetes autoscaling](/posts/kubernetes-autoscaling/) and [logging with the EFK stack](/posts/intro-to-logging-with-the-efk-stack/).

## When it goes wrong

| Symptom | Likely cause | Check / fix |
|---|---|---|
| `kubelet` restarts every few seconds before `init`/`join` | Normal: no config yet | Ignore until kubeadm has run |
| Preflight: `[ERROR FileContent--proc-sys-net-ipv4-ip_forward]` | IP forwarding off | Step 1 sysctls, `sudo sysctl --system` |
| Preflight: `running with swap on is not supported` | Swap still enabled | `swapoff -a`, fix `/etc/fstab`, check `swapon --show` |
| Preflight: `container runtime is not running` | containerd stopped or CRI plugin disabled | `systemctl status containerd`; remove `cri` from `disabled_plugins` |
| `init` hangs at `wait-control-plane` | kubelet can't start the static Pods | `journalctl -xeu kubelet`, `sudo crictl ps -a`, `sudo crictl logs <id>` |
| Control plane Pods restart every few minutes | cgroup driver mismatch | `SystemdCgroup = true` in containerd, `cgroupDriver: systemd` in kubelet |
| Nodes `NotReady`, CoreDNS `Pending` | No CNI plugin installed | Step 5 |
| CNI Pods crash-loop | Pod CIDR mismatch, or `br_netfilter` not loaded | Compare `podSubnet` with the plugin's config; `lsmod \| grep br_netfilter` |
| `join` fails with `token id ... is invalid` | Token expired (24 h) | `kubeadm token create --print-join-command` |
| `join` hangs at `[preflight] Running pre-flight checks` | Worker can't reach port 6443, or can't resolve the endpoint | `nc -zv k8s-api.lab.internal 6443`; check `/etc/hosts` and firewall |
| Pods on different nodes can't reach each other | Overlay traffic blocked | Open UDP 8472 (Flannel VXLAN); check node IPs with `kubectl get nodes -o wide` |
| `x509: certificate is valid for ..., not ...` from kubectl | Connecting via a name/IP not in the API server certificate | Add it to `apiServer.certSANs` and regenerate the API server cert |

### Starting over

If a node gets into a mess, `kubeadm reset` undoes what `init` or `join` did:[^14]

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube/config
sudo systemctl restart containerd
```

`reset` deliberately leaves the CNI configuration, iptables/IPVS rules and your kubeconfig behind, which is why you clean those up by hand. If you reset a worker, remove it from the cluster first with `kubectl drain <node> --ignore-daemonsets` and `kubectl delete node <node>` from the control plane.

## Cheat sheet

```text
Every node:
  unique hostname / MAC / product_uuid      swapoff -a + comment out swap in /etc/fstab
  modules: overlay, br_netfilter            sysctl: net.ipv4.ip_forward=1, bridge-nf-call-iptables=1
  containerd + SystemdCgroup = true         crictl → unix:///run/containerd/containerd.sock
  pkgs.k8s.io/core:/stable:/v1.37/          install kubelet kubeadm kubectl, hold/exclude them

Control plane:
  kubeadm config print init-defaults        kubeadm config images pull --config kubeadm-config.yaml
  kubeadm init --config kubeadm-config.yaml (always set controlPlaneEndpoint + podSubnet)
  cp /etc/kubernetes/admin.conf ~/.kube/config
  kubectl apply -f <CNI manifest>           → nodes Ready, CoreDNS Running

Workers:
  kubeadm join <endpoint>:6443 --token … --discovery-token-ca-cert-hash sha256:…
  new join command: kubeadm token create --print-join-command

HA:
  LB in front of :6443 → kubeadm init --upload-certs → join --control-plane --certificate-key …

Inspect:
  kubectl get nodes -o wide · kubectl get pods -A · journalctl -xeu kubelet · crictl ps -a
  ls /etc/kubernetes/{manifests,pki} · kubeadm certs check-expiration

Undo:
  kubeadm reset -f && rm -rf /etc/cni/net.d ~/.kube/config
```

## References

[^1]: [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/), Kubernetes documentation.
[^2]: [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), Kubernetes documentation (v1.37).
[^3]: [Ports and Protocols](https://kubernetes.io/docs/reference/networking/ports-and-protocols/), Kubernetes documentation.
[^4]: [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/), Kubernetes documentation: IPv4 forwarding, cgroup drivers, containerd configuration and the sandbox image.
[^5]: [pkgs.k8s.io: Introducing Kubernetes Community-Owned Package Repositories](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/), Kubernetes blog, with the 2024 update on the legacy repositories' shutdown.
[^6]: [kubeadm Configuration (v1beta4)](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/), Kubernetes API reference.
[^7]: [kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/), Kubernetes command reference: init workflow and phases.
[^8]: [Certificate Management with kubeadm: kubeconfig files for users](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/), Kubernetes documentation.
[^9]: [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/), Kubernetes documentation.
[^10]: [Flannel](https://github.com/flannel-io/flannel), flannel-io on GitHub: deploying Flannel with kubectl.
[^11]: [kubeadm join](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/), Kubernetes command reference: token-based discovery and CA pinning.
[^12]: [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/), Kubernetes documentation.
[^13]: [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/), Kubernetes documentation.
[^14]: [kubeadm reset](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/), Kubernetes command reference.
