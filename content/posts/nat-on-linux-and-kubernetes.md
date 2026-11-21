---
title: "NAT: From Your Home Router to Linux Netfilter to Kubernetes Services"
date: 2026-11-18
draft: true
description: "A practical guide to Network Address Translation: what SNAT, DNAT, masquerading and NAPT are, how Linux implements NAT with netfilter, conntrack, iptables and nftables, and where NAT hides inside Kubernetes Services, NodePorts and pod egress."
summary: "NAT is the reason your laptop, your containers and your Kubernetes Services can all reach each other with too few IP addresses. Here is how NAT works in general, how the Linux kernel does it with conntrack and netfilter, and where Kubernetes quietly rewrites your packets."
tags: ["networking", "linux", "nat", "iptables", "nftables", "kubernetes"]
---

Almost every packet you send today gets rewritten on the way. Your laptop has a private address like `192.168.1.23`, yet websites see a public address belonging to your router. A container listening on port 80 is reachable on port 8080 of the host. A Kubernetes Service has an IP address that no network interface anywhere actually owns. All three are the same trick: **Network Address Translation**, or NAT.

NAT is one of those technologies everybody uses and few people look at closely, until something breaks: the client IP in your logs is wrong, a connection hangs for exactly five seconds, or a pod can talk to every Service except its own. This post goes through NAT in three layers: the general idea, how the Linux kernel implements it, and where Kubernetes uses it (often without telling you).

## Part 1: NAT in general

### Why NAT exists

An IPv4 address is 32 bits, so there are about 4.3 billion of them, and far fewer are usable on the public internet. That ran out a long time ago. The fix was to set aside ranges that anyone may use internally but that are never routed on the public internet. RFC 1918 defines three of them:[^rfc1918]

| Range | Size | Typical use |
| --- | --- | --- |
| `10.0.0.0/8` | ~16.7 million addresses | Enterprises, cloud VPCs, Kubernetes pod networks |
| `172.16.0.0/12` | ~1 million addresses | Docker's default bridges, corporate networks |
| `192.168.0.0/16` | ~65,000 addresses | Home routers |

Thousands of homes all use `192.168.1.0/24` at the same time. That only works if something at the edge of each network rewrites the private source address into a public one before the packet leaves, and rewrites the reply back on the way in. That something is NAT.

NAT was meant as a stopgap until IPv6 arrived. It turned out to be useful for much more than saving addresses: it hides internal topology, lets you move services around behind a stable address, and implements load balancing. That is why it shows up in places where address scarcity is not the problem at all, like Kubernetes.

### The vocabulary

NAT rewrites addresses (and usually ports) in IP packet headers. Which header field it rewrites gives you the main types:

- **SNAT (Source NAT)** rewrites the *source* address of outgoing packets. This is what your home router does: `192.168.1.23` becomes `203.0.113.7`. Use it when many internal hosts need to share one outgoing address.
- **Masquerading** is a special case of SNAT where the new source address is not configured but taken from whatever address the outgoing interface currently has. It exists for interfaces with dynamic addresses, like a DSL line or a DHCP-configured uplink.
- **DNAT (Destination NAT)** rewrites the *destination* address of incoming packets. "Port forwarding" on your router is DNAT: traffic to `203.0.113.7:8080` is redirected to `192.168.1.50:80`. Load balancers and Kubernetes Services are DNAT.
- **NAPT (Network Address and Port Translation)**, also called PAT or "NAT overload", rewrites ports as well as addresses, so that many internal hosts can share a *single* public address. RFC 3022 calls this "Traditional NAT".[^rfc3022] When people say NAT in everyday speech, they almost always mean NAPT.
- **1:1 NAT (static NAT)** maps one internal address to one external address with no port rewriting. Cloud "elastic" or "floating" IPs often work this way.

### How NAPT keeps track

The key insight is that NAT is **stateful**. When the first packet of a connection goes out, the NAT device picks a translation and writes it into a table. Every later packet of the same connection, in both directions, is translated using that entry.

Say two laptops behind the same router both open a connection to the same web server at `198.51.100.10:443`:

