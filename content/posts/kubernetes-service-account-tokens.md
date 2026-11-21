---
title: "Using Kubernetes Service Account Tokens"
date: 2026-11-07
draft: true
description: "What Kubernetes service account tokens are, how Pods get them, how to use them inside and outside the cluster, how to validate and revoke them, and how to move off long-lived Secret-based tokens."
summary: "Every Pod in Kubernetes gets an identity and a token to prove it. Here is what that token is, how it gets into your Pod, how to use it to talk to the API server and to other systems, and how to keep it short-lived and safe."
tags: ["kubernetes", "security", "authentication", "service accounts", "rbac", "oidc"]
categories: ["Programming"]
---

Every Pod in Kubernetes runs as *someone*. That someone is a **ServiceAccount**, and the way a Pod proves who it is, to the API server or to anything else, is a **service account token**. Most of the time this happens without you noticing: a token file appears in the container and your client library picks it up. But as soon as you need a CI pipeline to deploy to a cluster, a Pod to talk to Vault, or a workload to get cloud credentials without a stored secret, you have to understand what these tokens are and how to use them properly.

This post covers what a service account token is, how it gets into a Pod, how to use it inside and outside the cluster, how other systems can validate it, and how to clean up the old long-lived tokens that are probably still lying around in older clusters.

Everything here is based on Kubernetes v1.37, the current release at the time of writing.

## Service accounts in one paragraph

Kubernetes has two kinds of identity. *Users* are humans, and Kubernetes has no User object; people are authenticated by something outside the cluster, such as client certificates or an OIDC provider. *Service accounts* are for processes, and they are real API objects: namespaced, cheap to create, and meant to be created per application[^1]. Every namespace gets one called `default`, and a Pod that doesn't name a service account runs as that one[^2].

A service account on its own can do nothing. What it is allowed to do is decided by RBAC: Roles and RoleBindings that grant permissions to the subject `system:serviceaccount:<namespace>:<name>`. The token only answers *who* you are; RBAC answers *what you may do*.

## What a token actually is

A service account token is a signed **JSON Web Token (JWT)**, so it's worth knowing what that is first.

### What's a JWT?

A JWT is a standard format for passing a set of facts, called *claims*, from one party to another in a way that can't be tampered with[^9]. It is a single string made of three parts, each base64url-encoded and separated by dots:

```text
eyJhbGciOiJSUzI1NiIsImtpZCI6Ik...  .  eyJhdWQiOlsiaHR0cHM6Ly9rdWJl...  .  Gk3pX0vQ2mR8...
└──────────── header ────────────┘     └──────────── payload ────────────┘     └─ signature ─┘
```

- The **header** says how the token was signed: the algorithm (`alg`, for example `RS256`) and the ID of the signing key (`kid`).
- The **payload** holds the claims: who the token is about, who it's for, when it expires, and so on.
- The **signature** is computed over the header and payload with the issuer's private key.

Two properties matter for everything that follows:

**A JWT is encoded, not encrypted.** Base64url is just a way of writing bytes as text. Anyone who has the token can decode it and read every claim. Never put anything secret in a claim, and treat the token itself as the secret.

**The signature makes it tamper-proof.** Change a single character in the header or payload, for example to claim a different service account, and the signature no longer matches. A receiver checks a token like this:

1. Read `kid` from the header and look up the matching public key. For Kubernetes, the keys are published at the cluster's JWKS endpoint, covered later in this post.
2. Verify the signature with that key.
3. Check the claims: is `iss` the issuer I trust, is `aud` me, has `exp` passed?

Kubernetes signs tokens with an asymmetric key, RSA or ECDSA (`RS256`, `ES256`, `ES384` or `ES512`)[^1]. Only the API server holds the private key; the public keys can be handed out freely, because they can verify tokens but not create them. That is what lets systems outside the cluster validate tokens without ever being able to forge one.

### Inside a service account token

