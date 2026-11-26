---
title: "Service Meshes on Kubernetes: What They Do, How They Work, and Whether You Need One"
date: 2026-11-25
draft: true
description: "What a service mesh is, the problems it solves (mTLS, traffic management, observability), how sidecar, ambient and eBPF data planes differ, a hands-on tour of Istio ambient mode, and how Istio, Linkerd and Cilium compare."
summary: "A service mesh moves encryption, retries, traffic shifting and telemetry out of your application code and into the network. Here is how meshes work on Kubernetes, how sidecar, ambient and eBPF designs differ, and how to tell whether you need one at all."
tags: ["kubernetes", "service mesh", "istio", "linkerd", "cilium", "envoy", "mtls", "gateway api", "networking"]
---

Kubernetes gives every pod an IP address and every Service a stable name, and then it more or less stops. It will route a packet from pod A to pod B, but it has no opinion on whether that packet should be encrypted, whether A is even *allowed* to talk to B, what to do when B is slow, or how long the request took.

For a long time, the answer was "the application handles it". Every service got an HTTP client library with retries, timeouts, circuit breakers, TLS configuration and metrics. That works, until you have forty services in four languages, each with its own slightly different retry logic and its own idea of what a timeout is.

A **service mesh** takes those concerns out of the application and puts them into the infrastructure. This post covers what a mesh does, how it works under the hood, the three data-plane designs you will run into (sidecars, ambient, eBPF), a hands-on walk through Istio, and the honest question of whether you need one at all.

## What a service mesh is

A service mesh is a dedicated infrastructure layer that handles **service-to-service communication**. It is made of two parts:

- **The data plane**: a set of proxies that sit in the path of every request between your services. They encrypt traffic, enforce policy, retry failed calls, and record metrics.
- **The control plane**: a set of controllers that watch the Kubernetes API and your mesh configuration, then push the resulting configuration and certificates out to the proxies.

```text
                      ┌──────────────────────┐
                      │    Control plane     │
                      │ (config, certs, CA)  │
                      └──────────┬───────────┘
                     config/certs│(xDS)
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
   ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
   │ Pod: frontend    │ │ Pod: orders      │ │ Pod: payments    │
   │ ┌─────┐ ┌──────┐ │ │ ┌──────┐ ┌─────┐ │ │ ┌──────┐ ┌─────┐ │
   │ │ app │►│proxy │═╪═╪►│proxy │►│ app │ │ │ │proxy │►│ app │ │
   │ └─────┘ └──────┘ │ │ └──────┘ └─────┘ │ │ └──────┘ └─────┘ │
   └──────────────────┘ └────────┬─────────┘ └────────▲─────────┘
                                 ╚════════ mTLS ══════╝
```

The application keeps making plain HTTP or gRPC calls to `http://orders`. The proxy next to it intercepts the call, wraps it in mutual TLS, picks a healthy endpoint, applies timeouts and retries, and reports what happened. The application doesn't know any of this is going on.

Most meshes use **Envoy** as their proxy, a high-performance L7 proxy originally built at Lyft, configured dynamically over a set of APIs known as **xDS**.[^1] Linkerd is the notable exception, with its own purpose-built proxy written in Rust.[^2]

## What a mesh gives you

### Mutual TLS and workload identity

This is the feature that sells most meshes. Every workload gets a cryptographic identity, usually derived from its Kubernetes ServiceAccount, and every connection between workloads is encrypted and authenticated in **both directions**.

Istio, for example, encodes identity as a **SPIFFE** ID,[^3] which looks like this:

```text
spiffe://cluster.local/ns/shop/sa/orders
```

The mesh runs its own certificate authority, issues short-lived certificates (typically valid for 24 hours), and rotates them automatically. No one has to generate keys, mount secrets or remember to renew anything.

This matters for two reasons. First, traffic inside the cluster is encrypted, which compliance teams like and which protects you from anyone sniffing the node network. Second, and more importantly, **identity is no longer an IP address**. A NetworkPolicy says "pods with label `app=frontend` may connect to port 8080". A mesh authorization policy says "the workload that can prove it is `sa/frontend` may call `GET /orders`". That's a much stronger statement, and it's the network half of a zero trust architecture.

### Traffic management

Because every request passes through a proxy that understands HTTP and gRPC, the mesh can make routing decisions per request rather than per connection:

- **Canary releases and traffic splitting**: send 5% of requests to `v2`, watch the error rate, increase gradually.
- **Header-based routing**: send requests with `x-user-group: beta` to the new version.
- **Retries and timeouts**: configured once, consistently, outside the code.
- **Circuit breaking and outlier detection**: stop sending traffic to an endpoint that keeps failing.
- **Fault injection**: deliberately add latency or errors to test how your system copes.
- **Better load balancing**: kube-proxy balances *connections*. With long-lived HTTP/2 or gRPC connections, that means one backend pod can end up with all the traffic. An L7 proxy balances *requests*.

That last point is easy to overlook and is, on its own, a common reason teams adopt a mesh for gRPC-heavy systems.

### Observability

Every proxy emits the same "golden signals" for every request: rate, errors and latency, broken down by source, destination, route and response code. You get a service dependency graph and consistent metrics across all services, regardless of language, without touching any code.

The mesh can also propagate and emit **tracing** spans, but there is a catch people often miss: the proxy can't link an incoming request to the outgoing requests the application makes in response. The application still has to forward trace headers (`traceparent` and friends). The mesh gives you the hops; you still need to pass the baton.

## How traffic gets into the proxy

The proxy has to see the traffic before it can do anything with it. How it gets there is the main way meshes differ from each other, and it has changed a lot in the last few years.

### Sidecars

The classic design. A **mutating admission webhook** injects an extra proxy container into every pod when it is created, plus an init container (or a CNI plugin) that sets up `iptables` rules redirecting all inbound and outbound traffic in the pod's network namespace through the proxy.

```text
┌──────────────────────── Pod ────────────────────────┐
│                                                     │
│   ┌───────┐  localhost   ┌────────────┐             │
│   │  app  │ ───────────► │   proxy    │ ──► network │
│   └───────┘   (iptables  │ (sidecar)  │             │
│                redirect) └────────────┘             │
└─────────────────────────────────────────────────────┘
```

Sidecars are well understood and give every workload full L7 features, but they come with real costs:

- **Resources.** One proxy per pod. A thousand pods means a thousand Envoys, each reserving CPU and memory.
- **Lifecycle problems.** Historically, the sidecar might not be ready when the app started, or might exit before the app finished draining. Jobs never completed because the sidecar kept running. Kubernetes fixed most of this with **native sidecar containers** (init containers with `restartPolicy: Always`), which start before and stop after the main containers; the feature went GA in Kubernetes 1.33.[^4]
- **Upgrades.** Updating the proxy means restarting every pod in the mesh.
- **Invasiveness.** The mesh modifies your pod spec, which surprises people the first time they see an extra container in `kubectl get pods`.

### Ambient (sidecar-less)

Istio's **ambient mode**, which reached general availability in Istio 1.24 in November 2024, splits the data plane into two layers:[^5]

- **ztunnel**: a lightweight, Rust-based proxy running as a DaemonSet, **one per node**. It handles L4 only: mTLS, identity, L4 authorization, and TCP telemetry. Traffic between ztunnels travels in a tunnel protocol called HBONE (HTTP CONNECT over mTLS).
- **Waypoint proxies**: optional Envoy deployments, typically **one per namespace or per service**, that you add only where you need L7 features such as HTTP routing, retries or path-based authorization.

```text
 Node 1                                 Node 2
┌──────────────────────────┐          ┌──────────────────────────┐
│ ┌──────┐                 │          │                 ┌──────┐ │
│ │ app  │──► ┌─────────┐  │  HBONE   │  ┌─────────┐ ──►│ app  │ │
│ └──────┘    │ ztunnel │══╪══(mTLS)══╪═►│ ztunnel │    └──────┘ │
│             └─────────┘  │    ║     │  └─────────┘             │
└──────────────────────────┘    ║     └──────────────────────────┘
                                ▼ (only if L7 is needed)
                         ┌──────────────┐
                         │   waypoint   │
                         │   (Envoy)    │
                         └──────────────┘
```

The pods themselves are untouched: no injected container, no restarts to join or leave the mesh, and proxy upgrades don't require rolling your workloads. You pay for L7 processing only where you use it. The trade-off is a newer, less battle-tested architecture and a shared per-node component, which makes noisy-neighbour and blast-radius questions slightly different from the sidecar model.

### eBPF and the CNI

