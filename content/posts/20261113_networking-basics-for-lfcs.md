---
title: "Networking Basics for the LFCS: Routing, Switching, and the Linux Commands That Go With Them"
date: 2026-11-13
draft: true
tags: ["networking", "linux", "lfcs", "routing", "switching", "certification", "namespaces"]
categories: ["networking"]
summary: "The networking theory you actually need for the LFCS exam — addressing, subnetting, switching, routing, NAT — mapped straight onto the Linux tools you'll use to configure and troubleshoot it."
---

Networking is a quarter of the LFCS exam. The current blueprint splits the exam into five domains: Operations Deployment (25%), **Networking (25%)**, Storage (20%), Essential Commands (20%), and Users and Groups (10%).[^1] The exam is two hours of hands-on tasks in a terminal. Nobody asks you to define the OSI model. They ask you to make a box route traffic, or bond two NICs, or figure out why a host can't reach the internet.

The official Networking competencies are:[^1]

- Configure IPv4 and IPv6 networking and hostname resolution
- Set and synchronize system time using time servers
- Monitor and troubleshoot networking
- Configure the OpenSSH server and client
- Configure packet filtering, port redirection, and NAT
- Configure static routing
- Configure bridge and bonding devices
- Implement reverse proxies and load balancers

This post covers the theory under all of that, focused on routing and switching, and then the Linux commands for each piece. If you understand *why* a packet goes where it goes, the commands become much easier to remember.

## Part 1: The mental model

### Layers, briefly

You only need two layers of the stack to reason about most LFCS tasks:

| Layer | Address | Device that cares | Linux concept |
|---|---|---|---|
| **Layer 2 — Data link** | MAC address (`52:54:00:ab:cd:ef`) | Switch | Bridge, bond, `ip link`, `ip neigh` |
| **Layer 3 — Network** | IP address (`192.168.1.10/24`) | Router | `ip addr`, `ip route`, forwarding, NAT |

Above that sits Layer 4 (TCP/UDP ports — what `ss` and firewall rules match on) and the applications (SSH, nginx, DNS).

The key rule: **switches move frames within a network using MAC addresses; routers move packets between networks using IP addresses.**

### Switching: how a Layer 2 network works

A switch connects hosts in the same **broadcast domain**. It learns which MAC address lives behind which port by watching the source address of incoming frames, and builds a MAC address table. When a frame arrives for a known MAC, it goes out one port. When the destination is unknown, or is the broadcast address `ff:ff:ff:ff:ff:ff`, the switch floods it out every port.

But hosts send to *IP* addresses, so how does a host find the MAC of a neighbour? With **ARP** (Address Resolution Protocol).[^2] The host broadcasts "who has 192.168.1.20?" and the owner replies with its MAC. The answer is cached in the neighbour table:

```bash
ip neigh show
# 192.168.1.1 dev eth0 lladdr 52:54:00:12:34:56 REACHABLE
```

IPv6 does the same job with Neighbor Discovery instead of ARP, and the same `ip neigh` command shows the results.

On Linux, a **bridge** is a software switch. It's what you use to connect VMs or containers to a physical network, and it's an exam topic in its own right (Part 5).

A note on **VLANs**: a VLAN splits one physical switch into several separate broadcast domains by tagging frames with a VLAN ID (802.1Q). Linux can create tagged sub-interfaces (`ip link add link eth0 name eth0.10 type vlan id 10`). It's worth knowing what they are, even though they aren't listed in the LFCS competencies.

### IP addressing and subnetting

An IPv4 address is 32 bits, written as four octets. The **prefix length** (CIDR notation, `/24`) says how many leading bits identify the network; the rest identify the host.[^3]

Worked example — `192.168.10.77/26`:

- `/26` → netmask `255.255.255.192`
- 32 − 26 = 6 host bits → 2⁶ = 64 addresses per subnet
- Subnets start at multiples of 64 in the last octet: .0, .64, .128, .192
- 77 falls in the .64 block, so:
  - **Network:** `192.168.10.64`
  - **Broadcast:** `192.168.10.127`
  - **Usable hosts:** `.65` – `.126` (62 hosts)

Being able to do this in your head is enough. If you blank, `ipcalc` (if installed) or Python's `ipaddress` module will do it for you:

```bash
python3 -c "import ipaddress; n=ipaddress.ip_network('192.168.10.77/26', strict=False); print(n, n.broadcast_address, n.num_addresses-2)"
```

Ranges to recognise:

- **Private IPv4:** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`[^4]
- **Loopback:** `127.0.0.0/8`, IPv6 `::1`
- **IPv6 link-local:** `fe80::/10` — every IPv6 interface has one automatically
- **IPv6 global unicast:** typically a `/64` per network segment

Two hosts can talk directly (via ARP and the switch) only if they're in the same subnet. Everything else needs a router.

### Routing: how a packet finds its way

Every Linux host has a **routing table**, even if it's not a router. When sending a packet, the kernel looks up the destination and picks the route with the **longest matching prefix**. A `/24` route beats a `/16`, which beats the default route `0.0.0.0/0`.

```bash
ip route show
# default via 192.168.1.1 dev eth0 proto dhcp metric 100
# 10.20.0.0/16 via 192.168.1.254 dev eth0
# 192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
```

Reading that table:

- `192.168.1.0/24 dev eth0` — a **connected** route. The kernel added it when the address was assigned. Destinations here are reached directly (ARP for them).
- `10.20.0.0/16 via 192.168.1.254` — a **static** route. Send to that gateway, which knows what to do next.
- `default via 192.168.1.1` — the **default gateway**. Anything not matched above goes here.

The most useful troubleshooting command on this topic asks the kernel which route it would pick:

```bash
ip route get 10.20.5.9
# 10.20.5.9 via 192.168.1.254 dev eth0 src 192.168.1.10
```

When multiple routes have the same prefix, the lower **metric** wins.

A host becomes a **router** when it forwards packets that aren't addressed to it. Linux does not do this by default — you have to turn on IP forwarding (Part 3). Also remember routing is two-way: the return traffic needs a route back. "I can ping out but get no reply" is very often a missing return route, or missing NAT.

### NAT

**Network Address Translation** rewrites addresses as packets pass through a router.

- **SNAT / masquerade:** private hosts going out to the internet get the router's public address as their source. Replies are translated back automatically by connection tracking.
- **DNAT / port forwarding:** traffic arriving at the router on a port gets redirected to an internal host and port.
- **Redirect:** a special DNAT to the local machine itself, e.g. port 8080 → port 80 on the same box.

That's all the theory. Now the commands.

## Part 2: Configuring addresses and hostname resolution

### Temporary configuration with `ip`

The `ip` command (iproute2) changes the running kernel state only. Everything disappears on reboot — useful for testing, wrong for an exam task that says "persistently".

```bash
ip -br addr                          # brief overview of all interfaces
ip link set eth1 up
ip addr add 192.168.50.10/24 dev eth1
ip addr add 2001:db8:50::10/64 dev eth1
ip addr del 192.168.50.10/24 dev eth1
```

### Persistent configuration

Which tool persists config depends on the distribution, and since the exam is no longer tied to a specific one,[^1] you should be comfortable with both of the common options.

**NetworkManager (`nmcli`)** — Rocky/RHEL/Fedora, and many desktops:[^5]

```bash
nmcli device status
nmcli connection show

nmcli connection modify eth1 \
  ipv4.method manual \
  ipv4.addresses 192.168.50.10/24 \
  ipv4.gateway 192.168.50.1 \
  ipv4.dns "1.1.1.1 9.9.9.9" \
  ipv6.method manual \
  ipv6.addresses 2001:db8:50::10/64

nmcli connection up eth1
```

Switch back to DHCP with `ipv4.method auto`. Note that `nmcli con mod` only changes the saved profile; it doesn't take effect until you `nmcli con up` it again.

**Netplan** — Ubuntu Server:[^6]

```yaml
# /etc/netplan/60-eth1.yaml
network:
  version: 2
  ethernets:
    eth1:
      dhcp4: false
      addresses:
        - 192.168.50.10/24
        - 2001:db8:50::10/64
      routes:
        - to: default
          via: 192.168.50.1
      nameservers:
        addresses: [1.1.1.1, 9.9.9.9]
