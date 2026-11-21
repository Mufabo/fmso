---
title: "Secrets in Kubernetes"
date: 2026-11-12
draft: true
description: "What Kubernetes Secrets are, how they are stored and delivered to Pods, why base64 is not encryption, how to encrypt them at rest with KMS v2, how to lock down access, and when to reach for Sealed Secrets, SOPS, External Secrets Operator or the Secrets Store CSI Driver."
summary: "A Kubernetes Secret is a convenient place to keep a password, but out of the box it is neither encrypted nor very private. Here is how Secrets actually work, where they leak, and how to make them worthy of the name."
tags: ["kubernetes", "security", "secrets", "encryption", "kms", "rbac", "gitops"]
categories: ["Programming"]
---

Almost every application needs something it shouldn't print in a log: a database password, an API key, a TLS private key. Kubernetes has a dedicated object for that, the **Secret**. The name suggests more protection than you get by default. A fresh cluster stores Secrets in plain text in etcd, anyone who can create a Pod in a namespace can read every Secret in it, and the values in your YAML are only base64-encoded.

None of that makes Secrets useless. It means they are a delivery mechanism with a few sharp edges, and you have to add the protection yourself. This post covers what a Secret is, how it gets from the API server into a container, where secrets leak, and what to do about each leak: encryption at rest, RBAC, and the external tools that keep secret values out of Git and sometimes out of the cluster altogether.

Everything here is based on Kubernetes v1.37, the current release at the time of writing.

## What a Secret is

A Secret is a namespaced API object that holds a small amount of sensitive data as key-value pairs[^1]. It works much like a ConfigMap; the difference is intent, and a handful of extra precautions Kubernetes takes because of that intent.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: shop
type: Opaque
data:
  username: c2hvcA==          # "shop"
  password: czNjcjN0LXBhc3M=  # "s3cr3t-pass"
```

Values in `data` must be base64-encoded. If you'd rather write plain text, use `stringData`; the API server merges it into `data` on write, and `stringData` wins if a key is in both[^1].

```yaml
stringData:
  username: shop
  password: s3cr3t-pass
```

A few constraints worth knowing:

- **Size.** A single Secret is limited to 1 MiB, to keep the API server and kubelets from running out of memory. A ResourceQuota can cap how many Secrets a namespace may have[^1].
- **Names.** Keys may contain only alphanumerics, `-`, `_` and `.`.
- **Namespaces.** A Pod can only use Secrets from its own namespace.

### Base64 is not encryption

This is the single most common misunderstanding, so it deserves its own heading. Base64 is a way of writing bytes as text. It exists so that binary data such as a certificate can sit inside JSON or YAML. It has no key and protects nothing[^2]:

```bash
kubectl get secret db-credentials -n shop -o jsonpath='{.data.password}' | base64 -d
# s3cr3t-pass
```

If you can read the Secret object, you can read the secret.

### Built-in types

The `type` field tells Kubernetes (and humans) what's inside, and for some types the API server checks that the expected keys exist[^1]:

| Type | Used for |
| --- | --- |
| `Opaque` | Anything. The default. |
| `kubernetes.io/tls` | A certificate and key, in `tls.crt` and `tls.key`. Used by Ingress and Gateway controllers. |
| `kubernetes.io/dockerconfigjson` | Registry credentials for pulling images. |
| `kubernetes.io/basic-auth` | `username` and `password`. |
| `kubernetes.io/ssh-auth` | An `ssh-privatekey`. |
| `kubernetes.io/service-account-token` | Legacy long-lived service account tokens. Avoid; see my post on service account tokens. |
| `bootstrap.kubernetes.io/token` | Node bootstrap tokens, used by kubeadm. |

You can also invent your own type. Convention is to prefix it with a domain you own, for example `example.com/payment-api-key`.

### Creating Secrets

The imperative way is the easiest way to avoid writing a secret into a file at all:

```bash
# From literals
kubectl create secret generic db-credentials -n shop \
  --from-literal=username=shop \
  --from-literal=password='s3cr3t-pass'

