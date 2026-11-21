---
title: "DNS: How the Internet Finds Anything"
date: 2026-11-13
draft: true
tags: ["networking", "dns", "internet", "security", "linux"]
categories: ["networking"]
summary: "A plain-language look at the Domain Name System: how a name like example.com becomes an IP address, how Linux resolves names, why dig doesn't see what your apps see, and how DNS fails."
---

Computers on the internet talk to each other using IP addresses like `93.184.215.14` or `2606:2800:21f:cb07:6820:80da:af6b:8b2c`. People don't remember numbers like that. They remember `example.com`.

The **Domain Name System (DNS)** connects the two. It's a globally distributed database that turns names into addresses (and much more), answers billions of queries a day, and has no single owner. It's often called the internet's phone book, but a phone book is one printed list. DNS is closer to a huge chain of referrals: nobody knows every answer, but everyone knows who to ask next.

## Before DNS: one text file

In the early ARPANET, every host's name and address lived in a single file, `HOSTS.TXT`, kept by the Network Information Center at SRI. Every machine downloaded a fresh copy regularly. As the network grew, the file got bigger, changed more often, and the bandwidth spent sending it around grew with the square of the number of hosts.[^1]

Paul Mockapetris designed the replacement. After early drafts in RFC 882 and 883, DNS was defined in November 1987 in **RFC 1034** (concepts) and **RFC 1035** (implementation and wire format).[^1][^2] Almost four decades later, every DNS message still uses the format from RFC 1035. Later extensions like EDNS(0) and DNSSEC were built to stay compatible with it.

Your system still has a descendant of `HOSTS.TXT`: `/etc/hosts` on Linux and macOS, `C:\Windows\System32\drivers\etc\hosts` on Windows. It's usually checked before DNS, though as we'll see in the Linux section, not by every tool.

## The namespace is a tree

DNS names are read right to left. Take `www.example.com.`. Note the trailing dot.

```text
                    .  (root)
          ┌─────────┼─────────┐
         com       org        uk
          │                   │
       example               co
          │                   │
         www                 bbc
```

- The **root** (`.`) sits at the top. The trailing dot is usually left out, but it's always there.
- **Top-level domains (TLDs)** sit below it: generic ones like `com`, `org`, `dev` and country-code ones like `uk`, `de`, `tr`.
- **Second-level domains** like `example` are what you register through a registrar.
- Everything below that (`www`, `api`, `mail`) is up to the domain owner.

The tree is split into **zones**. A zone is the part of the tree one party controls. The root zone **delegates** `com` to the operators of `com`. The `com` zone delegates `example.com` to whatever name servers the owner chose. That delegation is what lets DNS scale with no central authority keeping every record.[^1]

## Who answers your query

Four kinds of actor take part in a lookup:

1. **Stub resolver.** The small resolver library on your laptop or phone. It doesn't do much itself; it sends the question to a recursive resolver and waits.
2. **Recursive resolver.** Your ISP's resolver, your router, or a public one like `1.1.1.1`, `8.8.8.8`, or `9.9.9.9`. It does the legwork of chasing referrals and caches the results.
3. **Root servers.** They don't know where `www.example.com` is, but they know who runs `.com`.
4. **Authoritative servers.** The servers that actually hold a zone's records: the TLD servers for `com`, then the servers for `example.com`.

### A lookup, step by step

Say your resolver's cache is empty and you ask for `www.example.com`:

1. Your stub resolver asks the recursive resolver: *what's the A record for `www.example.com`?*
2. The recursive resolver asks a **root server**. The root answers with a referral: *I don't know, but here are the name servers for `com`.*
3. The resolver asks a **`com` server**, which replies: *here are the name servers for `example.com`.*
4. The resolver asks an **`example.com` server**, which gives the real answer: `93.184.215.14`.
5. The resolver caches every step and returns the answer to you.

You can watch this happen with `dig`:

```bash
dig +trace www.example.com
```

The output walks from the root down through each referral to the final answer.

## The 13 root servers (that aren't 13 servers)

How does a resolver know where the root servers are? It ships with a small **root hints** file listing 13 names, `a.root-servers.net` through `m.root-servers.net`, with their addresses.

