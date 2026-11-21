---
title: "Kubernetes the Hard Way, on Your Own Laptop"
date: 2026-11-20
draft: true
description: "A walkthrough of mmumshad's local edition of Kubernetes the Hard Way: what it builds, how it differs from Kelsey Hightower's original, what each lab teaches, and how to get through it without losing an evening to a typo."
summary: "kubeadm hides a lot. Kubernetes the Hard Way takes it all apart again: certificates, kubeconfigs, etcd, systemd units, TLS bootstrapping. Here is what the local, VirtualBox-based edition builds, what each lab teaches, and how to get the most out of it."
tags: ["kubernetes", "kubernetes the hard way", "cka", "pki", "etcd", "tls bootstrapping", "vagrant", "homelab"]
categories: ["Programming"]
---

Most people meet Kubernetes through a managed service or `kubeadm init`. Both are great at what they do, which is to hide everything. You get a working cluster and no idea why it works: where the certificates came from, who signed them, why the kubelet trusts the API server, or what actually happens when a node joins.

**Kubernetes the Hard Way** is the cure. It was written by Kelsey Hightower and has you build a cluster by hand, one binary, one certificate and one systemd unit at a time. This post is about a popular local fork of it by Mumshad Mannambeth (of KodeKloud fame), which runs entirely on your own machine instead of in a cloud account[^1].

## What you build

The local edition builds a small but highly available cluster with end-to-end TLS between all components and RBAC turned on[^1]. It uses five VMs:

| VM | Role |
| --- | --- |
| `controlplane01`, `controlplane02` | etcd, kube-apiserver, kube-controller-manager and kube-scheduler, all running as plain systemd services |
| `node01`, `node02` | Worker nodes: containerd, kubelet and kube-proxy |
| `loadbalancer` | HAProxy, balancing across both API servers. This is the address in your kubeconfig. |

Two details are worth calling out early. First, the control plane nodes are **not** registered as nodes, so `kubectl get nodes` only ever shows the two workers[^1]. There are no static Pods and no `/etc/kubernetes/manifests`: if you are used to kubeadm, the control plane will feel strangely invisible from inside the cluster. Second, running etcd on two members is a deliberate compromise. etcd wants an odd number of members for a useful quorum, and two members tolerate zero failures. The tutorial accepts that to save RAM; the point of having two control plane nodes is to show load balancing across API servers, not real fault tolerance[^1].

The default network layout is[^2]:

| Network | Range |
| --- | --- |
| VMs | `192.168.56.0/24` |
| Pods | `10.244.0.0/16` |
| Services | `10.96.0.0/16` (cluster DNS at `10.96.0.10`) |

Changing these is possible but fiddly, since the ranges are written into the docs and the CoreDNS manifest. Leave them alone on your first run.

## How it differs from the original

Kelsey Hightower's original originally targeted Google Cloud; its current version runs on any four ARM64 or AMD64 machines, with a jumpbox, one control plane node and two workers, using Kubernetes v1.32, containerd 2.1, CNI 1.6 and etcd 3.6[^3]. The local edition differs in a few ways that matter for learning[^1][^4]:

- **Local hypervisor instead of a cloud.** VirtualBox and Vagrant on Windows, Intel Macs and Linux; Multipass on Apple Silicon.
- **Two control plane nodes behind HAProxy**, instead of a single server.
- **TLS bootstrapping.** The first worker gets hand-made certificates; the second joins with a bootstrap token and requests its own. This is the most valuable addition, because it is how real clusters (and kubeadm) actually add nodes.
- **OpenSSL-style certificate files**: `.crt` and `.key` instead of the original's `.pem` and `-key.pem`, and a separate certificate for etcd rather than reusing the API server's.
- **Weave Net** as the pod network, instead of manually programmed routes.
- **A certificate verification script** (`cert_verify.sh`) that checks your work at key points, and an extra lab that runs the end-to-end conformance tests.

One thing it does *not* do: it is not a kubeadm cluster, and the README says so plainly to anyone coming from the CKA course[^1]. That is exactly the point. You see the parts kubeadm normally assembles for you.

## Before you start

The hardware requirements are real. The VirtualBox prerequisites ask for **16 GB of RAM, an 8-core CPU and 50 GB of free disk**, and warn that less may work but will be slow and can crash[^2]. Five VMs plus your host OS add up.

You need:

- VirtualBox and Vagrant (or Multipass on an M-series Mac).
- `git clone https://github.com/mmumshad/kubernetes-the-hard-way.git`
- Patience and attention to detail. The README is blunt: miss one step and the build breaks, and anything a check prints in red means you went wrong earlier[^1].

A good tip from the prerequisites: use **tmux with synchronized panes** for the steps you repeat on several machines. The Vagrant provisioner drops a `.tmux.conf` on the VMs where `CTRL+X` toggles synchronization; the pane divider turns red, and whatever you type goes to every pane[^2]. Use it for installing binaries on both control plane nodes. Turn it *off* before anything node-specific.

