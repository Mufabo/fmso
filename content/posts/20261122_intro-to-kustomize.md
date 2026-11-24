---
title: "Intro to Kustomize: Template-Free Kubernetes Configuration"
date: 2026-11-22
draft: true
description: "What Kustomize is, how bases and overlays work, the kustomization.yaml fields you'll use every day (resources, namespace, labels, images, replicas, patches, generators, components), the Kustomize v5 deprecations and how to migrate, how it compares to Helm, and how GitOps tools use it. With a hands-on lab that builds dev and prod overlays for one app."
summary: "You have one app and three environments, and the YAML differs by a replica count, an image tag and a hostname. Kustomize handles that without a single template: plain manifests as a base, small overlays on top, and kubectl already has it built in."
tags: ["kubernetes", "kustomize", "kubectl", "gitops", "configuration", "helm"]
categories: ["Programming"]
---

You have one application and three environments. The manifests for dev, staging and prod are 95% identical: prod wants five replicas instead of one, a pinned image tag, more memory and a different hostname. The tempting solution is to copy the folder three times. A month later the copies have drifted, someone fixed a probe in prod but not in staging, and nobody knows which differences are intentional.

**Kustomize** solves this without templates. You keep one set of ordinary, valid Kubernetes YAML as a **base**, and for each environment you write a small **overlay** that says only what is different. Kustomize merges them and prints the final manifests. It has been built into `kubectl` since v1.14, so you very likely already have it.

This post covers how Kustomize thinks, the fields you'll actually use, the Kustomize v5 changes that produce all those deprecation warnings, how it compares to Helm, and a lab you can run on any machine with `kubectl`.

## The idea: patch, don't template

Most configuration tools generate YAML from templates with placeholders (`replicas: {{ .Values.replicas }}`). The template isn't valid YAML until it's rendered, and the more options you add, the harder it is to read.

Kustomize takes the other route. The project describes itself as a template-free way to customize application configuration, working on raw YAML files and leaving the originals untouched.[^1] Every input is a normal Kubernetes manifest you could `kubectl apply` on its own. Customizations are expressed as **transformations**: "set the namespace on everything", "add these labels", "change this image tag", "merge this patch into that Deployment".

The upside: the base stays readable and testable, and you never need to learn a templating language. The downside: anything that isn't a structural change to YAML (loops, conditionals, string building) is deliberately out of scope.

## The kustomization file

Everything starts with a file named `kustomization.yaml` in a directory. That directory is a **kustomization**. The smallest useful one just lists resources:

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

Render it with either tool:

```bash
kubectl kustomize ./          # built into kubectl
kustomize build ./            # standalone binary, usually newer
```

Both print the combined YAML to stdout. Nothing is sent to the cluster. To apply, delete or diff directly, use `-k` instead of `-f`:[^2]

```bash
kubectl apply  -k ./
kubectl diff   -k ./
kubectl delete -k ./
```

The `kubectl` integration lags behind the standalone binary. `kubectl version --client` prints both versions; the embedded one has been on the v5 line since kubectl 1.27.[^3] If you need a feature from a recent release, install the standalone `kustomize` binary.

## Bases and overlays

The core pattern is a directory layout like this:

```text
app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── resources-patch.yaml
```

The base is a complete, environment-neutral version of the app. Each overlay is a kustomization whose `resources` points at the base, then adds its own changes:

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: shop-prod

images:
  - name: nginx
    newTag: "1.27.3"

replicas:
  - name: web
    count: 5

patches:
  - path: resources-patch.yaml
```

```text
           base/                    (plain manifests)
             │
     ┌───────┴────────┐
     ▼                ▼
overlays/dev     overlays/prod      (only the differences)
     │                │
     ▼                ▼
kubectl kustomize   kubectl kustomize
     │                │
