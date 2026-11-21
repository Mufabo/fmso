---
title: "IPAM in Kubernetes: Who Hands Out the IP Addresses?"
date: 2026-11-14
draft: true
description: "How IP address management works in Kubernetes: node pod CIDRs, CNI IPAM plugins, the block, pool and cloud-native models used by Calico, Cilium and the AWS VPC CNI, Service IP allocation with ServiceCIDR, and how to plan address space so you don't run out."
summary: "Every Pod and every Service in Kubernetes needs an IP address, but Kubernetes itself hands out surprisingly few of them. Here is who actually allocates pod and Service IPs, the main IPAM models CNI plugins use, and how to size your ranges before they run out."
tags: ["kubernetes", "networking", "ipam", "cni", "calico", "cilium", "platform engineering"]
categories: ["Programming"]
---

Every Pod in Kubernetes gets its own IP address. Every ClusterIP Service gets one too. In a busy cluster, thousands of addresses are handed out and taken back every hour. The job of deciding *which* address goes to *which* thing, making sure no two things get the same one, and getting addresses back when they are no longer used is called **IP Address Management (IPAM)**.

Most people never think about IPAM until something breaks: Pods stuck in `ContainerCreating` with `failed to allocate for range`, a new node that never becomes ready, or a cloud subnet that suddenly has no free addresses. This post explains how IPAM works in Kubernetes, from the control plane down to the CNI plugin, so that when it breaks you know where to look, and so you can plan your ranges before it does.

## Three kinds of addresses

A Kubernetes cluster deals with three separate sets of IP addresses, and different components are responsible for each:

| Addresses | Example range | Who allocates them |
| --- | --- | --- |
| **Node IPs** | `192.168.10.0/24` | Your infrastructure: DHCP, the cloud provider, or static config. Kubernetes just reads them. |
| **Pod IPs** | `10.244.0.0/16` | The CNI plugin's IPAM, sometimes with help from the control plane. |
| **Service ClusterIPs** | `10.96.0.0/12` | The kube-apiserver. |

The three ranges must not overlap with each other, or with any network your Pods need to reach (your office VPN, a peered VPC, an on-prem database).

Node IPs are outside Kubernetes' control, so the rest of this post is about the other two. Pod IPs are where nearly all the complexity is.

## Kubernetes doesn't assign pod IPs

This surprises people: the Kubernetes networking model *requires* every Pod to have a unique IP that every other Pod can reach without NAT, but Kubernetes itself does not assign pod IPs. That job is delegated to the **Container Network Interface (CNI)** plugin.

Here is what happens when a Pod is scheduled to a node:

```text
kubelet ──► container runtime (containerd / CRI-O)
                 │  creates the Pod sandbox and network namespace
                 ▼
            CNI plugin (e.g. bridge, calico, cilium)    "ADD"
                 │  creates the interface, sets up routes
                 ▼
            IPAM plugin (e.g. host-local, calico-ipam)
                 │  picks a free address and records it
                 ▼
            IP returned ──► written to Pod status (status.podIPs)
```

The CNI spec splits the work in two. The main plugin builds the network interface; the **IPAM plugin** only picks addresses. The main plugin calls the IPAM plugin named in the `ipam` section of its config, gets back an address (plus gateway and routes), and configures the interface with it. When the Pod is deleted, the runtime calls `DEL`, and the IPAM plugin releases the address.

Some CNIs use a generic IPAM plugin like `host-local`; others (Calico, Cilium, the AWS VPC CNI) ship their own. Either way, the IPAM plugin needs to answer one question: *which addresses am I allowed to hand out on this node?* There are a few different answers, and that is what separates the IPAM models below.

## Layer 1: giving each node a pod CIDR

The simplest answer is: give each node its own slice of the pod range, and let each node hand out addresses from its slice without talking to anyone else.

Kubernetes has a built-in controller for exactly this. The **node IPAM controller** runs inside `kube-controller-manager`. It watches for new nodes and allocates CIDRs from the available pool, and when a node is deleted, its CIDR is eventually released for reuse.[^nodecidr] It is turned on with three flags:

```text
kube-controller-manager \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.244.0.0/16 \
  --node-cidr-mask-size=24
```

