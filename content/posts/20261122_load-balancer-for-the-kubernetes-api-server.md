---
title: "Putting a Load Balancer in Front of the Kubernetes API Server"
date: 2026-11-22
draft: true
description: "How to make the Kubernetes control plane highly available with a load balancer in front of kube-apiserver: the controlPlaneEndpoint, TCP passthrough vs TLS termination, health checks, HAProxy plus keepalived, kube-vip, cloud load balancers, and the pitfalls (port clashes, certificate SANs, the kube-vip super-admin.conf workaround)."
summary: "Three control plane nodes give you nothing if every kubeconfig points at just one of them. Here is how to put a stable, health-checked address in front of kube-apiserver with HAProxy and keepalived, kube-vip, or a cloud load balancer."
tags: ["kubernetes", "kubeadm", "high availability", "haproxy", "keepalived", "kube-vip", "load balancing", "cka"]
categories: ["Programming"]
---

In [Deploy a Kubernetes Cluster with kubeadm](/posts/deploy-kubernetes-cluster-with-kubeadm/) I insisted on setting `controlPlaneEndpoint` to a DNS name, even for a single control plane node. This post is the payoff: turning that name into a **load-balanced, highly available address** in front of several API servers.

## Why the API server needs a load balancer

Every component that talks to the cluster talks to `kube-apiserver` on port 6443: kubectl, kubelets on every node, kube-proxy, CNI agents, controllers, operators, your CI pipeline. Each of them finds it through a single URL in a kubeconfig.

You can run three API servers, but if every kubeconfig says `https://10.0.0.10:6443`, losing `cp1` still takes the cluster's control plane down for every client. What you need is **one stable address that always reaches a healthy API server**:

```text
       kubectl · kubelets · kube-proxy · CI · operators
                            │
                            ▼
              k8s-api.lab.internal:6443   (VIP or LB address)
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
     cp1:6443          cp2:6443          cp3:6443
  kube-apiserver    kube-apiserver    kube-apiserver
     etcd              etcd              etcd       (stacked etcd)
```

The API server is stateless (all state is in etcd), so any instance can answer any request. That makes it an easy thing to load-balance. The work is in the details.

## Design decisions

### 1. Layer 4, not layer 7

Use **TCP passthrough** (layer 4). The load balancer forwards the TLS connection untouched, and the API server terminates TLS itself.

This matters because Kubernetes clients very often authenticate with **client certificates** (kubelets, controller manager, scheduler, the kubeadm admin user). If the load balancer terminates TLS, the client certificate ends at the load balancer and the API server never sees it.[^1] You *can* make L7 work with the front-proxy/request-header mechanism, but there's rarely a reason to.

A layer 4 load balancer also handles long-lived connections (watches, `kubectl exec`, `logs -f`, port-forwards) without fuss, as long as idle timeouts are generous.

### 2. A health check that means "this API server works"

A TCP check (port open) isn't enough: the port can accept connections while the API server is still starting or can't reach etcd. Check over HTTPS instead:

- **`/readyz`**: the API server is ready to serve traffic. It also turns unhealthy during graceful shutdown, so the load balancer drains it before it stops. Prefer this.
- **`/livez`**: the process is alive. Good for restarts, too lenient for routing.
- `/healthz`: deprecated in favour of the two above, but still widely used in examples.

These endpoints are readable without authentication by default (the `system:public-info-viewer` ClusterRole grants them to anonymous users).[^2] If you've disabled anonymous auth, check that your API server still allows anonymous access to the health endpoints (the `AuthenticationConfiguration` file can permit anonymous requests to just those paths), or fall back to a TCP check.

The API server's certificate is signed by the cluster CA, which the load balancer doesn't trust, so health checks typically skip verification (`verify none`, `curl -k`). That's fine: it's a liveness probe, not a security boundary.

### 3. The certificate must match the address

