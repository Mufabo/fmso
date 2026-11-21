---
title: "An Introduction to cert-manager on Kubernetes"
date: 2026-11-20
draft: true
description: "What cert-manager is, how its Issuers, Certificates and ACME challenges fit together, how to install it, get Let's Encrypt certificates for a Gateway or Ingress with HTTP-01 and DNS-01, run your own internal CA, and debug issuance when it gets stuck."
summary: "TLS certificates expire, and they are about to expire a lot faster. cert-manager turns them into ordinary Kubernetes resources that request and renew themselves. Here is how it works and how to set it up."
tags: ["kubernetes", "tls", "cert-manager", "letsencrypt", "acme", "gateway-api", "security"]
categories: ["Programming"]
---

Every HTTPS endpoint needs a certificate, and every certificate expires. For years the answer was a calendar reminder and a shell script. That was already fragile with 90-day Let's Encrypt certificates, and lifetimes are getting shorter: Let's Encrypt's opt-in `tlsserver` profile has issued 45-day certificates since May 2026, the default profile drops to 64 days in February 2027, and to 45 days in February 2028[^1]. Nobody should be renewing those by hand.

**cert-manager** is the standard way to automate this in Kubernetes. It's a CNCF project that adds certificates and certificate authorities to the Kubernetes API as custom resources. You describe the certificate you want, cert-manager gets it from a CA, stores it in a Secret, and renews it before it expires. This post covers the moving parts, installation, public certificates from Let's Encrypt, a private CA for internal services, and how to troubleshoot.

Everything here is based on cert-manager v1.21.3, the current release at the time of writing[^2]. Version 1.22 is due around now[^3], but nothing below should change.

## The resources

cert-manager introduces six custom resources. You write the first three; cert-manager creates the rest.

| Resource | Scope | What it is |
| --- | --- | --- |
| `Issuer` | Namespace | A CA that can sign certificates for that namespace only. |
| `ClusterIssuer` | Cluster | The same, but usable from any namespace. |
| `Certificate` | Namespace | "I want a certificate for these names, from this issuer, stored in this Secret." |
| `CertificateRequest` | Namespace | One signing request for one certificate. Created by cert-manager for each issuance or renewal. |
| `Order` | Namespace | An ACME order with the CA. Only for ACME issuers. |
| `Challenge` | Namespace | One proof of domain control (HTTP-01 or DNS-01) within an order. |

The flow for a Let's Encrypt certificate looks like this:

1. You create a `Certificate` (or annotate a Gateway or Ingress and cert-manager creates it for you).
2. cert-manager generates a private key, stores it in a temporary Secret, and creates a `CertificateRequest` containing a CSR.
3. The ACME issuer turns that into an `Order`, and the CA responds with one `Challenge` per domain name.
4. cert-manager solves each challenge, either by serving a token over HTTP or by creating a DNS TXT record.
5. The CA validates, signs, and cert-manager writes the certificate and key into the Secret named in the `Certificate`, as a `kubernetes.io/tls` Secret with `tls.crt`, `tls.key` and usually `ca.crt`.
6. Some time before expiry, cert-manager starts again at step 2.

For non-ACME issuers (a self-signed issuer, your own CA, Vault, a cloud private CA) steps 3 and 4 disappear: the issuer signs the CSR directly.

The Secret is the only thing your workload or Gateway sees. Everything else is cert-manager's bookkeeping, which is useful when something goes wrong, because each step leaves a resource with a status you can inspect.

## Installing cert-manager

The project recommends the Helm chart from its OCI registry[^4]:

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.3 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

`crds.enabled=true` lets the chart install the CRDs. The legacy chart repository at `charts.jetstack.io` still exists, but new releases show up there a few hours after the OCI registry[^4].

This creates three Deployments:

- **cert-manager**, the controller that does the actual work.
- **cert-manager-webhook**, an admission webhook that validates and defaults cert-manager resources. (If you want to know how those work, see my post on writing your own admission webhook.)
- **cert-manager-cainjector**, which copies CA bundles into webhook and APIService configurations. More on that later.

