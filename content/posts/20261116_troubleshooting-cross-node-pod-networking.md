---
title: "Pods Can't Reach Pods on Other Nodes: A Troubleshooting Guide"
date: 2026-11-16
draft: true
description: "A step-by-step method for debugging Kubernetes pod networking when Pods on the same node can talk to each other but pings to Pods on other nodes time out: CNI health, pod CIDR routes, tunnel interfaces, firewall ports for VXLAN, IP-in-IP and Geneve, kernel forwarding, and tracing packets with tcpdump."
summary: "Same-node pod traffic works, cross-node traffic times out. That symptom rules out half the stack straight away. Here is where to look next, in order, and how to use tcpdump to find exactly where the packet disappears."
tags: ["kubernetes", "networking", "cni", "calico", "cilium", "flannel", "troubleshooting", "cka"]
categories: ["Programming"]
---

It's a classic: Pods on the same node can ping each other fine, but pings to Pods on any other node just time out. You'll see it on fresh kubeadm clusters, in Vagrant labs, and in cloud clusters where someone tightened a security group.

The good news is that the symptom itself tells you a lot. This post walks through where to look, in the order I'd look.

## What the symptom tells you

When two Pods on the **same node** talk, the packet goes from one Pod's veth pair, through the node's bridge or routing table, into the other Pod's veth. It never leaves the machine. If that works, then:

- the Pods have IPs and their network namespaces are set up,
- the CNI plugin works on each node,
- the local bridge or routes are fine.

When two Pods on **different nodes** talk, the packet also has to:

1. find a route on the source node to the other node's pod CIDR,
2. usually get wrapped in a tunnel (VXLAN, IP-in-IP, Geneve), or be routed natively,
3. cross the physical or cloud network between the nodes,
4. get unwrapped and forwarded on the destination node,
5. make the same trip back.

So the problem is somewhere in that cross-node path. It belongs to the CNI's overlay or routing layer, and to the network underneath it. Kubernetes' networking model requires every Pod to reach every other Pod without NAT, but it's the CNI plugin that actually makes this work[^k8snet].

Also note that **kube-proxy isn't involved** here. You're pinging a Pod IP directly, not a Service, so Service rules don't matter.

## 1. Is the CNI healthy on every node?

Start with the cheapest check. The CNI agent runs as a DaemonSet, and if it's broken on one node, that node is cut off from the rest.

```bash
kubectl get pods -n kube-system -o wide
```

Look for `calico-node`, `cilium`, `kube-flannel-ds`, `weave-net` or whichever agent you use. Are any CrashLooping or not Ready? Is one missing on a node? Check the logs of anything unhealthy.

Most CNIs also have their own status tools:

```bash
# Cilium
cilium status
cilium-health status          # node-to-node reachability from Cilium's point of view

# Calico
calicoctl node status         # BGP peers should be "Established"
```

## 2. Do the nodes know how to reach each other's pod CIDRs?

Each node owns a slice of the pod network. Every other node needs a route to that slice.

