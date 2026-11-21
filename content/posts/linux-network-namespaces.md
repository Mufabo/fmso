---
title: "Linux Network Namespaces: Building Container Networking by Hand"
date: 2026-11-18
draft: true
description: "A hands-on introduction to Linux network namespaces: what they isolate, how to create them with ip netns, connecting them with veth pairs and a bridge, giving them internet access with NAT, and how Docker and Kubernetes use the same building blocks."
summary: "Every container gets its own network stack, and that stack is just a Linux network namespace. Here is what a network namespace isolates, how to build one by hand with veth pairs, a bridge and NAT, and how to use the same knowledge to debug Docker and Kubernetes networking."
tags: ["linux", "networking", "namespaces", "containers", "docker", "kubernetes"]
---

When you run `docker run nginx`, the container gets its own `eth0`, its own IP address, its own routing table and its own `localhost`. It can listen on port 80 even though another container on the same machine is also listening on port 80. None of this is magic, and none of it is Docker. It is a Linux kernel feature called a **network namespace**, and you can build the whole thing yourself with a handful of `ip` commands.

This post does exactly that. We start with an empty namespace, connect two of them with a virtual cable, put them on a virtual switch, give them internet access, and then look at how Docker and Kubernetes use the same pieces. Once you have built it by hand, container networking stops being a black box, and debugging it becomes a lot less guesswork.

You need a Linux machine (a VM is fine), root access, and the `iproute2` package, which every modern distribution ships.

## What a network namespace is

Linux namespaces wrap a global system resource so that processes inside the namespace see their own isolated copy of it.[^namespaces] There are several kinds: mount, PID, user, UTS, IPC, cgroup, time, and network. Containers are mostly a combination of all of them.

A network namespace gives its processes a private copy of the network stack.[^netns] That includes:

| Isolated per namespace | What it means in practice |
| --- | --- |
| **Network interfaces** | Each namespace has its own set of devices, including its own `lo`. |
| **IP addresses** | The same address can exist in two namespaces without conflict. |
| **Routing tables** | Each namespace has its own default route (or none). |
| **Firewall rules** | `iptables` / `nftables` rules apply only inside their namespace. |
| **Sockets and ports** | Two namespaces can both bind port 80. |
| **`/proc/net`, `/sys/class/net`** | Show only the namespace's own view. |
| **`net.*` sysctls** | e.g. `net.ipv4.ip_forward` can differ per namespace. |

Every process belongs to exactly one network namespace. When the system boots, everything lives in the **initial** (or "root", or "host") namespace. A network device, physical or virtual, also belongs to exactly one namespace at a time, but it can be moved between them.

You can see which namespace a process is in by looking at `/proc`:

```sh
$ readlink /proc/$$/ns/net
net:[4026531840]
```

That number is the namespace's inode. Two processes with the same number share a network stack. `lsns -t net` lists every network namespace on the system along with a process that is using it.

## Creating a namespace

The easiest tool for working with network namespaces is `ip netns`.[^ipnetns] Let's create two namespaces called `red` and `blue`:

```sh
sudo ip netns add red
sudo ip netns add blue
ip netns list
```

Now look inside one:

```sh
$ sudo ip netns exec red ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

That is the entire network of `red`: a single loopback interface, and it isn't even up. No `eth0`, no routes, no way out. `ip netns exec red <command>` runs any command inside the namespace, so you can just as well start a shell there with `sudo ip netns exec red bash` and poke around. `ip route`, `ss -tlnp` and `iptables -L` will all come back empty.

A useful shorthand: `ip -n red <...>` is the same as `ip netns exec red ip <...>`. I'll use it from here on.

Bring up loopback in both namespaces first, because a lot of software silently assumes `127.0.0.1` works:

```sh
sudo ip -n red link set lo up
sudo ip -n blue link set lo up
```

### What `ip netns add` actually does

A namespace normally lives only as long as some process is inside it. When the last process exits, the kernel destroys it. So how does `red` survive with no process in it?

`ip netns add red` creates an empty file at `/run/netns/red` and bind-mounts the new namespace's `/proc/<pid>/ns/net` handle onto it. As long as that bind mount exists, the kernel keeps the namespace alive. `ip netns exec` then opens that file and calls `setns()` to move the command into the namespace. This detail matters later: namespaces created by Docker or containerd are not always registered under `/run/netns`, so `ip netns list` won't show them.

`ip netns exec` does one more convenient thing: if `/etc/netns/red/resolv.conf` exists, it is bind-mounted over `/etc/resolv.conf` for the command. That is how you give a namespace its own DNS settings.

## Connecting two namespaces with a veth pair

To get traffic in or out of a namespace, you need an interface that crosses the boundary. The standard tool is a **veth pair**: two virtual Ethernet devices joined back to back, like the two ends of a cable. Whatever goes into one end comes out of the other.[^veth]

Create a pair and put one end in each namespace:

```sh
sudo ip link add veth-red type veth peer name veth-blue
sudo ip link set veth-red netns red
sudo ip link set veth-blue netns blue
```

Give each end an address on the same subnet and bring them up:

```sh
sudo ip -n red  addr add 10.0.0.1/24 dev veth-red
sudo ip -n blue addr add 10.0.0.2/24 dev veth-blue
sudo ip -n red  link set veth-red up
sudo ip -n blue link set veth-blue up
```

And test:

```sh
$ sudo ip netns exec red ping -c 2 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.051 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.043 ms
```

Two isolated network stacks, talking over a virtual cable. Notice that the host itself can't see either interface any more (`ip link` on the host no longer lists them), and it can't reach `10.0.0.1`: there is no route to that subnet in the host namespace.

To prove the port isolation, start a listener on the same port in both namespaces and on the host:

```sh
sudo ip netns exec red  python3 -m http.server 8000 &
sudo ip netns exec blue python3 -m http.server 8000 &
python3 -m http.server 8000 &
```

All three start without an "address already in use" error, because each one binds port 8000 in a different network stack. Stop them before moving on (`kill %1 %2 %3`).

## More than two: adding a bridge

A veth pair only connects two endpoints. With ten containers you don't want a mesh of cables; you want a switch. Linux has one built in: the **bridge**. You plug one end of each veth pair into the bridge, put the other end in a namespace, and the bridge forwards Ethernet frames between them.

This is exactly the layout Docker uses with its default `docker0` bridge:

```text
        host namespace
  ┌──────────────────────────────────────────┐
  │              br0  10.0.1.1/24            │
  │             ┌──────┴───────┐             │
  │          br-red         br-blue          │
  └─────────────┼──────────────┼─────────────┘
                │ veth         │ veth
  ┌─────────────┼───┐  ┌───────┼─────────────┐
  │   veth-red      │  │      veth-blue      │
  │   10.0.1.2      │  │      10.0.1.3       │
  │       red       │  │        blue         │
  └─────────────────┘  └─────────────────────┘
```

First remove the direct cable from the previous section. Deleting one end of a veth pair deletes both:

```sh
sudo ip -n red link del veth-red
```

Create the bridge in the host namespace and give it an address. That address will be the namespaces' gateway:

```sh
sudo ip link add br0 type bridge
sudo ip addr add 10.0.1.1/24 dev br0
sudo ip link set br0 up
```

Now wire up each namespace. The pattern is the same for both, so here it is as a loop:

```sh
i=2
for ns in red blue; do
  sudo ip link add veth-$ns type veth peer name br-$ns
  sudo ip link set veth-$ns netns $ns
  sudo ip link set br-$ns master br0
  sudo ip link set br-$ns up
  sudo ip -n $ns addr add 10.0.1.$i/24 dev veth-$ns
  sudo ip -n $ns link set veth-$ns up
  sudo ip -n $ns route add default via 10.0.1.1
  i=$((i + 1))
