---
title: "Communication Protocols: IP, ICMP, UDP, TCP, and HTTP"
date: 2026-11-21
draft: true
tags: ["networking", "tcp", "udp", "icmp", "http", "quic", "linux", "internet"]
categories: ["networking"]
summary: "A plain-language tour of the protocols that move your data: what IP, ICMP, UDP, TCP, and HTTP each do, how they stack on top of each other, and the Linux tools to watch them work."
---

When you run `curl https://example.com`, at least five protocols cooperate to get a page onto your screen. DNS finds the address. IP carries packets across networks. TCP makes the stream reliable. TLS encrypts it. HTTP says what you actually want. If something breaks along the way, ICMP is often the one that tells you.

Each of these protocols does one job and leans on the one below it. Once you see that split, a lot of networking stops feeling like magic: why `ping` works but the website doesn't, why video calls use UDP, why HTTP/3 left TCP behind.

This post goes through the main ones from the bottom up. It fits between my [networking basics post](/posts/20261113_networking-basics-for-lfcs/), which covers addressing and routing, and the [DNS post](/posts/20261113_dns-how-the-internet-finds-anything/), which covers name resolution.

## The stack: one job per layer

The internet uses the **TCP/IP model**, which has four layers. The OSI model has seven and is still the vocabulary people use ("Layer 4 load balancer", "Layer 7 firewall"), so here's how they line up:

| TCP/IP layer | OSI layer(s) | Protocols | Addresses | Unit of data |
|---|---|---|---|---|
| Application | 5–7 | HTTP, DNS, SSH, TLS, SMTP | URLs, hostnames | Message |
| Transport | 4 | TCP, UDP, QUIC | Ports | Segment / datagram |
| Internet | 3 | IPv4, IPv6, ICMP | IP addresses | Packet |
| Link | 1–2 | Ethernet, Wi-Fi, ARP | MAC addresses | Frame |

The idea that makes this work is **encapsulation**. Each layer takes the data from the layer above, puts its own header in front, and hands the result down. The receiver peels the headers off in reverse order.

```text
                              ┌──────────────────────────┐
 Application                  │ GET / HTTP/1.1  ...      │  HTTP message
                              └──────────────────────────┘
                    ┌────────┬──────────────────────────┐
 Transport          │TCP hdr │      HTTP message        │  TCP segment
                    └────────┴──────────────────────────┘
           ┌────────┬────────┬──────────────────────────┐
 Internet  │ IP hdr │TCP hdr │      HTTP message        │  IP packet
           └────────┴────────┴──────────────────────────┘
 ┌────────┬────────┬────────┬──────────────────────────┬─────┐
 │Eth hdr │ IP hdr │TCP hdr │      HTTP message        │ FCS │  Ethernet frame
 └────────┴────────┴────────┴──────────────────────────┴─────┘
```

A router in the middle only looks as far as the IP header. A switch only looks at the Ethernet header. Only the two endpoints read the TCP and HTTP parts. That's why you can swap out Wi-Fi for fibre, or HTTP for SSH, without touching the other layers.

## IP: best effort, nothing more

Everything in this post rides on the **Internet Protocol**. IPv4 was defined in RFC 791 in 1981,[^1] and IPv6 in RFC 8200.[^2] IP has one job: get a packet from one address to another, possibly across many networks.

It makes almost no promises. Packets can be lost, duplicated, delivered out of order, or corrupted in the payload. IP calls this **best-effort delivery**, and it's a feature, not a bug. Keeping the core simple is what let the internet grow. Reliability, if you need it, is the endpoints' problem.

A few IP header fields matter for the rest of this post:

- **Source and destination address.** Where it came from and where it's going.
- **TTL** (IPv4) or **Hop Limit** (IPv6). A counter that every router decrements by one. When it hits zero, the packet is dropped. This stops packets from looping forever, and as we'll see, it's also how `traceroute` works.
- **Protocol** (IPv4) or **Next Header** (IPv6). A number that says what's inside: `1` for ICMP, `6` for TCP, `17` for UDP, `58` for ICMPv6.[^3] This is how the receiving kernel knows which code should handle the payload.
- **Don't Fragment (DF) flag** (IPv4). Tells routers not to split a packet that's too big for the next link. IPv6 routers never fragment at all.

You can see the protocol numbers on any Linux box:

```bash
grep -E '^(icmp|tcp|udp|ipv6-icmp)\s' /etc/protocols
```

## ICMP: the network's error channel