Two things worth knowing about the chart:

- **Install it exactly once per cluster.** cert-manager manages cluster-scoped resources, so don't bundle it as a sub-chart of your own application chart[^4].
- **Uninstalling keeps the CRDs.** Since v1.15, `helm uninstall` leaves the CRDs in place on purpose, because deleting a CRD deletes every resource of that type in the cluster[^4]. Remove them yourself only when you mean it.

If you plan to use Gateway API, install the Gateway API CRDs first and enable support in cert-manager. It's no longer behind a feature gate, but it is off by default[^5]:

```bash
helm upgrade cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.3 \
  --namespace cert-manager \
  --reuse-values \
  --set config.gatewayAPI.enabled=true
```

cert-manager checks for the Gateway API CRDs at startup, so if you install them afterwards, restart the controller with `kubectl rollout restart deployment cert-manager -n cert-manager`[^5].

Verify the installation with `cmctl`, cert-manager's CLI:

```bash
cmctl check api --wait=2m
# The cert-manager API is ready
```

## Public certificates with Let's Encrypt

Let's Encrypt speaks **ACME**, a protocol in which you prove you control a domain by completing a challenge. cert-manager supports both kinds[^6]:

- **HTTP-01**: the CA fetches a token from `http://<your-domain>/.well-known/acme-challenge/<token>`. cert-manager starts a tiny solver Pod and routes that path to it through your Gateway or Ingress. Easy, but the domain must be reachable from the internet on port 80, and it can't issue wildcard certificates.
- **DNS-01**: the CA looks up a TXT record at `_acme-challenge.<your-domain>`. cert-manager creates it through your DNS provider's API. Works for wildcards and for services that aren't publicly reachable, but cert-manager needs credentials for your DNS zone.

### Start with staging

Let's Encrypt has strict rate limits in production. While you're getting things working, use the staging environment, which issues certificates from an untrusted CA but with far looser limits:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - name: public
                namespace: gateway
                kind: Gateway
```

A few fields need explaining:

- **`email`** is where Let's Encrypt sends account notices.
- **`privateKeySecretRef`** names the Secret holding your ACME *account* key, not a certificate key. cert-manager creates it on first use. It identifies you to the CA; if you lose it, you can register a new account, but you can no longer revoke certificates issued under the old one[^6].
- **`solvers`** says how to solve challenges. Here, HTTP-01 through a Gateway: cert-manager creates a temporary HTTPRoute attached to the `public` Gateway for each challenge. If you use Ingress instead, replace `gatewayHTTPRoute` with `ingress: { ingressClassName: <your-class> }`.

Once a staging certificate works end to end, create a second ClusterIssuer called `letsencrypt-production` with `server: https://acme-v02.api.letsencrypt.org/directory` and its own account key Secret, and switch your resources over.

### Choosing a profile

Since v1.18, the ACME issuer has a `profile` field that selects one of the CA's certificate profiles. Leave it out and Let's Encrypt uses `classic`[^6]:

```yaml
spec:
  acme:
    profile: tlsserver
```

`tlsserver` gets you 45-day certificates today[^1]. It's a cheap way to find out now whether anything in your setup assumes certificates last 90 days, rather than finding out in 2028.

### Requesting a certificate

The most explicit way is a `Certificate` resource:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: shop-example-com
  namespace: gateway