# From files (the file name becomes the key)
kubectl create secret generic api-key -n shop --from-file=./api-key.txt

# A TLS Secret
kubectl create secret tls web-tls -n shop --cert=tls.crt --key=tls.key

# Registry credentials
kubectl create secret docker-registry regcred -n shop \
  --docker-server=registry.example.com \
  --docker-username=ci --docker-password="$REGISTRY_TOKEN"
```

Be aware that `--from-literal` puts the value in your shell history. `--from-file`, or reading from an environment variable as in the last example, avoids that.

## How a Secret reaches a container

A Pod can consume a Secret in two ways: as files in a volume, or as environment variables.

### As files

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api
  namespace: shop
spec:
  containers:
    - name: api
      image: registry.example.com/shop-api:3.1.0
      volumeMounts:
        - name: db
          mountPath: /etc/secrets/db
          readOnly: true
  volumes:
    - name: db
      secret:
        secretName: db-credentials
        defaultMode: 0400
```

Each key becomes a file: `/etc/secrets/db/username` and `/etc/secrets/db/password`. Use `items` to pick specific keys or rename them.

What happens behind the scenes matters for security[^1]:

- A Secret is only sent to a node if a Pod scheduled there needs it.
- The kubelet writes the data to a **tmpfs**, a RAM-backed filesystem, so it never touches the node's disk.
- When the Pod is deleted, the kubelet deletes its copy.
- When the Secret changes, the kubelet updates the files in place, eventually. The delay is the kubelet sync period plus its cache delay, so think "within a minute or two", not instantly.

One catch: a Secret mounted with `subPath` is **never** updated[^1]. If you rotate a credential and your app reads it through a `subPath` mount, the Pod keeps the old value until it restarts.

### As environment variables

```yaml
containers:
  - name: api
    image: registry.example.com/shop-api:3.1.0
    env:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: db-credentials
            key: password
    # Or import every key at once:
    envFrom:
      - secretRef:
          name: db-credentials
```

Environment variables are convenient, and almost every framework reads them. They are also the leakier option:

- **They never update.** A process's environment is fixed when it starts. Rotating the Secret does nothing until the Pod restarts.
- **They spread.** Child processes inherit the whole environment. Crash reporters, debug pages and "print the config at startup" log lines love to dump it.
- **They're readable from outside the process.** Anyone who can read `/proc/<pid>/environ` on the node, or run `kubectl exec ... env`, sees them.

My rule of thumb: prefer files, and use environment variables only when the application gives you no other choice.

### Required or optional

By default a referenced Secret must exist. If it doesn't, the Pod's containers won't start, and the kubelet keeps retrying and reports an Event[^1]. Mark a reference `optional: true` if the application can run without it. If the Secret exists but a referenced key is missing, the Pod fails at startup.

### Only the containers that need it

Mounts and env references are per container. If a Pod has a web server and a signing sidecar, give the key only to the signer. A file-read bug in the web server then can't expose it[^1][^2]. This is cheap isolation and easy to forget.

### Immutable Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v7
immutable: true
stringData:
  password: s3cr3t-pass