With these settings, each node gets a `/24` (256 addresses) out of `10.244.0.0/16` and the result is written to the Node object:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODCIDRS:.spec.podCIDRs
```

```text
NAME     PODCIDRS
node-1   [10.244.0.0/24]
node-2   [10.244.1.0/24]
node-3   [10.244.2.0/24]
```

The default mask size is 24 for IPv4 and 64 for IPv6.[^azureipam] For dual-stack clusters you pass one cluster CIDR per family and set `--node-cidr-mask-size-ipv4` and `--node-cidr-mask-size-ipv6` instead.[^nodecidrmd] If you build clusters with kubeadm, `--pod-network-cidr` (or `networking.podSubnet`) is what ends up as `--cluster-cidr`.

The math is simple and worth doing before you build a cluster:

```text
max nodes      = 2^(node mask − cluster mask)
addresses/node = 2^(32 − node mask)
```

| Cluster CIDR | Node mask | Max nodes | Addresses per node |
| --- | --- | --- | --- |
| `/16` | `/24` | 256 | 256 |
| `/16` | `/25` | 512 | 128 |
| `/16` | `/26` | 1,024 | 64 |
| `/14` | `/24` | 1,024 | 256 |

The kubelet's default `maxPods` is 110. A `/24` per node leaves more than twice that many addresses, which is deliberate: it means a freshly released address doesn't have to be reused immediately, which avoids stale connections and conntrack entries hitting the wrong Pod.

Two limitations of the built-in controller matter in practice:

- **One size for every node.** The in-tree controller only supports a fixed node CIDR mask size for all nodes.[^azureipam] A tiny edge node and a 192-core monster get the same `/24`.
- **The settings are effectively frozen.** The cluster CIDR is a single range set at install time, and changing `--node-cidr-mask-size` on a running cluster is dangerous: people have reported that after changing it from `/24` to `/23`, new nodes could be given CIDRs that overlap with existing ones.[^maskbug]

There was an effort (KEP-2593) to fix both with a `ClusterCIDR` API that allowed multiple, discontiguous ranges and per-node-pool mask sizes. SIG Network decided not to build it into `kube-controller-manager` and instead created a separate node IPAM controller with its own CRDs.[^kep2593ml] That project, [`kubernetes-sigs/node-ipam-controller`](https://github.com/kubernetes-sigs/node-ipam-controller), watches Node and ClusterCIDR resources and sets `node.spec.podCIDRs`; to use it, you turn off the built-in controller with `--allocate-node-cidrs=false`.[^nodeipamctl]

Note that `node.spec.podCIDR` is only a *suggestion*. Some CNIs use it (Flannel, Cilium in `kubernetes` mode, Calico when configured with `host-local`); many ignore it completely and run their own IPAM, as we'll see.

## Layer 2: picking an address on the node

Once a node has a range, something has to pick individual addresses out of it. The reference implementation is the **`host-local`** IPAM plugin from the CNI project.

`host-local` allocates addresses out of a set of ranges and stores its state locally on the host filesystem, which guarantees uniqueness on a single host.[^hostlocal] The allocation strategy is loosely round-robin within each range.[^hostlocalgo] Its config is tiny:

```json
{
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{ "subnet": "10.244.1.0/24" }]
    ],
    "routes": [{ "dst": "0.0.0.0/0" }]
  }
}
```

And its "database" is just a directory of files. Allocated addresses are stored as files in `/var/lib/cni/networks/$NETWORK_NAME`.[^hostlocalgo] Each file is named after an IP and contains the ID of the container that owns it:

```bash
ls /var/lib/cni/networks/cbr0/
# 10.244.1.2  10.244.1.3  10.244.1.4  last_reserved_ip.0  lock
cat /var/lib/cni/networks/cbr0/10.244.1.3
# 6f1c9a...   (container ID)
```

That simplicity is also its weak point. If a `DEL` is never delivered (a node crash, a runtime bug, a forced cleanup), the file stays behind and the address is leaked. On a long-lived node with lots of Pod churn, leaked files can slowly fill the range until new Pods fail with `no IP addresses available in range set`. The fix is to compare the files against running containers and delete the stale ones, or put `dataDir` on a tmpfs so state is wiped on reboot.

Flannel is the classic user of this model: its CNI plugin reads the node's subnet and generates a config for the `bridge` plugin with `host-local` IPAM set to that subnet.[^flannel]

## The main IPAM models

Per-node ranges plus `host-local` is only one approach. Real CNIs use roughly four models, with different trade-offs between simplicity, address efficiency and integration with the underlying network.

### Model 1: a fixed range per node

*Used by: Flannel, kubenet-style setups, Cilium "Kubernetes host scope", Calico with `host-local`.*

Each node gets one CIDR (from the node IPAM controller or the CNI's own operator), and allocation within it is local. It's simple, needs no coordination at Pod creation time, and routing is easy: one route per node.

The cost is waste. Every node reserves its full range whether it runs 3 Pods or 100, and the range size is the same for every node. In a `/16` with `/24` per node you can only have 256 nodes, no matter how few Pods they run.

### Model 2: blocks claimed on demand (Calico)

Calico's own IPAM (`calico-ipam`) doesn't use `node.spec.podCIDR` at all. Instead, it divides an **IP pool** into fixed-size blocks and assigns those blocks to nodes.[^calicoblock] The default block is a `/26` (64 addresses).[^calicocidr]

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-pool
spec:
  cidr: 10.244.0.0/16
  blockSize: 26
  ipipMode: Never
  vxlanMode: CrossSubnet
  natOutgoing: true
  nodeSelector: all()
```