spec:
  secretName: shop-example-com-tls
  dnsNames:
    - shop.example.com
    - www.shop.example.com
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
```

Apply it and watch:

```bash
kubectl get certificate -n gateway -w
# NAME               READY   SECRET                 AGE
# shop-example-com   False   shop-example-com-tls   5s
# shop-example-com   True    shop-example-com-tls   48s
```

`rotationPolicy: Always` generates a fresh private key on every renewal instead of reusing the old one, which limits the damage from a leaked key. It's been the default since v1.18, but I like having it written down.

The Secret must live in the same namespace as the thing that uses it. For a Gateway, that's the Gateway's namespace, not the application's.

### Letting the Gateway ask for it

Writing a `Certificate` for every hostname gets old. cert-manager's **gateway-shim** can create them for you. Annotate the Gateway with an issuer, and for every listener that has a `hostname` and terminates TLS, cert-manager creates a Certificate whose Secret name matches the listener's `certificateRefs`[^5]:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public
  namespace: gateway
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-production
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
    - name: shop-https
      hostname: shop.example.com
      port: 443
      protocol: HTTPS
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - name: shop-example-com-tls
```

Keep the plain HTTP listener: the HTTP-01 solver needs it.

The Secret is created in the Gateway's namespace, which is a limitation[^5]: every new hostname means editing a shared Gateway. Gateway API's **ListenerSet** addresses this by letting application teams attach their own listeners. cert-manager supports annotating ListenerSets since v1.20, behind a feature gate (`config.gatewayAPI.enableListenerSet=true` and `config.featureGates.ListenerSets=true`), and it needs Gateway API v1.5 or later[^5]. (My post on the Gateway API covers listeners and routes in more detail.)

### Or the Ingress

The same works for Ingress with the **ingress-shim**[^7]. Add the annotation and a `tls` section:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
  namespace: shop
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-production
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-example-com-tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop
                port:
                  number: 80
```

cert-manager creates a Certificate named after the Secret, in the Ingress's namespace.

### Wildcards and private hosts with DNS-01

For `*.example.com`, or for a host that isn't reachable from the internet, use DNS-01. cert-manager supports a list of DNS providers out of the box (Cloudflare, Route 53, Google Cloud DNS, Azure DNS, DigitalOcean, RFC 2136 and others) and many more through webhook plugins[^8]. With Cloudflare, create an API token limited to *Zone → DNS → Edit* on the zone you need, and store it in a Secret:

```bash
kubectl create secret generic cloudflare-api-token \
  -n cert-manager \
  --from-literal=api-token="$CLOUDFLARE_API_TOKEN"
```

For a ClusterIssuer, Secrets it references must live in the **cluster resource namespace**, which is `cert-manager` by default. That trips up a lot of people.

Then add a DNS-01 solver. Solvers can have selectors, so one issuer can use DNS-01 for some names and HTTP-01 for the rest:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-production-account-key
    solvers:
      # Anything in example.com, including wildcards, via DNS
      - selector:
          dnsZones:
            - example.com
        dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
      # Everything else via HTTP
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - name: public
                namespace: gateway
                kind: Gateway
```

A wildcard certificate is then just another `dnsNames` entry:

```yaml
spec:
  secretName: wildcard-example-com-tls
  dnsNames:
    - example.com
    - "*.example.com"
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
```

If the challenges hang in `pending` with DNS-01, the usual cause is that cert-manager checks propagation against DNS servers that see a different view of your zone, such as an internal split-horizon resolver. The `--dns01-recursive-nameservers` and `--dns01-recursive-nameservers-only` controller flags point it at public resolvers instead.

## Your own CA for internal services

Let's Encrypt only issues for public domain names. For service-to-service TLS inside the cluster, for webhooks, or for `*.svc.cluster.local` names, you want your own CA. cert-manager can bootstrap one with two built-in issuer types[^9][^10]:

- **SelfSigned** signs a certificate with its own key. Useful for exactly one thing: creating a root CA.
- **CA** signs certificates with a CA key pair stored in a Secret.

The pattern is a three-step chain:

```yaml
# 1. A self-signed issuer, only used to create the root
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
# 2. The root CA certificate, signed by itself
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-root-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: Example Internal Root CA
  secretName: internal-root-ca
  duration: 87600h      # 10 years
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
---
# 3. An issuer that signs with that root
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-root-ca
```