IP silently drops packets it can't deliver. Without some feedback, a sender would never learn why its traffic vanished. That's what the **Internet Control Message Protocol** is for. ICMP was defined alongside IPv4 in RFC 792,[^4] and ICMPv6 in RFC 4443.[^5]

ICMP sits inside IP packets, but it isn't a transport protocol. It has no ports and carries no application data. It's the network layer talking about itself: "that host is unreachable", "your packet was too big", "your TTL ran out".

### The messages that matter

| ICMPv4 type | ICMPv6 type | Name | When you see it |
|---|---|---|---|
| 8 / 0 | 128 / 129 | Echo Request / Echo Reply | `ping` |
| 3 | 1 | Destination Unreachable | No route, host down, port closed (UDP), admin-prohibited |
| 3, code 4 | 2 | Fragmentation Needed / Packet Too Big | Packet larger than a link's MTU with DF set |
| 11 | 3 | Time Exceeded | TTL hit zero, used by `traceroute` |
| 5 | 137 | Redirect | A router telling you there's a better next hop |
| — | 133–136 | Router/Neighbor Solicitation and Advertisement | IPv6 Neighbor Discovery (replaces ARP) |

An ICMP error always includes the IP header and the first bytes of the packet that caused it. That lets the sender's kernel match the error to the right socket and tell the application something useful, like `Connection refused` or `No route to host`.

### ping

`ping` sends Echo Requests and waits for Echo Replies. It tells you whether a host is reachable at the IP layer and how long the round trip takes.

```bash
ping -c 4 1.1.1.1
ping -6 -c 4 2606:4700:4700::1111
ping -c 3 -s 1472 -M do 192.168.1.1   # 1472 + 8 ICMP + 20 IP = 1500, don't fragment
```

The last one is a quick MTU test. If the path can't carry a 1500-byte packet, you'll get a "message too long" or "Frag needed" error instead of a reply.

A successful ping only proves IP works. It says nothing about whether the web server on port 443 is running, and a failed ping doesn't prove the host is down, because plenty of hosts and firewalls drop Echo Requests.

### traceroute

`traceroute` is a clever abuse of TTL. It sends a probe with TTL 1, and the first router drops it and sends back an ICMP Time Exceeded. Then TTL 2, and the second router replies. And so on until the destination answers. Each Time Exceeded message reveals one router on the path.

```bash
traceroute example.com          # Linux default: UDP probes to high ports
traceroute -I example.com       # ICMP Echo probes (like Windows tracert)
traceroute -T -p 443 example.com  # TCP SYN probes to port 443, often gets through firewalls
mtr example.com                 # traceroute and ping combined, updated live
```

Rows of `* * *` usually mean a router doesn't send Time Exceeded messages, or a firewall drops them. They don't necessarily mean traffic is lost there.

### Don't block all ICMP

A common mistake is to block ICMP entirely "for security". It breaks things in subtle ways.

The worst case is **Path MTU Discovery**. A sender sets the DF flag and relies on routers to send back "Fragmentation Needed" (or ICMPv6 "Packet Too Big") when a packet is too large for some link along the way.[^6] If a firewall drops those messages, the sender never learns, keeps sending big packets, and they keep disappearing. The symptom is a classic: the TCP handshake works, small requests work, but large downloads or SSH sessions hang. This is called a **PMTUD black hole**. Modern stacks can probe for the MTU without ICMP,[^7] but you can't count on every host doing it.

In IPv6 it's even more serious: Neighbor Discovery runs over ICMPv6, so blocking it breaks basic connectivity on the local link. RFC 4890 lists which ICMPv6 messages are safe to filter and which you must let through.[^8] The sensible default is to allow Destination Unreachable, Packet Too Big, Time Exceeded, and Parameter Problem, and rate-limit Echo if you're worried about it.

## Ports: how one host runs many conversations

IP gets a packet to the right machine. But a machine runs a web server, an SSH daemon, a browser with forty tabs, and a DNS client all at once. **Ports** solve this. Both TCP and UDP add a 16-bit source port and destination port, so each conversation is identified by five values:

```text
(protocol, source IP, source port, destination IP, destination port)
(TCP,      192.168.1.10, 51734,    93.184.215.14, 443)
```

Servers listen on **well-known ports** assigned by IANA.[^9] Clients use a random **ephemeral port** for their side; on Linux that's taken from `net.ipv4.ip_local_port_range`, by default 32768–60999.

