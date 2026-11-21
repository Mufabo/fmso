---
title: "Authorization in Kubernetes"
date: 2026-11-16
draft: true
description: "How the Kubernetes API server decides whether a request is allowed: request attributes, the authorizer chain, RBAC, the Node authorizer, webhooks, structured authorization configuration, impersonation, kubelet authorization and the newest features in v1.36 and v1.37."
summary: "Authentication tells Kubernetes who you are. Authorization decides what you may do. Here is how that decision is made, how to configure it, how to check it, and where it stops."
tags: ["kubernetes", "security", "authorization", "rbac", "webhooks", "impersonation"]
categories: ["Programming"]
---

Every request to the Kubernetes API server goes through the same three gates. **Authentication** works out who is asking. **Authorization** decides whether that someone may do what they are asking. **Admission control** looks at the object itself and may change or reject it. This post is about the middle gate.

Authorization is where most of a cluster's security lives. A stolen token is only as dangerous as the permissions behind it, and a mistake in a single ClusterRoleBinding can quietly give a workload control of the whole cluster. Yet many people only ever meet authorization as "copy this RBAC YAML from the Helm chart".

This post covers what the API server actually authorizes, how the chain of authorizers works, RBAC in depth, the Node and Webhook authorizers, the new configuration file format, how to check permissions, impersonation, the kubelet's own API, and the features that arrived in the last few releases.

Everything here is based on Kubernetes v1.37, the current release at the time of writing.

## The request pipeline

When a request arrives at the API server, it passes through these stages in order[^1]:

```text
request ─▶ authentication ─▶ authorization ─▶ mutating admission ─▶ validation ─▶ validating admission ─▶ etcd
           "who are you?"    "may you do      "change the object"                "accept the object?"
                              this?"
```

Two things about this picture matter for everything that follows.

**Authorization happens before the object is read.** The authorizer sees *what kind* of request this is (who, which verb, which resource, which namespace, which name), but not the body. It can't say "you may create Pods, but only without `privileged: true`". That kind of rule belongs to admission control, which comes later.

**Authorization is per request.** There are no sessions. Every single API call, including each `watch`, is authorized on its own.

## What gets authorized: request attributes

The authorizer doesn't see an HTTP request. It sees a set of **attributes** that the API server extracts from it[^1]:

| Attribute | Example | Notes |
| --- | --- | --- |
| user | `jane`, `system:serviceaccount:demo:app` | From authentication. |
| groups | `developers`, `system:authenticated` | From authentication. |
| extra | `{"scopes": ["view"]}` | Arbitrary key/values from the authenticator. |
| verb | `get`, `list`, `create` | Derived from the HTTP method, see below. |
| API group | `""` (core), `apps`, `rbac.authorization.k8s.io` | |
| resource | `pods`, `deployments` | |
| subresource | `log`, `exec`, `status`, `scale` | `pods/log` is a different permission from `pods`. |
| namespace | `demo` | Empty for cluster-scoped resources. |
| name | `web-7d4f9` | Empty for `list`, `watch` of a collection, and `create`. |
| path | `/healthz`, `/metrics` | Only for **non-resource requests**. |

### HTTP methods become verbs

Kubernetes authorizes Kubernetes verbs, not HTTP methods. The mapping is[^1]:

| HTTP method | Verb |
| --- | --- |
| `GET`, `HEAD` on a single object | `get` |
| `GET`, `HEAD` on a collection | `list` |
| `GET` with `?watch=true` | `watch` |
| `POST` | `create` |
| `PUT` | `update` |
| `PATCH` | `patch` |
| `DELETE` on a single object | `delete` |
| `DELETE` on a collection | `deletecollection` |

Note that `list` returns the full objects, not just their names. Being allowed to `list` Secrets is the same as being allowed to read all of them.

Some verbs have nothing to do with HTTP and are checked by the API server or a controller for special actions[^1]:

- **`bind`** and **`escalate`** on Roles and ClusterRoles: create bindings to, or edit, roles containing permissions you don't have yourself (more on this below).
- **`impersonate`** on users, groups, service accounts and `userextras`: act as someone else.
- **`approve`** and **`sign`** on CertificateSigningRequests.
- **`use`**, historically used by PodSecurityPolicy and still used by some extensions.

## Decisions and the authorizer chain

An authorizer gives one of three answers[^1]:

- **Allow**: the request may proceed.
- **Deny**: the request is rejected.
- **No opinion**: this authorizer has nothing to say; ask the next one.

The API server is configured with an ordered list of authorizers. It asks them one at a time. The first one that says *allow* or *deny* wins, and the rest are never asked. If every authorizer says *no opinion*, the request is denied with `403 Forbidden`.

```text
          ┌──────┐   no opinion   ┌──────┐   no opinion   ┌─────────┐   no opinion
request ─▶│ Node │──────────────▶│ RBAC │──────────────▶│ Webhook │──────────────▶ 403
          └──┬───┘                └──┬───┘                └────┬────┘
             │ allow                  │ allow                   │ allow / deny
             ▼                        ▼                         ▼
          decision                 decision                  decision
```

This has a consequence that surprises people: **RBAC never says deny.** It only has allow rules, so it either allows or has no opinion. That means:

- If you put a webhook **after** RBAC, the webhook can only *add* permissions on top of RBAC. It never even sees requests RBAC has already allowed.
- If you want a webhook that can *forbid* things RBAC would allow, it has to come **before** RBAC in the chain.

There is one more thing to know: members of the group **`system:masters`** bypass authorization completely[^1]. The API server short-circuits before any authorizer is asked, so no RBAC rule or webhook can restrict them. The `admin.conf` kubeconfig that `kubeadm` creates historically used a certificate in this group, and newer versions use a separate `kubeadm:cluster-admins` group bound through RBAC for exactly this reason. Treat any credential in `system:masters` as a break-glass key that can't be revoked except by rotating the CA.

## The authorization modes

Kubernetes ships with these authorizers[^1]:

| Mode | What it does |
| --- | --- |
| `Node` | Lets kubelets access only what they need for the Pods scheduled on their own node. |
| `RBAC` | Role-based access control, configured with API objects. The one you use every day. |
| `Webhook` | Sends a `SubjectAccessReview` to an external HTTP service and uses its answer. |
| `ABAC` | Attribute-based access control from a static JSON-lines policy file. Legacy; changing it requires an API server restart. |
| `AlwaysAllow` | Allows everything. Only for testing. |
| `AlwaysDeny` | Denies everything. Only for testing. |

A typical production cluster uses `Node,RBAC`, optionally with one or more webhooks.

## Configuring the chain

### The old way: flags

For years, the chain was configured with flags on `kube-apiserver`:

```bash
kube-apiserver \
  --authorization-mode=Node,RBAC,Webhook \
  --authorization-webhook-config-file=/etc/kubernetes/authz-webhook.kubeconfig \
  --authorization-webhook-cache-authorized-ttl=5m \
  --authorization-webhook-cache-unauthorized-ttl=30s
```

This works, but it has real limits: only **one** webhook, no way to say which requests the webhook should even look at, and any change means restarting the API server.

### The new way: structured authorization configuration

The **AuthorizationConfiguration** file fixes all three. It went alpha in v1.29, beta in v1.30 and stable in v1.32[^5]. You point the API server at it with `--authorization-config`, which is mutually exclusive with `--authorization-mode` and the `--authorization-webhook-*` flags:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AuthorizationConfiguration
authorizers:
  - type: Node
    name: node

  # A policy engine that may DENY requests in tenant namespaces.
  # It comes before RBAC, so its "deny" wins over RBAC's "allow".
  - type: Webhook
    name: tenant-guard
    webhook:
      timeout: 3s
      subjectAccessReviewVersion: v1
      matchConditionSubjectAccessReviewVersion: v1
      authorizedTTL: 30s
      unauthorizedTTL: 30s
      # If the webhook is down, fall through to the next authorizer
      # instead of failing the request.
      failurePolicy: NoOpinion
      connectionInfo:
        type: KubeConfigFile
        kubeConfigFile: /etc/kubernetes/tenant-guard.kubeconfig
      # Only ask the webhook about resource requests in tenant namespaces.
      # Everything else skips it, with no network round-trip.
      matchConditions:
        - expression: has(request.resourceAttributes)
        - expression: request.resourceAttributes.namespace.startsWith('tenant-')
        - expression: "!('system:masters' in request.groups)"

  - type: RBAC
    name: rbac