done
```

The `br-*` ends stay in the host namespace and are attached to the bridge with `master br0`; they don't need IP addresses. The `veth-*` ends go into the namespaces and get the addresses. Interface names are limited to 15 characters, which is why real tools use short generated names like `vethb3f9a1c`.

Now `red` and `blue` can reach each other through the bridge, and, because the bridge has an address, the host can reach them too:

```sh
sudo ip netns exec red ping -c 1 10.0.1.3   # red → blue
ping -c 1 10.0.1.2                          # host → red
```

If the namespace-to-namespace ping fails but host-to-namespace works, you are probably on a machine with Docker installed. Docker sets the `iptables` `FORWARD` chain policy to `DROP`, and if `br_netfilter` is loaded, bridged traffic passes through that chain. Allow it explicitly:

```sh
sudo iptables -A FORWARD -i br0 -j ACCEPT
sudo iptables -A FORWARD -o br0 -j ACCEPT
```

## Reaching the outside world

The namespaces have a default route pointing at the host, but they still can't reach the internet. Two things are missing.

First, the host has to be willing to forward packets between interfaces, which Linux doesn't do by default:

```sh
sudo sysctl -w net.ipv4.ip_forward=1
```

Second, `10.0.1.0/24` is a private range that nobody on the internet knows how to route back to. The host has to rewrite the source address of outgoing packets to its own address, and undo it for the replies. That is source NAT, or masquerading:

```sh
sudo iptables -t nat -A POSTROUTING -s 10.0.1.0/24 ! -o br0 -j MASQUERADE
```

The `! -o br0` keeps traffic between the namespaces themselves from being NATed. Now:

```sh
$ sudo ip netns exec red ping -c 1 1.1.1.1
64 bytes from 1.1.1.1: icmp_seq=1 ttl=57 time=9.84 ms
```

For name resolution, give the namespace its own resolver config, which `ip netns exec` will mount automatically:

```sh
sudo mkdir -p /etc/netns/red
echo "nameserver 1.1.1.1" | sudo tee /etc/netns/red/resolv.conf
sudo ip netns exec red curl -sI https://example.com | head -1
```

Going the other way, publishing a service running inside a namespace on a host port, is destination NAT. This is roughly what `docker run -p 8080:80` sets up:

```sh
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.1.2:80
```

That rule catches traffic arriving from other machines. Connections made from the host itself to `localhost:8080` don't pass through `PREROUTING`, which is why Docker also adds rules in the `OUTPUT` chain or runs its `docker-proxy` process.

At this point you have rebuilt Docker's default bridge network: a bridge with a gateway address, a veth pair per container, a default route, masquerading for outbound traffic and DNAT for published ports.

## How containers use this

### Docker

Every container (unless started with `--network host`) gets a fresh network namespace, a veth pair with one end called `eth0` inside the container, and the other end attached to `docker0`. That `eth0` is not a real network card: it is just the container's end of the veth pair, renamed to `eth0` after being moved into the namespace, because a lot of software expects an interface by that name. Look at the host while a container is running and you'll see the host end:

```sh
$ ip link show master docker0
7: veth3c1e2f4@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP
```

The `@if6` suffix tells you the peer is interface index 6 in another namespace, which will be the container's `eth0`.

Docker does not register its namespaces under `/run/netns`, so `ip netns list` won't show them. Instead, enter the namespace through the container's process with `nsenter`:[^nsenter]

```sh
pid=$(docker inspect -f '{{.State.Pid}}' my-container)
sudo nsenter -t "$pid" -n ip addr
sudo nsenter -t "$pid" -n ss -tlnp
```

`-n` means "enter only the network namespace". The filesystem, and therefore the binaries, are still the host's. This is the single most useful trick in this post: you can run the host's `tcpdump`, `ss`, `dig` or `curl` inside a container's network stack, even when the container image is a minimal distroless image without a shell.

```sh
sudo nsenter -t "$pid" -n tcpdump -i eth0 -nn port 53
```

### Kubernetes

A Kubernetes Pod is a group of containers that share **one** network namespace. When the kubelet starts a Pod, the container runtime first creates a small "sandbox" (traditionally the `pause` container) whose only job is to hold the namespace open. The application containers are then started inside that same namespace.

That explains several Kubernetes behaviours that otherwise seem arbitrary:

- Containers in the same Pod reach each other on `localhost`, because they share one loopback interface.
- Two containers in the same Pod can't both listen on the same port, because they share one set of sockets.
- A Pod has a single IP address, no matter how many containers it has.
- A container restarting inside a Pod keeps the Pod's IP, because the namespace (held by the sandbox) survives.

Setting up the namespace's interfaces is delegated to the CNI plugin. The runtime calls the plugin with the path of the new namespace, and the plugin does more or less what we did by hand: create a veth pair, move one end into the namespace, assign an IP, add routes.[^cni] Plugins differ mostly in what they do with the host end: attach it to a bridge, route to it directly, or hand it to eBPF programs.

With containerd, pod namespaces usually *are* registered under `/run/netns` with names like `cni-1a2b3c4d-...`, so on a node you can often do:

```sh
sudo ip netns list
sudo ip netns exec cni-1a2b3c4d-... ip addr
```

If not, find a process of the Pod with `crictl inspect` or `lsns -t net` and use `nsenter -t <pid> -n` as with Docker.

## Quick experiments with `unshare`

You don't always need named namespaces. `unshare --net` starts a command in a brand-new network namespace that disappears when the command exits:

```sh
sudo unshare --net bash
ip link    # only lo, down
```

This is a quick way to test how a program behaves with no network at all. On most distributions you can do it without root by combining it with a user namespace: `unshare -rn bash`.[^unshare]

## Cleaning up

Deleting a namespace destroys virtual devices inside it, including veth ends, which also removes their peers in the host namespace. Physical devices that were moved into a namespace are returned to the initial namespace instead.[^netns]

```sh
sudo ip netns del red
sudo ip netns del blue
sudo ip link del br0
sudo iptables -t nat -D POSTROUTING -s 10.0.1.0/24 ! -o br0 -j MASQUERADE
sudo iptables -t nat -D PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.1.2:80
sudo rm -rf /etc/netns/red
```

If you added the `FORWARD` rules, remove them with the same commands using `-D` instead of `-A`.

## Wrapping up

A network namespace is a full, private copy of the Linux network stack. On its own it is useless, a box with only a loopback interface. What makes it useful is the plumbing you add around it:

- **veth pairs** to get packets across the namespace boundary,
- a **bridge** to connect many namespaces together,
- **routes** so the namespace knows where to send traffic,
- **IP forwarding and NAT** on the host to reach the outside world and to publish ports.

Docker, Podman, Kubernetes CNI plugins and most other container tools assemble exactly these pieces, with more automation and better names. The next time a container can't reach something, you can step into its namespace with `nsenter` and check the same things you just built: is the interface up, does it have an address, is there a default route, and is the host forwarding and NATing the traffic. For a deeper dive into the kernel side, the LWN series on namespaces is still the best reference.[^lwn]

[^namespaces]: [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html), Linux manual page.
[^netns]: [network_namespaces(7)](https://man7.org/linux/man-pages/man7/network_namespaces.7.html), Linux manual page.
[^ipnetns]: [ip-netns(8)](https://man7.org/linux/man-pages/man8/ip-netns.8.html), Linux manual page.
[^veth]: [veth(4)](https://man7.org/linux/man-pages/man4/veth.4.html), Linux manual page.
[^nsenter]: [nsenter(1)](https://man7.org/linux/man-pages/man1/nsenter.1.html), Linux manual page.
[^cni]: [Container Network Interface (CNI) Specification](https://www.cni.dev/docs/spec/), CNI project.
[^unshare]: [unshare(1)](https://man7.org/linux/man-pages/man1/unshare.1.html), Linux manual page.
[^lwn]: [Namespaces in operation, part 7: Network namespaces](https://lwn.net/Articles/580893/), Jake Edge, LWN.net.
