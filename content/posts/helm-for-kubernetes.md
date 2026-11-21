---
title: "Helm for Kubernetes: Charts, Releases and Templates Explained"
date: 2026-11-11
draft: true
description: "A practical introduction to Helm 4, the Kubernetes package manager: what charts, releases and values are, how to install and upgrade applications, how to write your own chart, and what changed from Helm 3."
summary: "Helm turns a pile of Kubernetes YAML into a versioned, configurable package you can install, upgrade and roll back with one command. Here is how it works, how to write your own chart, and what Helm 4 changes."
tags: ["kubernetes", "helm", "packaging", "deployment", "platform engineering"]
categories: ["Programming"]
---

Deploying a real application to Kubernetes rarely means one manifest. A typical web service needs a Deployment, a Service, a ConfigMap, a Secret, an Ingress or HTTPRoute, a ServiceAccount, maybe a HorizontalPodAutoscaler and a PodDisruptionBudget. Then you need slightly different versions of all of them for dev, staging and production. Copy-pasting YAML and editing it by hand works for a week.

Helm is the tool most of the ecosystem settled on to solve this. It calls itself the package manager for Kubernetes: it bundles manifests into a **chart**, lets you configure the chart with **values**, and tracks each installation as a **release** that you can upgrade and roll back.

This post covers Helm 4 (4.3.1 at the time of writing). Helm 4.0.0 was released at KubeCon in November 2025, the first new major version in six years.[^1] If you still run Helm 3, see the section on upgrading near the end: its support is winding down.

## The three core concepts

**Chart.** A directory (or a `.tgz` archive of one) containing templated Kubernetes manifests plus metadata. The layout is fixed:[^2]

```text
mychart/
  Chart.yaml        # name, version, description, dependencies
  values.yaml       # default configuration
  charts/           # dependency charts (subcharts)
  templates/        # templated manifests
  .helmignore       # files to leave out when packaging
```

**Values.** The configuration you feed into a chart. A chart ships defaults in `values.yaml`; you override them with your own files (`-f`) or individual flags (`--set`).

**Release.** One installation of a chart into a cluster, with a name. You can install the same chart many times under different release names. Each `install`, `upgrade` or `rollback` creates a new **revision** of the release.

The flow is simple:

```text
  chart templates  +  values  ──►  helm renders  ──►  plain YAML  ──►  Kubernetes API
                                                                         │
                         release record (revision N) stored in a Secret ◄┘
```

Helm has no server-side component. Helm 2 had one, called Tiller, which ran in the cluster with broad permissions; Helm 3 removed it. Today the `helm` CLI talks straight to the API server using your kubeconfig, so it can only do what your own RBAC allows. It records each release revision as a Secret in the release's namespace (named `sh.helm.release.v1.<release>.v<revision>`), which is how it knows what it installed last time.

## Installing Helm

Helm is a single binary. On macOS, `brew install helm`; on Linux, download a release from GitHub or use the install script described in the docs.[^3] Then check it:

```bash
helm version
```

Helm uses the same kubeconfig and current context as `kubectl`, so if `kubectl get nodes` works, Helm will too.

## Using someone else's chart

### Finding charts

Most public charts are indexed on Artifact Hub,[^4] which lets you search and read a chart's documented values before installing it.

Charts are distributed in two ways:

- **OCI registries**, the same registries that store container images (GHCR, Docker Hub, Quay, ECR, Harbor and so on). This is now the preferred way to publish charts.
- **Classic chart repositories**, plain HTTP servers with an `index.yaml`. You add these with `helm repo add`.

The examples below use `podinfo`, a small demo web app published as an OCI chart.

### Inspect before you install

Always look at a chart's values before installing it. They are the chart's API:

```bash
helm show values oci://ghcr.io/stefanprodan/charts/podinfo
```

To see exactly what will be sent to the cluster, render it locally:

```bash
helm template demo oci://ghcr.io/stefanprodan/charts/podinfo --set replicaCount=2
```

`helm template` doesn't contact the cluster at all, which makes it useful in CI and for reviewing changes.

### Install

```bash
helm install demo oci://ghcr.io/stefanprodan/charts/podinfo \
  --namespace demo --create-namespace \
  --set replicaCount=2 \
  --wait
```

`demo` is the release name. `--wait` makes Helm block until the resources are actually ready, not just accepted by the API server. In Helm 4 that waiting is built on kstatus, the same status library many controllers use, so it understands readiness for more resource types than Helm 3 did.[^5]

For anything beyond a couple of settings, put your overrides in a file instead of using `--set`:

```yaml
# demo-values.yaml
replicaCount: 3
resources:
  requests:
    cpu: 50m
    memory: 64Mi
ingress:
  enabled: false
```