```

The improvements:

- **Multiple webhooks**, each with its own name, endpoint, timeouts and cache settings.
- **`matchConditions`**: CEL expressions evaluated against the `SubjectAccessReview` *before* calling the webhook. All of them must be true for the webhook to be asked. This keeps a slow or fragile webhook off the critical path for requests it doesn't care about, such as those from system components.
- **`failurePolicy`**: `NoOpinion` (move on to the next authorizer) or `Deny` (reject the request) when the webhook can't be reached. Choose deliberately: `Deny` is safer for a guard webhook, but it can lock you out of your own cluster.
- **Automatic reload**: the API server watches the file and applies changes without a restart. If the new file is invalid, it keeps the last good configuration and reports the error in metrics.

The order of the list is the order of the chain, so everything said above about allow, deny and *no opinion* applies directly.

## RBAC in depth

RBAC is configured with four object types in the `rbac.authorization.k8s.io/v1` API group[^2]:

| | Namespaced | Cluster-scoped |
| --- | --- | --- |
| **Set of permissions** | `Role` | `ClusterRole` |
| **Grant to subjects** | `RoleBinding` | `ClusterRoleBinding` |

A role is a list of **rules**. A binding connects a role to **subjects**: users, groups and service accounts. There are no deny rules, and permissions from all bindings are simply added together.

### Roles and rules

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: shop
rules:
  # Manage Deployments and scale them
  - apiGroups: ["apps"]
    resources: ["deployments", "deployments/scale"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Read Pods and their logs, but not exec into them
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  # Read one specific ConfigMap only
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["shop-settings"]
    verbs: ["get"]
```

A few details that catch people out:

- **The core API group is the empty string** `""`, not `"core"` or `"v1"`.
- **Subresources are separate.** `pods` doesn't include `pods/log`, `pods/exec` or `pods/portforward`. `deployments` doesn't include `deployments/scale`.
- **`resourceNames` can't restrict `create`**, because the name isn't known at authorization time for a `POST` to a collection. It also can't restrict `list` or `watch` unless the client asks for a single name with a field selector `metadata.name=...`.
- **Wildcards** (`"*"`) match everything, including resources and verbs added in future versions and by CRDs. Avoid them outside break-glass roles.

### Bindings

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer
  namespace: shop
subjects:
  - kind: ServiceAccount
    name: ci
    namespace: ci-system   # service accounts from other namespaces are allowed
  - kind: Group
    name: shop-team        # from your OIDC provider, for example
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

`roleRef` can't be changed after the binding is created. To point a binding at a different role, delete and re-create it. This is deliberate: granting a binding's subjects a whole new role should be a visible action, not an edit[^2].

### The four combinations

The combination that is easy to miss is a **RoleBinding that refers to a ClusterRole**:

| Binding | Role | Effect |
| --- | --- | --- |
| RoleBinding | Role | Permissions in that one namespace. |
| RoleBinding | ClusterRole | The ClusterRole's namespaced permissions, **only in the binding's namespace**. |
| ClusterRoleBinding | ClusterRole | Permissions in all namespaces, plus cluster-scoped resources. |
| ClusterRoleBinding | Role | Not allowed. |

The second row is how you reuse one role definition across many namespaces: define a ClusterRole once, and bind it with a RoleBinding in each namespace where it should apply.

### Built-in roles

Kubernetes creates a set of default ClusterRoles, and you should use them before writing your own[^2]:

| ClusterRole | Intended use |
| --- | --- |
| `cluster-admin` | Everything, everywhere. Bound to `system:masters` by default. |
| `admin` | Full control within a namespace, including Roles and RoleBindings, but not the namespace itself or its ResourceQuota. Meant for a RoleBinding. |
| `edit` | Read and write most objects in a namespace, but not Roles or RoleBindings. |
| `view` | Read most objects in a namespace. **Excludes Secrets**, because reading them could reveal service account tokens and escalate. |

There are also many `system:` roles for control plane components. Don't edit them; the API server reconciles them back to their defaults on startup.

### Aggregated ClusterRoles

`admin`, `edit` and `view` grow automatically when you install CRDs, thanks to **aggregation**. A ClusterRole with an `aggregationRule` gets its rules filled in by a controller from every ClusterRole whose labels match:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backups-view
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
  - apiGroups: ["backup.example.com"]
    resources: ["backups"]
    verbs: ["get", "list", "watch"]
```

Anyone who already has `view` in a namespace can now read `Backup` objects there. This is how well-behaved operators make their CRDs work with the default roles. It's also worth auditing: installing a chart can silently extend `edit` or `view` across the whole cluster.

### Escalation prevention

RBAC has a built-in guard against the obvious attack: "I can create RoleBindings, so I'll bind myself to `cluster-admin`." By default you can only create or update a role, or a binding, if you **already hold every permission it grants**, in the same scope[^2].

Two verbs relax this on purpose:

- **`escalate`** on `roles` / `clusterroles` lets you create or edit roles with permissions you don't have.
- **`bind`** on `roles` / `clusterroles` lets you create bindings to roles with permissions you don't have.

That lets you build a delegated admin who can hand out the `edit` role, and nothing else, without holding it themselves:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: grant-edit
rules:
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["rolebindings"]
    verbs: ["create", "delete", "get", "list"]
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["clusterroles"]
    verbs: ["bind"]
    resourceNames: ["edit"]
```

Treat `bind`, `escalate` and `impersonate` as admin-level permissions whenever they aren't restricted with `resourceNames`[^9].

## The Node authorizer

Every kubelet needs to read the Pods scheduled on it, the Secrets and ConfigMaps those Pods mount, and to update its own Node and Pod statuses. Granting that with RBAC would mean every kubelet can read *every* Secret in the cluster, so one compromised node would expose everything.

The **Node authorizer** solves this. It only acts on requests from users named `system:node:<nodeName>` in the group `system:nodes`, and it keeps a graph of which Pods run on which node and which Secrets, ConfigMaps, PersistentVolumeClaims and service accounts they reference. A kubelet may read a Secret only if a Pod bound to its node uses it[^3].

It works together with the **NodeRestriction** admission plugin, which stops a kubelet from modifying other Node objects, or Pods bound to other nodes, and from setting labels in the `node-restriction.kubernetes.io/` prefix on its own Node. Always run both: `Node` first in the authorizer chain, and `NodeRestriction` in the admission plugin list.

## The Webhook authorizer

For policies RBAC can't express, the API server can ask an external service. It sends a **SubjectAccessReview** describing the request[^4]:

```json
{
  "apiVersion": "authorization.k8s.io/v1",
  "kind": "SubjectAccessReview",
  "spec": {
    "user": "jane",
    "groups": ["developers", "system:authenticated"],
    "resourceAttributes": {
      "namespace": "tenant-a",
      "verb": "delete",
      "group": "apps",
      "resource": "deployments",
      "name": "web"
    }
  }
}
```

The webhook answers in the `status`:

```json
{
  "apiVersion": "authorization.k8s.io/v1",
  "kind": "SubjectAccessReview",
  "status": {
    "allowed": false,
    "denied": true,
    "reason": "tenant-a is frozen for the release window"
  }
}
```

The combination of `allowed` and `denied` maps to the three decisions: `allowed: true` is *allow*, `denied: true` is *deny*, and both false is *no opinion*.