```

Setting `immutable: true` means the data can't be changed, only deleted and recreated, and the change can't be undone[^1]. Two benefits: nobody can break running applications with an accidental edit, and the kubelet stops watching the Secret, which takes real load off the API server in large clusters.

Immutable Secrets pair well with versioned names (`db-credentials-v7`). To rotate, create `-v8` and update the Deployment, which triggers a normal rolling update that you can roll back. Kustomize's `secretGenerator` does exactly this for you by appending a content hash to the name.

## Where Secrets leak

Before fixing anything, it helps to list every place a secret value lives in a typical cluster:

| Location | Who can read it |
| --- | --- |
| etcd, and every etcd backup | Anyone with access to etcd or the backup files. **Plain text by default.** |
| The API server | Anyone with `get`, `list` or `watch` on Secrets in that namespace. |
| Any Pod in the namespace | Anyone who can **create Pods** (or Deployments, Jobs…) in that namespace. |
| The node | The kubelet, and anything running privileged on that node. |
| The container | The application, its child processes, and anyone who can `exec` into it. |
| Your Git repository | Everyone who can clone it, forever, including in old commits. |
| CI logs, shell history, laptops | Whoever handles them. |

The rest of the post works through this table from the bottom of the stack up.

## Encrypting Secrets at rest

By default the API server writes resources to etcd exactly as they are; the default "encryption provider" is `identity`, which means none[^3]. Anyone with a copy of the etcd data directory or a snapshot can read every Secret. You can check for yourself on a kubeadm cluster:

```bash
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/shop/db-credentials | hexdump -C | head
```

If you can see `s3cr3t-pass` in the output, your Secrets are not encrypted.

### The EncryptionConfiguration

Encryption at rest is configured by a file passed to the API server with `--encryption-provider-config`. It lists, for each resource type, an ordered set of providers. The **first** provider encrypts new writes; all of them are tried, in order, to decrypt reads[^3].

The available providers are[^3]:

| Provider | What it does | Verdict |
| --- | --- | --- |
| `identity` | Nothing. | The default. Provides no confidentiality. |
| `aescbc` | AES-CBC, key in the config file. | Weak: vulnerable to padding-oracle attacks. Don't pick it for new setups. |
| `aesgcm` | AES-GCM, key in the config file. | Only safe with automated key rotation (the key must be rotated every 200,000 writes). |
| `secretbox` | XSalsa20-Poly1305, key in the config file. | Strong, but the key still sits on the control plane host. |
| `kms` v1 | Envelope encryption via an external KMS. | Deprecated since v1.28. |
| `kms` v2 | Envelope encryption via an external KMS. | Stable since v1.29. **The one to use.** |

The non-KMS providers have one shared weakness: the encryption key lives in a file on every control plane node. That protects you if someone steals an etcd backup, but not if someone gets onto a control plane host, because the key is right there next to the data[^3].

### KMS v2 and envelope encryption

The KMS provider uses **envelope encryption**. Kubernetes encrypts each resource with a **data encryption key (DEK)** using AES-GCM, and the DEK itself is encrypted by a **key encryption key (KEK)** that never leaves your KMS: a cloud KMS, Vault, or an HSM. etcd stores the ciphertext together with the encrypted DEK. To decrypt, the API server asks the KMS to unwrap the DEK[^3][^4].

An attacker now needs both a copy of etcd **and** access to your KMS. KMS v2 also fixed v1's performance problem: instead of calling the KMS for every single write, the API server derives a fresh DEK per encryption from a seed, and only goes back to the KMS when the KEK rotates[^3].

The API server talks to a **KMS plugin** over a local Unix socket. The plugin is usually a static Pod or systemd service on each control plane node, provided by your KMS vendor. Managed services like EKS, GKE and AKS set this up for you; on EKS, recent versions encrypt API data with KMS by default.

A KMS v2 configuration looks like this:

```yaml
# /etc/kubernetes/enc/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          apiVersion: v2
          name: my-kms
          endpoint: unix:///var/run/kmsplugin/socket.sock
          timeout: 3s
      - identity: {}   # lets the API server read Secrets written before encryption was on
```

Then point the API server at it, in `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
- --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
- --encryption-provider-config-automatic-reload=true
```

and mount `/etc/kubernetes/enc` into the static Pod. With automatic reload on, the API server picks up config changes within about a minute, so you can rotate keys without restarts[^3]. If you have several control plane nodes, the config must be identical on all of them, or some API servers won't be able to read what others wrote.

### Encrypting what's already there

Enabling encryption only affects **new writes**. Existing Secrets stay in plain text until they're written again. Rewrite them all:

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

It's safe to run more than once; conflicts just mean you retry[^3]. Then verify with `etcdctl` again: the stored value should now start with a prefix like `k8s:enc:kms:v2:my-kms:`.

Once you're sure everything is encrypted, remove the `identity` entry. From then on the API server refuses to read plain-text Secrets, which catches anything that slipped through.

### What encryption at rest doesn't do

It protects etcd and its backups. It does nothing against someone who can read Secrets **through the API server**, because the API server decrypts for any authorized caller. That's the job of the next section.

While you're at it, the docs suggest a few etcd hygiene items: only cluster administrators should reach etcd at all, peer traffic between etcd members should use TLS, and disks that held etcd data should be wiped when retired[^2].

## Locking down access with RBAC

### Reading Secrets

The obvious permissions are `get`, `list` and `watch` on `secrets`. Two things are less obvious[^2][^5]:

- **`list` is as good as `get`.** A list response contains the full objects, data included, not just their names. Granting `list` on Secrets in a namespace means granting read access to all of them. The same goes for `watch`.
- **`resourceNames` don't restrict `list`.** You can scope `get` to a single named Secret, but a `list` request can't be filtered that way.

So, for people and workloads that need a specific Secret through the API:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-db-credentials
  namespace: shop
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-credentials"]
    verbs: ["get"]
```

