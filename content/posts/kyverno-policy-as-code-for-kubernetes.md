---
title: "Kyverno: Policy as Code for Kubernetes"
date: 2026-11-20
draft: true
description: "What Kyverno is, how it plugs into the Kubernetes API server, and how to write validating, mutating, generating, image-verifying and deleting policies with the CEL-based policy types that replaced ClusterPolicy in Kyverno 1.20."
summary: "Kyverno lets you write cluster rules as Kubernetes resources: reject bad Pods, fill in safe defaults, create per-namespace boilerplate, verify image signatures and clean up old objects. With Kyverno 1.20 the old ClusterPolicy API is gone, so here is the tour using the new CEL-based policy types."
tags: ["kubernetes", "kyverno", "policy as code", "admission control", "security", "cel"]
categories: ["Programming"]
---

In my [post on admission webhooks](/posts/create-your-own-kubernetes-admission-webhook/) we built `pod-guard`, a small Go service that rejects Pods without a `team` label and fills in safe defaults. It worked, but it came with a lot of work around it: TLS certificates, a highly available Deployment, idempotent JSON patches, metrics and a break-glass plan. And it only knew two rules.

**Kyverno** is the answer to "I want rules like that, but I don't want to write and run a webhook for each one". It's a policy engine for Kubernetes: you install it once, and then you write policies as ordinary Kubernetes resources. Kyverno takes care of the webhook, the certificates and the reporting.

Kyverno has been around since 2020. It became a CNCF Graduated project in March 2026[^1], and it has just finished its biggest change since then. **Kyverno 1.20, released in November 2026, removed the old `ClusterPolicy` and `Policy` types.** Everything now uses a new family of policy types written in CEL, the same expression language Kubernetes uses for its built-in admission policies[^2][^3]. Many tutorials online still show `ClusterPolicy`, and that YAML no longer works on a current Kyverno. Everything in this post uses the new API.

Everything here is based on Kyverno 1.20 and Kubernetes v1.37. Kyverno 1.20 is tested against Kubernetes v1.34 to v1.37[^3].

## What Kyverno does

Kyverno runs in your cluster as a set of controllers. The most important one is an **admission controller**, a webhook that the API server calls on every matching create, update or delete, exactly like `pod-guard`. The others work in the background: they scan existing resources, create and sync generated resources, and delete things on a schedule.

What a policy can do depends on its type. There are five, all in the `policies.kyverno.io/v1` API group[^4]:

| Policy type | What it does | Example |
| --- | --- | --- |
| `ValidatingPolicy` | Allows, denies, warns about or audits resources. | "Every Pod needs a `team` label." |
| `MutatingPolicy` | Changes resources as they come in, or changes existing ones. | "Default the seccomp profile to `RuntimeDefault`." |
| `GeneratingPolicy` | Creates or copies resources when something else happens. | "Every new namespace gets a default-deny NetworkPolicy." |
| `ImageValidatingPolicy` | Checks container image signatures and attestations. | "Only run images signed by our CI pipeline." |
| `DeletingPolicy` | Deletes matching resources on a schedule. | "Delete completed Jobs every night." |

Each type has a **namespaced** version too: `NamespacedValidatingPolicy`, `NamespacedMutatingPolicy` and so on. These only apply in the namespace they're created in[^4]. That means a team can write policies for its own namespace without being allowed to change cluster-wide rules.

Kyverno also has a `PolicyException` type to exempt specific resources from a policy. And it writes the results of its checks to `PolicyReport` objects, which you can read with `kubectl`.

## Where it sits

```text
 kubectl apply
      │
      ▼
 ┌───────────┐   ┌──────────────────────────┐   ┌──────────────────────────┐   ┌──────┐
 │ AuthN /   │──▶│ Mutating admission       │──▶│ Validating admission     │──▶│ etcd │
 │ AuthZ     │   │  • MutatingPolicy        │   │  • ValidatingPolicy      │   │      │
 └───────────┘   │  • ImageValidatingPolicy │   │  • ImageValidatingPolicy │   └──────┘
                 │    (pins image digests)  │   │                          │
                 └────────────┬─────────────┘   └────────────┬─────────────┘
                              │                              │
                              ▼                              ▼
                 ┌─────────────────────────────────────────────────────────┐
                 │ Kyverno                                                 │
                 │  admission controller   ← webhook calls                 │
                 │  background controller  → GeneratingPolicy, mutate      │
                 │                           existing resources            │
                 │  reports controller     → PolicyReports for existing    │
                 │                           resources                     │
                 │  cleanup controller     → DeletingPolicy on a schedule  │
                 └─────────────────────────────────────────────────────────┘
```

