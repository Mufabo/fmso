---
title: "What Is Zero Trust?"
date: 2026-11-05
draft: true
tags: ["security", "zero-trust", "networking"]
summary: "Zero trust replaces the old castle-and-moat security model with one simple rule: never trust, always verify."
---

"Zero trust" shows up in nearly every security vendor's marketing these days, which makes it easy to dismiss as a buzzword. Underneath the hype is a useful idea, though: **never trust, always verify.** No user, device, or network connection gets trusted by default, even when it is already inside the corporate network.[^1]

## The model it replaces

For decades, network security followed a "castle and moat" design. You built a strong perimeter out of firewalls and VPNs, and anything that made it inside was treated as trusted.[^2]

That model has aged badly:

- **People work from everywhere.** The "inside" of the network is no longer a building.
- **Applications live in the cloud.** Much of what you are protecting isn't behind your firewall anyway.
- **One breach opens the gates.** An attacker who phishes a single laptop can often move laterally across the whole network, because everything inside trusts everything else.

Zero trust starts from the opposite assumption: the perimeter has already been breached, so location on the network can't be what grants access.[^3]

## The core principles

The most widely cited reference is NIST Special Publication 800-207, *Zero Trust Architecture*. Its ideas boil down to four principles.

### 1. Verify explicitly

Every request to access a resource is authenticated and authorized, using all the signals available:[^1]

- Who the user is
- Which device they are on, and whether it is healthy
- Where and when the request is coming from
- How sensitive the resource is

### 2. Least privilege

People and services get only the access they need, for only as long as they need it.[^1] An accountant doesn't need access to the build servers, and a web server doesn't need to talk to the HR database.

### 3. Assume breach

Design as if an attacker is already inside.[^1] That means:

- Segmenting the network so a compromise in one place can't spread
- Encrypting traffic internally, not just at the edge
- Logging and monitoring everything so you can spot unusual behavior

### 4. Evaluate continuously

Trust isn't granted once at login and then forgotten. It is re-checked throughout a session[^3] and can be revoked if something changes, such as a device falling out of compliance or a login suddenly coming from another country.

## What it looks like in practice

Organizations adopting zero trust typically invest in:

- **Strong identity.** Multi-factor authentication everywhere, ideally phishing-resistant methods like passkeys or hardware security keys.[^4]
- **Device posture checks.** Is the laptop managed, encrypted, and patched? If not, access is limited or denied.
- **Microsegmentation.** Workloads can only communicate with the specific systems they need.
- **Zero Trust Network Access (ZTNA).** Instead of a VPN that drops you onto the whole network, users get access to individual applications.
- **A policy engine.** A central decision point that weighs all these signals for each access request.[^3]

## A common misconception

Zero trust is not a product you can buy. Plenty of vendors sell tools that help, but zero trust is an architecture and a way of thinking. Most organizations adopt it gradually, usually starting with identity and their most sensitive systems, and expanding from there.[^1]

## The takeaway

The shift is simple to state and hard to finish: stop asking "is this request coming from inside our network?" and start asking "should *this* user, on *this* device, right now, have access to *this* resource?" Answer that question every time, and you're doing zero trust.

[^1]: [Zero Trust as a security foundation](https://learn.microsoft.com/en-us/security/zero-trust/zero-trust-overview), Microsoft Learn.
[^2]: Ward R, Beyer B. [BeyondCorp: A New Approach to Enterprise Security](https://research.google/pubs/beyondcorp-a-new-approach-to-enterprise-security/). *;login:*. 2014;39(6).
[^3]: Rose S, Borchert O, Mitchell S, Connelly S. [Zero Trust Architecture](https://doi.org/10.6028/NIST.SP.800-207), NIST Special Publication 800-207. National Institute of Standards and Technology, August 2020.
[^4]: [Implementing Phishing-Resistant MFA](https://www.cisa.gov/sites/default/files/publications/fact-sheet-implementing-phishing-resistant-mfa-508c.pdf), Cybersecurity and Infrastructure Security Agency (CISA) fact sheet.