Clients verify the API server's certificate against the name or IP they connected to. Whatever address the load balancer exposes, **its DNS name and/or IP must be in the API server certificate's SANs**. kubeadm adds `controlPlaneEndpoint` automatically; add anything else (the raw VIP, an external DNS name) under `apiServer.certSANs`:[^3]

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
controlPlaneEndpoint: k8s-api.lab.internal:6443
apiServer:
  certSANs:
    - k8s-api.lab.internal
    - 10.0.0.100          # the VIP
```

Get this wrong and kubectl reports `x509: certificate is valid for ..., not ...`.

### 4. Where the load balancer runs

| Option | How it works | Good for |
|---|---|---|
| **HAProxy + keepalived** on the control plane nodes | keepalived floats a virtual IP between nodes using VRRP; HAProxy on each node balances to all API servers | On-prem, bare metal, VMs on a flat L2 network |
| **kube-vip** as a static Pod | One component providing the VIP (ARP or BGP) and optionally load balancing | Same as above, with fewer moving parts |
| **Dedicated load balancer** (HAProxy/NGINX pair, F5, Citrix ADC) | Separate machines in front of the cluster | Environments that already have them |
| **Cloud load balancer** (AWS NLB, GCP passthrough NLB, Azure Load Balancer) | Managed L4 load balancer, backends = control plane instances | Self-managed clusters in the cloud |
| **DNS round robin** | Multiple A records | Not recommended: clients cache answers and nothing removes a dead node |

The kubeadm project documents the first two in detail.[^4]

## Option A: HAProxy + keepalived

This is the classic on-prem pattern. Lab addresses:

| Host | IP |
|---|---|
| VIP (`k8s-api.lab.internal`) | 10.0.0.100 |
| `cp1` | 10.0.0.10 |
| `cp2` | 10.0.0.11 |
| `cp3` | 10.0.0.12 |

Install on all three control plane nodes:

```bash
sudo apt-get install -y haproxy keepalived      # or dnf install
```

### The port clash

HAProxy runs on the same nodes as kube-apiserver, and kube-apiserver already listens on `0.0.0.0:6443`. So HAProxy can't also bind `*:6443`. Two ways out:

- **Different port**: HAProxy listens on `8443`, and `controlPlaneEndpoint` is `k8s-api.lab.internal:8443`. Simple, and what the kubeadm guide's templates assume.[^4]
- **Same port, bound only to the VIP**: set the API server's `bind-address` to the node IP and HAProxy to `10.0.0.100:6443`. Needs `net.ipv4.ip_nonlocal_bind=1` so HAProxy can bind the VIP on backup nodes that don't currently hold it.

I'll use port 8443.

### haproxy.cfg

```text
# /etc/haproxy/haproxy.cfg (same on all three nodes)
global
    log stdout format raw local0
    daemon

defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    timeout connect         5s
    timeout client          1h      # watches and exec sessions are long-lived
    timeout server          1h
    retries                 1

frontend apiserver
    bind *:8443
    default_backend apiserver

backend apiserver
    option httpchk
    http-check connect ssl
    http-check send meth GET uri /readyz
    http-check expect status 200
    balance roundrobin
    default-server inter 3s fall 3 rise 2
    server cp1 10.0.0.10:6443 check verify none
    server cp2 10.0.0.11:6443 check verify none
    server cp3 10.0.0.12:6443 check verify none
```

`mode tcp` is the passthrough; `option httpchk` with `http-check connect ssl` makes the *health check* speak HTTPS while traffic stays raw TCP.[^4]

Validate and start:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable --now haproxy
```

Before the first API server exists, all backends are down. That's expected.

### keepalived

keepalived elects one node to hold the VIP using VRRP and moves it when that node's health check fails.

The check script tests the *local* HAProxy, so a node only keeps the VIP while it can actually forward traffic:

```bash
# /etc/keepalived/check_apiserver.sh
#!/bin/sh
errorExit() { echo "*** $*" 1>&2; exit 1; }
curl -sfk --max-time 2 https://localhost:8443/readyz -o /dev/null \
  || errorExit "Error GET https://localhost:8443/readyz"
```