```text
Inside (original)                     Outside (translated)
192.168.1.23:51000 -> 198.51.100.10:443   203.0.113.7:40001 -> 198.51.100.10:443
192.168.1.42:51000 -> 198.51.100.10:443   203.0.113.7:40002 -> 198.51.100.10:443
```

Both laptops happened to choose source port `51000`. If the router only rewrote the address, the two connections would look identical from the outside and replies could not be told apart. So the router also rewrites the source port. When a reply arrives at `203.0.113.7:40002`, the router looks up the table, finds the entry, and rewrites the destination back to `192.168.1.42:51000`.

A connection is identified by its **5-tuple**: protocol, source address, source port, destination address, destination port. A NAT table entry maps an original 5-tuple to a translated one. This has some consequences worth remembering:

- **The ports are a finite resource.** One public IP has roughly 64,000 ports per protocol. NAT implementations can reuse the same public port for connections to *different* destinations, because the 5-tuple still differs, but many connections to the *same* destination from one public IP are limited by the port range. This is called **port exhaustion**, and it is a real problem for busy NAT gateways, for example in cloud environments where lots of workloads talk to the same API endpoint.
- **Entries expire.** TCP connections have a clear end (FIN or RST), but UDP and ICMP do not, so the NAT device forgets them after a timeout. A long-idle UDP "connection" or even a TCP connection with no keepalives can silently break when its entry is evicted.
- **Inbound connections do not work by default.** A packet arriving from outside with no matching entry has nowhere to go. That is why NAT is often mistaken for a firewall. It is not one, but it has a similar effect for unsolicited inbound traffic.

### NAT behaviour and peer-to-peer

Not all NATs behave the same. RFC 4787 describes the behaviours that matter for applications.[^rfc4787] The most important distinction is the **mapping behaviour**:

- **Endpoint-independent mapping**: an internal `ip:port` always gets the same external `ip:port`, no matter which destination it talks to. This is what the RFC recommends.
- **Address-dependent or address-and-port-dependent mapping**: the external port changes depending on the destination. Older literature calls the strictest version a "symmetric NAT".

This matters for peer-to-peer applications like video calls and games. Two peers behind NATs cannot connect to each other directly because neither accepts inbound connections. They work around this by asking a public server "what address do you see me as?" (that is the STUN protocol[^stun]), exchanging those addresses, and sending packets at each other at the same time so that both NATs create entries. This is called **hole punching**. It works with endpoint-independent mapping and fails with symmetric NAT, in which case the traffic has to be relayed through a server (TURN).

### NAT on a larger scale

Two more variants you will run into:

- **Carrier-grade NAT (CGNAT)**: many internet providers no longer give each customer a public IPv4 address. They put customers behind a provider-level NAT, so your traffic gets translated twice (once at home, once at the ISP). RFC 6598 reserved `100.64.0.0/10` for the network between the two.[^rfc6598] If you see a `100.64.x.x` address on your router's WAN side, you are behind CGNAT, and port forwarding on your home router will not help you.
- **NAT64**: translates between IPv6 and IPv4, letting IPv6-only clients reach IPv4-only servers.[^rfc6146] This is how many mobile networks run IPv6-only internally.

IPv6 has enough addresses that NAT is not needed to share them, and the IPv6 community generally discourages it. That does not mean IPv6 networks have no translation at all (NPTv6 prefix translation exists), but the default design is end-to-end addressing with a firewall, not NAT.

## Part 2: NAT on Linux

On Linux, NAT is implemented in the kernel by **netfilter**, the same framework that does packet filtering. The two pieces you need to understand are the netfilter hooks and the connection tracker.

### Netfilter hooks

Netfilter places hooks at fixed points along a packet's path through the kernel. Rules attached to a hook see every packet passing that point.

```text
                         incoming packet
                               |
                         [PREROUTING]   <- DNAT happens here
                               |
                        routing decision
                        /              \
             for this host           for another host
                   |                       |
               [INPUT]                 [FORWARD]
                   |                       |
            local process                  |
                   |                       |
               [OUTPUT]   <- DNAT for      |
                   |         locally       |
                   |         generated     |
                   |         packets       |
                    \                     /
                        routing decision
                               |
                         [POSTROUTING]  <- SNAT / MASQUERADE happen here
                               |
                         outgoing packet
```