Decoded, the payload of a modern service account token looks like this[^1]:

```json
{
  "aud": ["https://kubernetes.default.svc.cluster.local"],
  "exp": 1729605240,
  "iat": 1729601640,
  "nbf": 1729601640,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "jti": "aed34954-b33a-4142-b1ec-389d6bbb4936",
  "sub": "system:serviceaccount:my-namespace:my-serviceaccount",
  "kubernetes.io": {
    "namespace": "my-namespace",
    "serviceaccount": { "name": "my-serviceaccount", "uid": "14ee3fa4-..." },
    "pod":  { "name": "my-pod",  "uid": "5e0bd49b-..." },
    "node": { "name": "my-node", "uid": "646e7c5e-..." }
  }
}
```

The fields that matter:

| Claim | Meaning |
| --- | --- |
| `iss` | The issuer: who signed the token. Set by the API server's `--service-account-issuer` flag. |
| `sub` | The identity: `system:serviceaccount:<namespace>:<name>`. |
| `aud` | The audience: who the token is *for*. A token for Vault should not be accepted by the API server, and vice versa. |
| `exp`, `iat`, `nbf` | Expiry, issued-at and not-before times. Modern tokens always expire. |
| `jti` | A unique ID for this token, which shows up in audit logs so you can trace what a specific token did. |
| `kubernetes.io` | Private claims: namespace, service account, and the object the token is **bound** to (here a Pod, plus the Node it runs on). |

You can decode any token yourself by taking the middle segment:

```bash
kubectl create token my-sa | jq -R 'split(".") | .[1] | @base64d | fromjson'
```

Anyone who holds the token can use it until it expires. Treat it like a password.

## A short history: from Secrets to bound tokens

Service account tokens used to work very differently, and the old way still haunts many clusters, so it's worth knowing how we got here.

**Before v1.22: Secret-based tokens.** For every service account, a controller created a Secret of type `kubernetes.io/service-account-token` holding a JWT with no expiry and no audience. Pods mounted that Secret. The token was valid as long as the Secret existed, which was usually forever, and anyone who could read Secrets in the namespace could copy it and use it from anywhere.

**v1.22: bound tokens become the default.** The TokenRequest API and projected token volumes went stable[^3]. Pods now get tokens that are short-lived, have an audience, and are *bound* to the Pod: when the Pod is deleted, its tokens stop working.

**v1.24: no more automatic Secrets.** The control plane stopped generating token Secrets for new service accounts[^4]. This is the change that surprised people after upgrades: the Secret their tooling used to read the token from simply wasn't there.

**v1.29 and v1.30: cleanup of old tokens.** The legacy token cleaner, stable since v1.30, marks auto-generated token Secrets as invalid after a year of disuse, and deletes them after another year[^1][^4].

The direction is clear: tokens should be short-lived, tied to something, and never stored if you can avoid it.

## How a Pod gets its token

When you create a Pod, three components cooperate[^1]:

1. **The ServiceAccount admission controller** sets `spec.serviceAccountName` to `default` if you didn't set it, checks the service account exists, and (unless you opted out) adds a projected volume and mounts it into every container at `/var/run/secrets/kubernetes.io/serviceaccount`.
2. **The kubelet** sees the projected volume and calls the **TokenRequest API** to get a token for that service account, bound to that Pod, with the API server as audience.
3. **The kubelet keeps it fresh.** It rewrites the token file before it expires: when the token is older than 80% of its lifetime or older than 24 hours[^2].

The injected volume looks like this:

```yaml
- name: kube-api-access-<random-suffix>
  projected:
    sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          name: kube-root-ca.crt
          items:
            - key: ca.crt
              path: ca.crt
      - downwardAPI:
          items:
            - path: namespace
              fieldRef:
                fieldPath: metadata.namespace
```

So inside the container you find three files:

```text
/var/run/secrets/kubernetes.io/serviceaccount/
├── token       # the JWT
├── ca.crt      # CA bundle to verify the API server's certificate
└── namespace   # the Pod's namespace
```

The token is bound to the Pod. The API server rejects it once the Pod is gone, or more precisely, 60 seconds after the Pod's deletion timestamp, even if finalizers keep the object around[^2].

### Why `3607` seconds?

That odd number is a compatibility trick. Plenty of older client code reads the token file once at startup and never again, which would break after an hour. When the API server sees the default request of 3607 seconds, and `--service-account-extend-token-expiration` is on (the default), it issues a token that actually lives much longer, but records a `warnafter` time in the private claims at roughly the one-hour mark. If a client keeps using the token after that point, the request still succeeds but is flagged in the audit log as a stale token. This gives you a way to find clients that don't reload their tokens, without breaking them. The fix on the client side is simple: **re-read the token file regularly**. Re-reading every few minutes is enough[^2].

## Using the token from inside a Pod

### With a client library

Most of the time you don't touch the token at all. The official client libraries have an "in-cluster" mode that reads the three files above and connects to `https://kubernetes.default.svc`. In Go:

```go
config, err := rest.InClusterConfig()
if err != nil {
    panic(err)
}
clientset, err := kubernetes.NewForConfig(config)
```

`client-go` re-reads the token file periodically, so rotation by the kubelet is handled for you[^5].

### By hand

To see what's really happening, here is the same thing with `curl` from inside a container:

```bash
SA=/var/run/secrets/kubernetes.io/serviceaccount
TOKEN=$(cat $SA/token)
NS=$(cat $SA/namespace)

curl --cacert $SA/ca.crt \
     -H "Authorization: Bearer $TOKEN" \
     https://kubernetes.default.svc/api/v1/namespaces/$NS/pods
```

If you haven't granted the service account anything, this returns `403 Forbidden`. The token authenticated you; RBAC said no.

### Giving the service account permissions

Here is a complete example: a dedicated service account that may only read Pods in its own namespace, and a Deployment that uses it.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
  namespace: demo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-pods
  namespace: demo
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-read-pods
  namespace: demo
subjects:
  - kind: ServiceAccount
    name: pod-reader
    namespace: demo
roleRef:
  kind: Role
  name: read-pods
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels: { app: inventory }
  template:
    metadata:
      labels: { app: inventory }
    spec:
      serviceAccountName: pod-reader
      containers:
        - name: app
          image: registry.example.com/inventory:1.4.2
```

Check the result without deploying anything, by impersonating the service account:

```bash
kubectl auth can-i list pods -n demo --as=system:serviceaccount:demo:pod-reader
# yes
kubectl auth can-i delete pods -n demo --as=system:serviceaccount:demo:pod-reader
# no
```

Note that `serviceAccountName` can only be set when a Pod is created. To change it, you change the template and let the Pods be replaced[^2].

### Opting out

Most application Pods never talk to the Kubernetes API, and for them the token is pure risk: anyone who breaks into the container gets a credential for free. Turn it off, either for every Pod using a service account or for a single Pod:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web
automountServiceAccountToken: false
```

```yaml
spec:
  serviceAccountName: web
  automountServiceAccountToken: false
```

If both are set, the Pod's value wins[^2]. A good default is: one service account per application, automounting off, and turned back on only for the workloads that need it.

"But that service account has no permissions" is not a good reason to leave the token mounted. The token still makes the caller an authenticated identity, a member of groups like `system:authenticated` and `system:serviceaccounts`, so anything granted to those groups applies, and an attacker can use it to explore what the cluster exposes. Permissions also drift: someone binds a role to `default` or to a shared service account to fix an unrelated problem, and every Pod using it quietly gains that access. Even read-only access to Pods, Services or ConfigMaps tells an attacker what else is running and where to go next. Turning automounting off removes the credential instead of relying on RBAC staying right forever.

