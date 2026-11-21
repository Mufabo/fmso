---
title: "BGP: The Protocol Holding the Internet Together"
date: 2026-11-10
draft: true
tags: ["networking", "bgp", "internet", "routing"]
categories: ["networking"]
summary: "A plain-language look at the Border Gateway Protocol: what it does, how it decides where your packets go, and why it occasionally breaks the internet."
---

Every time you load a web page, your packets cross a patchwork of independently run networks — your ISP, a few transit providers, maybe a content delivery network — before they reach their destination. Nobody runs the whole thing. So how does any of those networks know where to send your traffic?

The answer is the **Border Gateway Protocol (BGP)**, the routing protocol that lets the internet's tens of thousands of networks tell each other which addresses they can reach.

## A protocol sketched on napkins

BGP's origins are famously informal. In 1989, Kirk Lougheed of Cisco and Yakov Rekhter of IBM sketched the first version over lunch at an IETF meeting, on two cafeteria napkins — hence its nickname, the "two-napkin protocol."[^1] It was published that same year as RFC 1105.[^2] The version everyone runs today, BGP-4, is defined in RFC 4271.[^3]

That history matters: BGP was designed for a much smaller internet where the operators largely knew and trusted each other. Many of its later problems trace back to that assumption.

## Autonomous systems: the building blocks

The internet is divided into **Autonomous Systems (ASes)** — networks under a single administrative control with a single routing policy. Each AS gets an **Autonomous System Number (ASN)** from a regional internet registry. Your ISP is an AS. So is Google, Cloudflare, and many universities.

BGP is how these ASes talk to each other. There are two flavours:

- **eBGP (external BGP)** — between routers in *different* ASes. This is the "internet routing" most people mean.
- **iBGP (internal BGP)** — between routers *within* the same AS, to share what was learned from the outside world.

## How BGP works

BGP is a **path-vector** protocol. Instead of advertising "I can reach 203.0.113.0/24 at a cost of 5," a BGP speaker advertises "I can reach 203.0.113.0/24, and the path is AS 64500 → AS 64510 → AS 64520."

Some key mechanics:

1. **Peering sessions run over TCP port 179.** Two routers (neighbours, or "peers") are configured manually; BGP doesn't discover neighbours automatically.[^3]
2. **Routes are announced and withdrawn.** When a network gains or loses reachability to a prefix, it sends UPDATE messages to its peers, which propagate them onward.
3. **The AS_PATH prevents loops.** If a router sees its own ASN already in a route's path, it rejects the route.
4. **Only the best route is used and passed on.** Each router picks one best path per prefix and advertises that to its neighbours.

### Choosing the "best" path

BGP isn't primarily about the *fastest* path — it's about *policy*. When a router has multiple routes to the same prefix, it walks through a decision process, roughly:

1. Highest **local preference** (an internal policy knob: "prefer our cheaper upstream")
2. Shortest **AS_PATH**
3. Lowest **origin** type
4. Lowest **MED** (a hint from a neighbour about which entry point they prefer)
5. Prefer eBGP-learned over iBGP-learned routes
6. Closest IGP next hop, then various tie-breakers