```bash
helm install demo oci://ghcr.io/stefanprodan/charts/podinfo -n demo -f demo-values.yaml
```

Values are merged in a fixed order, later sources winning: the chart's `values.yaml`, then each `-f` file in order, then `--set` flags.[^6] A common pattern is a shared `values.yaml` plus one small file per environment:

```bash
helm upgrade --install demo ./chart -f values/common.yaml -f values/prod.yaml
```

### Pin versions

Without `--version`, Helm installs the latest chart version, which means two runs of the same command can install different things. Pin it:

```bash
helm install demo oci://ghcr.io/stefanprodan/charts/podinfo --version 6.9.0
```

For OCI charts, Helm 4 can also install by **digest**. If the chart's content doesn't match the digest, Helm refuses to install it, which protects you against a tag being moved or tampered with:[^5]

```bash
helm install demo oci://ghcr.io/stefanprodan/charts/podinfo@sha256:<digest>
```

## Day two: upgrade, history, rollback

### Upgrade

Change a value and apply it:

```bash
helm upgrade demo oci://ghcr.io/stefanprodan/charts/podinfo -n demo \
  -f demo-values.yaml --set replicaCount=4
```

Two flags save a lot of grief in automation:

- `helm upgrade --install` installs the release if it doesn't exist and upgrades it if it does. This makes the same command safe to run on every CI pipeline run.
- `--rollback-on-failure` rolls the release back automatically if the upgrade fails. In Helm 3 this flag was called `--atomic`; Helm 4 renamed it, and renamed `--force` to `--force-replace`. The old names still work but print deprecation warnings.[^5]

One trap: by default `helm upgrade` uses the values you pass **now**, not the ones from the last release. If you installed with `-f prod.yaml` and upgrade without it, those settings silently revert to the chart defaults. Either always pass the same values files, or use `--reuse-values` (which keeps old values, but also ignores new defaults added in the new chart version). Passing the full set of files every time is the more predictable choice.

### Inspect a release

```bash
helm list -n demo                     # releases in a namespace (-A for all)
helm status demo -n demo              # current state and the chart's NOTES
helm get values demo -n demo          # the values you supplied
helm get values demo -n demo --all    # merged values, including defaults
helm get manifest demo -n demo        # the exact YAML Helm applied
helm history demo -n demo             # all revisions
```

### Roll back

```bash
helm history demo -n demo
REVISION  STATUS      CHART           DESCRIPTION
1         superseded  podinfo-6.9.0   Install complete
2         superseded  podinfo-6.9.0   Upgrade complete
3         deployed    podinfo-6.9.0   Upgrade complete

helm rollback demo 2 -n demo
```

A rollback doesn't rewind history; it creates a new revision (4) with the contents of revision 2. By default Helm keeps up to ten revisions per release; `--history-max` changes that.

### Uninstall

```bash
helm uninstall demo -n demo
```

This deletes the resources and the release records. Add `--keep-history` if you want to be able to see (and roll back to) what was there.

## Writing your own chart

### Scaffold

```bash
helm create myapp
```

This generates a complete, working chart for a stateless web app: a Deployment, Service, ServiceAccount, optional Ingress and autoscaler, a `NOTES.txt` shown after install, a test, and a `_helpers.tpl` file for reusable snippets. It's a good starting point, and also a good reference for idiomatic template style even if you end up deleting half of it.

### Chart.yaml

```yaml
apiVersion: v2
name: myapp
description: My web application
type: application
version: 0.3.0        # version of the chart itself
appVersion: "1.8.2"   # version of the app it deploys
```

Two versions, two jobs. `version` is the chart's own SemVer and must change whenever the chart changes. `appVersion` is informational and usually matches the container image tag.

`apiVersion: v2` is the chart format used since Helm 3, and it remains the format to use. Helm 4 adds an experimental `v3` chart API behind the `HELM_EXPERIMENTAL_CHART_V3` environment variable, but it's explicitly for testing and may change.[^5]

### Templates

Templates are Kubernetes manifests with Go template directives between `{{` and `}}`.[^7] A minimal Deployment:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
```

And the values it reads:

```yaml
# values.yaml
# replicaCount is the number of Pods to run.
replicaCount: 2

image:
  # image.repository is the container image to deploy.
  repository: ghcr.io/example/myapp
  # image.tag overrides the image tag. Defaults to the chart's appVersion.
  tag: ""

service:
  # service.port is the port the app listens on.
  port: 8080