The number 13 comes from the original design. The full list of root server names and addresses had to fit in a single 512-byte UDP response. These 13 **identities** are run by 12 independent organisations; Verisign runs two of them (A and J).[^3]

But 13 identities doesn't mean 13 machines. Each identity is served using **anycast**: the same IP address is announced over BGP from many places, and routing sends your query to the nearest one. ICANN's root server advisory committee puts the number of anycast instances at more than 1,500 worldwide,[^4] and root-servers.org publishes a live count, which is now around 2,000.[^3] The root server system has been attacked many times, but no attack has ever taken the whole system down.[^4]

If you'd like to skip the root servers entirely, RFC 8806 describes how a resolver can keep a local copy of the root zone.[^5]

## Records: what DNS actually stores

Every piece of data in DNS is a **resource record** with a name, a type, a class (almost always `IN` for internet), a TTL, and data. The common types:[^2]

| Type    | Purpose                                      | Example value                     |
|---------|----------------------------------------------|-----------------------------------|
| `A`     | IPv4 address                                 | `93.184.215.14`                   |
| `AAAA`  | IPv6 address                                 | `2606:2800:21f:cb07:...`          |
| `CNAME` | Alias to another name                        | `example.com.`                    |
| `MX`    | Mail server for the domain, with priority    | `10 mail.example.com.`            |
| `NS`    | Authoritative name servers for a zone        | `ns1.example.com.`                |
| `TXT`   | Free-form text (SPF, DKIM, domain checks)    | `"v=spf1 include:_spf.google.com ~all"` |
| `SOA`   | Zone metadata: primary server, serial, timers| —                                 |
| `PTR`   | Reverse lookup, address → name               | `www.example.com.`                |
| `SRV`   | Service location (host and port)             | `0 5 5060 sip.example.com.`       |
| `CAA`   | Which CAs may issue TLS certificates         | `0 issue "letsencrypt.org"`       |

A small zone file looks like this:

```text
$ORIGIN example.com.
$TTL 3600
@       IN  SOA   ns1.example.com. admin.example.com. (
                  2026111301 ; serial
                  7200       ; refresh
                  3600       ; retry
                  1209600    ; expire
                  3600 )     ; negative-cache TTL
@       IN  NS    ns1.example.com.
@       IN  NS    ns2.example.net.
@       IN  A     93.184.215.14
@       IN  AAAA  2606:2800:21f:cb07:6820:80da:af6b:8b2c
www     IN  CNAME example.com.
@       IN  MX    10 mail.example.com.
mail    IN  A     93.184.215.20
```

One well-known gotcha: a `CNAME` can't share a name with other records, so you can't put one at the zone apex (`example.com` itself) next to the `SOA` and `NS` records. Many DNS providers work around this with non-standard "ALIAS" or "CNAME flattening" features.

## Caching and TTLs: why DNS is fast

If every lookup went all the way to the root, DNS would be slow and the root servers would melt. It doesn't, thanks to **caching**. Every record has a **TTL (time to live)** in seconds. A resolver that has fetched a record can reuse it until the TTL runs out. Referrals are cached too, so after the first lookup your resolver already knows where `com` lives and won't ask the root again for a long time.

Failed lookups are cached as well. If a name doesn't exist, the resolver remembers that (an `NXDOMAIN`) for a time set by the zone's `SOA` record.[^6]

TTLs are a trade-off:

- **Long TTLs** (hours or days) mean fewer queries and better resilience if your name servers go down.
- **Short TTLs** (seconds to minutes) mean changes take effect quickly, which matters for failover and migrations.

A common habit before moving a service is to drop the TTL a day or two in advance, make the change, then raise it again. This is also why "DNS propagation" isn't really propagation. Nothing is being pushed out; old answers are just slowly expiring from caches around the world.

## Transport: UDP, TCP, and encrypted DNS

Classic DNS runs on **port 53**, mostly over UDP: one small request packet, one small response packet, no connection setup. Original UDP responses were capped at 512 bytes. Larger answers either set a "truncated" flag so the client retries over TCP, or use **EDNS(0)**, which lets client and server agree on bigger UDP packets.[^7]