**Cilium** comes at the problem from the other direction. It is already the CNI in many clusters, so it uses **eBPF** programs in the kernel to handle L3/L4 load balancing, network policy and observability without any proxy at all, and hands traffic to a **per-node Envoy** only when L7 processing is needed.[^6]

This is efficient and fits naturally if you already run Cilium. The main caveat is mutual authentication: Cilium's approach, based on SPIFFE identities with a separate handshake, isn't equivalent to per-connection mTLS in the Istio and Linkerd sense, and it was still marked beta at the time of writing. If encryption on the wire is the requirement, Cilium typically relies on WireGuard or IPsec node-to-node encryption instead, which encrypts traffic but authenticates nodes rather than workloads.

## Hands-on: Istio in ambient mode

Here is a minimal walk-through to make all this concrete. It assumes a test cluster (kind or minikube is fine) and the `istioctl` CLI.

### Install and add a namespace to the mesh

```sh
# Install Istio with the ambient profile (istiod, CNI node agent, ztunnel)
istioctl install --set profile=ambient --skip-confirmation

# Gateway API CRDs, used for routing below
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# Put a namespace into the mesh. No pod restarts needed.
kubectl create namespace shop
kubectl label namespace shop istio.io/dataplane-mode=ambient
```

From this moment, every pod in `shop` has mTLS for all traffic to other meshed pods, and ztunnel reports L4 telemetry for it. Compare that with sidecar mode, where you would label the namespace `istio-injection=enabled` and then restart every deployment to get the proxy injected.

### Require mTLS

By default, Istio runs in *permissive* mode: meshed workloads accept both mTLS and plain text, which makes gradual migration possible. Once everything is in the mesh, lock it down:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: shop
spec:
  mtls:
    mode: STRICT
```

Now any plain-text connection to a workload in `shop` is rejected.[^7]

### Authorize by identity

Allow only the `frontend` service account to reach `orders`:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: orders-allow-frontend
  namespace: shop
spec:
  selector:
    matchLabels:
      app: orders
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/shop/sa/frontend"]
```

Once an `ALLOW` policy selects a workload, anything that doesn't match a rule is denied.[^8] This policy is pure L4 (it only checks *who* is connecting), so ztunnel can enforce it without a waypoint. Add a rule on HTTP methods or paths and you'll need a waypoint for it to be enforced.

### Add a waypoint and split traffic

For L7 features, deploy a waypoint for the namespace and tell the namespace to use it:

```sh
istioctl waypoint apply --namespace shop --enroll-namespace
```

Now you can shift traffic with a standard Gateway API `HTTPRoute`. Note that the `parentRef` is a **Service**, not a Gateway. That's the mesh flavour of the Gateway API, defined by the GAMMA initiative,[^9] and it means the same resource works across every mesh that implements it:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orders-canary
  namespace: shop
spec:
  parentRefs:
    - group: ""
      kind: Service
      name: orders
      port: 8080
  rules:
    - backendRefs:
        - name: orders-v1
          port: 8080
          weight: 95
        - name: orders-v2
          port: 8080
          weight: 5
      timeouts:
        request: 2s
      retry:
        attempts: 3
        codes: [503]