What I wrote about webhooks in general still applies. Mutation runs before validation, Kyverno is in the critical path of every request it matches, and if it's down while its policies are set to fail closed, those requests are rejected. The difference is that you run *one* well-tested webhook instead of writing your own for each rule.

## Installing it

The usual way is the Helm chart:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

That's enough to try it out on a kind cluster. For production, run several replicas of the admission controller and read the installation and high-availability sections of the docs first[^5].

You'll also want the **Kyverno CLI**. It runs policies against YAML files without a cluster, which makes it the right tool for CI pipelines. It's available as a `kubectl` plugin (`kubectl krew install kyverno`) and as a standalone binary.

## ValidatingPolicy: saying no

Let's rebuild `pod-guard`'s validation rules as a Kyverno policy:

```yaml
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: pod-basics
spec:
  validationActions: [Deny]
  evaluation:
    background:
      enabled: true          # also check existing Pods and report on them
  autogen:
    podControllers:
      controllers: [deployments, statefulsets, daemonsets, jobs, cronjobs]
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  variables:
    - name: containers
      expression: "object.spec.containers + object.spec.?initContainers.orValue([])"
  validations:
    - expression: "object.metadata.?labels['team'].orValue('') != ''"
      message: "Pods must have a 'team' label."
    - expression: >
        variables.containers.all(c,
          image(c.image).containsDigest() ||
          (image(c.image).tag() != '' && image(c.image).tag() != 'latest'))
      message: "Images must use an explicit tag other than 'latest', or a digest."
```

If you've read my posts on [privilege escalation](/posts/preventing-privilege-escalation-in-kubernetes/) or admission webhooks, most of this will look familiar. `matchConstraints`, `variables` and `validations` with CEL `expression`s work just as they do in a native Kubernetes `ValidatingAdmissionPolicy`. Kyverno adds a few things on top:

- **`validationActions`** decides what happens when a check fails. `Deny` rejects the request, `Warn` lets it through with a warning, and `Audit` lets it through and only records the result in a policy report[^6]. A good rollout starts with `Audit`, looks at the reports, and switches to `Deny` once they're clean.
- **`evaluation.background`** has Kyverno re-check resources that already exist. Admission control only sees new requests. The background scan tells you about the Pods that were running before the policy existed[^6].
- **`autogen`** creates matching rules for Deployments, StatefulSets, Jobs and the other workload types. Without it, a Deployment with a bad Pod template is accepted, and the error only appears later, when the ReplicaSet controller tries to create the Pod. With autogen, `kubectl apply` on the Deployment fails right away, with the message where the user can see it[^6].
- **Kyverno's CEL libraries.** Functions like `image()` parse image references, so you don't have to split strings by hand the way I did in Go. Other libraries can read other cluster resources (`resource.Get`, `resource.List`), call HTTP endpoints, read cached data from a `GlobalContextEntry`, decode X.509 certificates, and more[^7].

The last point is the main reason to choose Kyverno over a plain `ValidatingAdmissionPolicy`. Native policies can only see the request itself and a parameter object. Kyverno policies can look things up.

### Trying it

```console
$ kubectl create namespace demo
$ kubectl -n demo run web --image=nginx --dry-run=server
Error from server: admission webhook "vpol.validate.kyverno.svc-fail" denied the request: ...
Pods must have a 'team' label.
Images must use an explicit tag other than 'latest', or a digest.
```

The exact wording of the error depends on the Kyverno version, but every failed check shows up. Then check what the background scan found in existing Pods:

```bash
kubectl get policyreports -A
kubectl -n demo describe policyreport <name>
```

### Testing it without a cluster

The same policy can be run against files in CI:

```bash
kyverno apply pod-basics.yaml --resource bad-pod.yaml
```