A node starts with one block. If it uses all the addresses in that block, it gets another one, and if no blocks are left in the pool, it can take addresses from blocks belonging to other hosts ("borrowing").[^calicov38] Block ownership is recorded in `BlockAffinity` resources, and allocations are stored in the Kubernetes API (or etcd), not on the node's disk.

This is more efficient than fixed ranges: a quiet node holds one `/26`, a busy one holds several. It also routes well, since each block is advertised as one route (over BGP, if you use it).

Things to know:

- **Block size is a trade-off.** Larger blocks mean fewer blocks per host and fewer routes; smaller blocks mean more routes. The docs advise having at least as many blocks in the pool as you have hosts.[^calicoippool]
- **You can't change block size on a live pool.** It can only be set when the pool is created, so changing it means creating a replacement pool and migrating workloads.[^calicoplan]
- **Borrowing breaks route aggregation.** A borrowed address needs its own `/32` route. If you want strict per-node blocks (for example, because your network can only route aggregated blocks), enable `strictAffinity`, at the cost of less efficient address use.[^calicoconf]
- **Pools are a policy tool.** You can have multiple pools, restrict them to nodes with a `nodeSelector` (for example, one pool per zone or rack), mark a pool `assignmentMode: Manual` so it's only used on request,[^calicoippool] and pin a namespace or Pod to a pool with annotations like `cni.projectcalico.org/ipv4pools`.

Useful commands:

```bash
calicoctl ipam show                  # usage per pool
calicoctl ipam show --show-blocks    # blocks and which node owns them
calicoctl ipam show --show-borrowed  # addresses taken from another node's block
calicoctl ipam check                 # find leaked or inconsistent allocations
```

### Model 3: operator-managed pools (Cilium)

Cilium supports several IPAM modes and you choose one at install time.[^ciliumipam] The two most common general-purpose ones are:

- **Cluster scope (`cluster-pool`, the default).** The Cilium operator manages a cluster-wide pool of CIDRs and hands out per-node subnets itself, instead of relying on `kube-controller-manager`.[^kodekloud] By default the pool is `10.0.0.0/8` with a `/24` per node.[^isovalent] The per-node CIDR is stored in the `CiliumNode` resource. You can add more CIDRs to the cluster pool when it fills up, but a node can't be given an extra CIDR once it has one.
- **Multi-pool.** Multiple pools, defined as `CiliumPodIPPool` resources, and the pool for a Pod is chosen based on workload annotations and node labels.[^ciliummulti] A Pod or namespace annotated with `ipam.cilium.io/ip-pool=<name>` gets its address from that pool. Nodes can also get extra CIDRs from a pool on demand, which solves cluster-pool's "one CIDR per node, forever" problem.

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumPodIPPool
metadata:
  name: batch-pool
spec:
  ipv4:
    cidrs:
      - 10.20.0.0/16
    maskSize: 24