Most applications don't need API access to Secrets at all: they get them through volumes or env vars, which the kubelet fetches on their behalf. Only controllers and operators should normally hold `list` or `watch` on Secrets.

### Creating Pods is reading Secrets

This one surprises people. A user who can't `get` a single Secret but **can create Pods** in a namespace can simply start a Pod that mounts any Secret there and prints it[^1][^2]:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: totally-innocent
  namespace: shop
spec:
  restartPolicy: Never
  containers:
    - name: cat
      image: busybox
      command: ["cat", "/s/password"]
      volumeMounts:
        - { name: s, mountPath: /s }
  volumes:
    - name: s
      secret:
        secretName: db-credentials
```

The same applies to anyone who can create Deployments, Jobs, CronJobs, StatefulSets or DaemonSets. There's no way around this inside a namespace, which leads to the most effective control there is: **namespaces are the boundary**. Put sensitive Secrets in namespaces where only the people and pipelines that deploy the owning workload can create Pods. Don't put a payment provider's API key in the namespace every developer deploys their experiments to.

### Privileged containers and nodes

A container running with `privileged: true` can read every Secret used by any Pod on its node[^1]. Pod Security Admission at the `baseline` or `restricted` level blocks privileged Pods in application namespaces, which closes that hole.

### Audit

Turn on API audit logging for Secrets at the `Metadata` level (never `Request` or `RequestResponse`, or the audit log becomes a copy of your Secrets). The Kubernetes docs suggest alerting on patterns such as one identity reading many Secrets in a short time[^2].

## Keeping secrets out of Git

GitOps wants every resource in Git. A Secret manifest in Git is a secret in Git, readable by everyone with access to the repository, and it stays in the history after you delete it[^2]. There are two broad ways out: store an **encrypted** version in Git, or store only a **reference** in Git and keep the value somewhere else.

One more trap first: if you `kubectl apply` a Secret with client-side apply, kubectl stores the whole manifest, values included, in the `kubectl.kubernetes.io/last-applied-configuration` annotation. Anyone who can see the object's metadata can then see the data. Use server-side apply (`kubectl apply --server-side`) or create Secrets another way.

### Sealed Secrets: encrypt for the cluster

Sealed Secrets, originally from Bitnami, has two parts: a controller in the cluster that holds a private key, and a CLI, `kubeseal`, that encrypts Secrets with the matching public key[^6]. Only the controller can decrypt.

```bash
# Write a normal Secret, but don't apply it
kubectl create secret generic db-credentials -n shop \
  --from-literal=password='s3cr3t-pass' \
  --dry-run=client -o yaml > secret.yaml

# Seal it, then throw the original away
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
rm secret.yaml
```

The result is safe to commit:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: shop
spec:
  encryptedData:
    password: AgBy8hCi...truncated...
```

When it's applied, the controller decrypts it and creates an ordinary Secret. By default the ciphertext is bound to the Secret's name and namespace, so someone can't copy it into a namespace they control and have it decrypted there.

Trade-offs: the secret value is still in Git, just encrypted, so anyone who ever gets the controller's private key can decrypt the entire history. Back up that key, because losing it means re-sealing everything. And the result is still a normal Kubernetes Secret, so everything earlier in this post still applies.