And when you inevitably break something, `vagrant destroy -f && vagrant up` gives you a clean slate[^2].

## The labs, and what each one teaches

The real value is not the finished cluster, which you will delete. It is the understanding you pick up on the way. Here is the lab sequence[^5] and what to pay attention to in each.

### 1–3. Prerequisites, compute resources, client tools

`vagrant up` creates the five VMs with fixed IPs and `/etc/hosts` entries. You then install `kubectl` and set up SSH from `controlplane01` to the other machines, because that node acts as your admin workstation for the rest of the tutorial. Not exciting, but the fixed-IP setup is what makes the certificate steps later predictable.

### 4. Certificate authority and TLS certificates

This is the lab that pays for the whole tutorial. You create your own CA and then a certificate for every identity in the cluster: admin, kube-controller-manager, kube-scheduler, kube-proxy, kube-apiserver, the API server's kubelet client, etcd, the service account signing key pair, and `node01`.

The thing to notice is that **the certificate *is* the identity**. Kubernetes has no user database: the API server reads the Common Name as the username and the Organization fields as groups. A certificate with `CN=system:kube-scheduler` *is* the scheduler as far as RBAC is concerned, and `O=system:masters` makes anyone holding it a cluster superuser. With OpenSSL, the admin certificate looks something like this:

```bash
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out admin.crt -days 1000
```

The API server certificate is the other one to study closely. Its Subject Alternative Names must cover every name and IP clients use to reach it: both control plane IPs, the load balancer, `127.0.0.1`, the first IP of the service range (`10.96.0.1`, the `kubernetes` Service), and `kubernetes.default.svc.cluster.local` and friends. Forget one and you get certificate errors that only show up much later, from a client you weren't thinking about. Run `cert_verify.sh` here, every time.

### 5. Kubeconfig files

