---
title: "Preventing Privilege Escalation in Kubernetes"
date: 2026-11-16
draft: true
description: "The ways an attacker turns a small foothold in a Kubernetes cluster into full control, and how to close each of them: RBAC, Pod Security Admission, user namespaces, node isolation, admission policies and detection."
summary: "Most Kubernetes compromises don't start with cluster-admin. They start with one weak Pod or one over-broad role and climb from there. Here is how that climb works, and how to take away every step of the ladder."
tags: ["kubernetes", "security", "rbac", "pod security", "user namespaces", "admission control"]
categories: ["Programming"]
---

Attackers rarely break into a Kubernetes cluster as `cluster-admin`. They get in through something small: a vulnerable web app in a container, a leaked CI token, a developer laptop with a kubeconfig. What they do next is **privilege escalation**: turning that small foothold into more access, step by step, until they control a node, the whole cluster, or the cloud account behind it.

This post is about breaking that chain. It looks at the four places where escalation happens, the specific techniques at each one, and the controls that stop them. It builds on my earlier posts on authorization and service account tokens, so RBAC basics are only touched on briefly here.

Everything here is based on Kubernetes v1.37, the current release at the time of writing.

## The escalation ladder

It helps to think of escalation as a ladder with four rungs. An attacker who gets onto any rung tries to climb to the next one:

```text
  ┌──────────────────────────────────────────────────────────────┐
  │ 4. Cloud account     node IAM role, metadata service, keys   │
  ├──────────────────────────────────────────────────────────────┤
  │ 3. Whole cluster     cluster-admin, every Secret, every node │
  ├──────────────────────────────────────────────────────────────┤
  │ 2. Node              root on the host, kubelet credentials   │
  ├──────────────────────────────────────────────────────────────┤
  │ 1. Pod / identity    code execution in a container, a token  │
  └──────────────────────────────────────────────────────────────┘
```

The moves between rungs fall into two families:

- **API escalation**: using Kubernetes permissions to get more Kubernetes permissions. Example: a service account that may create Pods starts a Pod running as a more powerful service account.
- **Runtime escalation**: breaking out of the container's isolation. Example: a privileged container mounts the host's disk and becomes root on the node.

Each family has its own defences, and you need both. Perfect RBAC doesn't help if any Pod can be privileged, and perfect Pod security doesn't help if a CI token can bind itself to `cluster-admin`.

## Part 1: API escalation

### RBAC's built-in guard

Kubernetes already blocks the most obvious attack. You can't create or edit a Role or binding that grants permissions you don't have yourself. A user with permission to create RoleBindings can't simply bind themselves to `cluster-admin`[^2].

The problem is that many *other* permissions lead to the same place by a detour.

### Permissions that are secretly admin

The Kubernetes RBAC good practices guide lists the permissions that let a holder escalate[^1]. The important ones:

| Permission | How it's used to escalate |
| --- | --- |
| `create` on Pods, Deployments, Jobs, CronJobs, DaemonSets, StatefulSets… | Start a Pod that runs as any service account in the namespace and mounts its token, or any Secret. Permission to create workloads in a namespace means having the permissions of every service account in it. |
| `get`, `list`, `watch` on Secrets | Read every credential in scope. `list` and `watch` return the contents, not just names. |
| `create` on `serviceaccounts/token` | Mint fresh tokens for any service account in the namespace. |
| `create` on `pods/exec`, `pods/attach`, `pods/ephemeralcontainers` | Get a shell in a container that runs with more privileges or more powerful credentials. |
| `bind`, `escalate` on roles | Skip the built-in guard described above. |
| `impersonate` on users, groups or service accounts | Act as a more powerful identity. |
| `create` on PersistentVolumes | Create a `hostPath` PersistentVolume and claim it. Pod security checks look at the Pod's volumes, and a PVC looks harmless. |
| `update`/`patch` on admission webhooks or policies | Turn off the guardrails, or add a mutating webhook that injects anything into every new Pod. |
| `approve` and `sign` on CertificateSigningRequests | Issue client certificates for arbitrary identities. Certificates can't be revoked until they expire. |
| `nodes/proxy` | Full access to every kubelet's API, including running commands in containers. |
| `patch` on Nodes | Change labels and taints so that sensitive workloads get scheduled onto a node the attacker already controls. |

The first row is the one most teams underestimate. "The CI pipeline can deploy to `payments`" sounds harmless, but if the `payments` namespace also contains a service account that can read Secrets cluster-wide, the CI pipeline effectively has that permission too.