### SOPS: encrypt the file

SOPS encrypts the *values* of a YAML or JSON file while leaving the keys readable, so diffs and reviews still make sense[^9]. It works with age keys, PGP, or cloud KMS keys:

```bash
sops --encrypt \
  --age age1qyz...recipient... \
  --encrypted-regex '^(data|stringData)$' \
  secret.yaml > secret.enc.yaml
```

Flux can decrypt SOPS files natively during reconciliation, and Argo CD and Helm have plugins for it. SOPS is a good fit if you want one encryption tool for Kubernetes and non-Kubernetes config alike, and if your key management lives in a cloud KMS anyway.

### External Secrets Operator: reference, don't store

External Secrets Operator (ESO) takes the other approach: the secret lives in a proper secret manager (Vault, AWS Secrets Manager, Google Secret Manager, Azure Key Vault, and many more), and Git contains only a pointer to it[^7]. ESO reads the value and writes a Kubernetes Secret, and keeps it in sync on a schedule.

First, a SecretStore that says how to reach the backend. Here, Vault, authenticating with the namespace's service account token:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: vault
  namespace: shop
spec:
  provider:
    vault:
      server: https://vault.example.com
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: shop
          serviceAccountRef:
            name: shop-eso
```

Then an ExternalSecret that says what to fetch:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: shop
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: db-credentials      # the Kubernetes Secret ESO will create
  data:
    - secretKey: password
      remoteRef:
        key: shop/db
        property: password
```

Both manifests are safe to commit. Rotation happens in the secret manager; ESO picks up the new value at the next refresh, and file-mounted Pods see it shortly after.

Notice the authentication: no static Vault token stored anywhere. ESO presents a Kubernetes service account token, which Vault validates. This is the pattern to aim for everywhere.

### Secrets Store CSI Driver: skip the Secret object

The Secrets Store CSI Driver, a Kubernetes SIG project, goes one step further. It mounts secrets from an external store directly into the Pod as a volume, so no Secret object needs to exist in etcd at all[^8]. Each backend has a provider plugin (AWS, Azure, GCP, Vault and others).

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: shop-db
  namespace: shop
spec:
  provider: vault
  parameters:
    vaultAddress: https://vault.example.com
    roleName: shop
    objects: |
      - objectName: password
        secretPath: secret/data/shop/db
        secretKey: password
---
apiVersion: v1
kind: Pod
metadata:
  name: api
  namespace: shop
spec:
  serviceAccountName: shop-api
  containers:
    - name: api
      image: registry.example.com/shop-api:3.1.0
      volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: shop-db