# resources are the container's resource requests and limits.
resources: {}
```

The pieces worth knowing:

- **Built-in objects.** `.Values` is the merged configuration; `.Release` holds the release name, namespace and whether this is an install or upgrade; `.Chart` is the contents of `Chart.yaml`; `.Capabilities` describes the cluster (Kubernetes version, available APIs). Built-ins start with a capital letter, which distinguishes them from your own values.[^6]
- **Pipelines and functions.** `{{ .Values.image.tag | default .Chart.AppVersion }}` reads left to right. Helm ships the Sprig function library plus its own helpers such as `toYaml`, `include`, `required` and `tpl`.
- **Whitespace control.** `{{-` and `-}}` trim whitespace on that side. Most template bugs are indentation bugs; `nindent N` (newline then indent by N) is the standard way to drop a block of YAML in at the right depth.
- **`with` and `if`.** `{{- with .Values.resources }}` only emits the block if the value is non-empty, and sets `.` to that value inside it.
- **Named templates.** `_helpers.tpl` defines snippets with `{{ define "myapp.labels" }}` that you reuse with `include`. Files starting with `_` are never rendered as manifests.
- **`required`.** `{{ required "image.repository is required" .Values.image.repository }}` fails the render with a clear message instead of producing a broken manifest.

The `checksum/config` annotation above is a well-known trick: it hashes the rendered ConfigMap into the Pod template, so changing the config changes the Pod spec and triggers a rolling restart. Without it, Pods keep running with the old config until something else restarts them.[^8]

### Designing values

`values.yaml` is the public interface of your chart, so it deserves the same care as an API. The official best practices recommend:[^6]

- **camelCase names**, starting with a lowercase letter.
- **Prefer flat over deeply nested**, because every level of nesting needs an existence check in the templates. Nest when it groups related settings meaningfully, like `image.repository` and `image.tag`.
- **Document every value** with a comment that begins with the value's name, as in the example above. Tools like helm-docs can turn those comments into a README.
- **Quote strings** that YAML might misread, such as version numbers or `"on"`.

Helm 4 also supports multiple YAML documents in a single values file, separated by `---`.[^5][^9]

You can enforce types and required fields by adding a `values.schema.json` (JSON Schema) to the chart. Helm validates values against it on `install`, `upgrade`, `lint` and `template`, which catches mistakes like `replicaCount: "three"` before anything reaches the cluster.

### Dependencies

To deploy your app together with, say, a Redis instance, declare it in `Chart.yaml`:

```yaml
dependencies:
  - name: redis
    version: "~20.0.0"
    repository: oci://registry.example.com/charts
    condition: redis.enabled
```

`helm dependency update` downloads it into `charts/` and writes a `Chart.lock` that pins the exact version. You configure the subchart from the parent's values under a key with its name (`redis:`), and `condition` lets users turn it off and bring their own.

### Hooks

Some steps must run at a particular point in the release lifecycle, such as database migrations before an upgrade. Annotate a resource, usually a Job, as a hook:

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

Helm creates the Job at that point, waits for it to finish and only then continues. If it fails, the release fails. Hooks are not tracked as part of the release, so the delete policy matters, or old Jobs pile up.[^10]

### Test, lint, package, publish

```bash
helm lint ./myapp                          # catch structural and template errors
helm template test ./myapp -f ci-values.yaml | kubeconform -strict   # validate output (kubeconform is a separate tool)
helm install myapp ./myapp --dry-run=server  # render and validate against the real API server
helm test myapp                            # run the chart's test Pods after install