Common uses are central policy engines (Open Policy Agent, Cedar-based authorizers), temporary freezes, and managed services that map cloud IAM identities to cluster permissions. Webhooks add latency and a dependency to every matching request, which is why `matchConditions` and the response cache settings matter.

### Selectors in authorization

Since v1.34, authorizers can also see the **field and label selectors** of `list`, `watch` and `deletecollection` requests (`AuthorizeWithSelectors`). That lets a webhook allow rules like "a node agent may list Pods, but only with `spec.nodeName=<its own node>`", which used to require giving it access to every Pod in the cluster.

## Checking permissions

You rarely need to read through all the bindings to find out what someone can do. The API server will tell you.

### As yourself

```bash
kubectl auth whoami
# ATTRIBUTE   VALUE
# Username    jane
# Groups      [developers system:authenticated]

kubectl auth can-i create deployments -n shop
# yes

kubectl auth can-i get pods --subresource=log -n shop
# yes

# Everything you can do in a namespace
kubectl auth can-i --list -n shop
```

Behind the scenes these use the **SelfSubjectReview**, **SelfSubjectAccessReview** and **SelfSubjectRulesReview** APIs, which every authenticated user may call about themselves.

### As someone else

```bash
kubectl auth can-i delete secrets -n shop --as=system:serviceaccount:ci-system:ci
kubectl auth can-i list nodes --as=jane --as-group=developers
```

`--as` uses impersonation, so you need the `impersonate` permission to run it. A cluster admin can also post a **SubjectAccessReview** directly, which is what webhook authorizers and tools like `kubectl auth can-i` do under the hood.

`--list` and SelfSubjectRulesReview are a best-effort summary. They only cover authorizers that can enumerate rules, which RBAC can and webhooks generally can't. For a definite answer about one action, ask `can-i` for that action.

### Finding out why

When a request fails with `Forbidden`, the error message includes the verb, resource and namespace that were checked. The API server's audit log records the decision and, in the `authorization.k8s.io/reason` annotation, the binding that allowed it, such as `RBAC: allowed by ClusterRoleBinding "ci-deployer" of ClusterRole "edit" to ServiceAccount "ci/ci-system"`. That annotation is the fastest way to answer "why does this account have access to that?".

## Impersonation

Impersonation lets one identity act as another by setting headers on the request: `Impersonate-User`, `Impersonate-Group`, `Impersonate-Uid` and `Impersonate-Extra-*`. The API server first checks that the caller may impersonate those values, then authorizes the actual request *as the impersonated identity*[^6]:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonate-readonly-auditor
rules:
  - apiGroups: [""]
    resources: ["users"]
    verbs: ["impersonate"]
    resourceNames: ["auditor"]