The placement is not arbitrary:

- **DNAT happens before routing** (in `PREROUTING`, or `OUTPUT` for packets generated on the host itself). The kernel must know the *real* destination before it decides where to send the packet. Rewrite the destination after routing and the packet goes the wrong way.
- **SNAT happens after routing** (in `POSTROUTING`). Only after routing does the kernel know which interface the packet leaves through, and therefore which source address makes sense. That is also why `MASQUERADE` can ask "what is the address of the outgoing interface?"

### Conntrack: the NAT table

The second piece is **conntrack**, the connection tracking subsystem. Conntrack follows every connection through the machine and stores an entry for each one, with its original and reply direction.

This leads to the most important fact about Linux NAT: **NAT rules are only evaluated for the first packet of a connection.** When a packet creates a new conntrack entry, the kernel walks the `nat` rules and records the chosen translation in the entry. Every later packet of that connection, in both directions, is translated from the conntrack entry, and the `nat` rules are never looked at again.

That has practical effects:

- Changing a NAT rule does not affect connections that already exist. They keep their old translation until the conntrack entry goes away.
- The reply direction is "un-NATed" automatically. You never write a rule for the return traffic.
- If conntrack loses track of a connection (the table is full, the entry expired, or the reply arrives through a different machine), NAT breaks for that connection.

You can look at the table with the `conntrack` tool[^conntrack] (package `conntrack` or `conntrack-tools`):

```sh
$ sudo conntrack -L -p tcp --dport 443
tcp      6 431999 ESTABLISHED src=10.0.0.2 dst=198.51.100.10 sport=51000 dport=443 \
    src=198.51.100.10 dst=203.0.113.7 sport=443 dport=51000 [ASSURED] mark=0 use=1
```

Read this as two tuples. The first is the original direction as the kernel first saw it: `10.0.0.2:51000 -> 198.51.100.10:443`. The second is what the *reply* is expected to look like: from `198.51.100.10:443` to `203.0.113.7:51000`. The difference between "where I sent it from" (`10.0.0.2`) and "where the reply goes to" (`203.0.113.7`) is the SNAT. The kernel kept the source port because it was free; it only changes ports when it has to.

A few other useful commands:

```sh
sudo conntrack -C                          # number of entries
sysctl net.netfilter.nf_conntrack_max      # table size limit
sudo conntrack -S                          # per-CPU stats, watch insert_failed and drop
sudo conntrack -E                          # stream events as connections come and go
sudo conntrack -D -d 198.51.100.10         # delete entries (forces re-evaluation of NAT rules)
```

When `nf_conntrack_count` hits `nf_conntrack_max`, the kernel drops new connections and logs `nf_conntrack: table full, dropping packet`. On busy NAT gateways and Kubernetes nodes, this is a classic source of mysterious packet loss.

### Hands-on: building a NAT router with namespaces

The easiest way to see this working is to turn your machine into a tiny router for a "private network" made of a network namespace. You need root and `iproute2`. Replace `eth0` with your real uplink interface (`ip route get 1.1.1.1` tells you which one it is).

First, create a namespace and connect it to the host with a veth pair, using a private `10.200.0.0/24` network:

```sh
sudo ip netns add client
sudo ip link add veth-host type veth peer name veth-client
sudo ip link set veth-client netns client

sudo ip addr add 10.200.0.1/24 dev veth-host
sudo ip link set veth-host up

sudo ip netns exec client ip addr add 10.200.0.2/24 dev veth-client
sudo ip netns exec client ip link set veth-client up
sudo ip netns exec client ip link set lo up
sudo ip netns exec client ip route add default via 10.200.0.1
```

The namespace can now reach the host, but not the internet:

```sh
$ sudo ip netns exec client ping -c1 -W1 1.1.1.1
1 packets transmitted, 0 received, 100% packet loss
```

Two things are missing. The host does not forward packets between interfaces, and even if it did, the replies would be addressed to `10.200.0.2`, which nobody on the internet can route back. Enable forwarding first:

```sh
sudo sysctl -w net.ipv4.ip_forward=1
```

Then add masquerading. With `iptables`:[^iptables-ext]

```sh
sudo iptables -t nat -A POSTROUTING -s 10.200.0.0/24 -o eth0 -j MASQUERADE
```

Or, the modern way, with `nftables`:[^nft-nat]

```sh
sudo nft add table ip lab
sudo nft add chain ip lab postrouting '{ type nat hook postrouting priority srcnat; }'
sudo nft add rule ip lab postrouting ip saddr 10.200.0.0/24 oifname "eth0" masquerade
```

(If your host has a restrictive `FORWARD` policy, for example because Docker is installed, you also need to allow forwarded traffic from `veth-host`.) Now it works, and conntrack shows you the translation:

```sh
$ sudo ip netns exec client ping -c1 1.1.1.1
64 bytes from 1.1.1.1: icmp_seq=1 ttl=57 time=12.3 ms

$ sudo conntrack -L -p icmp
icmp     1 29 src=10.200.0.2 dst=1.1.1.1 type=8 code=0 id=4242 \
    src=1.1.1.1 dst=192.168.1.23 type=0 code=0 id=4242 mark=0 use=1
```

The reply tuple goes to `192.168.1.23`, the host's own address: the namespace has been masqueraded behind it.

Now the other direction. Start a web server inside the namespace and publish it on port 8080 of the host with DNAT, exactly like a router's port forwarding or `docker run -p 8080:80`:

```sh
sudo ip netns exec client python3 -m http.server 80 &

# iptables
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.200.0.2:80

# or nftables
sudo nft add chain ip lab prerouting '{ type nat hook prerouting priority dstnat; }'
sudo nft add rule ip lab prerouting tcp dport 8080 dnat to 10.200.0.2:80
```

Another machine on your LAN can now open `http://<host-ip>:8080`. But try `curl localhost:8080` on the host itself, and it fails. Locally generated packets never pass `PREROUTING`; they go through `OUTPUT`. You need the same rule there (`iptables -t nat -A OUTPUT ...` or an nftables chain with `hook output`). Docker solves this with exactly such an `OUTPUT` rule, plus a userspace proxy as a fallback.

Clean up with `sudo ip netns del client` (which also removes the veth pair) and `sudo nft delete table ip lab` or by deleting the iptables rules.

### SNAT vs MASQUERADE

Both rewrite the source address, so which should you use?

- `SNAT --to-source 203.0.113.7` uses a fixed address. It is slightly cheaper, because the kernel does not look up the interface address for each new connection.
- `MASQUERADE` uses the outgoing interface's address at the time the connection starts. When the interface goes down, the kernel flushes all conntrack entries that were masqueraded through it, because their address is probably gone.