Vendors add their own steps (Cisco's "weight," for example), but the principle holds: business relationships come first. Networks generally prefer routes from customers (who pay them) over peers (free exchange) over upstream providers (whom they pay).

## A minimal example

Here's what a simple eBGP setup looks like in FRRouting, announcing one prefix to an upstream provider:

```text
router bgp 64500
 bgp router-id 192.0.2.1
 neighbor 198.51.100.1 remote-as 64510
 !
 address-family ipv4 unicast
  network 203.0.113.0/24
  neighbor 198.51.100.1 prefix-list OUT-FILTER out
 exit-address-family
!
ip prefix-list OUT-FILTER seq 10 permit 203.0.113.0/24
```

The outbound prefix list is the important part: it ensures this network only ever announces the address space it actually owns. Skipping filters like this one is how a lot of BGP incidents start.

## When BGP goes wrong

Because BGP largely trusts what neighbours announce, mistakes and malice can spread globally within minutes.

**Route hijacks.** In February 2008, Pakistan Telecom tried to block YouTube domestically by announcing a more specific route for YouTube's addresses. That announcement leaked to its upstream provider and spread worldwide, taking YouTube offline for most of the internet for around two hours.[^4] More-specific prefixes always win in routing, so the bogus route beat the legitimate one.

**Route leaks.** A network accidentally re-announces routes it learned from one provider to another, suddenly becoming an unintended transit path for huge amounts of traffic.

**Self-inflicted withdrawals.** On 4 October 2021, Facebook, Instagram, and WhatsApp disappeared for roughly six hours. A maintenance command cut off Facebook's backbone, and its DNS servers — designed to stop advertising their routes when they couldn't reach the data centres — withdrew their BGP announcements. With no routes to Facebook's DNS, the rest of the internet simply couldn't find it.[^5]

## Making BGP safer

BGP itself has no built-in way to verify that an AS is allowed to announce a prefix. Several layers have been bolted on over the years:

- **Prefix filtering and best practices.** RFC 7454 documents operational guidance for securing BGP sessions and filtering routes.[^6] The MANRS initiative encourages networks to commit to filtering, anti-spoofing, coordination, and global validation.[^7]
- **RPKI and Route Origin Validation.** Resource holders publish cryptographically signed Route Origin Authorizations (ROAs) saying which AS may originate their prefixes. Routers can then mark announcements as valid, invalid, or unknown, and drop the invalid ones.[^8] This would have stopped the YouTube hijack.
- **BGP Roles.** RFC 9234 lets peers declare their relationship (customer, provider, peer) when the session is established, so routers can automatically detect and block route leaks.[^9]
- **BGPsec.** RFC 8205 goes further and signs the entire AS path, not just the origin.[^10] It's the most thorough approach, but deployment has been minimal because of its computational cost and need for widespread adoption.

## Why it still works

For a protocol designed on napkins over three decades ago, BGP has scaled remarkably. It carries the global routing table — now around a million IPv4 prefixes — and lets independent networks with competing commercial interests cooperate without any central authority.

Its weaknesses are real, but the fixes are increasingly deployed. The next time a major site vanishes from the internet for a few hours, there's a good chance BGP is part of the story — and an equally good chance that the outage would have been worse a decade ago.

## References

[^1]: Computer History Museum, "The Two-Napkin Protocol" — [https://computerhistory.org/blog/the-two-napkin-protocol/](https://computerhistory.org/blog/the-two-napkin-protocol/)
[^2]: K. Lougheed, Y. Rekhter, "A Border Gateway Protocol (BGP)", RFC 1105 — [https://www.rfc-editor.org/rfc/rfc1105](https://www.rfc-editor.org/rfc/rfc1105)
[^3]: Y. Rekhter, T. Li, S. Hares, "A Border Gateway Protocol 4 (BGP-4)", RFC 4271 — [https://www.rfc-editor.org/rfc/rfc4271](https://www.rfc-editor.org/rfc/rfc4271)
[^4]: RIPE NCC, "YouTube Hijacking: A RIPE NCC RIS case study" — [https://www.ripe.net/publications/news/industry-developments/youtube-hijacking-a-ripe-ncc-ris-case-study](https://www.ripe.net/publications/news/industry-developments/youtube-hijacking-a-ripe-ncc-ris-case-study)
[^5]: Meta Engineering, "More details about the October 4 outage" — [https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/)
[^6]: J. Durand, I. Pepelnjak, G. Doering, "BGP Operations and Security", RFC 7454 — [https://www.rfc-editor.org/rfc/rfc7454](https://www.rfc-editor.org/rfc/rfc7454)
[^7]: MANRS — Mutually Agreed Norms for Routing Security — [https://www.manrs.org/](https://www.manrs.org/)
[^8]: P. Mohapatra et al., "BGP Prefix Origin Validation", RFC 6811 — [https://www.rfc-editor.org/rfc/rfc6811](https://www.rfc-editor.org/rfc/rfc6811)
[^9]: A. Azimov et al., "Route Leak Prevention and Detection Using Roles in UPDATE and OPEN Messages", RFC 9234 — [https://www.rfc-editor.org/rfc/rfc9234](https://www.rfc-editor.org/rfc/rfc9234)
[^10]: M. Lepinski, K. Sriram, "BGPsec Protocol Specification", RFC 8205 — [https://www.rfc-editor.org/rfc/rfc8205](https://www.rfc-editor.org/rfc/rfc8205)