The root CA Secret is in `cert-manager`, the cluster resource namespace, because `internal-ca` is a ClusterIssuer. Now any namespace can request internal certificates:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: orders-api
  namespace: shop
spec:
  secretName: orders-api-tls
  duration: 720h        # 30 days
  dnsNames:
    - orders-api
    - orders-api.shop.svc
    - orders-api.shop.svc.cluster.local
  usages:
    - server auth
    - client auth
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
```

Because nothing needs to be validated over the internet, internal certificates can be short-lived. 30 days or less is reasonable.

This setup is fine for a homelab or a development cluster. For production, think about where the root key lives. Here it's a Secret in etcd, so the advice from my post on Kubernetes Secrets applies in full: encrypt at rest, and restrict who can read Secrets or create Pods in `cert-manager`. A step up is an offline root that signs an intermediate, and only putting the intermediate in the cluster. Further still, cert-manager can delegate signing to **Vault**, **AWS Private CA**, **Google CAS** or a **Venafi/CyberArk** platform, so the CA key never enters the cluster.

### Distributing trust

Clients need to trust your root. Two companion projects help:

- **cainjector** (already installed) fills in `caBundle` fields on webhook configurations, CRD conversion webhooks and APIServices. Annotate the resource with `cert-manager.io/inject-ca-from: <namespace>/<certificate>` and it keeps the bundle current. Many operators use this so their webhooks get TLS without manual setup.
- **trust-manager** distributes CA bundles to applications. A `Bundle` resource collects certificates from Secrets, ConfigMaps or the public CA set, and writes them into a ConfigMap in every namespace, which Pods then mount.

## Renewal

By default, cert-manager renews when two thirds of a certificate's lifetime has passed. A 90-day certificate renews with about 30 days left, a 45-day one with about 15. You can override it per Certificate with `renewBefore` (a duration) or `renewBeforePercentage`[^11].

When the new certificate arrives, cert-manager updates the Secret in place. What happens next depends on the consumer:

- **Gateway and Ingress controllers** watch the Secret and reload it. Nothing to do.
- **Pods that mount the Secret as a volume** see the new files within a minute or two (except with `subPath` mounts, which never update). The application must re-read them. Many do this already; Go's `tls.Config.GetCertificate` makes it easy.
- **Pods that use environment variables, or read the file once at startup**, keep the old certificate until they restart. A tool like Stakater Reloader can roll the Deployment when the Secret changes.

If neither option suits you, the **csi-driver** project mounts certificates straight into Pods via a CSI volume, with a key generated on the node that never appears in a Secret.

To force a renewal, for example after changing an issuer:

```bash
cmctl renew shop-example-com -n gateway
```

## Troubleshooting

When a Certificate stays `READY False`, walk down the chain of resources. Each one's status and Events tell you where it's stuck[^12]:

```bash
# 1. The Certificate: what's the overall state?
kubectl describe certificate shop-example-com -n gateway

# 2. The current CertificateRequest
kubectl get certificaterequest -n gateway
kubectl describe certificaterequest shop-example-com-1 -n gateway