```

Cilium also has a `kubernetes` mode (use `node.spec.podCIDR` from the node IPAM controller), a CRD-backed mode where something external fills in the allowed addresses, and cloud modes for AWS ENI, Azure and GKE. Multi-pool pools must not overlap, because Cilium identifies endpoints by IP in its IPCache, so every pod IP must be unique in the cluster.[^ciliummultidocs]

Choose carefully: the safest path to change IPAM mode is a fresh cluster.[^ciliumstable] The only online migration currently available is from cluster-pool to multi-pool.[^ciliumipam]

### Model 4: real VPC addresses (cloud-native IPAM)

In the models above, pod IPs come from a private range that only the cluster knows about, and traffic between nodes is either encapsulated (VXLAN, Geneve, IP-in-IP) or routed with BGP. Cloud-native IPAM takes a different approach: **Pods get real addresses from the cloud network**, so the VPC routes them natively, security groups and flow logs see pod IPs, and no overlay is needed.

The best-known example is the **Amazon VPC CNI** on EKS. In its default mode, it attaches Elastic Network Interfaces (ENIs) to each node and assigns secondary IP addresses from the VPC subnet to them, then gives those to Pods. The catch is that each EC2 instance type has a limit on ENIs and addresses per ENI, which caps Pods per node (a small instance might fit only a dozen or so Pods).

**Prefix delegation** relaxes that cap. Instead of individual addresses, the CNI assigns `/28` prefixes (16 addresses each) to the ENI slots.[^awsprefixja] It keeps a warm pool of spare prefixes so Pods start quickly, controlled by environment variables on the `aws-node` DaemonSet:

- `WARM_PREFIX_TARGET`: how many spare prefixes to keep. The default is 1, which by default means a whole extra `/28` is allocated even if the current prefix is used by only one Pod.[^awsblog]
- `WARM_IP_TARGET` and `MINIMUM_IP_TARGET`: finer-grained targets in addresses; if set, they override `WARM_PREFIX_TARGET`.[^awsprocedure]

```bash
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
```

With prefix delegation, AWS's `max-pods-calculator.sh` gives 110 Pods for a `c5.4xlarge`, and 250 for a `c5.24xlarge`.[^setevoy]

Cloud-native IPAM moves the bottleneck from the cluster to the VPC. **Every Pod (plus every warm address) consumes a real subnet address**, so a `/24` node subnet that was fine for 50 VMs is far too small for 50 nodes running Pods. Prefix delegation adds a fragmentation problem: it needs contiguous free `/28`s in the subnet, which a subnet full of scattered single addresses may not have. The usual fixes are larger subnets, subnet CIDR reservations for prefixes, secondary VPC CIDRs (for example from `100.64.0.0/10`) with "custom networking", or IPv6, where prefix mode is the default and address space stops being a concern.[^hackmd]

Azure (Azure CNI, including its overlay and dynamic-allocation variants) and GKE (VPC-native clusters using alias IP ranges) offer equivalent models with their own limits and knobs.

### Bonus: cluster-wide IPAM for secondary networks (Whereabouts)

Some workloads (telco, storage, anything with a dedicated data-plane network) need *extra* interfaces on Pods, usually attached with Multus and a plugin like `macvlan` or `ipvlan`. These interfaces sit on a flat L2 network shared by all nodes, so per-node ranges don't work: a Pod must get the same kind of address whichever node it lands on, without colliding with Pods elsewhere.

**Whereabouts** is the standard answer. It's a cluster-wide IPAM plugin that tracks allocations in a shared store (Kubernetes custom resources or etcd) and always assigns the lowest available address in the range. It also has a "Fast IPAM" mode that pre-allocates slices of the range to nodes.[^whereabouts] Compared with `host-local`, which only knows about addresses on its own node, Whereabouts assigns addresses across the whole cluster.[^tanzu]

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: storage-net
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.50.0/24",
        "exclude": ["192.168.50.0/28"]
      }
    }
```

## Service IPs: a different allocator entirely

ClusterIPs have nothing to do with the CNI. They are virtual addresses allocated by the **kube-apiserver** when a Service is created, and they only exist as rules programmed by kube-proxy (or by the CNI's own Service implementation, such as Cilium's eBPF load balancer). No interface ever has a ClusterIP.