```

```bash
sudo chmod 600 /etc/netplan/60-eth1.yaml
sudo netplan try      # applies, rolls back after 120 s unless you confirm
sudo netplan apply
```

`netplan try` is your friend if you're working over SSH on the interface you're changing.

### Hostname and name resolution

```bash
hostnamectl set-hostname web01.example.lan
hostnamectl
```

Name lookups follow the order in `/etc/nsswitch.conf` — typically `hosts: files dns`, meaning `/etc/hosts` is checked before DNS. A static entry is just:

```text
# /etc/hosts
192.168.50.20   db01.example.lan db01
```

On systems running `systemd-resolved`, `/etc/resolv.conf` points at a local stub (`127.0.0.53`). Check the real upstream servers and test resolution with:

```bash
resolvectl status
resolvectl query db01.example.lan
getent hosts db01        # uses nsswitch, so it honours /etc/hosts too
dig example.com          # DNS only, ignores /etc/hosts
```

The difference between `getent` and `dig` is a classic troubleshooting clue: if `getent` works but `dig` doesn't, the name is coming from `/etc/hosts`.

## Part 3: Static routing and forwarding

### Adding routes

Temporary:

```bash
ip route add 10.20.0.0/16 via 192.168.1.254 dev eth0
ip route add default via 192.168.1.1
ip route del 10.20.0.0/16
ip -6 route add 2001:db8:99::/48 via 2001:db8:50::1
```

Persistent with NetworkManager — note the `+` to append rather than replace:

```bash
nmcli connection modify eth0 +ipv4.routes "10.20.0.0/16 192.168.1.254"
nmcli connection up eth0
```

Persistent with Netplan, under the interface:

```yaml
      routes:
        - to: 10.20.0.0/16
          via: 192.168.1.254
```

The full syntax, including metrics and route types such as `blackhole`, is in `man ip-route`.[^7]

### Turning a Linux box into a router

Forwarding is controlled by sysctls:[^8]

```bash
# Temporary
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv6.conf.all.forwarding=1

# Persistent
cat <<'EOF' | sudo tee /etc/sysctl.d/90-forwarding.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sudo sysctl --system
```

A typical exam-style scenario: a host with two interfaces (`eth0` on the internet side, `eth1` on `10.0.0.0/24`) must let the internal network reach the outside. That needs three things:

1. IP forwarding enabled
2. Masquerade (SNAT) on the outbound interface
3. Internal hosts using this box as their default gateway

Step 2 is packet filtering and NAT.

## Part 4: Packet filtering, port redirection, and NAT

Modern distributions use **nftables** as the kernel packet filter. `iptables` commands still exist on most systems, often as a compatibility layer over nftables, and firewalld (RHEL family) and ufw (Ubuntu) are front-ends on top. Know at least one well and be able to read the others.

### NAT with nftables

```bash
# /etc/nftables.conf (or a file it includes)
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        # Port redirection on this host: 8080 -> 80
        tcp dport 8080 redirect to :80

        # Port forwarding: external 2222 -> internal host's SSH
        iifname "eth0" tcp dport 2222 dnat to 10.0.0.5:22
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        # Masquerade the internal network going out eth0
        oifname "eth0" ip saddr 10.0.0.0/24 masquerade
    }
}
```

```bash
sudo nft -f /etc/nftables.conf
sudo nft list ruleset
sudo systemctl enable --now nftables     # load on boot
```

The nftables wiki has a good page covering all the NAT variants.[^9]

### The same thing with iptables

```bash
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-ports 80
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 2222 -j DNAT --to-destination 10.0.0.5:22
iptables -t nat -A POSTROUTING -o eth0 -s 10.0.0.0/24 -j MASQUERADE
iptables -t nat -L -n -v
```

Persist them with `iptables-save > /etc/iptables/rules.v4` (Debian/Ubuntu, `iptables-persistent`) or the distribution's equivalent.

### A basic filter

A minimal "allow SSH and web, drop everything else inbound" filter in nftables:

```text
table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        ct state established,related accept
        iif "lo" accept
        icmp type echo-request accept
        icmpv6 type { echo-request, nd-neighbor-solicit, nd-neighbor-advert, nd-router-advert } accept
        tcp dport { 22, 80, 443 } accept
    }
    chain forward {
        type filter hook forward priority filter; policy accept;
    }
}
```

The `inet` family covers IPv4 and IPv6 in one table. Don't forget the ICMPv6 neighbour discovery lines: dropping them breaks IPv6 entirely, the same way blocking ARP would break IPv4.

With **firewalld**, the same ideas are expressed as zones and services:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-forward-port=port=8080:proto=tcp:toport=80
firewall-cmd --permanent --add-masquerade
firewall-cmd --reload
```