# 3. For ACME: the Order and its Challenges
kubectl get order,challenge -n gateway
kubectl describe challenge <name> -n gateway
```

`cmctl status certificate shop-example-com -n gateway` shows the whole chain at once, and is usually where I start.

The usual suspects:

| Symptom | Likely cause |
| --- | --- |
| Challenge says `Waiting for HTTP-01 challenge propagation` | The solver route isn't reachable from the internet. Check DNS points at your Gateway or load balancer, port 80 is open, and nothing redirects `/.well-known/acme-challenge/` to HTTPS with a broken certificate. |
| Challenge says `Waiting for DNS-01 challenge propagation` | Wrong zone, missing API permissions, or cert-manager checking against a split-horizon resolver. |
| `secret "..." not found` on a ClusterIssuer | The referenced Secret isn't in the cluster resource namespace (`cert-manager`). |
| Order `errored` with `rateLimited` | You've hit Let's Encrypt limits. Wait, and use staging while experimenting. |
| Gateway annotation does nothing | Gateway API support isn't enabled, or cert-manager started before the Gateway API CRDs existed. |
| Browser warns about an untrusted issuer | You're still on the staging issuer. |

cert-manager also exposes Prometheus metrics, including `certmanager_certificate_expiration_timestamp_seconds` and `certmanager_certificate_ready_status`. An alert on any certificate that is not ready, or that expires within a week, catches nearly every failure long before users do. With 45-day certificates, a failed renewal leaves you about two weeks; that alert is worth five minutes of your time.

## Good practices

- **Use staging first**, then switch to production once issuance works.
- **Prefer ClusterIssuers for shared CAs** and namespaced Issuers when a team should manage its own.
- **Scope DNS credentials narrowly.** A token that can edit one zone's DNS records is enough; a global API key is not appropriate.
- **Protect the `cert-manager` namespace.** It holds your ACME account keys, DNS credentials and possibly your CA key. Few people should be able to read Secrets or create Pods there.
- **Rotate private keys on renewal** (`rotationPolicy: Always`, the default since v1.18).
- **Make sure workloads pick up renewed certificates**: watch the files, or restart on change.
- **Test short lifetimes now**, with the `tlsserver` profile or short internal certificates, before Let's Encrypt's defaults shrink.
- **Alert on expiry and readiness metrics.**

## Bottom line

- **cert-manager makes certificates Kubernetes resources**: Issuers describe CAs, Certificates describe what you want, and cert-manager handles the rest.
- **Install it once per cluster** from the OCI Helm chart, and enable Gateway API support if you use it.
- **Use Let's Encrypt for public names**: HTTP-01 for simple cases, DNS-01 for wildcards and private hosts.
- **Annotate Gateways or Ingresses** to get certificates without writing Certificate resources.
- **Bootstrap a private CA** with SelfSigned and CA issuers for internal TLS, and move the key out of the cluster when it matters.
- **Renewal is automatic, but reloading isn't always.** Check that your workloads notice, and alert on expiry.

## References

[^1]: [Decreasing Certificate Lifetimes to 45 Days](https://isrg.org/post/from-90-to-45/), ISRG / Let's Encrypt: timeline for the `tlsserver` and `classic` profiles.
[^2]: [Helm installation](https://cert-manager.io/docs/installation/helm/), cert-manager documentation: current version and install command.
[^3]: [Supported Releases](https://cert-manager.io/docs/releases/), cert-manager documentation: release dates, support windows and Kubernetes compatibility.
[^4]: [Helm installation](https://cert-manager.io/docs/installation/helm/), cert-manager documentation: OCI chart, CRD handling, uninstalling, and why not to use it as a sub-chart.
[^5]: [Annotated Gateway resource](https://cert-manager.io/docs/usage/gateway/), cert-manager documentation: enabling Gateway API support, gateway-shim, ListenerSet and the `gatewayHTTPRoute` solver.
[^6]: [ACME](https://cert-manager.io/docs/configuration/acme/), cert-manager documentation: ACME issuer configuration, HTTP-01 and DNS-01, account keys and profiles.
[^7]: [Securing Ingress resources](https://cert-manager.io/docs/usage/ingress/), cert-manager documentation: ingress-shim annotations.
[^8]: [Cloudflare DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/), cert-manager documentation: API tokens and solver configuration.
[^9]: [SelfSigned](https://cert-manager.io/docs/configuration/selfsigned/), cert-manager documentation: bootstrapping a CA issuer.
[^10]: [CA](https://cert-manager.io/docs/configuration/ca/), cert-manager documentation: issuing from a CA key pair in a Secret.
[^11]: [Certificate resource](https://cert-manager.io/docs/usage/certificate/), cert-manager documentation: duration, renewal, private key settings and rotation policy.
[^12]: [Troubleshooting](https://cert-manager.io/docs/troubleshooting/), cert-manager documentation: walking from Certificate to Challenge.