For most of Kubernetes' history, the range came from a single flag, `--service-cluster-ip-range`, configured independently on each apiserver, and allocations were tracked in one bitmap. If you ran out, the only fix was a disruptive reconfiguration.

That changed with **multiple Service CIDRs** (KEP-1880). It was alpha in 1.29 and beta in 1.31, and in Kubernetes v1.33 the two new APIs it introduced, `ServiceCIDR` and `IPAddress`, became generally available. Cluster administrators can now increase the number of addresses available for ClusterIP Services just by creating new `ServiceCIDR` objects.[^k8s133]

```bash
kubectl get servicecidr
```

```text
NAME         CIDRS          AGE
kubernetes   10.96.0.0/28   98m
```

The apiserver creates a default `ServiceCIDR` called `kubernetes` from its flags. To add room:

```yaml
apiVersion: networking.k8s.io/v1
kind: ServiceCIDR
metadata:
  name: extra-service-range
spec:
  cidrs:
    - 172.21.0.0/24
```

New Services then get addresses from the new range once the old one is full. Every allocated ClusterIP is now also an `IPAddress` object, so `kubectl get ipaddresses` shows exactly which Service owns which IP, and conflicts are caught by the API server instead of by a bitmap.

Rules worth knowing:

- A `ServiceCIDR` can have at most two CIDRs, one per IP family, and the field is immutable.[^servicecidr131] To change a range, create a new object.
- Deleting a `ServiceCIDR` is protected by a finalizer: you have to delete the Services that use its addresses first.[^doit]
- The first address of the default range is always taken by the `kubernetes.default` Service.[^k3s]
- Each range is split into a lower band reserved for Services that ask for a specific `clusterIP` (for example a fixed DNS address like `10.96.0.10`) and an upper band for dynamic allocation, so dynamically allocated Services rarely steal an address someone wanted to hard-code. This came from KEP-3070.
- Anything that assumed a single Service range (custom kube-proxy replacements, firewalls, CoreDNS reverse zones) needs to know about the new CIDRs.[^kep1880]

## Dual-stack

Dual-stack has been stable since v1.23. IPAM-wise it just means doing everything twice: two cluster CIDRs, two node mask sizes, two Service ranges, and an IPAM plugin that returns one address per family. With `host-local`, that's two range sets in `ranges`; with Calico, an IPv4 and an IPv6 pool; with Cilium, both families in the pool config. Pods see both addresses in `status.podIPs`, and Services choose families with `ipFamilyPolicy` and `ipFamilies`.

IPv6 is also the long-term answer to address exhaustion: a single `/64` per node is more addresses than you'll ever use, and in cloud-native modes it removes most of the warm-pool and fragmentation problems.

## Planning your address space

Most IPAM pain comes from decisions made in the first five minutes of building a cluster, which are then very hard to change. Some rules of thumb:

1. **Write down every range first.** Node subnets, pod CIDR, Service CIDR, and every network Pods will need to reach (VPNs, peered VPCs, on-prem, other clusters you might connect with a mesh later). None may overlap.
2. **Size for the cluster you'll have in three years.** Work out maximum nodes × addresses per node. With fixed ranges per node, the node count limit is usually hit long before the address limit.
3. **Match addresses per node to `maxPods`.** Roughly twice `maxPods` avoids immediate reuse. If you raise `maxPods` above about 125, a `/24` is no longer enough.
4. **In cloud-native modes, size the subnets, not just the cluster.** Count Pods plus warm addresses plus nodes plus load balancers, and leave room for node surges during upgrades.
5. **Use separate clusters' ranges that don't overlap**, even if you don't plan to connect them. Multi-cluster networking always arrives eventually.
6. **Choose your CNI's IPAM mode deliberately.** Switching is usually a rebuild.

## When IPAM goes wrong

Typical symptoms and where to look:

| Symptom | Likely cause | Where to look |
| --- | --- | --- |
| Pods stuck in `ContainerCreating`, events show `failed to allocate for range` or `no IP addresses available` | Node range full, or leaked allocations | `kubectl describe pod`; on the node, `/var/lib/cni/networks/`; `calicoctl ipam show --show-blocks`; `CiliumNode` status |
| New node stays `NotReady`, CNI never starts | Node never got a pod CIDR: cluster CIDR exhausted | `kubectl get node -o yaml` (empty `podCIDRs`); `kube-controller-manager` logs for `out of CIDRs` |
| Pods pending with `Too many pods` well below expected density | `maxPods` set from cloud ENI limits | `kubectl describe node` (allocatable pods); VPC CNI config |
| Pods slow to start on cloud nodes | Warm pool too small, cloud API calls on each Pod | CNI/ipamd logs; warm-target settings |
| Service creation fails with an internal error about allocating a ClusterIP | Service range full | `kubectl get servicecidr`, `kubectl get ipaddresses`; add a `ServiceCIDR` |
| Two Pods with the same IP, or traffic reaching the wrong Pod | Overlapping pools, or state lost after a datastore restore | IPAM consistency checks (`calicoctl ipam check`), CNI logs |

Some quick checks:

```bash
# Pod CIDR per node (if the node IPAM controller or Cilium kubernetes mode is used)
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDRs}{"\n"}{end}'

# Pods per node vs allocatable
kubectl describe node <node> | grep -A5 -E "Allocatable|Non-terminated"

# Recent IPAM failures across the cluster
kubectl get events -A --field-selector reason=FailedCreatePodSandBox

# Service ranges and allocations
kubectl get servicecidr
kubectl get ipaddresses | wc -l
```

## Bottom line

- **Kubernetes allocates very few IPs itself.** The apiserver hands out Service ClusterIPs, and `kube-controller-manager` can hand out per-node pod CIDRs. Individual pod IPs come from the CNI's IPAM plugin.
- **There are four main IPAM models:** fixed ranges per node (simple, wasteful), blocks claimed on demand (Calico), operator-managed pools (Cilium), and real VPC addresses (AWS VPC CNI and friends). Whereabouts covers cluster-wide IPAM for secondary networks.
- **Each model fails differently.** Fixed ranges run out of nodes, blocks fragment, pools need planning up front, and cloud-native IPAM exhausts your VPC subnets.
- **Service ranges are no longer fixed.** Since v1.33, you can add `ServiceCIDR` objects to grow the Service range without rebuilding the cluster.
- **Plan your ranges before you build.** Pod CIDR, node mask size, Service range and CNI IPAM mode are all hard or impossible to change later. IPv6 makes most of these problems go away.

## References