Plain DNS is **unencrypted**. Anyone on the path (your café Wi-Fi, your ISP) can see every name you look up and, in principle, change the answers. Several encrypted transports fix the privacy part:

- **DNS over TLS (DoT)** — DNS inside a TLS connection on port 853.[^8]
- **DNS over HTTPS (DoH)** — DNS queries as HTTPS requests on port 443, so they look like ordinary web traffic.[^9] Most browsers support it.
- **DNS over QUIC (DoQ)** — DNS over QUIC, with lower latency than TLS over TCP.[^10]

These protect the hop between you and your resolver. Your resolver still sees everything, so picking one you trust matters. **QNAME minimisation** helps on the other side: instead of sending the full name `www.example.com` to the root and the `com` servers, the resolver only tells each server as much as it needs (just `com` to the root, for example).[^11]

## When DNS goes wrong

There's an old sysadmin joke: *It's not DNS. There's no way it's DNS. It was DNS.* Because almost everything depends on name resolution, DNS failures look like everything is broken.

**Cache poisoning.** In 2008, Dan Kaminsky revealed a flaw that let attackers inject forged records into a resolver's cache by racing the real answer with fake ones. DNS used only a 16-bit transaction ID to match responses to queries, which is easy to guess with enough attempts. Vendors shipped a coordinated fix that also randomised the UDP source port, making forgery far harder.[^12]

**Attacks on DNS providers.** In October 2016, the Mirai botnet, made up of hacked cameras and routers, flooded the managed DNS provider Dyn. Sites that used Dyn as their only DNS provider, including Twitter, Reddit, and Netflix, were unreachable for large parts of the US for hours.[^13] Many companies have since added a second DNS provider.

**Self-inflicted outages.** In my [BGP post](/posts/20261110_bgp-the-protocol-holding-the-internet-together/), the October 2021 Facebook outage came up. Facebook's own DNS servers withdrew their BGP routes when they lost contact with the data centres. The DNS records were fine, but nobody could reach the servers holding them, so Facebook, Instagram, and WhatsApp simply disappeared.[^14]

**Everyday mistakes.** Expired domains, typos in records, forgotten delegations, and a `CNAME` pointing to a deleted cloud resource (which an attacker can then claim, a "subdomain takeover").

## DNSSEC: signing the answers

Encrypted transports hide queries, but they don't prove an answer is genuine. That's what the **DNS Security Extensions (DNSSEC)** do.[^15]

With DNSSEC, each zone signs its records. New record types carry the signatures (`RRSIG`) and public keys (`DNSKEY`). A parent zone vouches for its child's key using a `DS` record. This builds a **chain of trust** from the root down:

```text
root key (trust anchor)  →  signs .com DS  →  .com key  →  signs example.com DS  →  example.com key  →  signs records
```

A validating resolver checks every link. If a signature doesn't match, it refuses the answer and returns `SERVFAIL` instead of passing on a forged record. The root zone has been signed since 2010.

DNSSEC has a reputation for being fiddly. Expired signatures or botched key rollovers make a domain fail to resolve for every validating resolver, which is a strong incentive to automate. Adoption is uneven: many TLDs are signed, but a minority of second-level domains are.

## DNS on Linux

So far "the stub resolver" has been a black box. On Linux it's a small stack of pieces, and knowing how they fit together explains a lot of confusing debugging sessions.

When a program like `curl`, `ssh`, `ping`, or your browser wants an address, it usually doesn't speak DNS itself. It calls the C library function `getaddrinfo()`. In glibc, that function hands the question to the **Name Service Switch (NSS)**, which decides *where* to look. DNS is only one of the places it can look.

```text
 application (curl, ssh, ping …)
          │  getaddrinfo("example.com")
          ▼
 glibc NSS  ── reads /etc/nsswitch.conf
          │
   ┌──────┼───────────┬──────────────┬──────────────┐
 files   mdns         resolve         dns          myhostname
(/etc/hosts) (.local via Avahi) (systemd-resolved) (/etc/resolv.conf) (own hostname)
```