`kyverno test` goes a step further: you describe the expected result (pass or fail) for each resource in a test file and keep it in Git next to the policy. That's how you stop a policy change from quietly breaking deployments.

The **Kyverno Playground** is a web page where you paste a policy and a resource and see the result. It's the fastest way to debug a CEL expression[^2].

### ValidatingPolicy outside Kubernetes

A `ValidatingPolicy` with `evaluation.mode: JSON` checks any JSON payload, not just Kubernetes resources: a Terraform plan, a CI pipeline definition, an API request[^6]. You run these with the CLI or embed the engine in another program. It's a niche feature, but it means you can use one policy language in more than one place.

## MutatingPolicy: filling in defaults

`pod-guard` also set a `RuntimeDefault` seccomp profile on Pods that didn't have one. Here's that as a `MutatingPolicy`:

```yaml
apiVersion: policies.kyverno.io/v1
kind: MutatingPolicy
metadata:
  name: default-seccomp
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
  matchConditions:
    - name: seccomp-not-set
      expression: "!has(object.spec.securityContext) || !has(object.spec.securityContext.seccompProfile)"
  mutations:
    - patchType: ApplyConfiguration
      applyConfiguration:
        expression: >
          Object{
            spec: Object.spec{
              securityContext: Object.spec.securityContext{
                seccompProfile: Object.spec.securityContext.seccompProfile{
                  type: "RuntimeDefault"
                }
              }
            }
          }
```

An `ApplyConfiguration` mutation describes the fields you want, and the result is merged into the object the way server-side apply merges. You don't have to check whether `securityContext` exists before setting a field inside it, which is exactly the bookkeeping that made the hand-written JSON Patch in `pod-guard` fiddly. If you need precise control, `patchType: JSONPatch` lets you write JSON Patch operations in CEL instead[^8].

Two more features are worth knowing:

- **Mutating existing resources.** With `evaluation.mutateExisting.enabled: true`, a policy can change resources that already exist, not just incoming ones. An optional `targetMatchConstraints` lets one resource trigger a change to a different one. For example, a change to a ConfigMap can add an annotation to the Deployments that use it, so they roll out again. This runs asynchronously in the background controller, which may need extra RBAC permissions to edit the target resources[^8].
- **`reinvocationPolicy: IfNeeded`** reapplies the mutation if a later mutation changed the object, just like the webhook setting of the same name[^8]. The same rule applies: your mutations must be idempotent.

## GeneratingPolicy: creating resources automatically

Mutation and validation are what every webhook does. Generation is what makes Kyverno more than a webhook framework. A `GeneratingPolicy` watches for a trigger and creates other resources in response.

The classic example: every new namespace should start with a default-deny NetworkPolicy, so that teams have to open traffic on purpose.

```yaml
apiVersion: policies.kyverno.io/v1
kind: GeneratingPolicy
metadata:
  name: default-deny-netpol
spec:
  evaluation:
    synchronize:
      enabled: true          # recreate or repair it if someone deletes or edits it
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["namespaces"]
  variables:
    - name: nsName
      expression: "object.metadata.name"
    - name: netpol
      expression: >
        [{
          "kind": dyn("NetworkPolicy"),
          "apiVersion": dyn("networking.k8s.io/v1"),
          "metadata": dyn({"name": "default-deny"}),
          "spec": dyn({
            "podSelector": dyn({}),
            "policyTypes": dyn(["Ingress", "Egress"])
          })
        }]
  generate:
    - expression: "generator.Apply(variables.nsName, variables.netpol)"
```

`generator.Apply(namespace, [resources])` creates the listed resources in the target namespace. The `dyn()` wrappers are needed because CEL is strictly typed, and a map whose values have different types needs to be marked as dynamic[^9].

Instead of writing the resource inline, you can also **copy** an existing one. A common use is copying an image pull secret from a central namespace into every new namespace: read the source with `resource.Get("v1", "secrets", "default", "regcred")` and pass it to `generator.Apply`. With `synchronize` enabled, the copies are updated when the source changes[^9].

Two other settings matter:

- `evaluation.generateExisting.enabled` applies the policy once to resources that already exist when you create it. Without it, only namespaces created *after* the policy get a NetworkPolicy.
- `evaluation.orphanDownstreamOnPolicyDelete.enabled` controls what happens to generated resources when the policy is deleted. By default they are deleted too[^9].