Turning it off only removes the default API server token. You can still add your own projected volume with a custom audience, as described in the next section, so a Pod can have a token for Vault or a cloud provider and no token that works against the Kubernetes API. For workload identity, that is the cleanest setup.

## Tokens for other systems: custom audiences

The default token is meant for the Kubernetes API server. But the same mechanism can produce tokens for *any* system that trusts your cluster: Vault, an internal API, a message broker. You ask for them with your own projected volume and a custom audience:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: build-robot
  containers:
    - name: app
      image: registry.example.com/app:2.0.0
      volumeMounts:
        - name: vault-token
          mountPath: /var/run/secrets/tokens
          readOnly: true
  volumes:
    - name: vault-token
      projected:
        sources:
          - serviceAccountToken:
              path: vault-token
              audience: vault
              expirationSeconds: 7200
```

This gives the container a token at `/var/run/secrets/tokens/vault-token` with `aud: ["vault"]`, valid for two hours and refreshed by the kubelet[^2]. The `path` value is simply a file name you choose: it is relative to the volume's `mountPath`, so the two together make up the full location, and your application (for Vault, the Kubernetes auth `token_path` setting) must be told to read the token from there. Because the audience is `vault`, the Kubernetes API server will refuse it, so even if Vault (or whoever holds the token) leaks it, it can't be used against your cluster. And the reverse: Vault should refuse tokens whose audience isn't `vault`.

This is the most important habit with service account tokens: **one audience per receiver**. Never send your API server token to a third party.

The minimum lifetime you can ask for is 10 minutes, and the cluster may cap the maximum with `--service-account-max-token-expiration`.

## Tokens outside the cluster

Sometimes the thing that needs a token isn't a Pod: a CI pipeline, a script on a laptop, a monitoring system in another cluster.

### Short-lived tokens with `kubectl create token`

`kubectl create token` calls the TokenRequest API directly and prints a token[^2]:

```bash
# Default lifetime (one hour)
kubectl create token ci-deployer -n deploy

# A specific lifetime and audience
kubectl create token ci-deployer -n deploy --duration=30m --audience=https://kubernetes.default.svc.cluster.local
```

The API server may give you a shorter (or occasionally longer) lifetime than you asked for, so don't hard-code assumptions about expiry; read `exp` if you need it.

To use it with `kubectl`, put it into a kubeconfig:

```bash
TOKEN=$(kubectl create token ci-deployer -n deploy --duration=15m)

kubectl config set-cluster prod --server=https://k8s.example.com:6443 \
  --certificate-authority=ca.crt --embed-certs=true
kubectl config set-credentials ci-deployer --token="$TOKEN"
kubectl config set-context ci --cluster=prod --user=ci-deployer --namespace=deploy
kubectl config use-context ci

kubectl auth whoami
# Username: system:serviceaccount:deploy:ci-deployer
```

Of course, *something* has to be allowed to create the token in the first place, which is the chicken-and-egg problem of CI. If your CI system can run inside the cluster, or your CI provider can federate with your cluster's OIDC issuer (see below), you can avoid storing any Kubernetes credential at all.

### Binding a token to an object

Tokens from `kubectl create token` can be bound to a Pod, a Secret or a Node. The token then dies when that object is deleted[^1][^2]:

```bash
# Valid until it expires or node-001 is deleted
kubectl create token node-agent -n monitoring \
  --bound-object-kind Node --bound-object-name node-001 \
  --bound-object-uid "$(kubectl get node node-001 -o jsonpath='{.metadata.uid}')"
```

Binding to a **Secret** is a neat trick for revocation: create an empty Secret, bind a token to it, and when you want to kill the token, delete the Secret. Binding directly to a Node is stable since v1.33, and in v1.37 there is alpha support for binding to admission webhook configurations, used to authenticate the API server to webhooks[^1].

### Long-lived tokens, if you really must

Some tools still expect a token that never expires. You can still get one, but now you have to ask for it explicitly by creating a Secret with an annotation[^2]:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: legacy-tool-token
  namespace: tools
  annotations:
    kubernetes.io/service-account.name: legacy-tool
type: kubernetes.io/service-account-token
```

