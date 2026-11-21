---
title: "TLS Explained: How It Works, With HTTPS and etcd as Examples"
date: 2026-11-05
draft: true
description: "What TLS does, the cryptography it is built from, how the TLS 1.3 handshake works step by step, and how it is used in practice: by your browser for HTTPS and by etcd to protect the heart of a Kubernetes cluster with mutual TLS."
summary: "TLS is the protocol that puts the S in HTTPS and keeps your Kubernetes secrets safe on the wire. Here is what it actually does, how a TLS 1.3 handshake works, and how it looks in two very different settings: a browser talking to a website, and etcd talking to its clients and peers."
tags: ["tls", "https", "etcd", "security", "cryptography", "kubernetes", "certificates"]
categories: ["Programming"]
---

Almost every network connection you care about is wrapped in **TLS (Transport Layer Security)**. It is what turns HTTP into HTTPS, and it is what stops anyone on the network path between your Kubernetes API server and etcd from reading every Secret in your cluster.

Most people only meet TLS when it breaks: an expired certificate, a `certificate signed by unknown authority`, a `bad certificate` from a server that wanted a client certificate. Those errors make a lot more sense once you know what TLS is actually checking.

This post covers what TLS gives you, the building blocks it is made from, how the TLS 1.3 handshake works, and then two concrete examples:

- **HTTPS**, where a browser that has never seen a server before has to decide whether to trust it.
- **etcd**, where a small set of machines you control use **mutual TLS** to authenticate each other in both directions.

Everything here is based on TLS 1.3 (RFC 8446), etcd 3.6/3.7 and Go 1.24+, current at the time of writing.[^1][^2]

## What TLS gives you