## Part 5: Bridges and bonds

These are the two Layer 2 constructs on the exam, and people often mix them up:

- A **bridge** connects *several* interfaces into *one* switched network. Use it to put VMs or containers on the same LAN as the host.
- A **bond** combines *several* physical links into *one* logical link to the *same* network, for redundancy or extra bandwidth.

### Bridge

Temporary, with `ip`:

```bash
ip link add br0 type bridge
ip link set eth1 master br0
ip link set eth2 master br0
ip link set br0 up
ip addr add 192.168.60.1/24 dev br0     # the IP goes on the bridge, not the ports
bridge link show
```

Persistent, with NetworkManager:

```bash
nmcli connection add type bridge ifname br0 con-name br0 \
  ipv4.method manual ipv4.addresses 192.168.60.1/24
nmcli connection add type bridge-slave ifname eth1 master br0
nmcli connection add type bridge-slave ifname eth2 master br0
nmcli connection up br0
```

Persistent, with Netplan:

```yaml
network:
  version: 2
  ethernets:
    eth1: {}
    eth2: {}
  bridges:
    br0:
      interfaces: [eth1, eth2]
      addresses: [192.168.60.1/24]
```

### Bond

The kernel bonding driver supports several modes. The ones worth remembering:[^10]

| Mode | Name | What it does | Needs switch support? |
|---|---|---|---|
| 0 | `balance-rr` | Round-robin across links | Usually |
| 1 | `active-backup` | One link active, others standby | No |
| 2 | `balance-xor` | Hash-based link selection | Yes (static) |
| 4 | `802.3ad` | LACP link aggregation | Yes (LACP) |
| 6 | `balance-alb` | Adaptive load balancing | No |

If a task doesn't specify, `active-backup` is the safe choice because it works with any switch.

Temporary:

```bash
ip link add bond0 type bond mode active-backup miimon 100
ip link set eth1 down; ip link set eth1 master bond0
ip link set eth2 down; ip link set eth2 master bond0
ip link set bond0 up
ip addr add 192.168.70.10/24 dev bond0
cat /proc/net/bonding/bond0       # shows the mode and the active link
```

Persistent, with NetworkManager:

```bash
nmcli connection add type bond ifname bond0 con-name bond0 \
  bond.options "mode=active-backup,miimon=100" \
  ipv4.method manual ipv4.addresses 192.168.70.10/24
nmcli connection add type ethernet ifname eth1 master bond0
nmcli connection add type ethernet ifname eth2 master bond0
nmcli connection up bond0
```

Persistent, with Netplan:

```yaml
network:
  version: 2
  ethernets:
    eth1: {}
    eth2: {}
  bonds:
    bond0:
      interfaces: [eth1, eth2]
      addresses: [192.168.70.10/24]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
```

Test failover by downing the active link (`ip link set eth1 down`) and checking that `/proc/net/bonding/bond0` shows the other one taking over.

## Part 6: Monitoring and troubleshooting

When something doesn't work, go up the stack in order. Most problems show up in the first three steps.

```bash
# 1. Is the link up and does it have the right address?
ip -br link
ip -br addr

# 2. Can I reach my own subnet? Is ARP working?
ping -c 3 192.168.1.1
ip neigh

# 3. Which route will be used? Where does the path break?
ip route get 8.8.8.8
tracepath 8.8.8.8          # or traceroute / mtr

# 4. Does name resolution work?
resolvectl query example.com
getent hosts example.com

# 5. Is the service actually listening, and on which address?
ss -tulpn
ss -tn state established

# 6. Is a firewall dropping it?
nft list ruleset
firewall-cmd --list-all

# 7. What's really on the wire?
tcpdump -ni eth0 port 53
tcpdump -ni eth0 host 192.168.1.20 and icmp
```

A few patterns that come up all the time:

- **Pinging an IP works but pinging a name doesn't** → DNS. Check `resolvectl status` and `/etc/resolv.conf`.
- **A service listens on `127.0.0.1:80` instead of `0.0.0.0:80`** → it's only reachable locally. `ss -tlnp` shows this immediately.
- **Can reach the gateway but nothing beyond it** → default route missing or wrong, or the router isn't forwarding/NATing.
- **Works by IP from one side but not the other** → asymmetric routing or a missing return route.

Other useful tools: `ip -s link` (packet and error counters per interface), `ethtool eth0` (link speed and duplex), `nc -zv host 22` (quick TCP port check), and `journalctl -u NetworkManager` or `journalctl -u systemd-networkd` for config errors.

## Part 7: The rest of the domain, quickly

These aren't routing or switching, but they're in the same 25%, so here's the minimum.

### Time synchronisation

```bash
timedatectl                          # current status
timedatectl set-timezone Europe/Istanbul
timedatectl set-ntp true
```

With chrony, add servers in `/etc/chrony.conf` or `/etc/chrony/chrony.conf`:

```text
server time.cloudflare.com iburst
pool pool.ntp.org iburst
```

```bash
systemctl restart chronyd     # "chrony" on Debian/Ubuntu
chronyc sources -v
chronyc tracking
```

The chrony documentation covers the rest of the config directives.[^11]

### OpenSSH

```bash
ssh-keygen -t ed25519
ssh-copy-id user@host
```

Common server hardening in `/etc/ssh/sshd_config` (or a drop-in under `/etc/ssh/sshd_config.d/`):

```text
PermitRootLogin no
PasswordAuthentication no
Port 2222
AllowUsers alice bob
```

Always run `sshd -t` to validate the config before `systemctl reload sshd`, or you can lock yourself out. On SELinux systems, a non-standard port also needs `semanage port -a -t ssh_port_t -p tcp 2222`.

Client-side shortcuts go in `~/.ssh/config`:

```text
Host db
    HostName 192.168.50.20
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```

### Reverse proxy and load balancer

nginx is the usual tool for this. A reverse proxy that load-balances across two backends:[^12]

```nginx
# /etc/nginx/conf.d/app.conf
upstream app_backend {
    server 10.0.0.11:8080;
    server 10.0.0.12:8080;
    # least_conn;   # optional: send to the backend with fewest connections
}

server {
    listen 80;
    server_name app.example.lan;

    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
nginx -t && systemctl reload nginx
```

The default balancing method is round-robin. On SELinux systems, nginx connecting to backends may also need `setsebool -P httpd_can_network_connect 1`.

## Part 8: Network namespaces — a whole lab on one machine

Network namespaces aren't on the LFCS competency list, but they're worth learning for two reasons. They're how containers get their networking: each Docker container and each Kubernetes pod lives in its own network namespace. And they let you practise almost every routing, NAT and firewall task in this post on a single machine, without spinning up VMs.

### What a network namespace is

A network namespace is a separate copy of the kernel's networking stack. Processes inside one see only that namespace's interfaces, IP addresses, routing table, neighbour table, firewall rules, sockets and `/proc/net`.[^13] Two namespaces can both bind port 80 without clashing, and a firewall rule in one has no effect on the other, yet they all share the same kernel. To the processes inside, a namespace looks like a separate machine.

A new namespace contains only a loopback interface, and that interface starts out down. To connect namespaces you use a **veth pair**: two virtual interfaces joined like the two ends of a cable, so a packet sent into one end comes out of the other.[^14] You put one end in each namespace, or one end in a namespace and the other on the host or in a bridge.

The `ip netns` command manages named namespaces, and `ip netns exec` runs any command inside one.[^15] All of these need root.

```bash
ip netns add red
ip netns list
ip netns exec red ip -br addr      # only lo, and it's DOWN
ip netns exec red bash             # a shell inside the namespace; `exit` to leave
ip netns del red
```

Named namespaces appear as files under `/var/run/netns`. `lsns -t net` lists every network namespace on the system, including the unnamed ones that container runtimes create.

### Lab: two hosts and a router

This builds the scenario from Part 3, two subnets joined by a Linux router, with three namespaces instead of three VMs:

```text
[ red ] 10.0.1.2 ──── 10.0.1.1 [ router ] 10.0.2.1 ──── 10.0.2.2 [ blue ]
       veth-red       veth-r1             veth-r2       veth-blue
```