First, check that each node has a different, non-overlapping range:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,CIDR:.spec.podCIDR
```

(Some CNIs, like Calico with its own IPAM, ignore `spec.podCIDR` and track blocks themselves. My post on IPAM in Kubernetes covers the details.)

Then look at the routing table on a node:

```bash
ip route
```

You should see one route for every *other* node's pod range, pointing at a tunnel interface (`flannel.1`, `vxlan.calico`, `tunl0`, `cilium_host`) or directly at the other node's IP if you route without a tunnel. A missing route means the CNI never learned about that node, which takes you back to step 1 (agent logs, BGP status).

One more classic kubeadm mistake: the cluster CIDR given to kubeadm (`--pod-network-cidr`, `networking.podSubnet`) must match the pool configured in the CNI. If they don't agree, nodes get pod ranges that the CNI doesn't route.

## 3. Is the CNI using the right interface?

This is the most common cause in Vagrant, VirtualBox and any multi-NIC setup.

VirtualBox VMs usually have two interfaces: `eth0` on the NAT network, where **every VM has the same IP** (`10.0.2.15`), and `eth1` on the host-only or private network you actually want. Many CNIs pick the interface with the default route, which is `eth0`. Each node then tells the others "send my pod traffic to `10.0.2.15`", and that goes nowhere.

Check which address each node is advertising:

```bash
kubectl get nodes -o wide                                          # INTERNAL-IP
kubectl get node <node> -o yaml | grep -iE 'public-ip|IPv4Address'
```

Flannel stores the address in the `flannel.alpha.coreos.com/public-ip` annotation, and Calico stores it in `projectcalico.org/IPv4Address`. If they show the NAT address, fix the interface selection:

- **Flannel:** pass `--iface=eth1` (or `--iface-regex`) to flanneld[^flannelconf].
- **Calico:** set `IP_AUTODETECTION_METHOD=interface=eth1` (or `can-reach=<ip>`, or `cidr=<range>`) on `calico-node`[^calicoautodetect].
- **kubelet:** set `--node-ip` to the private address, so the node's `INTERNAL-IP` is right in the first place.

## 4. Is the network between the nodes letting tunnel traffic through?

A very common setup: someone opened SSH, 6443 for the API server and 10250 for the kubelet between nodes, and everything *looks* healthy. But the overlay uses its own ports, and those are being dropped.

Which ports you need depends on the CNI and mode:

| CNI / mode | What must be allowed between nodes |
| --- | --- |
| Flannel VXLAN | UDP 8472 (the Linux default VXLAN port)[^flannelbackends] |
| Cilium VXLAN | UDP 8472[^ciliumreq] |
| Cilium Geneve | UDP 6081[^ciliumreq] |
| Cilium WireGuard | UDP 51871[^ciliumreq] |
| Calico IP-in-IP (default) | IP protocol 4[^calicoreq] |
| Calico VXLAN | UDP 4789[^calicoreq] |
| Calico BGP | TCP 179[^calicoreq] |
| Calico WireGuard | UDP 51820[^calicoent] |

Watch out for **IP-in-IP**. It isn't TCP or UDP, it's its own IP protocol (number 4). Firewall rules written as "allow these TCP and UDP ports" never match it, so it gets dropped while everything else works. In AWS security groups, for example, you have to add a rule with protocol `4` specifically.

Cloud-specific things to check:

- **Security groups, NACLs, NSGs and VPC firewall rules** between the node subnets.
- **AWS source/destination check.** If you route pod traffic natively (no tunnel), packets leave a node with a pod IP as the source, not the node's IP. EC2 drops those unless source/destination check is disabled on the instances.
- **Native routing in general.** Without a tunnel, the network underneath has to know how to route pod CIDRs. Cilium's docs put it plainly: in direct routing mode, the network must allow routing of pod IPs[^ciliumreq].

## 5. Will the node kernel forward the packets?

Even when packets arrive at the right node, the kernel can still drop them.

```bash
sysctl net.ipv4.ip_forward              # must be 1
iptables -L FORWARD -n -v               # check the policy and the drop counters
```

A frequent cause is **Docker** being installed on the node alongside containerd. When Docker enables IP forwarding, it also sets the default policy of the iptables `FORWARD` chain to `DROP`[^dockerfw]. Forwarded pod traffic that no other rule explicitly accepts is then quietly dropped. Either remove Docker, add accept rules for the pod CIDR, or set `"ip-forward-no-drop": true` in `/etc/docker/daemon.json`[^dockerpf].

Two more kernel settings worth checking:

- **`rp_filter`:** strict reverse-path filtering (`1`) drops packets whose return route goes out a different interface than they came in on, which happens with asymmetric routing. Loose mode (`2`) is usually safe.
- **`br_netfilter`:** for bridge-based CNIs, the `br_netfilter` module and `net.bridge.bridge-nf-call-iptables=1` are expected by most setups.

## 6. Trace the packet

When the checks above don't reveal it, stop guessing and follow the packet. Start a ping from a Pod on node A to a Pod on node B, and capture traffic at each hop.

```bash
# Node A: does the encapsulated packet leave?
tcpdump -ni eth0 'udp port 8472 or ip proto 4'