The control plane fills in `token`, `ca.crt` and `namespace`. The token is valid until the Secret or the service account is deleted. Use this as a last resort, keep the RBAC tight, and don't add the Secret to the service account's `secrets` field, because then the legacy token cleaner treats it as auto-generated and may eventually invalidate it[^1].

## Validating tokens

If you are building the system that *receives* service account tokens, there are two ways to check them.

### Ask the API server: TokenReview

Send the token to the API server and it tells you whether it is valid and who it belongs to[^1]:

```yaml
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  token: <token>
  audiences: ["vault"]
```

```bash
kubectl create -o yaml -f tokenreview.yaml
```

The response contains `authenticated: true` (or not), the username, groups, and extra information such as the bound Pod's name and UID and the token's `jti`. TokenReview is not stored anywhere, so `kubectl get tokenreview` won't work; it's a question, not an object. The caller needs RBAC permission to create TokenReviews (the built-in `system:auth-delegator` ClusterRole grants it).

The big advantage: TokenReview checks that the bound object **still exists**. A token for a deleted Pod fails here.

### Validate offline: OIDC discovery

The alternative is to check the JWT signature yourself, which is what cloud providers do. The API server publishes what you need, in a format compatible with OpenID Connect[^2]:

- `{issuer}/.well-known/openid-configuration`: the discovery document, pointing to the keys.
- `/openid/v1/jwks`: the public keys that sign service account tokens.

```bash
kubectl get --raw /.well-known/openid-configuration | jq
kubectl get --raw /openid/v1/jwks | jq
```

A relying party fetches the keys, verifies the signature, then checks `iss`, `aud` and `exp`. Since many API servers aren't reachable from the internet, you can publish these two documents somewhere public (a bucket, a static site) and point `--service-account-jwks-uri` there. The issuer URL must use `https`.

There is a catch, and the documentation is explicit about it: **offline validation does not know whether the bound Pod still exists**. A token for a deleted Pod stays valid to an offline verifier until it expires[^1]. That's acceptable when tokens are short-lived, and it's one more reason to keep them that way. If you need the stronger guarantee, use TokenReview.

## Workload identity: trading tokens for cloud credentials

Offline validation is what makes **workload identity federation** possible. Instead of storing an AWS key or a GCP service account key in a Kubernetes Secret, you register your cluster's issuer URL with the cloud provider as a trusted OIDC provider. The flow is then:

1. The Pod gets a projected token with an audience the cloud provider expects (for example `sts.amazonaws.com`).
2. The Pod sends that token to the cloud provider's token exchange endpoint.
3. The provider fetches your cluster's JWKS, verifies the token, checks that `sub` matches the policy you wrote (for example "only `system:serviceaccount:payments:api`"), and returns short-lived cloud credentials.

No long-lived cloud secret exists anywhere. Managed services such as EKS, GKE and AKS wire this up for you, but it is built on exactly the mechanism above and works with self-managed clusters too, as long as the discovery documents are reachable by the provider[^6].

The same idea now reaches image pulls. With service account tokens for kubelet image credential providers (beta and enabled by default since v1.34), the kubelet can hand a credential provider plugin a token bound to the Pod that is pulling the image, and the plugin exchanges it for registry credentials. Pull access then follows the workload's identity rather than the node's, and you don't need `imagePullSecrets` at all[^7].

## Threat model: how tokens get abused

Most of the advice in this post makes more sense from the attacker's side. Here is how service account tokens typically get stolen and what an attacker does with them.

### How tokens leak