dev YAML           prod YAML        (complete, ready to apply)
```

Overlays can stack: a `prod-eu` overlay can use `../prod` as its resource and change only the region-specific bits. A kustomization can also list several bases, for example to deploy an app together with its database.

Older tutorials use a separate `bases:` field. It was merged into `resources:` long ago and is deprecated; anything you'd list under `bases` now goes under `resources`.[^4]

## The fields you'll use every day

The full reference lists many more,[^4] but these cover most real-world kustomizations.

### `namespace`, `namePrefix`, `nameSuffix`

```yaml
namespace: shop-prod
namePrefix: prod-
nameSuffix: -v2
```

`namespace` sets `metadata.namespace` on every namespaced resource. The prefix and suffix rename every resource. Kustomize knows where resources reference each other (a Deployment's `configMapRef`, a Service's name in an Ingress backend, a RoleBinding's subjects), and it updates those references too. That's the key difference from running `sed` over the files.

### `labels` and `commonAnnotations`

```yaml
labels:
  - pairs:
      app.kubernetes.io/part-of: shop
      team: payments
    includeSelectors: false    # default: don't touch selectors
    includeTemplates: true     # also label the pod templates
commonAnnotations:
  owner: payments@example.com
```

`labels` adds labels to every resource. With `includeSelectors: true` it also adds them to label selectors (Deployment `spec.selector`, Service `spec.selector`).[^5]

This is where the older `commonLabels` field caused real pain: it **always** changed selectors. A Deployment's `spec.selector` is immutable, so adding a new common label to a running app made `kubectl apply` fail, and you had to delete and recreate the Deployment. `commonLabels` is deprecated since v5.0.0 in favour of `labels`.[^6] Use `labels`, and only set `includeSelectors: true` for labels that will never change.

### `images`

```yaml
images:
  - name: nginx                          # matches image: nginx, nginx:1.25, ...
    newName: registry.example.com/nginx  # optional: different registry/repo
    newTag: "1.27.3"                     # or: digest: sha256:...
```

Changes the image of every container that uses `nginx`, in Deployments, StatefulSets, Jobs, CronJobs and so on. CI pipelines usually update this with a command instead of editing YAML:

```bash
cd overlays/prod
kustomize edit set image nginx=registry.example.com/nginx:1.27.3
```

### `replicas`

```yaml
replicas:
  - name: web
    count: 5
```

A shortcut for the most common patch. If a HorizontalPodAutoscaler manages the Deployment, leave replicas out of your manifests altogether, or every apply will fight the autoscaler.

### `patches`

For anything the dedicated fields don't cover, use `patches`. It accepts two kinds of patch, from a file or inline, and can target resources by kind, name, namespace, labels or annotations.[^7]

A **strategic merge patch** looks like a partial manifest. It's matched to its target by `apiVersion`, `kind` and `metadata.name`, then merged:

```yaml
# resources-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
        - name: web            # lists of containers merge by name
          resources:
            requests: { cpu: 500m, memory: 512Mi }
            limits:   { memory: 512Mi }
```

"Strategic" means it understands Kubernetes semantics: the `containers` list is merged by `name`, so this patch changes the `web` container and leaves any sidecars alone. A plain YAML merge would replace the whole list.

A **JSON 6902 patch** is a list of precise operations, handy for things a merge can't express, like removing a field or changing an element by index:

```yaml
patches:
  - target:
      kind: Ingress
      name: web
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: shop.example.com
      - op: remove
        path: /metadata/annotations/nginx.ingress.kubernetes.io~1rewrite-target
```

(`~1` is how JSON Pointer escapes a `/` inside a key.)

With a `target` selector, one patch can apply to many resources, for example adding a `securityContext` to every Deployment with `labelSelector: app.kubernetes.io/part-of=shop`.

### `configMapGenerator` and `secretGenerator`

Instead of writing ConfigMaps by hand, generate them from files, env files or literals:

```yaml
configMapGenerator:
  - name: web-config
    files:
      - nginx.conf
    literals:
      - LOG_LEVEL=info

secretGenerator:
  - name: web-credentials
    envs:
      - credentials.env