### /etc/nsswitch.conf: the order of lookups

The `hosts:` line in `/etc/nsswitch.conf` lists the sources in order.[^16] A typical line on a desktop distro with systemd-resolved looks something like this (the exact line varies by distribution):

```text
hosts: files mdns4_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] dns myhostname
```

Reading left to right:

- **`files`** checks `/etc/hosts`. A match here wins, and DNS is never asked.
- **`mdns4_minimal`** resolves `.local` names on your LAN via multicast DNS (Avahi). `[NOTFOUND=return]` means "if it's a `.local` name and mDNS can't find it, stop here".
- **`resolve`** asks systemd-resolved directly, over D-Bus/Varlink rather than the DNS protocol. `[!UNAVAIL=return]` means "unless resolved isn't running, trust its answer and stop".
- **`dns`** is the classic glibc DNS client. It reads `/etc/resolv.conf` and sends DNS packets itself. Here it's only a fallback for when resolved is down.
- **`myhostname`** makes sure your machine's own hostname (and `localhost`) always resolves, even with no network.

### /etc/hosts: local overrides

```text
127.0.0.1   localhost
::1         localhost
192.168.1.20  nas.home  nas
127.0.0.1   blog.test
```

Handy for development (`blog.test` → your local Hugo server), for blocking domains, or for pointing a name at a new server before you change real DNS. It only affects the machine it's on.

### /etc/resolv.conf: which server to ask

```text
nameserver 192.168.1.1
nameserver 1.1.1.1
search home.lan
options ndots:1 timeout:2 attempts:2
```

- **`nameserver`** lists up to three resolvers to query.
- **`search`** lists domains to append to short names, so `ping nas` can try `nas.home.lan`.
- **`ndots`** sets how many dots a name needs before it's tried as-is first, before the search list.[^17]

On most modern systems you shouldn't edit this file by hand. It's usually a symlink owned by systemd-resolved, NetworkManager, or `resolvconf`, and your changes get overwritten. Run `ls -l /etc/resolv.conf` to see who owns it.

`ndots` is a classic Kubernetes gotcha. Pods get `ndots:5` and several search domains by default, so a lookup for `api.example.com` (only two dots) is first tried as `api.example.com.default.svc.cluster.local` and friends before the real name. That means several extra queries per lookup. Using a fully qualified name with a trailing dot, or lowering `ndots` in the pod's `dnsConfig`, avoids it.[^18]

### systemd-resolved: the local resolver

Many distributions (Ubuntu, Fedora, Arch with the right setup) run **systemd-resolved**.[^19] It's a small local service that:

- **caches** answers for every program on the machine,
- keeps **separate DNS servers per network interface**, so a VPN can send `*.corp.example` to the company's DNS while everything else goes to your normal resolver ("split DNS"),
- can talk **DNS over TLS** upstream and validate **DNSSEC**, if enabled in `/etc/systemd/resolved.conf`,[^20]
- handles mDNS and LLMNR for local names.

Programs reach it two ways: through the `resolve` NSS module described above, or through a **stub listener** on `127.0.0.53`. In that setup `/etc/resolv.conf` just says:

```text
nameserver 127.0.0.53
options edns0 trust-ad
search .
```

This is why you often see `127.0.0.53` and wonder where your real DNS server went. It's behind resolved. To see the real upstream servers per interface:

```bash
resolvectl status          # DNS servers and domains per interface
resolvectl query example.com  # resolve the way resolved does
resolvectl statistics      # cache size, hits, misses
sudo resolvectl flush-caches  # clear the cache
```

Other setups use **dnsmasq** (often started by NetworkManager), **unbound**, or **nscd** as the local cache. The idea is the same: a local service in front of the real resolver.

### Seeing what your apps see: getent

To ask "what will my programs actually get for this name?", use `getent`. It goes through NSS exactly like `getaddrinfo()`, so it respects `/etc/hosts`, mDNS, resolved, and the search list:

```bash
getent hosts example.com     # first address
getent ahosts example.com    # all addresses, IPv4 and IPv6
getent hosts nas             # short name, search list applied
```

## Why dig ignores local DNS