Be careful with that default. Deleting a generating policy can delete the NetworkPolicies in every namespace it covered, which opens up traffic that was blocked.

## ImageValidatingPolicy: checking where images come from

Pinning image tags tells you *which* image runs. It doesn't tell you *who built it*. An `ImageValidatingPolicy` checks signatures and attestations made with Sigstore cosign or Notary before a Pod is admitted[^10]:

```yaml
apiVersion: policies.kyverno.io/v1
kind: ImageValidatingPolicy
metadata:
  name: signed-by-our-ci
spec:
  validationActions: [Deny]
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  matchImageReferences:
    - glob: "ghcr.io/example-org/*"
  attestors:
    - name: github-actions
      cosign:
        keyless:
          identities:
            - subject: "https://github.com/example-org/api/.github/workflows/release.yaml@refs/heads/main"
              issuer: "https://token.actions.githubusercontent.com"
        ctlog:
          url: "https://rekor.sigstore.dev"
  validations:
    - expression: >-
        images.containers.map(image,
          verifyImageSignatures(image, [attestors.github-actions])).all(e, e > 0)
      message: "Images must be signed by the release workflow of example-org/api."
```

This is **keyless** signing. Instead of a long-lived private key, the CI job signs with a short-lived certificate tied to its OIDC identity, here a specific GitHub Actions workflow on the `main` branch. The policy accepts only images signed by exactly that workflow. Images outside `ghcr.io/example-org/` aren't matched at all, so you'd normally combine this with a validating rule that only allows approved registries.

Image checks need network calls to the registry and the transparency log, so they're the slowest kind of policy. Kyverno caches the results. And because the API server has a timeout for webhook calls, test with the large images your cluster really uses before you switch to `Deny`.

## DeletingPolicy: cleaning up

The last type deletes resources on a schedule. For example, delete completed Jobs in `ci` every night:

```yaml
apiVersion: policies.kyverno.io/v1
kind: NamespacedDeletingPolicy
metadata:
  name: cleanup-completed-jobs
  namespace: ci
spec:
  schedule: "0 2 * * *"      # every day at 02:00
  matchConstraints:
    resourceRules:
      - apiGroups: ["batch"]
        apiVersions: ["v1"]
        resources: ["jobs"]
  conditions:
    - name: completed
      expression: "object.status.?conditions.orValue([]).exists(c, c.type == 'Complete' && c.status == 'True')"
```

`deletionPropagationPolicy` controls whether dependent objects are deleted too (`Foreground`, `Background`) or left in place (`Orphan`)[^11]. For Jobs you usually want the Pods to go as well. For one-off cleanups there's also a simpler option: put the `cleanup.kyverno.io/ttl` label on a resource, and Kyverno deletes it when the time runs out.

For Jobs specifically, the built-in `ttlSecondsAfterFinished` field is often enough. A DeletingPolicy is for the cases Kubernetes doesn't cover: old ConfigMaps, leftover preview environments, Pods stuck in `Failed`, or bare Pods without an owner.

## Exceptions

Sooner or later a policy will block something that really is allowed: a monitoring agent that needs `hostPath`, a vendor image you can't re-sign. Don't weaken the policy for everyone. Create an exception:

```yaml
apiVersion: policies.kyverno.io/v1
kind: PolicyException
metadata:
  name: allow-node-exporter
  namespace: monitoring
spec:
  policyRefs:
    - name: disallow-host-path
      kind: ValidatingPolicy
  matchConditions:
    - name: node-exporter-only
      expression: "object.metadata.name.startsWith('node-exporter')"
```

The exception lives in the namespace it applies to, names the policy it's exempt from, and narrows the match with CEL[^2]. Treat permission to create `PolicyException`s like permission to edit policies. Whoever can create one can switch off any rule for their own workloads. Restrict it with RBAC, and keep the exceptions in Git where changes get reviewed.

## Kyverno, native admission policies, or Gatekeeper?

Kubernetes now has `ValidatingAdmissionPolicy` (stable since v1.30) and `MutatingAdmissionPolicy` (stable since v1.36) built in. Both use CEL and run inside the API server, with no webhook at all. So why install Kyverno?