| Port | Protocol | Service |
|---|---|---|
| 22 | TCP | SSH |
| 25, 587 | TCP | SMTP (mail transfer, submission) |
| 53 | UDP, TCP | DNS |
| 67, 68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 123 | UDP | NTP |
| 443 | TCP, UDP | HTTPS (UDP for HTTP/3 over QUIC) |
| 853 | TCP, UDP | DNS over TLS / DNS over QUIC |
| 3306 / 5432 | TCP | MySQL / PostgreSQL |
| 6443 | TCP | Kubernetes API server |

On Linux, `/etc/services` has the full local list, and `ss` shows what's listening:

```bash
ss -tulpn     # TCP and UDP, listening sockets, with process names, numeric
```

## UDP: just add ports

The **User Datagram Protocol** is defined in RFC 768, which is three pages long.[^10] That says a lot. UDP adds exactly four fields to IP, in an 8-byte header:

```text
 0               16              31
┌───────────────┬───────────────┐
│  Source port  │  Dest port    │
├───────────────┼───────────────┤
│    Length     │   Checksum    │
├───────────────┴───────────────┤
│            Payload            │
└───────────────────────────────┘
```

That's it. UDP has no connection, no handshake, no acknowledgements, no retransmission, no ordering, and no congestion control. You hand the kernel a datagram, it goes out as one IP packet, and it either arrives or it doesn't. Message boundaries are kept: one `send()` on one side is one `recv()` on the other.

### Why would anyone want that?

Because sometimes reliability costs more than it's worth:

- **One question, one answer.** DNS, NTP, and DHCP send a tiny request and expect a tiny reply. A three-way handshake would triple the latency. If the answer doesn't come, the client just asks again.
- **Late data is useless data.** In a voice call or a game, a packet that arrives 300 ms late after a retransmission is worse than a packet that never arrives. Better to skip it and play the next one.
- **Broadcast and multicast.** TCP is strictly one-to-one. DHCP discovery, mDNS, and video multicast need UDP.
- **Building your own transport.** If TCP's behaviour doesn't suit you, you can build something better on top of UDP in user space. That's exactly what QUIC does, and why WireGuard and most VPNs use UDP too.

Anything you skip, the application has to handle itself if it cares. RFC 8085 is a good read on what a well-behaved UDP application should do, especially around congestion control.[^11]

### UDP's dark side: spoofing and amplification

With no handshake, nothing checks that a UDP packet's source address is real. An attacker can send a small query to a DNS, NTP, or memcached server with the **victim's** address as the source, and the server sends a much larger reply to the victim. Do that with thousands of servers and you get a reflection and amplification DDoS. This is why open resolvers are frowned on and why ISPs are encouraged to filter spoofed source addresses at the edge.

### Poking at UDP

```bash
# A UDP listener and a client
nc -u -l 9999                 # terminal 1
echo hello | nc -u 127.0.0.1 9999   # terminal 2

# A closed UDP port answers with ICMP Port Unreachable
sudo tcpdump -ni lo 'icmp or udp port 9998' &
echo hi | nc -u -w1 127.0.0.1 9998
```

That last example shows a nice detail: when a UDP packet hits a closed port, the kernel answers with **ICMP Destination Unreachable, code 3 (port unreachable)**. TCP handles the same case differently, as we'll see next.

## TCP: a reliable stream over an unreliable network

The **Transmission Control Protocol** turns IP's lossy, unordered packets into a reliable, ordered **byte stream** between two programs. The original spec, RFC 793 from 1981, was consolidated with four decades of fixes into **RFC 9293** in 2022.[^12]

TCP takes care of:

- **Connection setup and teardown**
- **Reliability**: lost data is retransmitted
- **Ordering**: data arrives in the order it was sent
- **Flow control**: a fast sender won't overwhelm a slow receiver
- **Congestion control**: senders back off when the network is overloaded

What TCP does *not* keep is message boundaries. If you `send()` 100 bytes and then another 100, the other side may `recv()` 200 at once, or 50 and 150. Application protocols on top of TCP have to frame their own messages, which is why HTTP has `Content-Length` headers and why so many protocols are line-based.

### The header

The TCP header is at least 20 bytes. The fields worth knowing:

- **Source and destination port**
- **Sequence number**: the position of this segment's first byte in the stream
- **Acknowledgement number**: the next byte the sender expects to receive
- **Flags**: `SYN` (start), `ACK` (acknowledgement field is valid), `FIN` (I'm done sending), `RST` (abort), `PSH`, `URG`, plus `ECE` and `CWR` for congestion signalling
- **Window**: how many more bytes the sender of this segment is willing to receive
- **Options**: maximum segment size (MSS), window scaling, selective acknowledgements (SACK), timestamps

### The three-way handshake

Before any data flows, the two sides agree on starting sequence numbers:

```text
 Client                                   Server (listening on :443)
   │                                         │
   │──── SYN  seq=x ───────────────────────▶ │
   │                                         │
   │ ◀─── SYN-ACK  seq=y, ack=x+1 ───────────│
   │                                         │
   │──── ACK  ack=y+1 ─────────────────────▶ │
   │                                         │
   │═══════════ data both ways ═════════════ │
```

The initial sequence numbers are random. That's not only for tidiness: predictable sequence numbers once let attackers inject data into other people's connections.

The handshake costs one full round trip before the client can send anything. Over a 100 ms path, that's 100 ms of pure waiting, and TLS adds more on top. A lot of the protocol work of the past decade (TCP Fast Open,[^13] TLS 1.3, QUIC) has been about cutting these round trips.

If the client sends a SYN to a port where nothing is listening, the server's kernel replies with **RST**. That's what produces `Connection refused`. Compare this with UDP's ICMP Port Unreachable. If nothing replies at all, a firewall is probably dropping the SYN silently, and you get a timeout instead.

### Reliability: sequence numbers and acknowledgements

Every byte has a sequence number. The receiver acknowledges the highest contiguous byte it has received. If the sender doesn't get an ACK in time, it retransmits.

Two mechanisms detect loss:

- **Retransmission timeout (RTO).** If no ACK arrives within a timer based on the measured round-trip time, resend.[^14]
- **Fast retransmit.** If the receiver gets segments 1, 2, 4, 5, 6, it keeps acknowledging "I still need 3". Three duplicate ACKs tell the sender segment 3 was lost, and it resends without waiting for the timer.

With **SACK** (selective acknowledgement), the receiver can also say "I have 4–6, just missing 3", so the sender doesn't resend data that already arrived.

### Flow control: the receive window

Each side advertises a **receive window**: how much buffer space it has left. The sender may not have more unacknowledged data in flight than that. If an application stops reading, its buffer fills, the window drops to zero, and the sender pauses. Window scaling lets the window grow far beyond the 64 KB the original 16-bit field allowed, which matters on fast, long-distance links.

### Congestion control: being polite to the network

Flow control protects the receiver. **Congestion control** protects the network in between. In the mid-1980s, the early internet suffered "congestion collapse": routers were overloaded, packets were dropped, senders retransmitted, which made the overload worse. Van Jacobson's fixes in 1988 introduced the ideas TCP still uses.[^15]

The sender keeps a **congestion window** (`cwnd`), separate from the receiver's window, and only sends the smaller of the two:

1. **Slow start.** Begin with a small window (10 segments on modern stacks)[^16] and roughly double it every round trip.
2. **Congestion avoidance.** After a threshold, grow more slowly.
3. **Back off on loss.** A lost packet is taken as a sign of congestion, so the window shrinks sharply.[^14]

Different algorithms tune this curve. Linux defaults to **CUBIC**,[^17] which is loss-based. Google's **BBR** instead models the path's bandwidth and round-trip time and tries not to fill router buffers in the first place. You can check and change it per host:

```bash
sysctl net.ipv4.tcp_congestion_control
sysctl net.ipv4.tcp_available_congestion_control
```

Slow start is why a fresh connection is slow for the first few round trips, and why reusing connections (HTTP keep-alive, connection pools) makes such a difference.

### Closing a connection, and TIME_WAIT

Each direction closes separately with a `FIN` that the other side acknowledges, so a clean close is usually four segments. One side can finish sending while still receiving, which is called a half-close.

The side that closes first enters **TIME_WAIT** and stays there for twice the maximum segment lifetime. On Linux that's a fixed 60 seconds. It looks wasteful, but it protects against delayed packets from the old connection being mistaken for part of a new one on the same ports. A busy client that opens and closes thousands of short connections can run out of ephemeral ports because of it, which is another argument for reusing connections.

`RST` is the rude version: abort immediately, no goodbye. You see it when a process crashes, when a firewall kills a connection, or when data arrives for a connection the kernel no longer knows about.

### Watching TCP on Linux

```bash
# All TCP sockets with their state
ss -tan

# Count connections by state
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c

# Internal details: RTT, cwnd, retransmits, congestion algorithm
ss -tin dst 1.1.1.1

# Only handshakes and resets on the wire
sudo tcpdump -ni any 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'
```

In `ss -tan` you'll see the state names from the TCP state machine: `LISTEN`, `SYN-SENT`, `SYN-RECV`, `ESTAB`, `FIN-WAIT-1`, `FIN-WAIT-2`, `CLOSE-WAIT`, `TIME-WAIT`, `LAST-ACK`. A pile of `CLOSE-WAIT` sockets almost always means an application bug: the other side closed, but your program never called `close()`.

### TCP's catch: head-of-line blocking

Because TCP guarantees order, one lost segment holds up everything behind it. Even if segments 4 through 100 have arrived, the application can't read any of them until segment 3 is retransmitted. For a single file download that's fine. For many independent streams multiplexed on one connection, like an HTTP/2 page with fifty images, one loss stalls all of them. Keep that in mind for the HTTP section.

## TCP vs UDP at a glance

| | TCP | UDP |
|---|---|---|
| Connection | Yes, three-way handshake | No |
| Reliability | Retransmits lost data | None |
| Ordering | Guaranteed | None |
| Message boundaries | No, byte stream | Yes, one datagram per send |
| Flow and congestion control | Built in | None (application's job) |
| Header size | 20–60 bytes | 8 bytes |
| One-to-many | No | Broadcast and multicast |
| Closed port response | RST | ICMP Port Unreachable |
| Typical uses | Web, SSH, email, databases, file transfer | DNS, DHCP, NTP, VoIP, games, VPNs, QUIC |

Neither is "better". TCP is a great default when you need every byte in order. UDP is the right choice when latency matters more than completeness, or when you want to build your own rules.

## TLS: the layer people forget

Strictly speaking, TLS isn't part of the four-layer picture, but almost all real-world traffic now goes through it. **TLS** sits between TCP and the application, and gives you three things:

- **Encryption**: nobody on the path can read the data
- **Integrity**: nobody can change it without being detected
- **Authentication**: the server proves its identity with a certificate (and the client can too, with mutual TLS)

TLS 1.3, defined in RFC 8446, cut the handshake to a single round trip, removed old and weak algorithms, and encrypts more of the handshake itself.[^18] So a fresh HTTPS connection over TCP costs one round trip for TCP and one for TLS before the first HTTP byte can be sent. Certificate management in Kubernetes is covered in my [cert-manager post](/posts/20261120_intro-to-cert-manager-on-kubernetes/).

## HTTP: the language of the web

The **Hypertext Transfer Protocol** is an application-layer, request–response protocol. A client sends a request, the server sends a response. Since 2022, HTTP's meaning (methods, status codes, headers) is defined once in RFC 9110,[^19] separately from the wire format of each version.

### What a request looks like

HTTP/1.1 is plain text, which makes it easy to see:

```bash
curl -v http://example.com/ 2>&1 | grep -E '^[<>]'
```

```text
> GET / HTTP/1.1
> Host: example.com
> User-Agent: curl/8.10.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/html; charset=UTF-8
< Content-Length: 1256
< Cache-Control: max-age=3600
<
```

A request has a **method**, a **path**, a protocol version, **headers**, and optionally a **body**. The response has a **status code**, headers, and usually a body. The blank line separates headers from body. That's essentially the whole protocol; everything else is agreed meaning on top.

### Methods

| Method | Meaning | Safe | Idempotent |
|---|---|---|---|
| `GET` | Fetch a resource | Yes | Yes |
| `HEAD` | Like GET, headers only | Yes | Yes |
| `POST` | Submit data, create something, trigger an action | No | No |
| `PUT` | Replace a resource with this one | No | Yes |
| `PATCH` | Partially modify a resource[^20] | No | No |
| `DELETE` | Remove a resource | No | Yes |
| `OPTIONS` | Ask what's allowed (used by CORS preflight) | Yes | Yes |

**Safe** means the request shouldn't change anything on the server. **Idempotent** means sending it twice has the same effect as sending it once. This matters in practice: a client, proxy, or load balancer may automatically retry an idempotent request after a network error, but must not blindly retry a `POST`, or you might get charged twice.

### Status codes

The first digit tells you the category:[^19]

| Class | Meaning | Common examples |
|---|---|---|
| `1xx` | Informational | `101 Switching Protocols` (WebSocket upgrade) |
| `2xx` | Success | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | Redirection | `301 Moved Permanently`, `302 Found`, `304 Not Modified` |
| `4xx` | Client error | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests` |
| `5xx` | Server error | `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

`502` and `504` are worth remembering for anyone running reverse proxies or ingress controllers. They're the proxy telling you the **backend** behind it failed or didn't answer in time, not that the proxy itself is broken.

### HTTP is stateless

Each request stands on its own. The server isn't required to remember anything between requests. Logins, shopping carts, and sessions are built on top with **cookies** or tokens (like `Authorization: Bearer ...`) that the client sends with every request. Statelessness is a big part of why HTTP scales so well: any server behind a load balancer can answer any request.

### The versions

| Version | Year | Transport | Key ideas |
|---|---|---|---|
| HTTP/0.9 | 1991 | TCP | One line: `GET /page`. HTML only, no headers. |
| HTTP/1.0 | 1996 | TCP | Headers, status codes, content types. One request per connection. |
| HTTP/1.1 | 1997 | TCP | Persistent connections, the `Host` header (many sites per IP), chunked transfer.[^21] |
| HTTP/2 | 2015 | TCP + TLS | Binary framing, many streams multiplexed on one connection, header compression.[^22] |
| HTTP/3 | 2022 | QUIC over UDP | Independent streams with no TCP head-of-line blocking, faster setup, connection migration.[^23] |

**HTTP/1.1** can only handle one request at a time per connection. Browsers worked around this by opening around six parallel connections per site, and developers bundled files and used sprite sheets to cut down on requests.

**HTTP/2** fixed that by splitting messages into binary frames and interleaving many streams on one TCP connection. But it ran into TCP's head-of-line blocking: one lost packet stalls every stream on the connection. On a lossy mobile network, HTTP/2 can do worse than HTTP/1.1 with its six independent connections.

**HTTP/3** solves this by leaving TCP behind.

### QUIC: rebuilding TCP on top of UDP

**QUIC** (RFC 9000) is a transport protocol that runs in user space on top of UDP.[^24] It reimplements what TCP gives you (reliability, ordering, flow and congestion control) and adds:

- **Independent streams.** Loss on one stream only delays that stream. The others keep going.
- **TLS 1.3 built in.** The transport and crypto handshakes are combined, so a new connection takes one round trip, and a resumed one can send data immediately (0-RTT).
- **Connection IDs instead of the 4-tuple.** When your phone switches from Wi-Fi to mobile data, your IP changes. A TCP connection would break. A QUIC connection keeps going.
- **Encrypted transport headers.** Middleboxes can't see or meddle with most of the transport state, which also means they can't "ossify" the protocol by depending on details of it.

Why build on UDP instead of making a new IP protocol number? Because the internet is full of firewalls and NAT devices that only pass TCP and UDP. A new protocol number would never get through. UDP is the escape hatch, which brings us back to the point from earlier: UDP's simplicity is what makes it useful to build on.

Servers advertise HTTP/3 with an `Alt-Svc` header (or a DNS `HTTPS` record), so browsers usually start on HTTP/2 and switch once they know HTTP/3 is available. If UDP 443 is blocked, they quietly fall back.

```bash
curl -sI https://cloudflare.com | grep -i alt-svc
curl --http3 -sI https://cloudflare.com | head -1   # needs a curl built with HTTP/3 support
```

## Putting it together: one request, every layer

Here's what happens on a cold `curl https://example.com/` with nothing cached:

1. **DNS over UDP.** The resolver sends a query to port 53 and gets back an IP address. One round trip, if the resolver has it cached.
2. **ARP or Neighbor Discovery.** Your machine needs the MAC address of its default gateway to put the first frame on the wire. Usually already cached.
3. **TCP handshake.** `SYN`, `SYN-ACK`, `ACK` with the server on port 443. One round trip.
4. **TLS 1.3 handshake.** Agree on keys, verify the certificate. One round trip.
5. **HTTP request and response.** `GET /` goes out, `200 OK` and the HTML come back. One round trip, plus however long the transfer takes under slow start.
6. **Along the way, IP** carries every one of these packets hop by hop, each router decrementing the TTL and picking the next hop from its routing table, which across the internet is shaped by [BGP](/posts/20261110_bgp-the-protocol-holding-the-internet-together/).
7. **If something fails, ICMP** may tell you: no route to host, packet too big, TTL exceeded.

`curl` will show you the time spent in each phase:

```bash
curl -so /dev/null https://example.com -w '\
dns:      %{time_namelookup}s\n\
tcp:      %{time_connect}s\n\
tls:      %{time_appconnect}s\n\
first byte: %{time_starttransfer}s\n\
total:    %{time_total}s\n'
```

The numbers are cumulative. If `tcp` minus `dns` is roughly the same as `tls` minus `tcp`, you're looking at one round trip each, just as the list above predicts.

## Seeing it on the wire

`tcpdump` shows the real packets. A few filters cover most needs:

```bash
sudo tcpdump -ni any icmp or icmp6        # all ICMP
sudo tcpdump -ni any udp port 53          # DNS queries and answers
sudo tcpdump -ni any 'tcp port 443 and (tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0)'  # HTTPS setup and teardown
sudo tcpdump -ni any udp port 443         # QUIC / HTTP/3
sudo tcpdump -ni any -w capture.pcap      # save everything, open later in Wireshark
```

Wireshark is the best way to actually *see* these protocols. Open a capture, right-click a TCP packet, and choose "Follow → TCP Stream" to see the whole conversation reassembled. For HTTPS you'll only see the encrypted bytes unless you export the session keys, which browsers and curl do if you set the `SSLKEYLOGFILE` environment variable.

## The "etc.": other protocols you'll meet

| Protocol | Layer | Transport | What it does |
|---|---|---|---|
| ARP | Link | — (Ethernet directly) | Maps IPv4 addresses to MAC addresses on the local network[^25] |
| DHCP | Application | UDP 67/68 | Hands out IP addresses, gateways, and DNS servers |
| DNS | Application | UDP/TCP 53 | Turns names into addresses (see the [DNS post](/posts/20261113_dns-how-the-internet-finds-anything/)) |
| NTP | Application | UDP 123 | Keeps clocks in sync |
| SSH | Application | TCP 22 | Encrypted remote shell, file transfer, port forwarding |
| SMTP / IMAP | Application | TCP 25, 587 / 993 | Sending and reading email |
| WebSocket | Application | TCP (via HTTP upgrade) | Full-duplex messaging inside an HTTP connection |
| gRPC | Application | HTTP/2 | Remote procedure calls with Protocol Buffers |
| SCTP | Transport | IP protocol 132 | Message-based, multi-stream, multi-homed transport, used in telecom and by WebRTC data channels[^26] |
| BGP | Application | TCP 179 | Routing between networks on the internet (see the [BGP post](/posts/20261110_bgp-the-protocol-holding-the-internet-together/)) |

## Why the layers matter

Every protocol here does one thing and leaves the rest to someone else. IP moves packets and doesn't care if they arrive. ICMP reports problems but carries no data. UDP adds ports and nothing more. TCP adds reliability at the cost of latency and head-of-line blocking. HTTP defines what to ask for and doesn't care how the bytes get there, which is why it could move from TCP to QUIC without changing what a `GET` or a `404` means.

That separation is also your best troubleshooting tool. Work up the stack: can you `ping` it (IP and ICMP)? Does the TCP handshake complete (`ss`, `tcpdump`, `curl -v`)? Does TLS succeed? What status code comes back? The layer where things stop working is usually where the problem is.

## References

[^1]: J. Postel, "Internet Protocol", RFC 791 — [https://www.rfc-editor.org/rfc/rfc791](https://www.rfc-editor.org/rfc/rfc791)
[^2]: S. Deering, R. Hinden, "Internet Protocol, Version 6 (IPv6) Specification", RFC 8200 — [https://www.rfc-editor.org/rfc/rfc8200](https://www.rfc-editor.org/rfc/rfc8200)
[^3]: IANA, "Assigned Internet Protocol Numbers" — [https://www.iana.org/assignments/protocol-numbers/protocol-numbers.xhtml](https://www.iana.org/assignments/protocol-numbers/protocol-numbers.xhtml)
[^4]: J. Postel, "Internet Control Message Protocol", RFC 792 — [https://www.rfc-editor.org/rfc/rfc792](https://www.rfc-editor.org/rfc/rfc792)
[^5]: A. Conta, S. Deering, M. Gupta, "Internet Control Message Protocol (ICMPv6) for the Internet Protocol Version 6 (IPv6) Specification", RFC 4443 — [https://www.rfc-editor.org/rfc/rfc4443](https://www.rfc-editor.org/rfc/rfc4443)
[^6]: J. Mogul, S. Deering, "Path MTU Discovery", RFC 1191 — [https://www.rfc-editor.org/rfc/rfc1191](https://www.rfc-editor.org/rfc/rfc1191)
[^7]: G. Fairhurst et al., "Packetization Layer Path MTU Discovery for Datagram Transports", RFC 8899 — [https://www.rfc-editor.org/rfc/rfc8899](https://www.rfc-editor.org/rfc/rfc8899)
[^8]: E. Davies, J. Mohacsi, "Recommendations for Filtering ICMPv6 Messages in Firewalls", RFC 4890 — [https://www.rfc-editor.org/rfc/rfc4890](https://www.rfc-editor.org/rfc/rfc4890)
[^9]: IANA, "Service Name and Transport Protocol Port Number Registry" — [https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
[^10]: J. Postel, "User Datagram Protocol", RFC 768 — [https://www.rfc-editor.org/rfc/rfc768](https://www.rfc-editor.org/rfc/rfc768)
[^11]: L. Eggert, G. Fairhurst, G. Shepherd, "UDP Usage Guidelines", RFC 8085 — [https://www.rfc-editor.org/rfc/rfc8085](https://www.rfc-editor.org/rfc/rfc8085)
[^12]: W. Eddy (Ed.), "Transmission Control Protocol (TCP)", RFC 9293 — [https://www.rfc-editor.org/rfc/rfc9293](https://www.rfc-editor.org/rfc/rfc9293)
[^13]: Y. Cheng et al., "TCP Fast Open", RFC 7413 — [https://www.rfc-editor.org/rfc/rfc7413](https://www.rfc-editor.org/rfc/rfc7413)
[^14]: M. Allman, V. Paxson, E. Blanton, "TCP Congestion Control", RFC 5681 — [https://www.rfc-editor.org/rfc/rfc5681](https://www.rfc-editor.org/rfc/rfc5681)
[^15]: V. Jacobson, M. Karels, "Congestion Avoidance and Control", SIGCOMM 1988 — [https://ee.lbl.gov/papers/congavoid.pdf](https://ee.lbl.gov/papers/congavoid.pdf)
[^16]: J. Chu et al., "Increasing TCP's Initial Window", RFC 6928 — [https://www.rfc-editor.org/rfc/rfc6928](https://www.rfc-editor.org/rfc/rfc6928)
[^17]: L. Xu et al., "CUBIC for Fast and Long-Distance Networks", RFC 9438 — [https://www.rfc-editor.org/rfc/rfc9438](https://www.rfc-editor.org/rfc/rfc9438)
[^18]: E. Rescorla, "The Transport Layer Security (TLS) Protocol Version 1.3", RFC 8446 — [https://www.rfc-editor.org/rfc/rfc8446](https://www.rfc-editor.org/rfc/rfc8446)
[^19]: R. Fielding, M. Nottingham, J. Reschke (Eds.), "HTTP Semantics", RFC 9110 — [https://www.rfc-editor.org/rfc/rfc9110](https://www.rfc-editor.org/rfc/rfc9110)
[^20]: L. Dusseault, J. Snell, "PATCH Method for HTTP", RFC 5789 — [https://www.rfc-editor.org/rfc/rfc5789](https://www.rfc-editor.org/rfc/rfc5789)
[^21]: R. Fielding, M. Nottingham, J. Reschke (Eds.), "HTTP/1.1", RFC 9112 — [https://www.rfc-editor.org/rfc/rfc9112](https://www.rfc-editor.org/rfc/rfc9112)
[^22]: M. Thomson, C. Benfield (Eds.), "HTTP/2", RFC 9113 — [https://www.rfc-editor.org/rfc/rfc9113](https://www.rfc-editor.org/rfc/rfc9113)
[^23]: M. Bishop (Ed.), "HTTP/3", RFC 9114 — [https://www.rfc-editor.org/rfc/rfc9114](https://www.rfc-editor.org/rfc/rfc9114)
[^24]: J. Iyengar, M. Thomson (Eds.), "QUIC: A UDP-Based Multiplexed and Secure Transport", RFC 9000 — [https://www.rfc-editor.org/rfc/rfc9000](https://www.rfc-editor.org/rfc/rfc9000)
[^25]: D. Plummer, "An Ethernet Address Resolution Protocol", RFC 826 — [https://www.rfc-editor.org/rfc/rfc826](https://www.rfc-editor.org/rfc/rfc826)
[^26]: R. Stewart, M. Tüxen, K. Nielsen, "Stream Control Transmission Protocol", RFC 9260 — [https://www.rfc-editor.org/rfc/rfc9260](https://www.rfc-editor.org/rfc/rfc9260)