Rule of thumb: static address, use SNAT; dynamic address (DHCP, PPPoE, or an address you don't control, like the node IP in an autoscaling group), use MASQUERADE. Kubernetes uses MASQUERADE almost everywhere, for that reason.

### Port allocation and `--random-fully`

When the kernel must pick a new source port, by default it tries to keep the original port, and if that is taken it searches for a free one in a fairly predictable way. Under high connection rates from many sources to the same destination, two connections on different CPUs can pick the same port at the same moment. One of them loses the race at conntrack insert time, and the packet is dropped (it shows up in `conntrack -S` as `insert_failed`).

`iptables` offers `--random` and `--random-fully` for `MASQUERADE` and `SNAT`; nftables has the `random` and `fully-random` flags. `--random-fully` makes port selection fully random, which greatly reduces these collisions. Kubernetes and most CNI plugins enable it when the kernel supports it.

## Part 3: NAT in Kubernetes

The Kubernetes networking model makes a promise that sounds like the opposite of NAT: every pod gets its own IP address, and **pods can communicate with all other pods on any node without NAT**.[^k8s-model] A pod sees its own IP address, and other pods see that same address when it talks to them.

That promise is about pod-to-pod traffic only. As soon as Services, nodes or the outside world are involved, Kubernetes uses NAT heavily. There are four places to know.

### 1. Services are DNAT

A ClusterIP Service gets a virtual IP, say `10.96.45.12`. No interface on any node has this address, and nothing answers ARP for it. It only exists as NAT rules. On every node, **kube-proxy** watches Services and EndpointSlices and programs rules that DNAT traffic sent to the virtual IP to one of the backing pods.[^k8s-vip]

In the classic `iptables` mode, you can see the chain of rules for a Service with three endpoints:

```sh
$ sudo iptables -t nat -S | grep -E 'default/web|KUBE-SVC-ABC|KUBE-SEP-'
-A KUBE-SERVICES -d 10.96.45.12/32 -p tcp -m comment --comment "default/web cluster IP" \
    -m tcp --dport 80 -j KUBE-SVC-ABC
-A KUBE-SVC-ABC ! -s 10.244.0.0/16 -d 10.96.45.12/32 -p tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-ABC -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-1
-A KUBE-SVC-ABC -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-2
-A KUBE-SVC-ABC -j KUBE-SEP-3
-A KUBE-SEP-1 -s 10.244.1.5/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-1 -p tcp -m tcp -j DNAT --to-destination 10.244.1.5:8080
...
```

(Real chain names are hashes like `KUBE-SVC-NPX46M4PTMTKRN6Y`; they are shortened here.) Walking through it:

1. `KUBE-SERVICES` is jumped to from `PREROUTING` (traffic from pods) and `OUTPUT` (traffic from the node itself). It matches the ClusterIP and port and jumps to the Service's chain.
2. `KUBE-SVC-ABC` picks an endpoint at random. The probabilities are chosen so the result is uniform: the first rule matches 1/3 of the time, the second matches 1/2 of the remaining 2/3, and the last catches the rest.
3. `KUBE-SEP-1` (Service EndPoint) performs the actual `DNAT` to `10.244.1.5:8080`.

Because of conntrack, this happens once per connection. All later packets of a TCP connection stick to the same pod, and the reply from the pod gets its source rewritten back to `10.96.45.12:80` automatically, so the client never knows which pod answered.

This is also why "load balancing" with ClusterIP is per connection, not per request. A client with one long-lived HTTP/2 or gRPC connection sends all requests to the same pod, no matter how many replicas exist.

The newer `nftables` mode (generally available since Kubernetes 1.33) does the same translation with nftables maps instead of a long linear list of rules, which scales much better with thousands of Services. CNI plugins like Cilium can replace kube-proxy entirely and do the translation in eBPF, often at the socket level, so the packet is addressed to the pod from the start and no per-packet NAT happens.[^cilium-kpr] The concepts stay the same: a virtual IP gets translated to a pod IP.

### 2. Masquerading for NodePorts and external traffic

DNAT alone is not always enough. Consider a NodePort Service: a client at `198.51.100.50` connects to node A on port `30080`, and kube-proxy DNATs the connection to a pod that happens to run on node B.

```text
client 198.51.100.50 --> node A:30080 --DNAT--> pod 10.244.2.7 (on node B)
```

The pod replies to `198.51.100.50`, its default route sends the reply out through node B, and the client receives a packet from node B with the pod's IP as the source. The client expected a reply from node A, so it drops it. The connection never gets established, because the reverse translation would have had to happen on node A, where the conntrack entry lives.

The fix is to also SNAT the packet on node A, so the pod sees node A as the client and replies go back through it. That is what `KUBE-MARK-MASQ` is for. It does not masquerade anything itself; it sets a firewall mark (`0x4000`) on the packet. Later, in `POSTROUTING`, kube-proxy masquerades every marked packet:

```sh
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m mark ! --mark 0x4000/0x4000 -j RETURN
-A KUBE-POSTROUTING -j MARK --set-xmark 0x4000/0x0
-A KUBE-POSTROUTING -j MASQUERADE --random-fully
```

The mark is set early (during DNAT, when kube-proxy knows the packet came from outside the pod network) and acted on late (during SNAT, the only place SNAT is allowed). The `! -s 10.244.0.0/16` rule in the Service chain above does the same for ClusterIP traffic that originates outside the pod CIDR, such as from the node itself.

The side effect is well known: **the pod loses the client's real IP address.** Your access logs show node A's IP instead of `198.51.100.50`.[^k8s-source-ip] If you need the real IP, set `externalTrafficPolicy: Local` on the NodePort or LoadBalancer Service. Then kube-proxy only forwards to pods on the node that received the traffic and skips the masquerade. The cost is that nodes without a local pod drop the traffic (load balancers detect this through a health check port) and load can become uneven across pods. For HTTP traffic, the other option is to let an ingress or load balancer add an `X-Forwarded-For` header or use the PROXY protocol.

### 3. Hairpin traffic

A pod that connects to a Service whose chosen backend turns out to be *itself* creates a special case. The DNAT rewrites the destination to the pod's own IP, so the packet leaves the pod and is sent straight back into it. The reply would then go from the pod directly to itself, skipping the conntrack entry, and the reverse translation never happens.

That is why `KUBE-SEP-1` above has the rule `-s 10.244.1.5/32 -j KUBE-MARK-MASQ`: if the source is the endpoint itself, masquerade it so the reply has to go back through the node. On bridge-based CNIs, the bridge port also needs **hairpin mode**, which lets a packet go out of the same bridge port it came in on. If you have ever seen a pod that can reach every Service except the one it backs, a missing hairpin setting is the first suspect.

### 4. Pod egress to the outside world

Pod IPs come from the cluster's pod CIDR, usually a private range that the network outside the cluster knows nothing about. When a pod connects to something outside the cluster, such as a database in another VPC or a public API, the reply would come back to an address nobody can route. So traffic leaving the cluster is masqueraded to the node's IP.

Who does this depends on your setup. Many CNI plugins install their own masquerade rule for traffic from the pod CIDR to anything outside the cluster. Cloud providers often run the **ip-masq-agent**, which takes a list of `nonMasqueradeCIDRs` and masquerades everything else.[^ip-masq-agent] Some cloud CNIs give pods real VPC addresses, in which case traffic inside the VPC needs no NAT at all and only internet-bound traffic is translated, often by a NAT gateway outside the cluster.

This has a few practical consequences:

- **Firewalls outside the cluster see node IPs, not pod IPs.** If you want to allow "just this one workload" on an external database, IP-based rules do not work; every pod on that node shares the same source address. Egress gateways (available in several CNIs and service meshes) solve this by routing selected traffic through a dedicated node with a fixed IP.
- **Port exhaustion moves to the node.** All pods on a node share the node's ports for outbound connections to the same destination. A node running many pods that all call the same external endpoint can run out, and so can a cloud NAT gateway in front of a large cluster.
- **Getting the non-masquerade list wrong breaks things quietly.** If traffic to a peered network gets masqueraded when it shouldn't, the remote side sees unexpected source IPs. If traffic that should be masqueraded isn't, replies are lost.

### Conntrack trouble in Kubernetes

Because every Service connection goes through conntrack, Kubernetes nodes stress it more than most machines. Three problems show up repeatedly:

- **Full conntrack table.** A node with many short-lived connections fills the table and starts dropping packets. kube-proxy sets `nf_conntrack_max` based on the number of CPUs (configurable via its `conntrack` settings), but nodes running proxies, DNS resolvers or scrapers can need more. Watch `nf_conntrack_count` against `nf_conntrack_max` in your node metrics.
- **The DNS five-second delay.** glibc resolvers send the `A` and `AAAA` queries for a name in parallel, from the same socket, to the CoreDNS ClusterIP. Both UDP packets need a new conntrack entry with the same 5-tuple, and on older kernels one could lose the insert race and be dropped. The resolver then waits for its 5-second timeout and retries. Symptoms are occasional DNS lookups that take almost exactly five seconds; `conntrack -S` shows `insert_failed` going up. Kernel fixes reduced this a lot, and the common mitigations are NodeLocal DNSCache (which avoids conntrack for DNS from pods) or the `single-request-reopen` resolver option.
- **Stale UDP entries.** When a Service's backend pod is replaced, existing UDP conntrack entries still point to the old pod IP and traffic keeps going to a pod that no longer exists. kube-proxy deletes such entries when endpoints change, which is why it needs the `conntrack` binary on older versions. If you write your own UDP NAT rules, you have to do the same.

## Debugging checklist

When traffic through NAT misbehaves, on a Linux box or a Kubernetes node, I go through roughly this list:

1. **Is forwarding on?** `sysctl net.ipv4.ip_forward` must be `1` on anything that routes.
2. **What do the rules say?** `sudo iptables -t nat -S` and `sudo nft list ruleset`. On modern distributions `iptables` is often a frontend to nftables (`iptables -V` shows `nf_tables`), and rules from both tools end up in the same kernel.
3. **Do the rules match?** `sudo iptables -t nat -L -v -n` shows packet counters per rule. A counter that does not move means the rule never matches.
4. **What did conntrack decide?** `sudo conntrack -L` with filters for the address or port shows the actual translation. Remember that existing connections keep their old translation after a rule change; delete the entry or open a new connection.
5. **Is the return path symmetric?** NAT only works if replies pass through the machine that holds the conntrack entry. Asymmetric routing is the most common reason for "the SYN goes out but the connection never completes".
6. **Is conntrack healthy?** `conntrack -C` vs. `nf_conntrack_max`, and `conntrack -S` for `insert_failed` and `drop`.
7. **Capture on both sides.** `tcpdump -ni any host 10.244.1.5` on the node shows the packet before and after translation, since it appears on the pod's veth and on the uplink with different addresses.

## Wrapping up

NAT is a small idea, rewriting addresses in packet headers and remembering how you did it, used in a lot of places. The home router uses it to share one address. Linux implements it with a few netfilter hooks and the conntrack table, where the first packet decides and every later one follows. Kubernetes promises pod-to-pod traffic without NAT, and then uses DNAT for every Service, masquerading for NodePorts, hairpins and egress, and relies on conntrack for all of it.

Once you know where the rewrites happen, the strange symptoms make sense. The wrong client IP is masquerading. Requests that stick to one pod are conntrack. The five-second DNS delay is a conntrack race. The pod that cannot reach its own Service is a hairpin. And the reply that never arrives is almost always a path that skipped the machine holding the conntrack entry.

[^rfc1918]: [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918), IETF.
[^rfc3022]: [RFC 3022: Traditional IP Network Address Translator (Traditional NAT)](https://www.rfc-editor.org/rfc/rfc3022), IETF.
[^rfc4787]: [RFC 4787: Network Address Translation (NAT) Behavioral Requirements for Unicast UDP](https://www.rfc-editor.org/rfc/rfc4787), IETF.
[^stun]: [RFC 8489: Session Traversal Utilities for NAT (STUN)](https://www.rfc-editor.org/rfc/rfc8489), IETF.
[^rfc6598]: [RFC 6598: IANA-Reserved IPv4 Prefix for Shared Address Space](https://www.rfc-editor.org/rfc/rfc6598), IETF.
[^rfc6146]: [RFC 6146: Stateful NAT64](https://www.rfc-editor.org/rfc/rfc6146), IETF.
[^conntrack]: [conntrack(8)](https://man7.org/linux/man-pages/man8/conntrack.8.html), Linux manual page.
[^iptables-ext]: [iptables-extensions(8)](https://man7.org/linux/man-pages/man8/iptables-extensions.8.html), Linux manual page (see the `MASQUERADE`, `SNAT` and `DNAT` targets).
[^nft-nat]: [Performing Network Address Translation (NAT)](https://wiki.nftables.org/wiki-nftables/index.php/Performing_Network_Address_Translation_(NAT)), nftables wiki.
[^k8s-model]: [Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/), Kubernetes documentation.
[^k8s-vip]: [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/), Kubernetes documentation.
[^cilium-kpr]: [Kubernetes Without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/), Cilium documentation.
[^k8s-source-ip]: [Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/), Kubernetes documentation.
[^ip-masq-agent]: [IP Masquerade Agent User Guide](https://kubernetes.io/docs/tasks/administer-cluster/ip-masq-agent/), Kubernetes documentation.