[^nodecidr]: [How to configure pod network CIDR allocation per node](https://oneuptime.com/blog/post/2026-02-09-pod-network-cidr-per-node/view), OneUptime.
[^nodecidrmd]: [How to configure pod network CIDR allocation per node (dual-stack flags)](https://oneuptime.com/blog/post/2026-02-09-pod-network-cidr-per-node/markdown), OneUptime.
[^azureipam]: [Node IPAM controller](https://cloud-provider-azure.sigs.k8s.io/topics/ipam/), Cloud Provider Azure documentation.
[^maskbug]: [Support variable sized pod-CIDRs for different-sized nodes, issue #90922](https://github.com/kubernetes/kubernetes/issues/90922), kubernetes/kubernetes.
[^kep2593ml]: [KEP-2593: Enhanced NodeIPAM to support Discontiguous Cluster CIDR](https://groups.google.com/g/kubernetes-sig-network/c/nts1xEZ--gQ/m/2aTOUNFFAAAJ), kubernetes-sig-network mailing list. See also [KEP-2593](https://www.kubernetes.dev/resources/keps/2593/).
[^nodeipamctl]: [kubernetes-sigs/node-ipam-controller](https://github.com/kubernetes-sigs/node-ipam-controller), GitHub.
[^hostlocal]: [host-local IP address management plugin](https://www.cni.dev/plugins/current/ipam/host-local/), CNI documentation.
[^hostlocalgo]: [host-local README](https://github.com/containernetworking/plugins/tree/main/plugins/ipam/host-local), containernetworking/plugins.
[^flannel]: [Flannel CNI plugin](https://pkg.go.dev/github.com/flannel-io/cni-plugin), flannel-io.
[^calicoblock]: [Configure BlockAffinity Behavior in Calico](https://oneuptime.com/blog/post/2026-03-13-configure-blockaffinity-behavior-calico/markdown), OneUptime.
[^calicocidr]: [Troubleshoot Node CIDR Planning in Calico](https://oneuptime.com/blog/post/2026-03-13-troubleshoot-node-cidr-planning-calico/markdown), OneUptime.
[^calicov38]: [IP pool resource (v3.8)](https://docs.tigera.io/archive/v3.8/reference/resources/ippool), Calico documentation.
[^calicoippool]: [IP pool resource](https://docs.tigera.io/calico/latest/reference/resources/ippool), Calico documentation.
[^calicoplan]: [Configure Node CIDR Planning with Calico](https://oneuptime.com/blog/post/2026-03-13-configure-node-cidr-planning-calico/view), OneUptime.
[^calicoconf]: [How to Use calicoctl ipam configure](https://oneuptime.com/blog/post/2026-03-15-calicoctl-ipam-configure/markdown), OneUptime.
[^ciliumipam]: [IP Address Management (IPAM)](https://docs.cilium.io/en/latest/network/concepts/ipam/), Cilium documentation.
[^kodekloud]: [Cilium IP Address Management (IPAM)](https://notes.kodekloud.com/docs/Prep-Course-Cilium-Certified-Associate-CCA-Certification/Exploring-Cilium/IP-Address-Management-IPAM/page.md), KodeKloud.
[^isovalent]: [Overcoming Kubernetes IP Address Exhaustion with Cilium](https://isovalent.com/blog/post/overcoming-kubernetes-ip-address-exhaustion-with-cilium/), Isovalent.
[^ciliummulti]: [Multi-Pool IPAM](https://docs.cilium.io/en/stable/network/concepts/ipam/multi-pool/), Cilium documentation.
[^ciliummultidocs]: [Multi-Pool (Beta): limitations](https://cilium.readthedocs.io/en/stable/network/concepts/ipam/multi-pool/), Cilium documentation.
[^ciliumstable]: [IP Address Management (IPAM), stable docs](https://cilium.readthedocs.io/en/stable/network/concepts/ipam/), Cilium documentation.
[^awsprefixja]: [Prefix Mode for Linux](https://docs.aws.amazon.com/eks/latest/best-practices/prefix-mode-linux.html), Amazon EKS Best Practices Guide.
[^awsblog]: [Amazon VPC CNI increases pods per node limits](https://aws.amazon.com/blogs/containers/amazon-vpc-cni-increases-pods-per-node-limits), AWS Containers Blog.
[^awsprocedure]: [Increase the available IP addresses for your Amazon EKS node](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses-procedure.html), Amazon EKS User Guide.
[^setevoy]: [AWS: VPC Prefix and the maximum of Pods on Kubernetes WorkerNodes](https://itnext.io/aws-vpc-prefix-and-the-maximum-of-pods-on-kubernetes-workernodes-b670d2d3a303), Arseny Zinchenko.
[^hackmd]: [Prefix delegation notes (IPv6 prefix mode)](https://hackmd.io/@sheetjos/pd), based on the EKS Best Practices Guide.
[^whereabouts]: [Whereabouts IPAM CNI plugin](https://github.com/k8snetworkplumbingwg/whereabouts), k8snetworkplumbingwg.
[^tanzu]: [Install Multus with Whereabouts](https://docs.vmware.com/en/VMware-Tanzu-Packages/latest/tanzu-packages/packages-whereabouts-mc.html), VMware Tanzu documentation.
[^k8s133]: [Kubernetes v1.33: Octarine](https://kubernetes.io/blog/2025/04/23/kubernetes-v1-33-release/), Kubernetes blog.
[^servicecidr131]: [ServiceCIDR v1beta1 API reference](https://v1-31.docs.kubernetes.io/docs/reference/kubernetes-api/cluster-resources/service-cidr-v1beta1), Kubernetes documentation. See also the [v1 reference](https://kubernetes.io/docs/reference/kubernetes-api/networking/service-cidr-v1/).
[^doit]: [Scaling Kubernetes: How to Seamlessly Expand Service IP Ranges](https://www.doit.com/blog/scaling-kubernetes-how-to-seamlessly-expand-service-ip-ranges), DoiT.
[^k3s]: [How to Configure K3s Service CIDR](https://oneuptime.com/blog/post/2026-03-20-k3s-service-cidr/markdown), OneUptime.
[^kep1880]: [KEP-1880: Multiple Service CIDRs](https://www.kubernetes.dev/resources/keps/1880/), Kubernetes enhancements.