Here's the part that trips people up: **`dig` doesn't use any of that.**

`dig` (like `host` and `nslookup`, which come from the same BIND tools) is a **DNS protocol tool**.[^21] It doesn't call `getaddrinfo()` and never goes near NSS. It builds a DNS packet itself, reads the `nameserver` line from `/etc/resolv.conf` (unless you give it `@server`), sends the query, and prints exactly what comes back.

That means `dig`:

- **ignores `/etc/hosts`** — your `blog.test` override doesn't exist as far as `dig` is concerned,
- **ignores `/etc/nsswitch.conf`** — the lookup order, `myhostname`, and everything else in it,
- **ignores mDNS** — `nas.local` won't resolve,
- **ignores the search list by default** — `dig nas` asks for the literal name `nas.` unless you add `+search`,
- **ignores nscd or other NSS-level caches.**

You can see it for yourself:

```bash
$ echo "127.0.0.1 blog.test" | sudo tee -a /etc/hosts

$ getent hosts blog.test
127.0.0.1       blog.test

$ dig +short @1.1.1.1 blog.test
$                       # nothing: 1.1.1.1 has never heard of it

$ ping -c1 blog.test
PING blog.test (127.0.0.1) 56(84) bytes of data.
```

There's one wrinkle on systems with systemd-resolved. If `/etc/resolv.conf` points at `127.0.0.53`, a plain `dig example.com` *does* go to resolved, through its DNS stub, so resolved's cache and per-interface routing still apply. By default, resolved also answers from `/etc/hosts` (`ReadEtcHosts=yes`), so a hosts entry may show up in `dig` there.[^20] That's resolved doing it, not `dig`. Point `dig` at any other server with `@` and the local view is gone.

This isn't a bug. The two tools answer different questions:

| Question | Tool |
|----------|------|
| What does a DNS server say about this name? | `dig`, `dig @server` |
| What will my applications get for this name? | `getent hosts`, `getent ahosts` |
| What does systemd-resolved return, and from which interface? | `resolvectl query` |

The classic symptom is "`dig` works but `curl` doesn't" (or the reverse). Usually it's one of these: a stale `/etc/hosts` entry, a VPN that only installed its DNS servers on one interface in resolved, or a short name that only works with the search list. Check with `getent` and `resolvectl` before blaming the DNS server.

Some programs skip NSS too. Firefox and Chrome with DNS over HTTPS turned on send queries straight to their DoH provider. Go programs often use Go's own resolver, which reads `/etc/hosts` and `/etc/resolv.conf` but doesn't load NSS modules like `resolve` or `mdns` (it falls back to glibc in some cases).[^22] Alpine-based containers use musl, which has no NSS at all, just `/etc/hosts` and `/etc/resolv.conf`.

## Poking at DNS yourself

A few commands go a long way. Remember that `dig` shows what a DNS server says, not what your applications see:

```bash
# Basic lookup
dig example.com

# A specific record type
dig example.com MX
dig example.com AAAA

# Ask a specific resolver
dig @1.1.1.1 example.com

# Just the answer
dig +short example.com

# Follow the delegation from the root
dig +trace example.com

# Reverse lookup
dig -x 1.1.1.1

# Check DNSSEC
dig +dnssec example.com

# Which root server instance answered you?
dig @k.root-servers.net hostname.bind CH TXT +short

# What your applications actually get (uses /etc/hosts, NSS, resolved)
getent ahosts example.com
resolvectl query example.com
```

## Why it still works

DNS was designed when the internet had a few thousand hosts. It now handles hundreds of millions of domains, and its core ideas (a hierarchical namespace, delegation, caching with TTLs) have barely changed since 1987. What changed is everything bolted on around it: anycast for resilience, DNSSEC for authenticity, and encrypted transports for privacy.

The next time a page loads instantly, it's because a resolver somewhere remembered the answer. And the next time half the internet goes down, there's a decent chance it was DNS.

## References