```bash
sudo chmod +x /etc/keepalived/check_apiserver.sh
```

```text
# /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
    enable_script_security
    script_user root
}

vrrp_script check_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state MASTER              # BACKUP on cp2 and cp3
    interface eth0            # the interface on 10.0.0.0/24
    virtual_router_id 51      # same on all nodes, unique on the L2 segment
    priority 101              # 100 on cp2 and cp3
    authentication {
        auth_type PASS
        auth_pass 42k8sVIP    # max 8 characters
    }
    virtual_ipaddress {
        10.0.0.100/24
    }
    track_script {
        check_apiserver
    }
}
```

The `weight -2` with priorities 101/100 means: when the master's check fails, its effective priority drops to 99, below the backups, and the VIP moves.[^4]

```bash
sudo systemctl enable --now keepalived
ip -brief addr show eth0       # 10.0.0.100 should appear on cp1
```

Things that break VRRP: firewalls dropping **IP protocol 112** (VRRP isn't TCP or UDP), cloud networks that don't allow gratuitous ARP or unknown IPs on an interface (on AWS, Azure and GCP use a cloud load balancer instead), and two clusters on one VLAN sharing a `virtual_router_id`.

### Running them as static Pods instead

Both can also run as static Pods from `/etc/kubernetes/manifests/`, so the kubelet manages them alongside the control plane. The manifests need `hostNetwork: true`, keepalived needs the `NET_ADMIN`, `NET_BROADCAST` and `NET_RAW` capabilities, and the config files are mounted from the host.[^4] It's tidier, but OS services are easier to debug when the kubelet itself is the problem.

## Option B: kube-vip

[kube-vip](https://kube-vip.io) replaces both keepalived and HAProxy with one static Pod per control plane node. It advertises the VIP either with **ARP** (layer 2, leader election through a Kubernetes Lease) or **BGP** (layer 3, peering with your routers), and can optionally load-balance across API servers too.[^4]

Generate the manifest **before** `kubeadm init`, on the first control plane node:

```bash
export VIP=10.0.0.100
export INTERFACE=eth0
export KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases \
  | jq -r ".[0].name")

sudo ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip \
  /kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --arp \
    --leaderElection \
  | sudo tee /etc/kubernetes/manifests/kube-vip.yaml
```

Because kube-vip holds the VIP on port 6443 directly (no separate proxy on the node), there's no port clash: `controlPlaneEndpoint` can be `k8s-api.lab.internal:6443`.

The static Pod approach exists precisely because of kubeadm's bootstrap order: the VIP has to answer *during* `kubeadm init`, before there's a cluster to run a DaemonSet in.[^5]

### The super-admin.conf workaround

Since Kubernetes v1.29, kubeadm writes `admin.conf` for a user that only gets `cluster-admin` through a ClusterRoleBinding *created during init*, plus a separate `super-admin.conf` in `system:masters`. kube-vip's generated manifest mounts `admin.conf`, which can't yet acquire the leader-election Lease when kube-vip starts. Without a leader there's no VIP, and `kubeadm init` times out waiting for the API server at the VIP.[^6]

The documented workaround is to point kube-vip at `super-admin.conf` **on the first node only, for the init**, then switch back:[^4][^7]

```bash
# before kubeadm init, on cp1 only
sudo sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' \
  /etc/kubernetes/manifests/kube-vip.yaml

sudo kubeadm init --config kubeadm-config.yaml --upload-certs

# after init succeeds
sudo sed -i 's#path: /etc/kubernetes/super-admin.conf#path: /etc/kubernetes/admin.conf#' \
  /etc/kubernetes/manifests/kube-vip.yaml
```

On `cp2` and `cp3`, generate the manifest unchanged (with `admin.conf`) before `kubeadm join --control-plane`; by then the binding exists.

Newer kubeadm versions also support `ControlPlaneKubeletLocalMode`, where the control plane kubelet talks to its *local* API server during bootstrap instead of the endpoint, which removes part of this chicken-and-egg problem.[^8] Check the feature's status for your version before relying on it.

Note that `admin.conf`'s client certificate expires after a year. kube-vip's docs recommend giving it its own kubeconfig and rotating that, rather than leaning on the admin credential long-term.[^4]

## Option C: a cloud load balancer

On AWS, Azure or GCP, VRRP and ARP-based VIPs mostly don't work, and you don't need them. Create a **layer 4, TCP** load balancer:

- **Listener**: TCP 6443 (not TLS, not HTTPS).
- **Targets**: the control plane instances on port 6443.
- **Health check**: HTTPS `GET /readyz`, expecting 200. If the platform's HTTPS check insists on certificate validation, use a TCP check.
- **Internal or external**: internal unless you really need the API on the internet. If external, restrict source ranges.
- **Idle timeout**: raise it if the default is short, or watches and `kubectl exec` sessions get cut.

Then point a DNS record (`k8s-api.example.internal`) at the load balancer and use that as `controlPlaneEndpoint`.

One gotcha: **hairpinning**. Some cloud L4 load balancers don't let a backend reach *itself* through the load balancer address, so a control plane node's kubelet connecting to the endpoint can time out whenever it's routed back to itself. Check your provider's docs. kubeadm sidesteps most of this because control plane components use the local API server address in their own kubeconfigs,[^5] but kubelets and anything else that uses the endpoint are still affected.

## Bootstrap the cluster through the load balancer

Whichever option you picked, the kubeadm part is the same:[^3]

```yaml
# kubeadm-config.yaml on cp1
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.0.10
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
controlPlaneEndpoint: k8s-api.lab.internal:8443   # :6443 with kube-vip or a cloud LB
networking:
  podSubnet: 10.244.0.0/16
apiServer:
  certSANs:
    - k8s-api.lab.internal
    - 10.0.0.100
```

```bash
# cp1
sudo kubeadm init --config kubeadm-config.yaml --upload-certs

# cp2, cp3 (command printed by init)
sudo kubeadm join k8s-api.lab.internal:8443 --token ... \
  --discovery-token-ca-cert-hash sha256:... \
  --control-plane --certificate-key ...

# workers
sudo kubeadm join k8s-api.lab.internal:8443 --token ... \
  --discovery-token-ca-cert-hash sha256:...
```

`--upload-certs` stores the shared CA and service account keys, encrypted, in a Secret so joining control plane nodes can fetch them. The uploaded certificates are deleted after two hours; regenerate with `sudo kubeadm init phase upload-certs --upload-certs`.

## Adding a load balancer to an existing cluster

If you built the cluster with a DNS name as `controlPlaneEndpoint` (as in my kubeadm post), this is easy: build the load balancer, repoint the name at it, and add more control plane nodes with `kubeadm join --control-plane`. Nothing else changes.

If the endpoint is a node's IP address, it's much more work, because that address is baked into the API server certificate, every kubeconfig on every node, the `kubeadm-config` and `cluster-info` ConfigMaps, and the kube-proxy ConfigMap. You need to add the new address to the certificate SANs, regenerate the API server certificate, update all of those, and restart the kubelets. It's doable, but it's the strongest argument for always setting `controlPlaneEndpoint` to a name from day one.

## Test the failover

Don't trust HA you haven't broken on purpose.

```bash
# from a workstation: watch the API continuously
while true; do
  kubectl get --raw /readyz >/dev/null 2>&1 && echo "$(date +%T) ok" || echo "$(date +%T) FAIL"
  sleep 1
done
```

In another terminal, take a control plane node out:

```bash
# on cp1: stop the API server by moving its static Pod manifest away
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
# ...watch the loop, then restore
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

Then try harder: power off the node holding the VIP (`ip addr` shows which one). With HAProxy + keepalived you should see at most a few failed requests while the VIP moves; HAProxy's stats or logs show the backend going `DOWN` and back `UP`.

With **three** control plane nodes and stacked etcd, you can lose **one**. Lose two and etcd loses quorum: the API servers stay up but can't read or write, and `/readyz` fails everywhere. The load balancer can't fix that.

## When it goes wrong

| Symptom | Likely cause | Check / fix |
|---|---|---|
| `kubeadm init` hangs at `wait-control-plane` / times out on the endpoint | VIP isn't up yet | `ip addr` for the VIP; kube-vip logs (`crictl logs`), super-admin.conf workaround |
| HAProxy fails to start: `cannot bind socket [0.0.0.0:6443]` | Port clash with kube-apiserver | Use a different frontend port, or bind HAProxy to the VIP only |
| All HAProxy backends `DOWN` | Health check failing | `curl -k https://10.0.0.10:6443/readyz` from the LB host; anonymous auth disabled? |
| `x509: certificate is valid for ..., not ...` | LB address not in SANs | Add to `apiServer.certSANs`, regenerate the API server cert |
| `kubectl exec` / `logs -f` / watches drop after a minute or so | LB idle timeout too short | Raise client/server timeouts (HAProxy) or LB idle timeout (cloud) |
| VIP on two nodes at once (split brain) | VRRP packets blocked | Allow IP protocol 112 between nodes; check `virtual_router_id` |
| VIP never moves on failure | Check script not failing, or weights wrong | Run the script by hand; `weight` must push the master below the backups |
| A control plane node can't reach the endpoint, others can | Cloud LB hairpinning | Provider docs; make local components use the local API server |
| Everything is `DOWN` after losing 2 of 3 nodes | etcd quorum lost | Restore a node, or restore etcd from a snapshot |

## Cheat sheet

```text
Design:
  L4 TCP passthrough on 6443 (client certs must reach the API server)
  Health check: HTTPS GET /readyz → 200, skip cert verification
  LB name/IP in apiServer.certSANs; controlPlaneEndpoint = DNS name
  Long idle timeouts (watches, exec, logs -f)

HAProxy + keepalived:
  haproxy: mode tcp, bind *:8443, option httpchk + http-check connect ssl, check verify none
  keepalived: VRRP VIP, check_apiserver.sh curls local :8443/readyz, weight -2
  allow IP proto 112; not on AWS/Azure/GCP

kube-vip:
  static Pod manifest in /etc/kubernetes/manifests BEFORE kubeadm init
  --controlplane --arp --leaderElection (or --bgp)
  first node, k8s ≥1.29: mount super-admin.conf for init, switch back to admin.conf

Cloud:
  TCP 6443 listener, HTTPS /readyz health check, internal, DNS name → LB, watch for hairpinning

kubeadm:
  kubeadm init --config … --upload-certs
  kubeadm join <endpoint> … --control-plane --certificate-key …

Test:
  loop kubectl get --raw /readyz; stop an apiserver; power off the VIP holder
  3 control plane nodes tolerate 1 failure (etcd quorum)
```

## References

[^1]: [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/), Kubernetes documentation: X.509 client certificates and the authenticating proxy.
[^2]: [Kubernetes API health endpoints](https://kubernetes.io/docs/reference/using-api/health-checks/), Kubernetes documentation.
[^3]: [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/), Kubernetes documentation.
[^4]: [Options for Software Load Balancing](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md), kubernetes/kubeadm on GitHub: keepalived, HAProxy, static Pods and kube-vip.
[^5]: [kube-vip: Static Pods](https://kube-vip.io/docs/installation/static/), kube-vip documentation.
[^6]: [kube-vip issue #684: kube-vip requires super-admin.conf with Kubernetes 1.29](https://github.com/kube-vip/kube-vip/issues/684), kube-vip on GitHub.
[^7]: [KubeVIP cluster template](https://capn.linuxcontainers.org/reference/templates/kube-vip.html), Cluster API Provider Incus: an automated version of the super-admin.conf workaround.
[^8]: [kubespray issue #11229](https://github.com/kubernetes-sigs/kubespray/issues/11229), kubernetes-sigs/kubespray on GitHub: discussion of ControlPlaneKubeletLocalMode and kube-vip bootstrap.