```

The generated object gets a **content hash suffix**, like `web-config-7b2k9h4m5f`, and Kustomize rewrites every reference to `web-config` to use the hashed name. When the content changes, the name changes, so the Deployment's pod template changes and Kubernetes rolls out new pods. That fixes the classic "I updated the ConfigMap but the pods still have the old config" problem.[^8] Old ConfigMaps are left behind; `kubectl apply --prune` or your GitOps tool cleans them up. If something outside Kustomize needs a stable name, turn the suffix off with `options: { disableNameSuffixHash: true }`.

A warning about `secretGenerator`: it only base64-encodes values, like any Secret. Never commit the source files in plain text. Pair Kustomize with SOPS (via the ksops plugin or Flux's built-in decryption) or the External Secrets Operator; see my post on [Kubernetes Secrets]({{< relref "20261112_kubernetes-secrets.md" >}}).

### `replacements`

Sometimes you need to copy a value from one resource into another, say a Service's name into an environment variable of a different Deployment. `replacements` does that:

```yaml
replacements:
  - source:
      kind: Service
      name: api
      fieldPath: metadata.name
    targets:
      - select:
          kind: Deployment
          name: web
        fieldPaths:
          - spec.template.spec.containers.[name=web].env.[name=API_HOST].value
```

Because the source is read *after* other transformations, the value includes any prefix or suffix. This replaces the old `vars` field, deprecated in v5.0.0.[^9]

## Components: optional features

Overlays form a hierarchy, which breaks down when features don't follow environments. Suppose some environments need monitoring, some need an external database, and the combinations vary. You'd need an overlay for every combination.

**Components** solve this. A component is a reusable bundle of resources and patches that any kustomization can opt into:[^10]

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - servicemonitor.yaml
patches:
  - path: metrics-port-patch.yaml
```

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base
components:
  - ../../components/monitoring
  - ../../components/external-db
```

Note the different `apiVersion` and `kind`. Unlike an ordinary base, a component's patches apply to the resources of the kustomization that includes it, which is what makes it a mix-in.

## Kustomize v5: what the warnings mean

If you run old manifests through a current Kustomize, you'll see warnings like this:

```text
# Warning: 'patchesStrategicMerge' is deprecated. Please use 'patches' instead.
Run 'kustomize edit fix' to update your Kustomization automatically.
```

Kustomize v5.0.0 deprecated several fields. They still work in the `v1beta1` API, but you should migrate:

| Deprecated | Replacement |
|---|---|
| `bases` | `resources` |
| `patchesStrategicMerge` | `patches` (with `path` or inline `patch`)[^11] |
| `patchesJson6902` | `patches` (with `target` and an inline `patch`) |
| `commonLabels` | `labels` (decide on `includeSelectors` explicitly)[^6] |
| `vars` | `replacements`[^9] |

`kustomize edit fix` rewrites the kustomization in place for most of these. One catch: `commonLabels` becomes `labels` with `includeSelectors: true`, which keeps the old behaviour, so no immutable-selector surprise, but also no improvement. Converting `vars` needs the extra `--vars` flag and is marked experimental, so check the output.[^9] Either way, render before and after and diff:

```bash
kustomize build overlays/prod > before.yaml
(cd overlays/prod && kustomize edit fix)
kustomize build overlays/prod > after.yaml
diff before.yaml after.yaml
```

## Things that trip people up

- **Files must live inside the kustomization.** By default Kustomize refuses to load a file outside the kustomization's directory tree (`security; file '...' is not in or below '...'`). Referencing another kustomization like `../../base` is fine; referencing a single file like `../../shared/patch.yaml` is not. Restructure, or use `--load-restrictor LoadRestrictionsNone` if you really must.
- **Output order isn't your file order.** Kustomize sorts output so that, for example, Namespaces and CRDs come before the things that depend on them. Usually that's what you want.
- **Remote resources.** `resources` can point at a Git URL, like `https://github.com/org/repo//deploy/base?ref=v1.4.0`. Always pin `ref` to a tag or commit: an unpinned URL means a push to someone else's repo changes your next deploy.
- **Patches that match nothing.** A strategic merge patch whose name doesn't match any resource is an error, which is good. A typo in a `target` selector, though, can silently match nothing. Render and read the output.
- **The two kinds of "Kustomization".** Flux has a custom resource also called `Kustomization` (`kustomize.toolkit.fluxcd.io`). It tells the Flux controller where to find a kustomize directory; it isn't a `kustomization.yaml`.