helm package ./myapp                       # produces myapp-0.3.0.tgz
helm registry login registry.example.com
helm push myapp-0.3.0.tgz oci://registry.example.com/charts
```

Note that in Helm 4, `helm registry login` takes only the registry's domain name, not a full URL with a path.[^5] Helm 4 also produces reproducible chart archives, so packaging the same source twice yields the same bytes and the same digest.[^1]

## Helm 4: what changed

Helm 4 is designed so that charts deployable with Helm 3 still deploy, and releases managed by Helm 3 can be upgraded by Helm 4 without migration.[^11] Most of the work went into the internals. The changes you are likely to notice:

- **Server-side apply.** New releases use Kubernetes server-side apply instead of Helm's own three-way merge, which gives clearer conflict handling when another tool or controller also manages fields on the same objects. Existing releases keep using whichever apply method they were created with on upgrade and rollback, so everything installed by Helm 3 stays on client-side apply unless you pass `--server-side` explicitly.[^5]
- **kstatus-based waiting**, described above.
- **Plugins rebuilt.** A new plugin system with an optional WebAssembly runtime for sandboxed plugins. Existing plugins keep working.[^5]
- **Post-renderers are plugins.** `--post-renderer` now takes a plugin name rather than a path to an executable. If you pipe Helm output through Kustomize with a script, this is the change most likely to break your pipeline.[^5]
- **Renamed flags:** `--atomic` → `--rollback-on-failure`, `--force` → `--force-replace`.
- **Content-based caching**, digest installs for OCI charts, and logging through Go's `slog` for SDK users.[^1]

### Helm 3 support

Helm 3's last feature release was on September 9, 2026, and it now only gets security fixes. Those end on **February 10, 2027**; after that there will be no Helm 3 releases at all, including Kubernetes client library updates.[^11] That last part matters: an unmaintained Helm 3 will gradually fall behind new Kubernetes versions.

The upgrade itself is mostly swapping the binary. The maintainers' checklist is to try your charts against a non-production cluster, update CI scripts for the renamed flags, convert any post-renderer to a plugin and check `helm registry login` calls.[^11]

## Pitfalls and good habits

**CRDs.** Helm installs CustomResourceDefinitions placed in a chart's `crds/` directory on first install, but deliberately never upgrades or deletes them, because deleting a CRD deletes every custom resource of that type in the cluster. Helm 4 didn't change this.[^12] Many charts work around it by templating CRDs as ordinary resources behind a flag; others expect you to apply CRD updates yourself before upgrading. Read the chart's upgrade notes.

**Secrets in values.** Values files end up in Git, and every revision's values are stored in the release Secret. Don't put real passwords in `values.yaml`. Reference existing Secrets by name, or use something like External Secrets Operator or Sealed Secrets.

**Drift.** If someone runs `kubectl edit` on a Helm-managed object, Helm doesn't notice until the next upgrade. The `helm-diff` plugin shows what an upgrade would change, which is worth running before every production upgrade.

**Release state lives in the cluster.** A namespace deleted out from under a release takes its history with it. A release stuck in `pending-upgrade` (for example after a CI job was killed mid-upgrade) blocks further upgrades until you roll it back.

**Don't template what you don't need to.** A chart that exposes every field of every resource as a value is just YAML with extra steps. Expose what users actually change, and set sensible defaults for the rest.

## Helm vs Kustomize, and GitOps

Kustomize, built into `kubectl`, takes the opposite approach: no templates, just plain YAML plus patches layered on top for each environment. It's simpler to read and has nothing to learn, but it has no notion of a package, a version, a release or a rollback, and it doesn't handle "install this third-party app with these settings" nearly as well.

In practice, many teams use both: Helm for third-party software and anything distributed to others, Kustomize for in-house apps with a few environment overlays. GitOps tools like Argo CD and Flux render Helm charts natively, so you can keep the chart and values in Git and let the controller do the `helm upgrade`. Note that Argo CD renders charts with `helm template` and applies the result itself, so `helm list` won't show those apps; Flux's helm-controller does create real Helm releases.

## Cheat sheet

```bash
# Explore
helm show values <chart>                      # what can I configure?
helm template <name> <chart> -f values.yaml   # what will it create?

# Install and upgrade (idempotent)
helm upgrade --install <name> <chart> -n <ns> --create-namespace \
  --version <x.y.z> -f values.yaml --wait --rollback-on-failure

# Inspect
helm list -A
helm status <name> -n <ns>
helm get values <name> -n <ns> --all
helm history <name> -n <ns>

# Undo
helm rollback <name> <revision> -n <ns>
helm uninstall <name> -n <ns>

# Build
helm create <chart>
helm lint <chart>
helm package <chart>
helm push <chart>-<ver>.tgz oci://<registry>/<path>
```

Helm's model is small: templates plus values make manifests, and every apply is a numbered revision you can return to. Once that clicks, most of the rest is reading a chart's `values.yaml` carefully and pinning your versions.

## References

[^1]: [Helm 4 Released](https://helm.sh/blog/helm-4-released), Helm blog, November 2025.
[^2]: [Charts](https://helm.sh/docs/topics/charts/), Helm documentation.
[^3]: [Installing Helm](https://helm.sh/docs/intro/install/), Helm documentation.
[^4]: [Artifact Hub](https://artifacthub.io/), CNCF.
[^5]: [Helm 4 Overview](https://helm.sh/docs/overview), Helm documentation (v4.3.1).
[^6]: [Chart Best Practices: Values](https://helm.sh/docs/chart_best_practices/values/), Helm documentation.
[^7]: [The Chart Template Developer's Guide](https://helm.sh/docs/chart_template_guide/), Helm documentation.
[^8]: [Chart Development Tips and Tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/), Helm documentation.
[^9]: [Helm v4: key changes from v3](https://dev.to/ilya-lesikov/helm-v4-key-changes-from-v3-2i26), Ilya Lesikov, DEV Community.
[^10]: [Chart Hooks](https://helm.sh/docs/topics/charts_hooks/), Helm documentation.
[^11]: [Helm 3 End of Life](https://helm.sh/blog/helm-v3-end-of-life), Helm blog, June 2026.
[^12]: [Helm Improves Kubernetes Package Management with Biggest Release in 6 Years](https://www.infoq.com/news/2025/11/helm-4), InfoQ, November 2025.