- **A compromised container.** Code execution in the container, or a bug that lets someone read files (a path traversal, a debug endpoint), gives them `/var/run/secrets/kubernetes.io/serviceaccount/token`.
- **Readable Secrets.** Anyone allowed to `get` or `list` Secrets in a namespace can read every long-lived token Secret in it. `list` is enough: it returns the contents, not just the names.
- **A compromised node.** The kubelet holds the tokens of every Pod on its node. The Node authorizer and the NodeRestriction admission plugin limit a kubelet to Pods scheduled on its own node, which keeps one bad node from becoming a whole-cluster breach.
- **Careless handling.** Tokens printed in CI logs, pasted into tickets, or baked into kubeconfig files on laptops.

### What an attacker does with one

The first step is to find out what the token is allowed to do:

```bash
kubectl auth can-i --list --token="$STOLEN_TOKEN"
```

Then they look for a way to get more. Several permissions are more powerful than they look[^8]:

| Permission | Why it's dangerous |
| --- | --- |
| `create` on Pods (or Deployments, Jobs, CronJobs…) | The new Pod can run as **any** service account in the namespace and mount its token. In practice, permission to create workloads in a namespace means having the permissions of every service account in it. |
| `get` / `list` on Secrets | Reads long-lived token Secrets, and every other secret in the namespace. |
| `create` on `serviceaccounts/token` | Mints fresh tokens for service accounts directly through the TokenRequest API. |
| `bind`, `escalate`, `impersonate` | Lets the holder grant themselves roles they don't have, or act as another user or service account. |

### How they stay in

A short-lived token expires, so an attacker who wants to keep access turns it into something longer-lasting:

- **Creating a long-lived token Secret** for a service account they control. It never expires and survives the original Pod being deleted.
- **Adding a RoleBinding or ClusterRoleBinding** that gives their service account more permissions.
- **Creating a new service account** with an innocent-looking name.

### How the defences map to the threats

| Threat | Defence |
| --- | --- |
| Token stolen from a container | Turn off automounting where it isn't needed; keep lifetimes short. |
| Token used against another system | One audience per receiver. |
| Escalation through workload creation | Only grant `create` on Pods and workloads to accounts you'd trust with every service account in that namespace; keep powerful service accounts in their own namespace. |
| Long-lived tokens read from Secrets | Don't create them; restrict `get`/`list` on Secrets; audit `kubernetes.io/service-account-token` Secrets. |
| Persistence through new Secrets or bindings | Use audit logging and alerts on creation of token Secrets, RoleBindings and ClusterRoleBindings. |
| Use after the Pod is gone | Bound tokens die with their Pod; TokenReview checks this, offline validation doesn't. |

## Revoking tokens

There is no "revoke this token" button. TokenRequest tokens aren't stored, so there is no list to remove them from[^1]. Your options, from narrowest to broadest:

| Situation | What to do |
| --- | --- |
| A Pod's token leaked | Delete the Pod. Its bound tokens stop working. |
| A token bound to a Secret or Node | Delete that object. |
| A long-lived Secret-based token | Delete the Secret. |
| Any token of a service account, including ones you can't track | Delete and re-create the service account. Its UID changes, which invalidates **every** token issued for it. |
| The signing key itself is compromised | Rotate the service account signing key. This invalidates all tokens in the cluster. |

And the best option: keep lifetimes short so a leaked token is worthless soon anyway.

## Auditing old tokens

If your cluster has been upgraded from versions older than v1.24, it probably still contains long-lived token Secrets. Find them:

```bash
kubectl get secrets -A --field-selector type=kubernetes.io/service-account-token
```

The control plane tracks their use for you. Each one gets a `kubernetes.io/legacy-token-last-used` label with the date it was last used. The legacy token cleaner, part of `kube-controller-manager`, runs once a day and, for auto-generated tokens that aren't mounted by any Pod and haven't been used for a year, adds a `kubernetes.io/legacy-token-invalid-since` label. From then on, using the token fails. If nobody complains for another year, the Secret is deleted. Both periods are set with `--legacy-service-account-token-clean-up-period`[^1].