```

It's useful for admins testing permissions and for proxies that authenticate users themselves. It's also one of the most dangerous permissions you can grant: classic impersonation is all or nothing. If you may impersonate a node or a powerful service account, you get **everything** that identity can do.

### Constrained impersonation

**Constrained impersonation**, alpha in v1.35 and beta (enabled by default) in v1.36, splits this into two separate checks[^6]:

1. May the caller impersonate this identity at all?
2. May the caller perform **this specific action** while impersonating it?

The second check uses new verbs of the form `impersonate-on:<mode>:<verb>`, so you can say "this agent may impersonate the node it runs on, but only to `get` Pods". The motivating example is node agents such as CSI drivers or readiness reporters, which today need broad cluster-wide permissions to do something that only concerns their own node[^6].

## Authorizing the kubelet API

The kubelet has its own HTTPS API on each node (port 10250) for logs, exec, metrics and more. With `--authorization-mode=Webhook` on the kubelet, it asks the API server, with a SubjectAccessReview, whether the caller may access it[^11]. The request is mapped to the `nodes` resource and a subresource based on the path:

| Kubelet path | Resource / subresource |
| --- | --- |
| `/metrics/*` | `nodes/metrics` |
| `/stats/*` | `nodes/stats` |
| `/logs/*` | `nodes/log` |
| `/spec/*` | `nodes/spec` |
| `/configz` | `nodes/configz` |
| `/healthz` | `nodes/healthz` |
| `/pods` | `nodes/pods` |
| everything else | `nodes/proxy` |

Before **fine-grained kubelet authorization**, the last three fell through to `nodes/proxy`. Monitoring agents that only wanted to read Pod lists or health checks were given `nodes/proxy`, which also covers running commands in containers through the kubelet. Fine-grained authorization went alpha in v1.32, beta in v1.33, and stable in v1.36, where its feature gate is locked on[^7]. If your monitoring stack still has `nodes/proxy`, check whether `nodes/metrics`, `nodes/stats` and `nodes/pods` are enough, and remove it.

## Where authorization stops

Authorization answers "may this identity perform this verb on this resource?". Many real policies need more than that:

- "Developers may create Pods, but not privileged ones."
- "Nobody may delete the `production` namespace, not even admins."
- "Services of type `LoadBalancer` need an approval label."

None of these can be expressed in RBAC, because they depend on the object's content, or need to deny what RBAC allows. Today you handle them in **admission control**, with **ValidatingAdmissionPolicy** (stable since v1.30), Pod Security Admission, or admission webhooks such as Kyverno and Gatekeeper[^10]. The usual pattern is: RBAC grants a bit more than you'd like, and an admission policy takes back what's dangerous.

### Conditional authorization

That split is what **conditional authorization** aims to close. It's alpha in v1.37 behind the `ConditionalAuthorization` feature gate, and disabled by default[^8].

The idea: an authorizer can now answer **Conditional** in addition to allow, deny and no opinion. Along with that answer it returns conditions, for example "only if `spec.storageClassName == 'dev'`". The API server carries the conditions through the pipeline, and a built-in admission plugin, `AuthorizationConditionsEnforcer`, evaluates them against the actual object during validating admission[^8]. The authorizer gets a guarantee that its policy is enforced, without having to rely on some separate admission webhook being installed and configured correctly. `kubectl auth can-i` can show users the conditions they are subject to.

For now, conditions only apply to write requests (`create`, `update`, `patch`, `delete`, `deletecollection`) and connect requests, and in v1.37 they come from webhook authorizers that opt in. It's the foundation for future work such as richer RBAC rules, and worth watching, not running in production yet.

## Common pitfalls

Several permissions look harmless and aren't[^9]:

| Permission | Why it's dangerous |
| --- | --- |
| `create` on Pods or any workload (Deployments, Jobs, CronJobs…) | The new Pod can run as **any** service account in the namespace and mount any Secret there. Workload creation in a namespace means having the permissions of everything in it. |
| `get`, `list` or `watch` on Secrets | `list` and `watch` return the contents. Reads service account tokens and every credential in scope. |
| `create` on `serviceaccounts/token` | Mints tokens for service accounts directly. |
| `create` on `pods/exec`, `pods/attach` | A shell in someone else's container, with their credentials. |
| `nodes/proxy` | Full access to the kubelet API, including exec, on every node. |
| `bind`, `escalate`, `impersonate` without `resourceNames` | Effectively cluster-admin. |
| `update` on `validatingwebhookconfigurations`, `mutatingwebhookconfigurations` or admission policies | Can disable the guardrails that admission control provides. |
| `patch` on Nodes | Can change labels and taints that steer sensitive workloads onto a node the attacker controls. |
| Anything granted to `system:authenticated` | Applies to every identity in the cluster, including every service account. |

Other habits that cause trouble:

- **Binding roles to the `default` service account.** Every Pod that doesn't set `serviceAccountName` gets those permissions.
- **Using ClusterRoleBindings for namespaced needs.** A RoleBinding to a ClusterRole gives the same permissions, in one namespace.
- **Long-lived `system:masters` credentials** on laptops and in CI. They bypass every authorizer and can't be revoked.
- **Never reviewing aggregated roles** after installing charts and operators.

## Good practices

- **Run `Node,RBAC`** as the base of the chain, together with the NodeRestriction admission plugin.
- **Use the AuthorizationConfiguration file** if you run webhooks. Put deny-capable webhooks before RBAC, use `matchConditions` to keep them off the hot path, and choose `failurePolicy` deliberately.
- **Grant to groups, not users**, and let your identity provider manage membership.
- **Prefer namespaced grants.** Use RoleBindings, also when the role is a ClusterRole.
- **Start from `view`, `edit` and `admin`** before writing custom roles, and avoid wildcards.
- **Treat workload creation, Secret reads, `bind`, `escalate`, `impersonate` and `nodes/proxy` as privileged**, and review who has them.
- **Keep `system:masters` for break-glass only**, and use an RBAC-bound admin group for everyday administration.
- **Check before and after changes** with `kubectl auth can-i --as=...`, and use the audit log's authorization reason to trace unexpected access.
- **Move monitoring off `nodes/proxy`** now that fine-grained kubelet authorization is stable.
- **Use admission policies for content rules** (ValidatingAdmissionPolicy, Pod Security Admission), and keep an eye on conditional authorization.

## Bottom line

- **Authorization decides what an authenticated identity may do**, based on request attributes: user, groups, verb, resource, subresource, namespace and name. It never sees the request body.
- **Authorizers form a chain.** The first *allow* or *deny* wins; if all have *no opinion*, the request is denied. RBAC only allows, so a webhook that must deny goes before it.
- **RBAC is additive**, built from Roles, ClusterRoles and their bindings, and protects itself against escalation unless you grant `bind` or `escalate`.
- **The Node authorizer and NodeRestriction** confine each kubelet to its own Pods, and **fine-grained kubelet authorization** removes the need for `nodes/proxy`.
- **Structured authorization configuration** (stable since v1.32) gives you multiple webhooks, CEL match conditions and reload without restart.
- **New in the last releases**: selector-aware authorization (v1.34), constrained impersonation (beta in v1.36), and conditional authorization (alpha in v1.37), which starts to bring object content into the authorization decision.

## References

[^1]: [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/), Kubernetes documentation: request attributes, verb mapping, special verbs, authorization modes, decisions and the authorizer chain.
[^2]: [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), Kubernetes documentation: Roles, bindings, aggregation, default roles and privilege escalation prevention.
[^3]: [Using Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/), Kubernetes documentation.
[^4]: [Webhook Mode](https://kubernetes.io/docs/reference/access-authn-authz/webhook/), Kubernetes documentation: the SubjectAccessReview request and response format.
[^5]: [KEP-3221: Structured Authorization Configuration](https://www.kubernetes.dev/resources/keps/3221/), Kubernetes Enhancement Proposal (alpha v1.29, beta v1.30, stable v1.32), and [Kubernetes 1.30: Multi-Webhook and Modular Authorization Made Much Easier](https://kubernetes.io/blog/2024/04/26/multi-webhook-and-modular-authorization-made-much-easier/), Kubernetes blog.
[^6]: [User Impersonation](https://kubernetes.io/docs/reference/access-authn-authz/user-impersonation/), Kubernetes documentation, and [KEP-5284: Constrained Impersonation](https://www.kubernetes.dev/resources/keps/5284/) (alpha v1.35, beta v1.36).
[^7]: [Kubernetes v1.36: Fine-Grained Kubelet API Authorization Graduates to GA](https://kubernetes.io/blog/2026/04/24/kubernetes-v1-36-fine-grained-kubelet-authorization-ga/), Kubernetes blog, and [KEP-2862](https://www.kubernetes.dev/resources/keps/2862/).
[^8]: [KEP-5681: Conditional Authorization](https://www.kubernetes.dev/resources/keps/5681/), Kubernetes Enhancement Proposal (alpha in v1.37).
[^9]: [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/), Kubernetes documentation: privilege escalation risks.
[^10]: [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/), Kubernetes documentation.
[^11]: [Kubelet authentication/authorization](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/), Kubernetes documentation.