## Kustomize vs Helm

They solve overlapping problems from opposite ends. (I've covered Helm in its own post: [Helm for Kubernetes]({{< relref "20261111_helm-for-kubernetes.md" >}}).)

| | Kustomize | Helm |
|---|---|---|
| Approach | Patches plain YAML | Renders Go templates with values |
| Input files | Valid Kubernetes manifests | Templates, not valid YAML until rendered |
| Logic | None: no loops or conditionals | Full template language |
| Packaging and versioning | None; you use Git | Charts, versions, repositories, OCI registries |
| Release tracking and rollback | No; `kubectl` or GitOps handle state | Yes, `helm rollback` |
| Installed | Built into `kubectl` | Separate binary |
| Best for | Your own apps across environments | Distributing apps for others to install |

The rule of thumb many teams land on: **Helm to consume third-party software, Kustomize for your own manifests.** And they combine well. Kustomize's `helmCharts` field renders a chart and lets you patch the result, for the "the chart doesn't expose that one setting" problem.[^12] It needs the `helm` binary and the `--enable-helm` flag:

```yaml
helmCharts:
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.11.3
    releaseName: ingress
    namespace: ingress-nginx
    valuesInline:
      controller:
        replicaCount: 2
```

```bash
kustomize build --enable-helm .
```

## Kustomize and GitOps

Kustomize fits GitOps naturally, because an overlay directory in Git is exactly "the desired state of this environment".

- **Argo CD** detects a `kustomization.yaml` in an Application's path and runs Kustomize itself. An Application can also override images, name prefixes and similar fields without a commit.[^13]
- **Flux** has a dedicated kustomize-controller that builds and applies a path from a Git or OCI source, with pruning, health checks and SOPS decryption built in.[^14]

A common repository layout:

```text
apps/
  shop/
    base/
    overlays/{dev,staging,prod}/
clusters/
  prod-eu/      # points Argo CD / Flux at apps/shop/overlays/prod, etc.
```

Promotion to the next environment then means changing a `newTag` in the next overlay in a pull request, which is reviewable, auditable and easy to revert.

## Hands-on lab: one app, two environments

All you need is `kubectl` (1.27 or newer). Rendering doesn't require a cluster; the last step applies to one if you have it (kind, minikube, k3d all work).

```bash
mkdir -p /tmp/kz/{base,overlays/dev,overlays/prod}
cd /tmp/kz
```

**1. The base.** A Deployment, a Service and a generated ConfigMap.

```bash
cat > base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: web-config
EOF

cat > base/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 80
EOF

cat > base/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
configMapGenerator:
  - name: web-config
    literals:
      - GREETING=hello
labels:
  - pairs:
      app.kubernetes.io/part-of: kz-demo
EOF

kubectl kustomize base
```

In the output, look at the ConfigMap's name (`web-config-<hash>`) and at the Deployment's `configMapRef`: Kustomize updated the reference to the hashed name. The `part-of` label is on both resources, but not in the selectors.

**2. The dev overlay.** Its own namespace and a name prefix.

```bash
cat > overlays/dev/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: kz-dev
namePrefix: dev-
configMapGenerator:
  - name: web-config
    behavior: merge
    literals:
      - GREETING=hello-from-dev
EOF

kubectl kustomize overlays/dev
```

Everything is now called `dev-web` and lives in `kz-dev`. The `configMapRef` points at `dev-web-config-<new hash>`: the merged ConfigMap has different content, so a different hash. `behavior: merge` adds to the base's generator instead of creating a second ConfigMap.

**3. The prod overlay.** More replicas, a pinned image, resource limits.

```bash
cat > overlays/prod/resources-patch.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  template:
    spec:
      containers:
        - name: web
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { memory: 128Mi }
EOF

cat > overlays/prod/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: kz-prod
images:
  - name: nginx
    newTag: "1.27-alpine"
replicas:
  - name: web
    count: 3
patches:
  - path: resources-patch.yaml
  - target:
      kind: Service
      name: web
    patch: |-
      - op: add
        path: /metadata/annotations
        value:
          environment: production
EOF

kubectl kustomize overlays/prod
```

Check the result: `replicas: 3`, `image: nginx:1.27-alpine`, the `resources` block in the `web` container, and the annotation on the Service.

**4. Compare the environments.**

```bash
diff <(kubectl kustomize overlays/dev) <(kubectl kustomize overlays/prod)
```

This is the real payoff: the difference between environments is visible and reviewable, not buried in three copied folders.

**5. Optional: apply to a cluster.**

```bash
kubectl create namespace kz-prod
kubectl apply -k overlays/prod
kubectl -n kz-prod get deploy,svc,cm

# Change the config and watch a rollout happen by itself:
sed -i 's/GREETING=hello/GREETING=hi/' base/kustomization.yaml
kubectl diff -k overlays/prod          # new ConfigMap name, changed Deployment
kubectl apply -k overlays/prod
kubectl -n kz-prod rollout status deploy/web

kubectl delete -k overlays/prod
kubectl delete namespace kz-prod
```

(On macOS, use `sed -i ''`.) Then clean up with `rm -rf /tmp/kz`.

## Cheat sheet

```text
Render:            kubectl kustomize <dir>    |   kustomize build <dir>
Apply/diff/delete: kubectl apply -k <dir>  ·  kubectl diff -k <dir>  ·  kubectl delete -k <dir>
Versions:          kubectl version --client  (embedded kustomize: v5.x since kubectl 1.27)

kustomization.yaml (apiVersion kustomize.config.k8s.io/v1beta1, kind Kustomization)
  resources:          files, other kustomizations (../../base), pinned Git URLs
  namespace / namePrefix / nameSuffix   references updated automatically
  labels:             - pairs: {...}  includeSelectors: false (default)
  commonAnnotations:  {...}
  images:             - name, newName, newTag | digest
  replicas:           - name, count
  patches:            - path: file.yaml   |   - target: {...}  patch: | (SMP or JSON 6902)
  configMapGenerator / secretGenerator   hash suffix → automatic rollouts
  replacements:       copy a field from one resource into others
  components:         opt-in features (apiVersion .../v1alpha1, kind Component)
  helmCharts:         render a chart, then patch it (needs --enable-helm)

Deprecated in v5 → use:  bases→resources · patchesStrategicMerge/patchesJson6902→patches
                         commonLabels→labels · vars→replacements    (kustomize edit fix)
CLI edits:         kustomize edit set image nginx=nginx:1.27 · kustomize edit add resource x.yaml
Gotchas:           files must be inside the kustomization · pin remote refs · generators aren't encryption
Rule of thumb:     Helm for third-party software, Kustomize for your own manifests
```

## References

[^1]: [Kustomize](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/), Kubectl and Kustomize documentation.
[^2]: [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/), Kubernetes documentation.
[^3]: [kubernetes-sigs/kustomize](https://github.com/kubernetes-sigs/kustomize), GitHub (kubectl integration and version table).
[^4]: [Kustomization file reference](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/), Kubectl and Kustomize documentation.
[^5]: [labels](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/labels/), Kustomize reference.
[^6]: [commonLabels](https://kustomize.sigs.k8s.io/references/kustomize/kustomization/commonlabels), Kustomize reference.
[^7]: [patches](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/), Kustomize reference.
[^8]: [configMapGenerator](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/configmapgenerator/), Kustomize reference.
[^9]: [vars](https://kubectl.docs.k8s.io/references/kustomize/kustomization/vars/), Kustomize reference (deprecation and migration to replacements).
[^10]: [Kustomize Components](https://kubectl.docs.kubernetes.io/guides/config_management/components/), Kubectl and Kustomize documentation.
[^11]: [patchesStrategicMerge](https://kubectl.docs.k8s.io/references/kustomize/kustomization/patchesstrategicmerge/), Kustomize reference.
[^12]: [helmCharts](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/helmcharts/), Kustomize reference.
[^13]: [Kustomize](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/), Argo CD documentation.
[^14]: [Kustomize controller](https://fluxcd.io/flux/components/kustomize/), Flux documentation.