```

The value is fetched when the Pod starts, using the Pod's own service account identity, so access can be granted per workload rather than per namespace. The downsides: if the external store is unreachable, new Pods can't start, and applications that insist on environment variables need the driver's optional sync-to-Secret feature, which puts the value back into etcd.

### Which one?

| | Value stored in Git | Value stored in etcd | Rotation | Needs external store |
| --- | --- | --- | --- | --- |
| Plain Secret manifest | Yes, readable | Yes | Manual | No |
| Sealed Secrets | Yes, encrypted | Yes | Re-seal and commit | No |
| SOPS | Yes, encrypted | Yes | Re-encrypt and commit | Optional (KMS keys) |
| External Secrets Operator | No | Yes | Automatic, in the store | Yes |
| Secrets Store CSI Driver | No | No (unless synced) | Automatic, in the store | Yes |

For a homelab or a small team without a secret manager, Sealed Secrets or SOPS is a big step up from nothing for very little effort. Once you have a real secret manager, ESO is the most compatible choice, because applications keep consuming normal Secrets. The CSI driver is for when "this value must never be in etcd" is a hard requirement.

## The best secret is no secret

Many credentials in Kubernetes Secrets don't need to exist at all. The Kubernetes docs list alternatives to Secrets, and they mostly come down to **identity instead of passwords**[^1]:

- **Workload identity for cloud access.** Instead of an AWS access key in a Secret, the Pod presents its projected service account token to the cloud provider and gets short-lived credentials back. EKS, GKE and AKS all support this, and it works for self-managed clusters too.
- **Service account tokens for in-cluster services.** If both sides run in the cluster, the client can authenticate with a projected token with a custom audience, and the server validates it with TokenReview. (I covered this in detail in my post on service account tokens.)
- **Credential providers for image pulls.** Instead of `imagePullSecrets` with a long-lived registry password, a kubelet credential provider plugin fetches pull credentials on demand[^10]. Since v1.34 it can even use the Pod's service account token, so pull access follows the workload.
- **Certificates from cert-manager or a custom signer** for TLS and mTLS, issued and renewed automatically.
- **Dynamic database credentials** from Vault, created per Pod with a short TTL, so a leaked password expires on its own.

Each of these turns a long-lived, copyable secret into something short-lived and tied to an identity. That's the same direction Kubernetes itself took with service account tokens.

## Good practices

**Cluster administrators**

- **Turn on encryption at rest** with KMS v2, rewrite existing Secrets, verify with `etcdctl`, then remove `identity`.
- **Restrict etcd** to cluster administrators, use TLS between etcd members, and treat etcd backups as secret material.
- **Grant `get`, `list` and `watch` on Secrets sparingly.** Remember `list` and `watch` reveal everything in the namespace.
- **Treat Pod creation as Secret access.** Use namespaces as the boundary for sensitive Secrets.
- **Block privileged Pods** in application namespaces with Pod Security Admission.
- **Audit Secret access** at `Metadata` level and alert on unusual reads.

**Developers**

- **Mount Secrets as files**, not environment variables, and give them only to the containers that need them.
- **Never log secret values**, and be careful with crash reporters and debug endpoints that dump the environment.
- **Don't commit Secret manifests.** Use Sealed Secrets, SOPS, or references with External Secrets Operator.
- **Don't use `kubectl apply` client-side** on Secrets; the values end up in an annotation.
- **Avoid `subPath` mounts** for Secrets you plan to rotate.
- **Use immutable, versioned Secrets** for values that change through deployments.
- **Prefer identity over secrets** wherever a workload-identity option exists.

## Bottom line

- **A Secret is a delivery mechanism, not a vault.** By default it's base64 in YAML and plain text in etcd.
- **Encrypt at rest with KMS v2** so etcd and its backups are useless without your KMS.
- **RBAC and namespaces decide who can read Secrets**, and the ability to create Pods counts as reading.
- **Inside the Pod, prefer files over environment variables**, scoped to the containers that need them.
- **Keep values out of Git**, either encrypted (Sealed Secrets, SOPS) or referenced from a secret manager (External Secrets Operator, Secrets Store CSI Driver).
- **The best secret is the one you don't have**: use workload identity and short-lived credentials wherever you can.

## References

[^1]: [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/), Kubernetes documentation: Secret types, size limits, volumes and environment variables, update propagation, `subPath`, immutable Secrets, tmpfs storage, and alternatives to Secrets.
[^2]: [Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/), Kubernetes documentation: least-privilege access, etcd management, external Secret stores, and advice for developers.
[^3]: [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/), Kubernetes documentation: EncryptionConfiguration, the available providers and their strength, verifying encryption, rewriting existing data, key rotation and automatic reload.
[^4]: [Using a KMS provider for data encryption](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/), Kubernetes documentation: KMS v2 configuration and envelope encryption.
[^5]: [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/), Kubernetes documentation: risks of `list` and `watch` on Secrets and of workload creation.
[^6]: [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets), GitHub: controller and `kubeseal` CLI for one-way encrypted Secrets.
[^7]: [External Secrets Operator](https://external-secrets.io/latest/), project documentation: SecretStore, ExternalSecret and supported providers.
[^8]: [Kubernetes Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/), Kubernetes SIGs documentation: SecretProviderClass, providers and sync-to-Secret.
[^9]: [SOPS: Secrets OPerationS](https://getsops.io/), project documentation: encrypting YAML and JSON values with age, PGP and cloud KMS.
[^10]: [Configure a kubelet image credential provider](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-credential-provider/), Kubernetes documentation.