Every component that talks to the API server gets a kubeconfig: the CA to trust, the server address, and its client certificate. Note which components point at the **load balancer** (the workers' kubelet and kube-proxy) and which point at **`127.0.0.1`** (the scheduler and controller manager, which run next to an API server). After this lab, a kubeconfig is no longer a magic file; it's three pieces of information you could write by hand.

### 6. Data encryption config

By default, Secrets are stored in etcd base64-encoded, which is to say not encrypted at all. This lab writes an `EncryptionConfiguration` with an AES-CBC key and points the API server at it with `--encryption-provider-config`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded 32-byte key>
      - identity: {}
```

The smoke test later proves it works by reading a Secret straight out of etcd and seeing ciphertext with a `k8s:enc:aescbc:v1:key1` prefix instead of your data[^6]. In production you would use a KMS provider instead of a key in a file, but the mechanism is the same.

### 7. Bootstrapping etcd

You install etcd on both control plane nodes as a systemd service, with mutual TLS for both client traffic (port 2379) and peer traffic (2380). Then you query it directly with `etcdctl member list`. The lesson: etcd is just a database that happens to hold the entire cluster state. Everything else in the control plane is stateless and could be thrown away and restarted.

### 8. Bootstrapping the control plane

kube-apiserver, kube-controller-manager and kube-scheduler, each as a systemd unit with a long list of flags, then HAProxy on the load balancer. Read every flag. This is where you learn what the API server is really configured with: where etcd is, which CA verifies client certificates, which key signs service account tokens, which admission plugins run, which ranges Services get. It's also where you discover that "the control plane" is three ordinary Linux processes.

### 9–10. Container runtime and the first worker

Install containerd and runc on both workers, then configure `node01` the "manual" way: its own certificate with `CN=system:node:node01` and `O=system:nodes`, which is exactly what the Node authorizer and NodeRestriction admission plugin need to limit a kubelet to its own node's objects. Then the kubelet and kube-proxy, again as systemd services with config files.

### 11. TLS bootstrapping the second worker

This is where the local edition goes beyond the original. Hand-crafting a certificate for every node doesn't scale, so real clusters let a new kubelet request its own. The flow[^7]:

1. You create a **bootstrap token**: a Secret in `kube-system` of type `bootstrap.kubernetes.io/token`. The API server, started with `--enable-bootstrap-token-auth`, accepts it as a weak identity in the `system:bootstrappers` group.
2. The kubelet on `node02` starts with a *bootstrap kubeconfig* that contains only that token.
3. It creates a **CertificateSigningRequest** for a client certificate.
4. RBAC bindings let the bootstrappers group create CSRs, and let the controller manager **auto-approve** node client certificates. The controller manager signs them with the cluster CA.
5. The kubelet writes its new certificate and switches to it. With certificate rotation on, it renews it the same way before expiry.

Then you look at `kubectl get csr` and find one still `Pending`: the kubelet's *serving* certificate, which the API server uses when it connects back to the kubelet. Kubernetes deliberately does not auto-approve those, because nothing in the CSR proves the node really owns the IPs and names it asks for. You approve it by hand with `kubectl certificate approve`. Understanding why that distinction exists teaches you more about cluster security than most courses.

### 12. Remote kubectl

Configure an admin kubeconfig that goes through the load balancer, and you are working the way you would with any other cluster. `kubectl get nodes` shows two `NotReady` nodes, which brings us to the next lab.

### 13. Pod networking

The nodes stay `NotReady` until there is a CNI plugin. The tutorial deploys Weave Net, after which pods get IPs from `10.244.0.0/16` and can reach each other across nodes. A caveat for 2026: Weaveworks shut down in early 2024 and Weave Net is no longer actively developed. It still works fine for a lab, but if you want to practice with something you will meet in production, swapping in Flannel or Cilium here is a good exercise in its own right, since it only requires the pod CIDR to match.

### 14. API server to kubelet RBAC

Try `kubectl logs` or `kubectl exec` before this lab and it fails. Those commands go from the API server *to the kubelet*, and the kubelet authorizes the API server like any other client. This lab creates a ClusterRole for the kubelet's `nodes/proxy`, `nodes/log`, `nodes/stats` and similar subresources and binds it to the API server's kubelet client identity. It's a short lab that clears up a common misconception: traffic in a cluster doesn't only flow *to* the API server.

### 15. DNS add-on

Deploy CoreDNS at `10.96.0.10`, the address every kubelet was configured to hand to Pods, and test resolution of `kubernetes.default` from a busybox Pod. If you want to know how Service names turn into IPs, this is where it comes together.

### 16–17. Smoke test and end-to-end tests

The smoke test checks the things that cut across the whole stack: Secret encryption in etcd, Deployments, port forwarding, logs, exec, and a NodePort Service. The final lab runs the upstream end-to-end tests against your cluster. They take a while, and a clean result is a nice piece of evidence that you really did build a conformant Kubernetes.

## Tips for getting through it

- **Check which machine you are on before every command.** Most failures in this tutorial come from running a step on the wrong node. Put the hostname in your prompt in a bright colour.
- **Run `cert_verify.sh` whenever the tutorial tells you to.** It catches SAN and file-name mistakes immediately rather than three labs later[^1].
- **Read the logs, not just the status.** `journalctl -u kube-apiserver -f` and `journalctl -u kubelet -f` will tell you exactly which certificate or flag is wrong.
- **Snapshot the VMs after each major lab.** Rolling back to "etcd works" is much faster than starting over.
- **Type the commands, at least the first time.** Pasting a block of twenty commands teaches you nothing. Typing the API server flags teaches you the API server.
- **Break it on purpose afterwards.** Stop etcd on one node and watch what happens with two members. Delete the `kube-apiserver-to-kubelet` binding and try `kubectl logs`. Let a kubelet certificate expire. That's where the real learning is.

## Is it worth it?

If you only ever use managed Kubernetes, you can get by without ever doing this. But the first time a certificate expires, a node won't join, or `kubectl exec` mysteriously returns `Forbidden`, you'll wish you had. The README puts the target audience well: people who plan to support a production cluster and want to understand how the parts fit together[^1].

It's also excellent CKA preparation, even though the exam uses kubeadm. Troubleshooting a broken control plane is much easier once you have built one from nothing and know which file, which flag, and which certificate each component depends on.

Set aside a weekend, make coffee, and expect to break it at least once.

## Bottom line

- **Kubernetes the Hard Way** builds a cluster by hand so you understand every component, certificate and connection.
- **mmumshad's edition** runs locally on VirtualBox/Vagrant or Multipass, with two control plane nodes behind HAProxy and two workers. Budget 16 GB of RAM.
- **The key lessons**: a certificate is an identity, kubeconfigs are just CA + server + credentials, the control plane is a handful of systemd services, and nodes join with TLS bootstrapping and CSRs.
- **It's not kubeadm**, which is exactly why it's worth doing.

## References

[^1]: [Kubernetes The Hard Way (local edition)](https://github.com/mmumshad/kubernetes-the-hard-way), Mumshad Mannambeth, GitHub: cluster layout, platform support, CKA note and troubleshooting advice.
[^2]: [Prerequisites: Kubernetes The Hard Way on VirtualBox](https://github.com/mmumshad/kubernetes-the-hard-way/blob/master/VirtualBox/docs/01-prerequisites.md), GitHub: hardware requirements, network ranges and tmux tips.
[^3]: [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way), Kelsey Hightower, GitHub: the original tutorial, its component versions and lab list.
[^4]: [Differences to the original](https://github.com/sam-perrin/kubernetes-the-hard-way/blob/master/docs/differences-to-original.md), from a fork of the local edition.
[^5]: [Lab files in the local edition](https://github.com/mmumshad/kubernetes-the-hard-way/tree/master/docs), GitHub.
[^6]: [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/), Kubernetes documentation.
[^7]: [TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/), Kubernetes documentation.