### Defences for API escalation

**Grant less, and grant it narrowly.**

- Use namespaced RoleBindings instead of ClusterRoleBindings wherever you can.
- Avoid wildcards in rules. A `"*"` covers resources that don't exist yet, including future CRDs.
- Use `resourceNames` for sensitive verbs like `bind`, `impersonate` and access to specific Secrets.
- Start from the built-in `view`, `edit` and `admin` roles. Notably, `view` excludes Secrets on purpose[^2].

**Separate powerful identities from the people and pipelines that create workloads.** Since anyone who can create Pods in a namespace can borrow any service account in it, keep high-privilege service accounts (operators, controllers, backup tools) in their **own namespaces**, where only platform administrators can create workloads.

**Turn off token automounting** for service accounts and Pods that don't call the Kubernetes API. A token that isn't there can't be stolen.

**Treat CSR approval, admission configuration and PersistentVolume creation as admin-only.** These are easy to grant by accident to "platform" or "storage" roles.

**Review regularly.** Ask the API server what an identity can actually do instead of reading YAML:

```bash
# Everything a service account can do in a namespace
kubectl auth can-i --list -n payments --as=system:serviceaccount:ci-system:deployer

# Who can read Secrets in payments? (needs a tool such as rbac-tool or kubectl-who-can)
kubectl who-can get secrets -n payments
```

Open source tools like `rbac-tool`, `kubectl-who-can` and `rakkess` turn the bindings into readable reports, and CNAPP or KSPM tools flag risky combinations automatically.

### Enforce RBAC rules with admission policies