If something does break because of an invalidated token, the request is recorded with an audit annotation and the `invalid_legacy_auto_token_uses_total` metric goes up. Removing the `invalid-since` label re-enables the token temporarily, but the real fix is to move that client to a short-lived token[^1].

A quick sweep:

```bash
kubectl get secrets -A \
  --field-selector type=kubernetes.io/service-account-token \
  -L kubernetes.io/legacy-token-last-used \
  -L kubernetes.io/legacy-token-invalid-since
```

## Good practices

- **One service account per application.** Don't let unrelated workloads share `default` and its permissions.
- **Turn off automounting** for workloads that don't call the Kubernetes API.
- **Grant the minimum RBAC**, scoped to a namespace with Role and RoleBinding where possible. Check with `kubectl auth can-i --as=...`.
- **Use a separate audience for every receiver.** API server tokens go to the API server, Vault tokens to Vault.
- **Keep lifetimes short** and make clients re-read the token file. Watch the audit log for stale-token warnings.
- **Prefer `kubectl create token` and federation** over long-lived Secret-based tokens. If you must create one, document why.
- **Use TokenReview** when you need to know a bound object still exists; offline validation can't tell you.
- **Audit and remove legacy token Secrets** in clusters that predate v1.24.
- **Protect the signing key.** Since v1.36 the API server can delegate signing to an external signer, such as an HSM or cloud KMS, so the private key never sits on the control plane's disk[^1].

## Bottom line

- **A service account token is a signed JWT** that says which service account you are, who the token is for, and when it expires. RBAC decides what it can do.
- **Pods get bound, short-lived tokens automatically**, through a projected volume that the kubelet keeps refreshed. When the Pod dies, the token dies.
- **Use projected volumes with custom audiences** to give workloads tokens for other systems, and `kubectl create token` for anything outside the cluster.
- **Other systems validate tokens** with TokenReview or with the cluster's OIDC discovery documents, which is the basis for keyless cloud access and, increasingly, image pulls.
- **Long-lived Secret-based tokens are legacy.** Find them, replace them, and let the cleaner do the rest.

## References

[^1]: [Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/), Kubernetes documentation: bound tokens, private claims, TokenReview, the token volume, legacy token cleanup, revocation and external signing.
[^2]: [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/), Kubernetes documentation: automounting, `kubectl create token`, long-lived tokens, token projection, kubelet refresh and service account issuer discovery.
[^3]: [KEP-1205: Bound Service Account Tokens](https://www.kubernetes.dev/resources/keps/1205/), Kubernetes Enhancement Proposal: the TokenRequest API and projected token volumes, stable in v1.22.
[^4]: [KEP-2799: Reduction of Secret-based Service Account Tokens](https://www.kubernetes.dev/resources/keps/2799/), Kubernetes Enhancement Proposal: ending auto-generated token Secrets and cleaning up unused ones.
[^5]: [Accessing the Kubernetes API from a Pod](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/), Kubernetes documentation.
[^6]: [Workload identity](https://docs.cloud.google.com/anthos/clusters/docs/multi-cloud/aws/concepts/workload-identity), Google Cloud documentation: how a cluster's built-in OIDC provider and projected tokens are used for identity federation.
[^7]: [Kubernetes v1.34: Service Account Token Integration for Image Pulls Graduates to Beta](https://kubernetes.io/blog/2025/09/03/kubernetes-v1-34-sa-tokens-image-pulls-beta/), Kubernetes blog, and [KEP-4412](https://www.kubernetes.dev/resources/keps/4412/).
[^8]: [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/), Kubernetes documentation: privilege escalation risks, including workload creation, reading Secrets, and the `bind`, `escalate` and `impersonate` verbs.
[^9]: [RFC 7519: JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519), IETF.