```bash
# Three "machines"
ip netns add red
ip netns add router
ip netns add blue

# Two "cables"
ip link add veth-red  type veth peer name veth-r1
ip link add veth-blue type veth peer name veth-r2

# Plug the cable ends into the machines
ip link set veth-red  netns red
ip link set veth-r1   netns router
ip link set veth-r2   netns router
ip link set veth-blue netns blue

# Addresses
ip -n red    addr add 10.0.1.2/24 dev veth-red
ip -n router addr add 10.0.1.1/24 dev veth-r1
ip -n router addr add 10.0.2.1/24 dev veth-r2
ip -n blue   addr add 10.0.2.2/24 dev veth-blue

# Bring everything up
for ns in red router blue; do ip -n "$ns" link set lo up; done
ip -n red    link set veth-red up
ip -n router link set veth-r1 up
ip -n router link set veth-r2 up
ip -n blue   link set veth-blue up

# Default gateways on the two "hosts"
ip -n red  route add default via 10.0.1.1
ip -n blue route add default via 10.0.2.1
```

`ip -n NAME ...` is shorthand for `ip netns exec NAME ip ...`.

Network sysctls are per namespace, so forwarding has to be enabled inside the router namespace itself. Depending on the kernel, a new namespace may or may not copy the host's setting, so set it explicitly:

```bash
ip netns exec router sysctl -w net.ipv4.ip_forward=1
ip netns exec red ping -c 2 10.0.2.2          # works through the router
ip netns exec red ip route get 10.0.2.2       # via 10.0.1.1
```

Now turn forwarding off with `ip netns exec router sysctl -w net.ipv4.ip_forward=0` and run the ping again. It fails, even though both hosts can still ping their own gateway. That's the "can reach the gateway but nothing beyond it" pattern from Part 6, reproduced on purpose.

### Practising NAT and filtering in the lab

Each namespace has its own nftables ruleset, so the Part 4 configs work here unchanged once you swap in the veth names. For example, masquerade traffic leaving the router towards blue:

```bash
ip netns exec router nft add table ip nat
ip netns exec router nft add chain ip nat postrouting '{ type nat hook postrouting priority srcnat; }'
ip netns exec router nft add rule ip nat postrouting oifname "veth-r2" masquerade

# In one terminal, watch what blue receives
ip netns exec blue tcpdump -ni veth-blue icmp
# In another, ping from red
ip netns exec red ping -c 2 10.0.2.2
```

With the rule in place, tcpdump shows the pings arriving from `10.0.2.1` instead of `10.0.1.2`. Delete the table with `ip netns exec router nft delete table ip nat` and the real source address comes back.

Services work the same way. `ip netns exec blue python3 -m http.server 80` starts a web server that exists only in blue, `ip netns exec red curl 10.0.2.2` reaches it through the router, and `ss -tlnp` on the host doesn't show it at all, because the host is a different namespace.

To tear the lab down, delete the namespaces. That also removes the veth pairs, because deleting one end of a veth pair deletes the other:

```bash
for ns in red router blue; do ip netns del "$ns"; done
```

### How containers use this

Once you've built the lab, container networking stops being magic:

- **Docker** (default bridge network) creates a namespace for each container, puts one end of a veth pair inside it as `eth0`, attaches the other end to the `docker0` bridge on the host, and adds a masquerade rule for outbound traffic. That's a bridge from Part 5 plus NAT from Part 4. `--network host` skips the namespace, so the container uses the host's network stack directly.
- **Kubernetes** gives each pod one network namespace that all of its containers share, which is why containers in the same pod can talk over `localhost`. The CNI plugin creates the veth pair and the routes for each pod.

Docker's namespaces aren't registered under `/var/run/netns`, so `ip netns list` won't show them. Use `nsenter` with the container's PID instead:[^16]

```bash
pid=$(docker inspect -f '{{.State.Pid}}' mycontainer)
nsenter -t "$pid" -n ip -br addr
nsenter -t "$pid" -n ss -tlnp
```

This runs the host's tools inside the container's network namespace, which also works for minimal images that don't include `ip` or `ss`.

## A practice checklist