TLS sits between TCP and the application protocol (HTTP, etcd's gRPC, SMTP, and so on). The application writes plaintext into a TLS connection; TLS sends encrypted records over TCP. It provides three guarantees:

- **Confidentiality.** Nobody on the path can read the data.
- **Integrity.** Nobody on the path can change the data without being detected.
- **Authentication.** You know who is on the other end. Usually only the server proves its identity; with **mutual TLS (mTLS)** the client does too.

The third point is the one people underestimate. Encryption without authentication is nearly worthless: if you can't tell whether you are talking to the real server or to an attacker in the middle, you could be setting up a perfectly encrypted channel straight to the attacker. Most of the operational pain of TLS (certificates, CAs, SANs, expiry) exists to solve authentication.

What TLS does *not* do:

- It doesn't hide **who** you are talking to. IP addresses are visible, and so (usually) is the hostname, as we'll see.
- It doesn't protect data **at rest**. etcd's data on disk is not encrypted by TLS.
- It doesn't say anything about whether the server is **trustworthy**. A phishing site can have a perfectly valid certificate for `paypa1-login.com`.

## The building blocks

TLS is not one algorithm. It is a protocol that combines four kinds of cryptography, each doing one job.

### 1. Symmetric encryption: protecting the data

The bulk of the data is encrypted with a **symmetric** cipher: both sides use the same secret key. Symmetric ciphers are fast, which matters when you are streaming video or replicating a database.

TLS 1.3 only uses **AEAD** ciphers (Authenticated Encryption with Associated Data), which encrypt and integrity-protect in one step. Each record carries a tag; if a single bit is changed in transit, decryption fails and the connection is torn down. There are only three cipher suites in TLS 1.3:

| Cipher suite | Notes |
| --- | --- |
| `TLS_AES_128_GCM_SHA256` | The most common. Very fast on CPUs with AES instructions. |
| `TLS_AES_256_GCM_SHA384` | Same with a bigger key. |
| `TLS_CHACHA20_POLY1305_SHA256` | Fast in software, popular on phones and older CPUs. |

The hard question is: how do two machines that have never met agree on a shared secret key, over a network where everyone can listen?

### 2. Key exchange: agreeing on a secret in public

This is what **Diffie-Hellman** solves. Each side generates a random private value and sends a matching public value. Each side combines its own private value with the other's public value, and both end up with the **same shared secret**. Someone who saw both public values can't compute it.

TLS 1.3 uses **ephemeral** Diffie-Hellman: fresh key pairs for every connection, thrown away afterwards. That gives **forward secrecy**: if a server's long-term private key is stolen next year, an attacker who recorded today's traffic still can't decrypt it, because the keys that protected it no longer exist anywhere. (TLS 1.2 also allowed RSA key exchange, which had no forward secrecy. TLS 1.3 removed it.)

The classic choice is **X25519**, Diffie-Hellman on an elliptic curve. The current default in browsers and Go is the hybrid **X25519MLKEM768**: an X25519 exchange and an ML-KEM-768 exchange (a post-quantum algorithm standardised by NIST as FIPS 203) done side by side, with both results mixed into the key.[^3] The connection stays secure as long as *either* algorithm holds. The point is to stop "harvest now, decrypt later": an attacker recording traffic today in the hope of breaking it with a future quantum computer.

### 3. Digital signatures: proving identity

Key exchange gives you a shared secret with *someone*. To know it is the right someone, the server signs the handshake with its long-term private key (RSA, ECDSA or Ed25519), and the client checks the signature with the server's public key.

But that just moves the problem: how does the client know which public key belongs to `example.com`?

### 4. Certificates: binding a key to a name

A **certificate** (X.509) is a small document that says, in effect, *"this public key belongs to `example.com`, valid from date A to date B"*, signed by a **Certificate Authority (CA)**. The important fields:

| Field | What it is |
| --- | --- |
| Subject | Who the certificate is for. The Common Name (`CN`) is here, but for hostnames it is legacy. |
| Subject Alternative Name (SAN) | The DNS names and IP addresses the certificate is valid for. **This is what clients check.** |
| Issuer | The CA that signed it. |
| Validity | `notBefore` and `notAfter`. |
| Public key | The key the server will sign with. |
| Key usage / Extended key usage | What the key may be used for: `serverAuth`, `clientAuth`, signing other certificates. |

Certificates form a **chain**: the server's (leaf) certificate is signed by an intermediate CA, which is signed by a root CA. The client has a **trust store**, a list of root certificates it trusts in advance. Validation means building a chain from the leaf up to one of those roots and checking every signature, date and constraint along the way.

Who is in the trust store is the big difference between our two examples. For HTTPS, it's the ~150 public roots shipped by your OS and browser. For etcd, it's usually exactly one CA that you created yourself.

## The TLS 1.3 handshake

Here is how the four pieces come together. TLS 1.3 needs just **one round trip** before the client can send application data:

```text
   Client                                                Server

   ClientHello
     supported versions, cipher suites,
     key_share (X25519MLKEM768, X25519),
     server_name (SNI), ALPN
                          ───────────────────────►
                                                     ServerHello
                                                       chosen cipher suite,
                                                       key_share
                                     ── from here everything is encrypted ──
                                                     {EncryptedExtensions}
                                                     {CertificateRequest}   ← only for mTLS
                                                     {Certificate}
                                                     {CertificateVerify}
                                                     {Finished}
                          ◄───────────────────────
   {Certificate}          ← only for mTLS
   {CertificateVerify}    ← only for mTLS
   {Finished}
                          ───────────────────────►
   [Application data]     ◄──────────────────────►  [Application data]
```

Step by step:

1. **ClientHello.** The client says which TLS versions and cipher suites it supports and, crucially, *guesses* which key exchange the server will pick and sends its public value right away (`key_share`). It also sends the hostname it wants (**SNI**, Server Name Indication) so a server hosting many sites can pick the right certificate, and the application protocols it speaks (**ALPN**, e.g. `h2` or `http/1.1`).
2. **ServerHello.** The server picks a cipher suite and key exchange and sends its own public value. Both sides can now compute the shared secret, and from it derive handshake keys (TLS 1.3 uses HKDF to derive a whole tree of keys). Everything after this point is encrypted.
3. **EncryptedExtensions.** The rest of the server's choices, such as the selected ALPN protocol.
4. **CertificateRequest** (mTLS only). "Show me your certificate too", including which CAs the server accepts.
5. **Certificate.** The server's certificate chain.
6. **CertificateVerify.** A signature, made with the server's private key, over a hash of the whole handshake so far. This proves the server actually holds the private key for the certificate, and ties that proof to *this* handshake so it can't be replayed elsewhere.
7. **Finished.** A MAC over the whole handshake using a key derived from the shared secret. If anyone tampered with any message (for example, stripped the strong cipher suites out of the ClientHello to force a downgrade), the Finished values won't match.
8. The client validates the certificate chain, checks the signature and Finished message, sends its own certificate (mTLS) and Finished, and starts sending data.

If the client guessed the wrong key exchange in step 1, the server sends a **HelloRetryRequest** and it costs one extra round trip. Clients that have connected before can use **session resumption** with a pre-shared key, and even send data in the very first flight (**0-RTT**), at the price of that early data being replayable, so it must only be used for idempotent requests.

TLS 1.2 is still widely supported and needs two round trips. It also allowed a long list of weak options that TLS 1.3 removed: RSA key exchange, CBC-mode ciphers, SHA-1, renegotiation, compression. If you control both ends (as with etcd), there's rarely a reason not to require TLS 1.3.

## Example 1: HTTPS

You type `https://example.com` into a browser. Here is what TLS does in that setting.

### The connection

After DNS and a TCP connection to port 443 (or a QUIC connection over UDP for HTTP/3, which embeds the same TLS 1.3 handshake inside QUIC), the browser sends a ClientHello with:

- `server_name: example.com`
- ALPN: `h2`, `http/1.1`
- key shares for `X25519MLKEM768` and `X25519`

You can watch the result with `curl -v`:

```bash
curl -v https://example.com -o /dev/null
```

```text
* Connected to example.com (93.184.215.14) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519MLKEM768 / id-ecPublicKey
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=example.com
*  start date: Oct 12 00:00:00 2026 GMT
*  expire date: Jan 10 23:59:59 2027 GMT
*  subjectAltName: host "example.com" matched cert's "example.com"
*  issuer: C=US; O=Some CA; CN=Some Intermediate CA
*  SSL certificate verify ok.
```

Every message from the handshake diagram is there, in order. (The exact output depends on your curl build; older TLS libraries will show `X25519` instead of the hybrid group.)

### What the browser checks

The interesting part of HTTPS is that the browser has **no prior relationship** with the server. All trust comes from the public CA system. Before sending a single byte of HTTP, the browser checks:

1. **The chain.** The leaf certificate chains through intermediates to a root in the browser's trust store, and every signature is valid.
2. **The name.** `example.com` appears in the certificate's **SAN**. The `CN` field is ignored by modern browsers. This is the check that stops an attacker with a valid certificate for `attacker.com` from impersonating `example.com`.
3. **The dates.** Now is between `notBefore` and `notAfter`.
4. **Key usage.** The certificate is allowed to be used for `serverAuth`.
5. **Certificate Transparency.** Chrome and Safari require proof that the certificate was logged in public CT logs. Since every publicly trusted certificate is published, a CA that mis-issues a certificate for your domain can be caught, and you can monitor the logs for certificates issued for your names.
6. **Proof of possession.** The `CertificateVerify` signature checks out with the certificate's public key.

The client does **not** present a certificate. The server authenticates the user at the HTTP level instead, with passwords, cookies, OAuth tokens and so on, all protected by the TLS channel.

### Revocation and shrinking lifetimes

What if a server's private key is stolen? The certificate can be revoked, but revocation checking on the web has always worked badly: online checks (OCSP) were slow, leaked your browsing to the CA, and browsers ignored failures anyway. Browsers have moved to pushing compressed revocation lists to clients instead (Chrome's CRLSets, Firefox's CRLite), and Let's Encrypt has dropped OCSP entirely.

The industry's real answer is to make certificates expire quickly, so a stolen key is only useful for a short time. In April 2025 the CA/Browser Forum voted (ballot SC-081v3) to cut the maximum lifetime of public TLS certificates in stages: 200 days from March 2026, 100 days from March 2027, and 47 days from March 2029.[^4] Let's Encrypt is moving faster: from 90 days to 64 days in February 2027 and 45 days in February 2028, and it already offers six-day certificates.[^5] The practical consequence: **if certificate renewal on your public sites isn't fully automated (ACME), fix that now.**

### What's still visible

Even with HTTPS, someone on the network sees your IP, the server's IP, timing and sizes, and the SNI hostname, because it travels in the unencrypted ClientHello. **Encrypted Client Hello (ECH)** fixes the last one by encrypting the ClientHello with a key the server publishes in DNS. It is supported by Chrome, Firefox and large CDNs, but not yet by most servers.

### HSTS

One last HTTPS-specific detail: if a user types `example.com` without `https://`, the first request goes out over plain HTTP, and an attacker can intercept it before any redirect to HTTPS. The `Strict-Transport-Security` header tells browsers to only ever use HTTPS for that site in the future, and the HSTS preload list bakes that into browsers in advance.

## Example 2: etcd

etcd is a distributed key-value store that keeps its data consistent across a cluster using the Raft consensus algorithm. Its most famous user is Kubernetes: every object in a cluster, including every Secret, lives in etcd. Anyone who can talk to etcd unauthenticated owns the cluster. Anyone who can read the traffic between etcd members can read every Secret.

The TLS here is the same protocol as in your browser. What changes is the **trust model**:

- You control every machine, so you don't need public CAs. You run **your own CA**, and the trust store has exactly one entry: your CA certificate.
- Both sides are machines, so both sides can and should present certificates. This is **mutual TLS**.
- There are **two separate channels** to protect.

### The two channels

```text
                     client traffic (port 2379)
   kube-apiserver ─────────────────────────────┐
   etcdctl        ──────────────────────┐      │
                                        ▼      ▼
                                   ┌──────────────┐
                                   │    etcd-1    │
                                   └──────────────┘
                                     ▲          ▲
               peer traffic (2380)   │          │   peer traffic (2380)
                                     ▼          ▼
                          ┌──────────────┐  ┌──────────────┐
                          │    etcd-2    │◄►│    etcd-3    │
                          └──────────────┘  └──────────────┘
```

- **Client traffic** (port 2379): the API server, `etcdctl`, backup tools. etcd is the TLS server; clients connect to it.
- **Peer traffic** (port 2380): Raft messages and snapshots between members. Every member is both a server (accepting connections from peers) and a client (dialling peers).

Each channel has its own set of flags, which is the first thing that confuses people:

| Purpose | Client channel | Peer channel |
| --- | --- | --- |
| This member's certificate | `--cert-file` | `--peer-cert-file` |
| This member's private key | `--key-file` | `--peer-key-file` |
| Require certificates from the other side | `--client-cert-auth` | `--peer-client-cert-auth` |
| CA to verify the other side's certificate | `--trusted-ca-file` | `--peer-trusted-ca-file` |
| Self-signed certs, generated at startup | `--auto-tls` | `--peer-auto-tls` |

Plus two flags that apply to both: `--tls-min-version` / `--tls-max-version` and `--cipher-suites`.[^6]

### Server TLS vs mutual TLS

If you only set `--cert-file` and `--key-file`, etcd does **server TLS**, just like a website: clients can verify etcd and the traffic is encrypted, but etcd accepts anyone who connects. That is the HTTPS model, and for etcd it's not enough.

Adding `--client-cert-auth` and `--trusted-ca-file` turns on **mutual TLS**. etcd sends a `CertificateRequest` in the handshake, and a client without a certificate signed by your CA is rejected before it can send a single request. The same goes for peers with `--peer-client-cert-auth`: a machine that isn't a member can't join the Raft conversation.

With mTLS, the certificate also becomes an **identity**. If you enable etcd's authentication (RBAC), the client certificate's Common Name is used as the etcd username, so a certificate with `CN=backup` can be given read-only access.

### Building it: a CA and certificates

In production you'd use your platform's tooling (kubeadm, cert-manager, Vault, cfssl). Doing it once by hand with OpenSSL shows what's really needed.

**1. Create a CA.** This is the only certificate everyone will trust:

```bash
openssl req -x509 -new -noenc \
  -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
  -keyout ca.key -out ca.crt -days 3650 \
  -subj "/CN=etcd-ca"
```

Guard `ca.key` carefully: anyone holding it can mint a certificate that your whole cluster will accept.

**2. Create a certificate for each member.** The SANs must include every name and IP that clients and peers will use to reach this member. Since a member is a TLS server *and* a TLS client towards its peers, it needs both extended key usages:

```bash
openssl req -new -noenc \
  -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
  -keyout etcd1.key -out etcd1.csr \
  -subj "/CN=etcd1" \
  -addext "subjectAltName=DNS:etcd1,DNS:localhost,IP:10.0.0.11,IP:127.0.0.1" \
  -addext "extendedKeyUsage=serverAuth,clientAuth"

openssl x509 -req -in etcd1.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out etcd1.crt -days 365 -copy_extensions copy
```

Repeat for `etcd2` (10.0.0.12) and `etcd3` (10.0.0.13).

**3. Create client certificates.** One per client, with only `clientAuth`:

```bash
openssl req -new -noenc \
  -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
  -keyout apiserver-etcd-client.key -out apiserver-etcd-client.csr \
  -subj "/CN=kube-apiserver" \
  -addext "extendedKeyUsage=clientAuth"

openssl x509 -req -in apiserver-etcd-client.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out apiserver-etcd-client.crt -days 365 -copy_extensions copy
```

For this example one CA signs everything. Many setups use separate CAs for peer and client certificates, so that a client certificate can never be used to pose as a cluster member.

### Running etcd with TLS

```bash
etcd --name etcd1 \
  --listen-client-urls https://10.0.0.11:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.0.11:2379 \
  --cert-file etcd1.crt --key-file etcd1.key \
  --client-cert-auth --trusted-ca-file ca.crt \
  --listen-peer-urls https://10.0.0.11:2380 \
  --initial-advertise-peer-urls https://10.0.0.11:2380 \
  --peer-cert-file etcd1.crt --peer-key-file etcd1.key \
  --peer-client-cert-auth --peer-trusted-ca-file ca.crt \
  --tls-min-version TLS1.3 \
  --initial-cluster etcd1=https://10.0.0.11:2380,etcd2=https://10.0.0.12:2380,etcd3=https://10.0.0.13:2380 \
  --initial-cluster-state new
```

Note the `https://` in every URL. With `http://`, the certificate flags are simply not used for that listener.

Talking to it with `etcdctl` needs the same three pieces as any mTLS client: the CA to verify the server, plus its own certificate and key:

```bash
etcdctl --endpoints https://10.0.0.11:2379 \
  --cacert ca.crt \
  --cert apiserver-etcd-client.crt \
  --key apiserver-etcd-client.key \
  endpoint health
```

What happens in that handshake, compared to HTTPS:

- `etcdctl` checks that etcd's certificate chains to `ca.crt` (not to the public roots) and that `10.0.0.11` is in its SANs.
- etcd sends a `CertificateRequest`. `etcdctl` responds with its certificate and a `CertificateVerify` proving it holds the private key.
- etcd checks the client certificate chains to `ca.crt` and is valid for `clientAuth`.
- Both are Go programs using Go's `crypto/tls`. With TLS 1.3 required, the cipher suite is one of the three above. With builds on Go 1.24 or later, the key exchange is the hybrid post-quantum `X25519MLKEM768` by default, with no configuration at all.

One quirk: `--cipher-suites` only affects TLS 1.2. Go doesn't let you choose TLS 1.3 suites, because all three are considered safe.

### How Kubernetes does it

If you build a cluster with kubeadm, all of this is done for you. Look in `/etc/kubernetes/pki/etcd/` on a control-plane node and you'll find:

| File | Purpose |
| --- | --- |
| `ca.crt`, `ca.key` | A dedicated etcd CA, separate from the Kubernetes cluster CA. |
| `server.crt` | etcd's client-facing certificate (`--cert-file`). |
| `peer.crt` | Used for member-to-member traffic (`--peer-cert-file`). |
| `healthcheck-client.crt` | Used by the liveness probe to call etcd. |

And in `/etc/kubernetes/pki/`, `apiserver-etcd-client.crt`, which the API server presents to etcd via `--etcd-certfile`, `--etcd-keyfile` and `--etcd-cafile`.[^7]

kubeadm certificates are valid for one year, and `kubeadm certs check-expiration` shows when each one expires. Upgrading the cluster renews them; clusters that are never upgraded have a famous habit of going down exactly one year after they were built.

### Common etcd TLS errors

Almost every etcd TLS problem maps directly to one of the checks in the handshake:

| Error | What it means |
| --- | --- |
| `x509: certificate signed by unknown authority` | The client's CA file isn't the CA that signed the server certificate (or vice versa). |
| `x509: certificate is valid for 10.0.0.11, not 10.0.0.12` | The address you connected to isn't in the certificate's SANs. Common after a node gets a new IP. |
| `remote error: tls: bad certificate` | The *other* side rejected your certificate: wrong CA, expired, or missing `clientAuth`. Check the logs on the other side. |
| `x509: certificate has expired or is not yet valid` | Expired certificate, or clocks out of sync between machines. |
| `tls: first record does not look like a TLS handshake` | One side is speaking plain HTTP: an `http://` URL on one end and `https://` on the other. |
| Peer rejected with a SAN/IP error | When `--peer-client-cert-auth` is on, etcd also checks the connecting peer's address against the SANs in that peer's certificate, so peer certificates need correct IP SANs, not just DNS names. |

And one warning: `--auto-tls` and `--peer-auto-tls` generate self-signed certificates at startup. The traffic is encrypted, but nobody is verified, which brings back exactly the man-in-the-middle problem TLS exists to solve. Fine for a test, not for anything real.

## HTTPS and etcd side by side

| | HTTPS (public website) | etcd |
| --- | --- | --- |
| Who is trusted | ~150 public root CAs shipped with the OS/browser | One private CA you created |
| Who issues certificates | Public CAs, via ACME (Let's Encrypt etc.) | You, kubeadm, cert-manager, Vault |
| Client certificate | No; users authenticate in HTTP | Yes; mutual TLS on both channels |
| Name the client checks | Hostname in the URL, against SAN | Hostname or IP from the endpoint URL, against SAN |
| Typical lifetime | 90 days today, heading to 45 | Usually one year, set by you |
| Revocation | CRLs pushed by browsers, short lifetimes | Practically none; rotate the CA if a key leaks |
| Extra protections | Certificate Transparency, HSTS, ECH | Network isolation, separate peer and client CAs |
| TLS versions | 1.2 and 1.3, for old clients | Can require 1.3 |

Same protocol, same handshake, same cryptography. The difference is entirely in **who decides whom to trust**.

## A small debugging toolkit

These three commands will diagnose most TLS problems, on the web or in etcd.

Look inside a certificate:

```bash
openssl x509 -in etcd1.crt -noout -text
```

Check the `Subject Alternative Name`, `Not After`, `Issuer` and `Extended Key Usage` lines first.

Check a certificate actually chains to a CA:

```bash
openssl verify -CAfile ca.crt etcd1.crt
```

Do a real handshake and see what the server sends:

```bash
# A website
openssl s_client -connect example.com:443 -servername example.com </dev/null

# etcd, presenting a client certificate
openssl s_client -connect 10.0.0.11:2379 \
  -CAfile ca.crt -cert apiserver-etcd-client.crt -key apiserver-etcd-client.key </dev/null
```

Look for `Verify return code: 0 (ok)`, the negotiated protocol and cipher, and the certificate chain the server actually presented (a missing intermediate is a classic cause of "works in my browser, fails in curl").

## Bottom line

- **TLS gives you confidentiality, integrity and authentication.** Authentication is the hard part, and it's what certificates are for.
- **It combines four tools:** ephemeral key exchange to agree on a secret (now hybrid post-quantum by default), symmetric AEAD ciphers to protect the data, signatures to prove identity, and certificates to bind keys to names.
- **TLS 1.3 needs one round trip**, encrypts most of the handshake, and removed every legacy option that caused trouble in the past.
- **HTTPS trusts the public CA system** and authenticates only the server. Certificate lifetimes are shrinking fast, so automate renewal.
- **etcd trusts a private CA** and uses mutual TLS on two channels, client and peer. Use `https://` everywhere, turn on `--client-cert-auth` and `--peer-client-cert-auth`, get the SANs right, require TLS 1.3, and keep an eye on expiry dates.

[^1]: [RFC 8446: The Transport Layer Security (TLS) Protocol Version 1.3](https://www.rfc-editor.org/rfc/rfc8446), IETF.
[^2]: [Announcing etcd v3.7](https://etcd.io/blog/2026/announcing-etcd-3.7/), etcd blog.
[^3]: [Go 1.24 release notes: crypto/tls](https://go.dev/doc/go1.24#cryptotlspkgcryptotls), on `X25519MLKEM768` becoming the default.
[^4]: [Ballot SC-081v3: Introduce Schedule of Reducing Validity and Data Reuse Periods](https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/), CA/Browser Forum.
[^5]: [Profiles](https://letsencrypt.org/docs/profiles/), Let's Encrypt documentation: certificate profiles, including six-day certificates.
[^6]: [Transport security model](https://etcd.io/docs/v3.6/op-guide/security/), etcd documentation: TLS flags, client and peer certificates, and examples.
[^7]: [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/), Kubernetes documentation: the certificates kubeadm generates for etcd and the API server.