[^1]: P. Mockapetris, "Domain Names — Concepts and Facilities", RFC 1034 — [https://www.rfc-editor.org/rfc/rfc1034](https://www.rfc-editor.org/rfc/rfc1034)
[^2]: P. Mockapetris, "Domain Names — Implementation and Specification", RFC 1035 — [https://www.rfc-editor.org/rfc/rfc1035](https://www.rfc-editor.org/rfc/rfc1035)
[^3]: Root Server Technical Operations Association — [https://root-servers.org/](https://root-servers.org/)
[^4]: ICANN, "Root Server System Advisory Committee (RSSAC) FAQ" — [https://www.icann.org/groups/rssac/faq](https://www.icann.org/groups/rssac/faq)
[^5]: W. Kumari, P. Hoffman, "Running a Root Server Local to a Resolver", RFC 8806 — [https://www.rfc-editor.org/rfc/rfc8806](https://www.rfc-editor.org/rfc/rfc8806)
[^6]: M. Andrews, "Negative Caching of DNS Queries (DNS NCACHE)", RFC 2308 — [https://www.rfc-editor.org/rfc/rfc2308](https://www.rfc-editor.org/rfc/rfc2308)
[^7]: J. Damas, M. Graff, P. Vixie, "Extension Mechanisms for DNS (EDNS(0))", RFC 6891 — [https://www.rfc-editor.org/rfc/rfc6891](https://www.rfc-editor.org/rfc/rfc6891)
[^8]: Z. Hu et al., "Specification for DNS over Transport Layer Security (TLS)", RFC 7858 — [https://www.rfc-editor.org/rfc/rfc7858](https://www.rfc-editor.org/rfc/rfc7858)
[^9]: P. Hoffman, P. McManus, "DNS Queries over HTTPS (DoH)", RFC 8484 — [https://www.rfc-editor.org/rfc/rfc8484](https://www.rfc-editor.org/rfc/rfc8484)
[^10]: C. Huitema, S. Dickinson, A. Mankin, "DNS over Dedicated QUIC Connections", RFC 9250 — [https://www.rfc-editor.org/rfc/rfc9250](https://www.rfc-editor.org/rfc/rfc9250)
[^11]: S. Bortzmeyer, R. Dolmans, P. Hoffman, "DNS Query Name Minimisation to Improve Privacy", RFC 9156 — [https://www.rfc-editor.org/rfc/rfc9156](https://www.rfc-editor.org/rfc/rfc9156)
[^12]: CERT/CC, "Vulnerability Note VU#800113: Multiple DNS implementations vulnerable to cache poisoning" — [https://www.kb.cert.org/vuls/id/800113](https://www.kb.cert.org/vuls/id/800113)
[^13]: Cloudflare Learning Center, "What is the Mirai Botnet?" — [https://www.cloudflare.com/learning/ddos/glossary/mirai-botnet/](https://www.cloudflare.com/learning/ddos/glossary/mirai-botnet/)
[^14]: Meta Engineering, "More details about the October 4 outage" — [https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/)
[^15]: R. Arends et al., "DNS Security Introduction and Requirements", RFC 4033 — [https://www.rfc-editor.org/rfc/rfc4033](https://www.rfc-editor.org/rfc/rfc4033)
[^16]: Linux manual page, "nsswitch.conf(5)" — [https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html](https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html)
[^17]: Linux manual page, "resolv.conf(5)" — [https://man7.org/linux/man-pages/man5/resolv.conf.5.html](https://man7.org/linux/man-pages/man5/resolv.conf.5.html)
[^18]: Kubernetes documentation, "DNS for Services and Pods" — [https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
[^19]: systemd, "systemd-resolved.service(8)" — [https://www.freedesktop.org/software/systemd/man/latest/systemd-resolved.service.html](https://www.freedesktop.org/software/systemd/man/latest/systemd-resolved.service.html)
[^20]: systemd, "resolved.conf(5)" — [https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html)
[^21]: ISC, "BIND 9 Manual Pages: dig — DNS lookup utility" — [https://bind9.readthedocs.io/en/latest/manpages.html](https://bind9.readthedocs.io/en/latest/manpages.html)
[^22]: Go standard library, "net package: Name Resolution" — [https://pkg.go.dev/net#hdr-Name_Resolution](https://pkg.go.dev/net#hdr-Name_Resolution)