| | Native admission policies | Kyverno | OPA Gatekeeper |
| --- | --- | --- | --- |
| Language | CEL | CEL, plus Kyverno's libraries | Rego, with CEL support in newer versions |
| Runs in | API server | Webhook and background controllers | Webhook and audit controller |
| Validate / mutate | Yes / yes | Yes / yes, including existing resources | Yes / yes |
| Generate resources | No | Yes | No |
| Verify image signatures | No | Yes | Through external providers |
| Scheduled cleanup | No | Yes | No |
| Look up other resources or HTTP APIs | No (parameter objects only) | Yes | Yes (data replication, external data) |
| Reports on existing resources | No | PolicyReports | Audit results on constraints |
| Test without a cluster | Limited | `kyverno apply`, `kyverno test` | `gator` |
| Extra component to run | None | Yes | Yes |

My rule of thumb:

- **Simple field checks and defaults** with no lookups: use native admission policies. There's nothing extra to install, and no webhook that can go down.
- **Anything that needs generation, image verification, cleanup, lookups or reporting:** Kyverno. If your team already writes Kubernetes YAML and some CEL, there's not much new to learn.
- **Policies shared with non-Kubernetes systems** that already use OPA, or a team that knows Rego well: Gatekeeper.

The CEL-based policy types also make it easier to move between Kyverno and native policies. For validating and mutating policies that use no Kyverno-only features, Kyverno can generate the matching native `ValidatingAdmissionPolicy` or `MutatingAdmissionPolicy`, so the check runs in the API server instead of the webhook[^12][^8]. You get Kyverno's tooling for writing and reporting, and the reliability of the API server for enforcement.

## Running Kyverno in production

Kyverno is a webhook, so everything from the [webhook post](/posts/create-your-own-kubernetes-admission-webhook/) applies:

- **Run it highly available.** Use several admission controller replicas spread across nodes, with a PodDisruptionBudget. A Kyverno outage combined with fail-closed policies blocks deployments.
- **Don't let it gate itself or its dependencies.** Keep `kube-system`, the `kyverno` namespace, your CNI and DNS out of Kyverno's scope. Its default configuration already excludes some system namespaces. Check what yours excludes before you add policies that match everything.
- **Roll out with `Audit` first.** Watch the policy reports, then switch to `Deny` namespace by namespace.
- **Protect the policies.** Permission to create, edit or delete Kyverno policies or exceptions is effectively permission to switch off your guardrails, and a malicious `MutatingPolicy` can inject anything into every new Pod. Treat these permissions like cluster-admin, as described in the [privilege escalation post](/posts/preventing-privilege-escalation-in-kubernetes/).
- **Be careful with lookups.** Policies that call HTTP endpoints or read many resources add latency to every matching request. Since Kyverno 1.18, HTTP calls to loopback addresses and cloud metadata services are blocked by default, and namespaced policies can't make HTTP calls unless you turn that on[^13]. Leave those defaults on.
- **Keep up with releases.** Kyverno releases roughly every three months, and since 1.18 only the current and the previous release get patches[^14]. A cluster running a Kyverno version that's two releases old no longer gets security fixes.
- **Read the hardening guide.** The project published a threat model and hardening guide written by ControlPlane[^15].

## Migrating from ClusterPolicy

If you have existing `ClusterPolicy`, `Policy`, `CleanupPolicy` or old-style `PolicyException` resources, you must migrate them **before** upgrading to 1.20. Kyverno 1.19, released in August 2026, was the last version to support them fully[^2]. The mapping is mostly one to one:

| Old | New |
| --- | --- |
| `ClusterPolicy` / `Policy` with `validate` rules | `ValidatingPolicy` / `NamespacedValidatingPolicy` |
| `mutate` rules | `MutatingPolicy` |
| `generate` rules | `GeneratingPolicy` |
| `verifyImages` rules | `ImageValidatingPolicy` |
| `CleanupPolicy` / `ClusterCleanupPolicy` | `DeletingPolicy` |
| `PolicyException` in `kyverno.io/v2` | `PolicyException` in `policies.kyverno.io/v1` |