Spin up two or three VMs on a private network, or use network namespaces as in Part 8, and work through these until you can do each one without notes:

1. Give an interface a static IPv4 and IPv6 address that survives a reboot, using both `nmcli` and Netplan.
2. Add a persistent static route to a subnet behind a second VM.
3. Turn one VM into a router: enable forwarding persistently and masquerade an internal network.
4. Forward port 2222 on the router to SSH on an internal VM.
5. Redirect port 8080 to 80 on the same host.
6. Build a bridge from two interfaces and give it an address.
7. Build an `active-backup` bond, then down one link and prove failover works.
8. Add a host to `/etc/hosts` and explain why `getent` finds it but `dig` doesn't.
9. Sync time against a specific NTP server and verify it with `chronyc`.
10. Disable SSH password logins, move SSH to a new port, and connect using a `~/.ssh/config` alias.
11. Put nginx in front of two backends and confirm requests alternate between them.
12. Break something on purpose (wrong gateway, service bound to localhost, a firewall drop rule) and find it using only the troubleshooting commands above.
13. Build the red/router/blue namespace lab from scratch, then use `nsenter` to inspect a running container's interfaces and listening ports.

If you can do all thirteen, the networking quarter of the exam shouldn't hold any surprises.

## References

[^1]: The Linux Foundation, "Linux Foundation Certified System Administrator (LFCS) — Domains & Competencies" — [https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/](https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/)
[^2]: D. Plummer, "An Ethernet Address Resolution Protocol", RFC 826 — [https://www.rfc-editor.org/rfc/rfc826](https://www.rfc-editor.org/rfc/rfc826)
[^3]: V. Fuller, T. Li, "Classless Inter-domain Routing (CIDR)", RFC 4632 — [https://www.rfc-editor.org/rfc/rfc4632](https://www.rfc-editor.org/rfc/rfc4632)
[^4]: Y. Rekhter et al., "Address Allocation for Private Internets", RFC 1918 — [https://www.rfc-editor.org/rfc/rfc1918](https://www.rfc-editor.org/rfc/rfc1918)
[^5]: NetworkManager, "nmcli reference manual" — [https://networkmanager.dev/docs/api/latest/nmcli.html](https://networkmanager.dev/docs/api/latest/nmcli.html)
[^6]: Netplan documentation — [https://netplan.readthedocs.io/en/stable/](https://netplan.readthedocs.io/en/stable/)
[^7]: Linux manual page, "ip-route(8)" — [https://man7.org/linux/man-pages/man8/ip-route.8.html](https://man7.org/linux/man-pages/man8/ip-route.8.html)
[^8]: Linux kernel documentation, "IP Sysctl" — [https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
[^9]: nftables wiki, "Performing Network Address Translation (NAT)" — [https://wiki.nftables.org/wiki-nftables/index.php/Performing_Network_Address_Translation_(NAT)](https://wiki.nftables.org/wiki-nftables/index.php/Performing_Network_Address_Translation_(NAT))
[^10]: Linux kernel documentation, "Linux Ethernet Bonding Driver HOWTO" — [https://www.kernel.org/doc/html/latest/networking/bonding.html](https://www.kernel.org/doc/html/latest/networking/bonding.html)
[^11]: chrony documentation — [https://chrony-project.org/documentation.html](https://chrony-project.org/documentation.html)
[^12]: nginx, "Module ngx_http_upstream_module" — [https://nginx.org/en/docs/http/ngx_http_upstream_module.html](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
[^13]: Linux manual page, "network_namespaces(7)" — [https://man7.org/linux/man-pages/man7/network_namespaces.7.html](https://man7.org/linux/man-pages/man7/network_namespaces.7.html)
[^14]: Linux manual page, "veth(4)" — [https://man7.org/linux/man-pages/man4/veth.4.html](https://man7.org/linux/man-pages/man4/veth.4.html)
[^15]: Linux manual page, "ip-netns(8)" — [https://man7.org/linux/man-pages/man8/ip-netns.8.html](https://man7.org/linux/man-pages/man8/ip-netns.8.html)
[^16]: Linux manual page, "nsenter(1)" — [https://man7.org/linux/man-pages/man1/nsenter.1.html](https://man7.org/linux/man-pages/man1/nsenter.1.html)