# Node B: does it arrive?
tcpdump -ni eth0 'udp port 8472 or ip proto 4'

# Node B: is it delivered to the pod side after unwrapping?
tcpdump -ni <tunnel-or-veth-interface> icmp
```

Adjust the filter for your CNI (`udp port 4789` for Calico VXLAN, `udp port 6081` for Geneve). Then read the result:

| What you see | Where the problem is |
| --- | --- |
| Nothing leaves node A | Source node: missing route, CNI agent broken, wrong tunnel interface |
| Leaves node A, never reaches node B | The network between: firewall, security group, wrong interface or IP advertised |
| Reaches node B, doesn't reach the Pod | Destination node: FORWARD chain, routes, NetworkPolicy |
| Reaches the Pod, no reply comes back | The return path: routes or `rp_filter` on the way back |

Cilium users can skip some of this with `cilium monitor` or Hubble (`hubble observe --verdict DROPPED`), which tells you exactly where a packet was dropped and why.

## Less likely suspects

- **NetworkPolicy.** A default-deny ingress policy would usually block same-node traffic too, so it doesn't fit the symptom well. Still, policies can select Pods by labels or namespaces in ways that happen to match the cross-node case. A quick `kubectl get netpol -A` rules it out.
- **MTU.** Tunnels add headers (50 bytes for VXLAN, 20 for IP-in-IP), so the Pod MTU must be smaller than the node MTU. A *mismatch* gives a different symptom, though: small pings work, but large packets and TCP transfers hang. If even a basic ping times out, it isn't MTU. You can test with `ping -s 1400 -M do <pod-ip>`.

## Bottom line

- Same-node works and cross-node fails means the problem is in **the path between nodes**: the CNI's overlay or routing, and the network underneath.
- Work outwards: **CNI health → routes → advertised interface → firewall → kernel forwarding**, then use **tcpdump** to see exactly where the packet disappears.
- In practice, most cases are **the CNI using the wrong interface** (especially in VirtualBox and Vagrant labs) or **tunnel traffic blocked between nodes**, with IP-in-IP (protocol 4) being the one firewall rules most often miss.

## References

[^k8snet]: [Services, Load Balancing, and Networking: the Kubernetes network model](https://kubernetes.io/docs/concepts/services-networking/), Kubernetes documentation.
[^flannelconf]: [Flannel configuration](https://github.com/flannel-io/flannel/blob/master/Documentation/configuration.md), flannel-io.
[^flannelbackends]: [Flannel backends](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md), flannel-io.
[^calicoautodetect]: [Configure IP autodetection](https://docs.tigera.io/calico/latest/networking/ipam/ip-autodetection), Calico documentation.
[^calicoreq]: [System requirements: network requirements](https://docs.tigera.io/calico/latest/getting-started/bare-metal/requirements), Calico documentation. See also the [Kubernetes requirements page](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements).
[^calicoent]: [System requirements](https://docs.tigera.io/calico-enterprise/3.21/getting-started/install-on-clusters/requirements), Calico Enterprise documentation.
[^ciliumreq]: [System Requirements: firewall rules](https://docs.cilium.io/en/stable/operations/system_requirements/), Cilium documentation.
[^dockerfw]: [Docker with iptables](https://docs.docker.com/engine/network/firewall-iptables/), Docker documentation.
[^dockerpf]: [Packet filtering and firewalls](https://docs.docker.com/network/packet-filtering-firewalls/), Docker documentation.