The field names change along the way. `validationFailureAction` becomes `validationActions`, `background` becomes `evaluation.background.enabled`, `context` entries become `variables`, and `exclude` blocks become `matchConstraints.excludeResourceRules` or `matchConditions`[^16]. The biggest change is the language: JMESPath expressions and `pattern` overlays become CEL. The migration guide has field-by-field mappings and a JMESPath-to-CEL guide[^16].

Two things make this less painful than it sounds:

1. **Your existing CLI tests keep working.** You can run the same `kyverno test` suite against the migrated policies and check that they give the same results[^2].
2. **Policy reports don't change**, so dashboards and alerts built on them keep working[^2].

Upgrade to 1.19 first, migrate and test there while both APIs work side by side, and only then move to 1.20. After upgrading, `kyverno migrate` rewrites stored objects to the current storage version[^2].

## Bottom line

- **Kyverno is a policy engine for Kubernetes.** You write policies as Kubernetes resources, and Kyverno runs the webhook, the background scans and the reporting.
- **Five policy types** cover validation, mutation, generation, image verification and scheduled deletion. Each has a namespaced version for team-owned rules.
- **Since Kyverno 1.20, everything is CEL** in `policies.kyverno.io/v1`. `ClusterPolicy` is gone, and older tutorials no longer apply.
- **What Kyverno adds over native admission policies** is lookups, resource generation, image signature checks, cleanup, background reports and a CLI for testing in CI.
- **It's still a webhook in your critical path.** Run it highly available, keep it away from its own dependencies, roll out with `Audit` first, and guard who can change policies and exceptions.

## References

[^1]: [Cloud Native Computing Foundation Announces Kyverno's Graduation](https://www.cncf.io/announcements/2026/03/24/cloud-native-computing-foundation-announces-kyvernos-graduation/), CNCF, March 24, 2026.
[^2]: [Announcing Kyverno Release 1.19](https://kyverno.io/blog/2026/08/20/announcing-kyverno-release-1.19), Kyverno blog: feature parity for CEL-based policy types, deprecation and removal timeline, migration notes.
[^3]: [Kyverno Releases](https://kyverno.io/docs/releases/), Kyverno documentation: supported releases and Kubernetes versions.
[^4]: [Policy Types Overview](https://kyverno.io/docs/policy-types/overview/), Kyverno documentation.
[^5]: [Installation](https://kyverno.io/docs/installation/), Kyverno documentation.
[^6]: [ValidatingPolicy](https://kyverno.io/docs/policy-types/validating-policy), Kyverno documentation: validation actions, background evaluation, autogen and JSON mode.
[^7]: [CEL Libraries](https://kyverno.io/docs/policy-types/cel-libraries/), Kyverno documentation.
[^8]: [MutatingPolicy](https://kyverno.io/docs/policy-types/mutating-policy), Kyverno documentation: ApplyConfiguration and JSONPatch mutations, mutating existing resources, reinvocation, autogen and MutatingAdmissionPolicy generation.
[^9]: [GeneratingPolicy](https://kyverno.io/docs/policy-types/generating-policy), Kyverno documentation.
[^10]: [ImageValidatingPolicy](https://kyverno.io/docs/policy-types/image-validating-policy), Kyverno documentation.
[^11]: [DeletingPolicy](https://kyverno.io/docs/policy-types/deleting-policy), Kyverno documentation.
[^12]: [Announcing Kyverno Release 1.14](https://www.cncf.io/blog/2025/05/14/announcing-kyverno-release-1-14/), CNCF blog: introduction of ValidatingPolicy and ImageValidatingPolicy, and generating ValidatingAdmissionPolicies from ValidatingPolicy.
[^13]: [Announcing Kyverno Release 1.18](https://kyverno.io/blog/2026/04/24/announcing-kyverno-release-1.18/), Kyverno blog: HTTP call hardening and the new patch support model.
[^14]: [Announcing Kyverno Release 1.18](https://www.cncf.io/blog/2026/05/05/announcing-kyverno-release-1-18/), CNCF blog: "main + 1" patch support.
[^15]: [Kyverno blog](https://kyverno.io/blog): announcement of the end-user threat model and hardening guide by ControlPlane.
[^16]: [Migrating to CEL Policies](https://kyverno.io/docs/guides/migration-to-cel/), Kyverno documentation: field mappings from ClusterPolicy to the CEL-based policy types.