RBAC can't say "nobody except the platform team may bind `cluster-admin`". It has no deny rules. A **ValidatingAdmissionPolicy**, stable since v1.30, can[^7]:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: restrict-cluster-admin-bindings
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["rbac.authorization.k8s.io"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["clusterrolebindings", "rolebindings"]
  validations:
    - expression: >
        object.roleRef.name != 'cluster-admin' ||
        'platform-admins' in request.userInfo.groups
      message: "Only members of platform-admins may bind cluster-admin."
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: restrict-cluster-admin-bindings
spec:
  policyName: restrict-cluster-admin-bindings
  validationActions: ["Deny"]
```

The same pattern works for other things RBAC can't express: refusing PersistentVolumes of type `hostPath`, refusing bindings to `system:authenticated` or `system:anonymous`, or refusing Pods that use a list of protected service accounts outside approved namespaces.

Make sure the policies themselves are protected. Whoever can edit or delete a ValidatingAdmissionPolicy can remove it.

## Part 2: Runtime escalation, from Pod to node

A container is a group of Linux processes that share the host's kernel, isolated with namespaces, cgroups, capabilities and security modules. Every setting that weakens that isolation is a possible way out. The most important ones:

| Setting | Why it lets you escape |
| --- | --- |
| `privileged: true` | All capabilities, all devices, no seccomp or AppArmor. Mount the host disk and you're root on the node. |
| `hostPath` volumes | Direct access to the host filesystem. Mounting `/`, `/var/run/containerd/containerd.sock` or `/var/lib/kubelet` is effectively node root. |
| `hostPID: true` | See and signal all host processes; with enough capabilities, enter their namespaces with `nsenter`. |
| `hostNetwork: true` | Bypass NetworkPolicy, reach services bound to the node's localhost, sniff traffic. |
| `hostIPC: true` | Access shared memory of other processes on the host. |
| Added capabilities, especially `SYS_ADMIN`, `SYS_PTRACE`, `SYS_MODULE`, `NET_ADMIN` | `SYS_ADMIN` alone allows mounting filesystems and many known escapes. |
| Running as root (UID 0) | Root in the container is UID 0 on the host too, unless user namespaces are used. Any kernel or runtime bug that breaks isolation then lands you as host root. |
| `allowPrivilegeEscalation` not set to `false` | A process can gain privileges through setuid binaries or file capabilities. |
| No seccomp profile | The full set of system calls is reachable, which means the full kernel attack surface. |

My post on Docker privileged containers goes deeper into what `privileged` actually turns off.

### `allowPrivilegeEscalation` in one paragraph

This field is often misunderstood. It doesn't control whether the container is privileged. It sets the Linux `no_new_privs` flag on the container's processes. With the flag set, `execve` can never give a process more privileges than its parent had, so setuid binaries like `sudo` or `su`, and files with capabilities, stop working as an escalation path[^11]. It's automatically `true` (that is, escalation allowed) when the container is privileged or has `CAP_SYS_ADMIN`[^13]. Set it to `false` everywhere.

### Pod Security Admission

You don't need to check every one of these settings by hand. The **Pod Security Standards** define three profiles, and the built-in **Pod Security Admission** controller enforces them per namespace[^3][^4]:

| Profile | Meant for |
| --- | --- |
| `privileged` | No restrictions. Only for trusted system components such as CNI and CSI agents. |
| `baseline` | Blocks the known escalations: privileged containers, host namespaces, `hostPath`, dangerous capabilities, unsafe sysctls. Most unmodified images run fine. |
| `restricted` | Baseline plus current hardening: must run as non-root, must set `allowPrivilegeEscalation: false`, must drop `ALL` capabilities (only `NET_BIND_SERVICE` may be added back), must set a seccomp profile of `RuntimeDefault` or `Localhost`, and only a short list of volume types. |

You turn it on with namespace labels. A safe rollout is to warn and audit at `restricted` while enforcing `baseline`, and switch enforcement once the warnings are gone:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.37
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

Pinning `enforce-version` means a cluster upgrade can't suddenly tighten the rules and block deployments. Bump it deliberately.

You can also set a cluster-wide default in the API server's admission configuration, so new namespaces start at `baseline` or `restricted` without anyone remembering to label them, and exempt only the namespaces that really need `privileged`[^4].

Pod Security Admission only checks Pods. Higher-level objects like Deployments get warnings, but the actual rejection happens when the controller tries to create the Pod, so read the `warn` output on `kubectl apply`.

### A hardened Pod

This passes the `restricted` profile and adds two things it doesn't require: a read-only root filesystem and user namespaces.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: payments
spec:
  replicas: 2
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      serviceAccountName: api
      automountServiceAccountToken: false
      hostUsers: false                    # user namespace: root in the Pod is not root on the node
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: api
          image: registry.example.com/payments/api:3.2.1
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

### User namespaces: root in the container is no longer root on the host

The biggest change in this area in years is **user namespaces**, stable since v1.36[^5]. Setting `hostUsers: false` maps UIDs inside the Pod to a range of unprivileged UIDs on the host. A process running as UID 0 in the container is a nobody on the node.

Why that matters: many container escapes, through kernel bugs or runtime bugs, end with the attacker running as whatever UID they had inside the container. Without user namespaces, that is real host root. With them, it's an unprivileged user that can't read other Pods' files, load kernel modules or touch the kubelet's credentials. Capabilities granted inside the Pod only apply inside its own user namespace, too.

Things to know before you switch it on[^6]:

- It needs a recent Linux kernel with ID-mapped mount support and a container runtime that supports it, such as containerd 2.x or CRI-O.
- It can't be combined with `hostNetwork`, `hostPID` or `hostIPC`.
- It doesn't fix kernel vulnerabilities. The kernel is still shared, and a bug that is exploitable from an unprivileged user still works. User namespaces reduce what a successful escape gets you; they don't make escapes impossible.

Since v1.36, **MutatingAdmissionPolicy** is stable as well[^8], which makes it easy to default `hostUsers: false` for every new Pod that doesn't explicitly ask for host users:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicy
metadata:
  name: default-user-namespaces
spec:
  failurePolicy: Fail
  reinvocationPolicy: IfNeeded
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
  matchConditions:
    - name: hostusers-not-set
      expression: "!has(object.spec.hostUsers)"
    - name: no-host-namespaces
      expression: >
        !(has(object.spec.hostNetwork) && object.spec.hostNetwork) &&
        !(has(object.spec.hostPID) && object.spec.hostPID) &&
        !(has(object.spec.hostIPC) && object.spec.hostIPC)
  mutations:
    - patchType: ApplyConfiguration
      applyConfiguration:
        expression: >
          Object{ spec: Object.spec{ hostUsers: false } }
---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionPolicyBinding
metadata:
  name: default-user-namespaces
spec:
  policyName: default-user-namespaces
  matchResources:
    namespaceSelector:
      matchLabels:
        security.example.com/user-namespaces: "default"
```

Roll it out namespace by namespace with the label, because some images assume they can write files owned by real host UIDs.

### Beyond the container boundary

For workloads you really don't trust, such as customer code, CI jobs building arbitrary repositories or AI agents running generated code, namespaces and seccomp may not be enough because the kernel is still shared. Sandboxed runtimes like **gVisor** or **Kata Containers**, selected per Pod with a `RuntimeClass`, put a second kernel or a lightweight VM between the workload and the host.

## Part 3: From node to cluster

Suppose an attacker does get root on a node. The next goal is the rest of the cluster. A kubelet necessarily holds credentials, and every Pod on the node has its own tokens and Secrets mounted. The question is how far those reach.

### The Node authorizer and NodeRestriction

The **Node authorizer** limits each kubelet to the Pods scheduled on its own node, and to the Secrets, ConfigMaps and volumes those Pods use. The **NodeRestriction** admission plugin stops a kubelet from modifying other Nodes or Pods bound to other nodes, and from setting labels with the `node-restriction.kubernetes.io/` prefix on its own Node[^9]. Together, they turn "one node compromised" from a whole-cluster breach into a breach of what ran on that one node. Make sure both are on: `--authorization-mode` should include `Node` (or the `Node` authorizer should be in your AuthorizationConfiguration), and `--enable-admission-plugins` should include `NodeRestriction`.

Use the `node-restriction.kubernetes.io/` label prefix for any label you use to steer sensitive workloads. A compromised kubelet can't set it on its own Node, so it can't attract those workloads.

### Lock down the kubelet API

Every kubelet serves an HTTPS API on port 10250 that can list Pods, read logs and run commands in containers. Check that[^10]:

- **Anonymous access is off**: `--anonymous-auth=false` (or `authentication.anonymous.enabled: false` in the kubelet config).
- **Authorization is delegated to the API server**: `--authorization-mode=Webhook`, so every call is checked with RBAC.
- **Nobody except the API server has `nodes/proxy`.** With fine-grained kubelet authorization stable since v1.36, monitoring agents can use `nodes/metrics`, `nodes/stats`, `nodes/pods` and `nodes/healthz` instead[^12].

### Keep crown jewels away from shared nodes

Since a node compromise exposes everything on that node, don't let untrusted workloads share nodes with powerful ones. Run controllers that hold cluster-wide permissions, ingress controllers with TLS keys and secret-management tools on a dedicated node pool, with taints so ordinary workloads can't be scheduled there, and node affinity (based on a `node-restriction.kubernetes.io/` label) so the sensitive workloads only land there.

### Don't let the control plane be a target

- Don't run application workloads on control plane nodes.
- Encrypt Secrets at rest in etcd, ideally with a KMS provider, and limit network access to etcd to the API servers.
- Keep `system:masters` credentials for break-glass only. They bypass authorization entirely and can't be revoked without rotating the cluster CA.

## Part 4: From the cluster to the cloud

On a managed cluster, the last rung is often the most damaging: the cloud account. The classic path is the **instance metadata service** at `169.254.169.254`. Any Pod that can reach it can often fetch the node's cloud credentials, and node roles tend to be broad, with permissions to pull images, attach disks and change networking.

Defences:

- **Block Pods from the metadata service.** On AWS, require IMDSv2 with a hop limit of 1, so requests from inside containers don't get a response. On every platform, a NetworkPolicy that denies egress to `169.254.169.254/32` is a cheap second layer.
- **Use workload identity instead of node roles.** Give each workload its own cloud role, through its projected service account token, and keep the node role down to the minimum the kubelet needs.
- **Keep cloud credentials out of Secrets.** A long-lived cloud key in a Secret is one `get secrets` away from anyone who escalates in that namespace.

## Detecting escalation attempts

Prevention will never be perfect, so watch for the moves described above. The API server audit log is the main source. Alert on:

- Creation or modification of ClusterRoleBindings and RoleBindings, especially to `cluster-admin`, `admin` or anything with `bind`, `escalate` or `impersonate`.
- Creation of `kubernetes.io/service-account-token` Secrets, and TokenRequests for service accounts that normally never get one.
- `pods/exec`, `pods/attach` and ephemeral containers in production namespaces.
- Pods created with `privileged`, `hostPath`, host namespaces or added capabilities, and Pods using a powerful service account outside its usual namespace.
- `Forbidden` responses in bursts from one identity: someone probing with `kubectl auth can-i --list` or trying permissions one by one.
- Changes to admission webhooks and admission policies.

Runtime detection tools such as **Falco** or **Tetragon** cover what the audit log can't see: a shell spawned in a container, reads of `/var/run/secrets/kubernetes.io/serviceaccount/token`, `nsenter` or `mount` calls, connections to the metadata service.

## A checklist

**Identities and RBAC**

- [ ] No wildcards in custom roles; `view`/`edit`/`admin` used where possible.
- [ ] Workload creation, Secret reads, `serviceaccounts/token`, `pods/exec`, PersistentVolume creation, CSR approval and admission configuration are restricted to people and pipelines you'd trust with every credential they can reach.
- [ ] Powerful service accounts live in namespaces where only administrators can create workloads.
- [ ] `bind`, `escalate` and `impersonate` are only granted with `resourceNames`.
- [ ] Token automounting is off by default.
- [ ] Admission policies block bindings to `cluster-admin` and to broad groups.

**Workloads**

- [ ] Every namespace has a Pod Security level; application namespaces enforce `restricted`.
- [ ] `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]`, `runAsNonRoot: true` and `seccompProfile: RuntimeDefault` everywhere.
- [ ] `hostUsers: false` for every workload that doesn't need host namespaces.
- [ ] Sandboxed runtimes for untrusted code.

**Nodes and control plane**

- [ ] `Node` authorizer and `NodeRestriction` enabled.
- [ ] Kubelet anonymous access off, webhook authorization on, no `nodes/proxy` outside the API server.
- [ ] Sensitive workloads on dedicated, tainted nodes selected with `node-restriction.kubernetes.io/` labels.
- [ ] Secrets encrypted at rest; `system:masters` used only for break-glass.

**Cloud**

- [ ] Pods can't reach the metadata service.
- [ ] Workload identity instead of node roles or static keys.

**Detection**

- [ ] Audit logging with alerts on bindings, token creation, exec, privileged Pods and admission changes.
- [ ] Runtime detection on nodes.

## Bottom line

- **Escalation is a climb**: from a Pod or token, to a node, to the cluster, to the cloud account. Take away the rungs at every level, not just one.
- **API escalation hides in harmless-looking permissions.** Creating workloads, reading Secrets, minting tokens, creating PersistentVolumes and approving certificates can all lead to admin. Separate powerful service accounts into their own namespaces and back RBAC with admission policies.
- **Runtime escalation is mostly closed by Pod Security Admission.** Enforce `restricted` for applications, and add `hostUsers: false`, which since v1.36 makes container root an unprivileged user on the host.
- **Contain node compromises** with the Node authorizer, NodeRestriction, a locked-down kubelet and dedicated nodes for sensitive workloads.
- **Cut the path to the cloud** by blocking the metadata service and using workload identity.
- **Assume something will get through** and watch the audit log and runtime events for the moves in this post.

## References

[^1]: [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/), Kubernetes documentation: privilege escalation risks, including workload creation, Secrets, PersistentVolume creation, CSRs, webhooks, `nodes/proxy`, `bind`, `escalate` and `impersonate`.
[^2]: [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), Kubernetes documentation: privilege escalation prevention and default roles.
[^3]: [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/), Kubernetes documentation: the `privileged`, `baseline` and `restricted` profiles.
[^4]: [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/), Kubernetes documentation: namespace labels, modes, versions and cluster-wide defaults.
[^5]: [Kubernetes v1.36: User Namespaces in Kubernetes are finally GA](https://kubernetes.io/blog/2026/04/23/kubernetes-v1-36-userns-ga/), Kubernetes blog, and [KEP-127: Support User Namespaces](https://www.kubernetes.dev/resources/keps/127/).
[^6]: [User Namespaces](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/), Kubernetes documentation: requirements and limitations.
[^7]: [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/), Kubernetes documentation.
[^8]: [Mutating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/mutating-admission-policy/), Kubernetes documentation, and [Amazon EKS and Amazon EKS Distro Now Support Kubernetes 1.36](https://builder.aws.com/content/3F7xMMfym9HPoYxY8aYz7dMkY2w/amazon-eks-and-amazon-eks-distro-now-support-kubernetes-136), AWS: user namespaces and mutating admission policies GA in v1.36.
[^9]: [Using Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/), Kubernetes documentation.
[^10]: [Kubelet authentication/authorization](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/), Kubernetes documentation.
[^11]: [No New Privileges Flag](https://docs.kernel.org/userspace-api/no_new_privs.html), Linux kernel documentation.
[^12]: [Kubernetes v1.36: Fine-Grained Kubelet API Authorization Graduates to GA](https://kubernetes.io/blog/2026/04/24/kubernetes-v1-36-fine-grained-kubelet-authorization-ga/), Kubernetes blog.
[^13]: [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/), Kubernetes documentation.