```

Clients keep calling `orders:8080`. Ninety-five percent of requests go to `orders-v1`, five percent to `orders-v2`, each request gets a two-second timeout, and 503s are retried. (The `retry` field is newer than the rest of `HTTPRoute`, so check your mesh's Gateway API support; Istio's own `VirtualService` resource can do the same thing if it isn't available.)

### Look at it

```sh
istioctl ztunnel-config workloads        # what ztunnel knows about each pod
istioctl dashboard kiali                 # service graph (requires the Kiali addon)
```

## Istio, Linkerd and Cilium compared

| | **Istio** | **Linkerd** | **Cilium** |
|---|---|---|---|
| Data plane | Sidecar (Envoy) or ambient (ztunnel + Envoy waypoints) | Sidecar (linkerd2-proxy, Rust) | eBPF in the kernel + per-node Envoy for L7 |
| mTLS | Yes, automatic | Yes, on by default | Mutual auth (beta) or WireGuard/IPsec node encryption |
| Feature set | Very broad: routing, multi-cluster, egress control, VM support, extensibility | Deliberately small: mTLS, retries, timeouts, traffic split, metrics | Strong at L3/L4 and network policy; L7 via Envoy |
| Operational complexity | Highest, though ambient lowers it considerably | Lowest of the three | Low if Cilium is already your CNI |
| Configuration | Istio CRDs + Gateway API | Linkerd CRDs + Gateway API | CiliumNetworkPolicy + Gateway API |
| CNCF status | Graduated | Graduated | Graduated |

A note on Linkerd licensing: since early 2024, the open source project publishes only **edge** releases. Versioned **stable** releases come from Buoyant, the company behind Linkerd, as Buoyant Enterprise for Linkerd, which is free for small companies but paid beyond that.[^10] The code is still Apache-licensed, but if you want stable releases with backported fixes from upstream, check the current terms before choosing it.

Roughly:

- **Linkerd** if you want mTLS, golden metrics and sane retries with the least possible operational overhead, and you don't need the long tail of features.
- **Istio ambient** if you want the full feature set, plan to grow into multi-cluster or fine-grained L7 policy, or need egress control, and want to avoid sidecar overhead.
- **Cilium** if it's already your CNI and your needs are mostly L4 policy, observability and occasional L7 routing.

## Do you actually need one?

A mesh is not free. Even in ambient mode, you're adding a control plane to keep running and upgrade, a new class of failure ("is it my app or the mesh?"), extra latency on every hop (small, usually well under a millisecond per proxy for L4, more for L7), and a sizeable set of new CRDs your team has to understand. Debugging a request that fails somewhere between two proxies is a skill in itself.

**You probably need a mesh if:**

- You must encrypt and authenticate all service-to-service traffic (compliance, multi-tenant clusters, zero trust requirements), and doing it per application isn't realistic.
- You have many services in several languages and want consistent retries, timeouts and telemetry without reimplementing them everywhere.
- You run lots of gRPC or HTTP/2 and are suffering from connection-level load balancing.
- You do frequent progressive delivery and want traffic splitting that doesn't depend on replica counts.

**You probably don't need one if:**

- You have a handful of services, mostly in one language, where a good shared client library covers retries and metrics.
- Your security requirement is really "pods in namespace A shouldn't reach namespace B". A plain **NetworkPolicy** does that, with no extra moving parts.
- Your traffic is mostly north-south. An ingress controller or Gateway API implementation at the edge may be all you need.
- No one on the team has time to own it. An unowned mesh is worse than no mesh.

If you're unsure, a reasonable path is to start small: install a mesh in permissive mode, enrol one or two namespaces, use it for mTLS and metrics only, and add traffic management when a concrete problem asks for it. Ambient mode makes this especially cheap, since joining and leaving the mesh is a namespace label rather than a fleet-wide restart.

## Summary

- A service mesh moves **encryption, identity, traffic control and telemetry** out of application code and into proxies managed by a central control plane.
- Its biggest win is **workload identity with automatic mTLS**: policies based on *who* is calling, not which IP they happen to have.
- **Sidecars** give every pod a full proxy at a high resource and lifecycle cost; **ambient** splits L4 (per node) from L7 (per namespace, on demand); **eBPF** meshes push as much as possible into the kernel.
- The **Gateway API**, via GAMMA, is becoming the common language for mesh routing, so `HTTPRoute` skills carry across implementations.
- A mesh solves real problems at a real operational cost. Adopt it when you have the problems, not because the architecture diagram looks more complete with one.

[^1]: [What is Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy), Envoy Proxy documentation.
[^2]: [Architecture](https://linkerd.io/2-edge/reference/architecture/), Linkerd documentation.
[^3]: [SPIFFE Overview](https://spiffe.io/docs/latest/spiffe-about/overview/), SPIFFE project.
[^4]: [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/), Kubernetes documentation.
[^5]: [Fast, Secure, and Simple: Istio's Ambient Mode Reaches General Availability in v1.24](https://istio.io/latest/blog/2024/ambient-reaches-ga/), Istio blog, November 2024.
[^6]: [Service Mesh](https://docs.cilium.io/en/stable/network/servicemesh/), Cilium documentation.
[^7]: [PeerAuthentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/), Istio documentation.
[^8]: [Authorization Policy](https://istio.io/latest/docs/reference/config/security/authorization-policy/), Istio documentation.
[^9]: [The GAMMA Initiative (Gateway API for Service Mesh)](https://gateway-api.sigs.k8s.io/concepts/gamma/), Kubernetes Gateway API project.
[^10]: [Linkerd Releases](https://linkerd.io/releases/), Linkerd project.
