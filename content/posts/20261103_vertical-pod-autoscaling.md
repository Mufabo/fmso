---
title: "Vertical Pod Autoscaling in Kubernetes: What It Is and How It Works"
date: 2026-11-03
draft: true
description: "What the Vertical Pod Autoscaler does, why right-sizing CPU and memory requests matters, and how VPA's recommender, updater and admission controller work together, including the new in-place update modes."
summary: "Most Pods ask for the wrong amount of CPU and memory. The Vertical Pod Autoscaler watches what they actually use and fixes their requests for you. Here is what it does, why you need it, and how it works under the hood."
tags: ["kubernetes", "autoscaling", "vpa", "resource management", "platform engineering"]
categories: ["Programming"]
---

Every container in Kubernetes is supposed to say how much CPU and memory it needs. Almost nobody gets those numbers right. They are guessed when the service is first deployed, copied from another service, or bumped up after an outage and never lowered again. The **Vertical Pod Autoscaler (VPA)** exists to replace those guesses with numbers based on what your containers actually use.

This post covers what VPA is, why it is needed, what it does, how it does it, and where it fits next to the other autoscalers.

Everything here is based on Kubernetes v1.37 and VPA 1.8, the current releases at the time of writing.

## Horizontal vs vertical scaling

There are two ways to give an application more capacity:

- **Horizontal scaling** adds more copies. Three Pods become six. This is what the Horizontal Pod Autoscaler (HPA) does.
- **Vertical scaling** makes each copy bigger. A Pod with 500m CPU and 512Mi of memory gets 1 CPU and 1Gi. This is what VPA does.

Horizontal scaling is usually the better tool for handling load, but it only works if the application can run as many interchangeable copies. Plenty of things can't, or can't easily: databases, caches, a singleton controller, a legacy app, a batch job. And even for workloads that scale horizontally, *each* copy still needs a correctly sized request. HPA decides how many Pods you run; it doesn't decide how big they are.

## Why requests matter so much

To see why VPA is useful, you need to know what Kubernetes does with the numbers in `resources`:

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    memory: 1Gi
```

- **Requests** are what the scheduler uses. A Pod is only placed on a node with enough *unrequested* capacity. Once placed, the request is reserved for that Pod whether it uses it or not. The CPU request also sets the container's share of CPU when the node is busy.
- **Limits** are enforced at runtime. A container that goes over its CPU limit is throttled. A container that goes over its memory limit is killed (OOMKilled).

So requests decide how many Pods fit on a node, and therefore how many nodes you pay for. Getting them wrong hurts in both directions:

**Requests too high (over-provisioning).** The scheduler reserves capacity that sits idle. Nodes look full while running at 15% CPU. The cluster autoscaler adds nodes you don't need. In most real clusters, this is where the money goes: it is common for actual usage to be a fraction of what is requested.

**Requests too low (under-provisioning).** The scheduler packs too many Pods onto a node. When they all get busy, they compete for CPU and slow down. Memory is worse: if the node runs out, the kubelet starts evicting Pods, and if a container exceeds its memory limit, it is killed. Low requests also put Pods in a lower quality-of-service class, so they are the first to be evicted under pressure.

On top of that, the right numbers change over time. Traffic grows, a new feature uses more memory, a library upgrade makes something faster. A request that was right in January is wrong by June, and nobody revisits it.

Right-sizing by hand means watching dashboards for every container in every service, picking numbers, and redeploying, then doing it again every few months. That doesn't scale past a handful of services. VPA automates it.

## What VPA does

In one sentence: **VPA watches the actual CPU and memory usage of the containers in a workload, computes what their requests should be, and (optionally) applies those values to the Pods.**

You tell it which workload to watch with a `VerticalPodAutoscaler` object:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Off"
```

The `targetRef` can point at a Deployment, StatefulSet, DaemonSet, ReplicaSet, Job, CronJob, or any custom resource that implements the `scale` subresource (for example, an Argo Rollout).

After a while, VPA fills in a recommendation in the object's status:

```bash
kubectl describe vpa web-vpa
```

```text
Status:
  Recommendation:
    Container Recommendations:
      Container Name:  web
      Lower Bound:
        Cpu:     120m
        Memory:  262144k
      Target:
        Cpu:     210m
        Memory:  340Mi
      Uncapped Target:
        Cpu:     210m
        Memory:  340Mi
      Upper Bound:
        Cpu:     1100m
        Memory:  1.2Gi
```

There are four numbers per container:

| Field | Meaning |
| --- | --- |
| `target` | The request VPA thinks the container should have. This is the value it applies. |
| `lowerBound` | The minimum that still looks safe. If the current request is below this, the Pod is badly under-provisioned. |
| `upperBound` | The maximum that is still reasonable. If the current request is above this, the Pod is badly over-provisioned. |
| `uncappedTarget` | What the target would be if you hadn't set any minimum or maximum in the resource policy. |

With `updateMode: "Off"`, that's all VPA does: it recommends and you decide. With the other modes, it applies the target to your Pods, as described below.

## How VPA works

VPA is not built into Kubernetes. It is a separate project in the [`kubernetes/autoscaler`](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) repository that you install into your cluster (many managed services, such as GKE and EKS, offer it as an add-on or ship their own implementation of the same API). It consists of a CustomResourceDefinition plus three components, each with one job:

```text
          metrics-server                        VerticalPodAutoscaler
   (current CPU/memory per container)            objects (your config)
                │                                        │
                ▼                                        ▼
        ┌───────────────┐   writes recommendation   ┌─────────┐
        │  Recommender  │ ────────────────────────► │   VPA   │
        └───────────────┘     into VPA status       │ status  │
                                                    └────┬────┘
                         reads recommendation            │
              ┌──────────────────────────────────────────┤
              ▼                                          ▼
        ┌───────────┐  evict or resize Pod     ┌─────────────────────┐
        │  Updater  │ ───────────────────────► │ Admission controller │
        └───────────┘                          │ (sets requests on    │
                                               │  every new Pod)      │
                                               └─────────────────────┘
```

### The recommender: working out the right numbers

The recommender is the brain. Roughly once a minute, it reads current usage for every container from the [Metrics API](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/) (usually served by `metrics-server`), and it watches for OOM events. It feeds those samples into a model per container and writes the result into the status of each VPA object.

The model is the interesting part. VPA doesn't just take an average, because averages are a bad basis for a request: half the time the container would use more than it asked for. It doesn't try to predict future load either. It builds a statistical picture of past usage, weighted toward recent days, and reads percentiles from it. Here is how, step by step.

#### Step 1: collect samples, per workload and container

Once a minute, the recommender reads current usage for every running container. Samples are grouped by **workload and container name**, not by Pod. All replicas of the `web` container in a Deployment feed the same model, which is why every replica gets the same recommendation. It also means a Pod that is replaced doesn't take its history with it.

#### Step 2: put samples into decaying histograms

Each container gets two histograms, one for CPU and one for memory. The buckets grow exponentially, each about 5% wider than the last. CPU buckets start at 0.01 cores and memory buckets at about 10 MB, so the histograms are precise for small containers and still reach very large ones.

CPU and memory are recorded differently, because they fail differently:

- **CPU is compressible.** A container that wants more CPU than it gets is slowed down, not killed. So every CPU sample goes into the histogram. Each sample is weighted by the container's CPU request at the time (with a small minimum), so samples from containers with bigger requests count for more.
- **Memory is incompressible.** A container that needs more memory than its limit is killed. So what matters is the highest point, not the typical one. VPA divides time into aggregation windows (24 hours by default, eight of them kept) and records only the **peak** in each window. When usage beats the current window's peak, the old peak is taken out of the histogram and the new one goes in. Since VPA 1.7 you can set the window length per VPA object.

Recency comes from **exponential decay**, with a half-life of 24 hours by default for both histograms. A sample from yesterday weighs half as much as one from now; one from a week ago weighs about 1/128. In practice the last few days dominate. This lets recommendations follow a workload as it changes, while still remembering daily patterns.

#### Step 3: learn from OOM kills

When a container is OOMKilled, its real peak is unknown, because it would have used more if it had been allowed to. So VPA adds a synthetic memory sample above the usage at the time of the kill:

```text
sample = max(memory_at_oom + 100MB, memory_at_oom × 1.2)
```

Both numbers are recommender flags (`--oom-min-bump-up-bytes` and `--oom-bump-up-ratio`). This pushes the recommendation up immediately, instead of waiting for the histogram to slowly catch up while the container keeps dying.

#### Step 4: read percentiles and add a margin

From each histogram, VPA takes three percentiles and adds a safety margin (15% by default, `--recommendation-margin-fraction`):

| Value | Percentile | Then multiplied by |
| --- | --- | --- |
| Lower bound | 50th | margin × confidence adjustment |
| Target | 90th | margin |
| Upper bound | 95th | margin × confidence adjustment |

So the target roughly means: *what this container used 90% of the time, weighted toward recent days, plus 15% headroom*. For memory, "used" means the daily peaks, so the target covers most daily peaks plus headroom.

#### Step 5: adjust the bounds for how much history there is

A workload that has been watched for ten minutes shouldn't get the same tight range as one watched for two weeks. VPA measures **confidence** in days of history: the smaller of how long the workload has been observed and how many samples it has, divided by one day's worth of samples. Then it widens the bounds when confidence is low:

```text
upper bound × (1 + 1/confidence)
lower bound × (1 + 0.001/confidence)^-2
```

Some concrete values:

| History | Upper bound multiplier | Lower bound multiplier |
| --- | --- | --- |
| None | infinite | 0 |
| 5 minutes | ×289 | ×0.6 |
| 12 hours | ×3 | ≈×1.0 |
| 1 day | ×2 | ≈×1.0 |
| 7 days | ×1.14 | ≈×1.0 |

This matters because the updater only acts on Pods whose requests fall outside the bounds (or that differ substantially from the target after running for a while). Wide bounds early on mean VPA rarely touches a workload it knows little about. The lower bound tightens within the first hour, because running out of resources is the urgent case; the upper bound takes days, because being too generous is the cheap mistake.

#### Step 6: apply floors, caps and constraints

Finally the raw numbers are adjusted:

- **Per-Pod minimums.** By default every Pod gets at least 25m CPU and 250Mi memory (`--pod-recommendation-min-cpu-millicores`, `--pod-recommendation-min-memory-mb`). The minimum is split **evenly** across the Pod's containers. This catches people out: a tiny single-container Pod that uses 30Mi will be recommended 250Mi, and the recommendation will never move. Lower the flag if you run many small containers.
- **Your bounds.** `minAllowed` and `maxAllowed` from the resource policy are applied. The value before capping is kept as `uncappedTarget`, so you can see when your caps are holding VPA back.
- **Other constraints.** `controlledResources` and any `LimitRange` in the namespace are applied. Limits aren't recommended at all; they are scaled later, when the recommendation is applied, to keep the ratio from your Pod template.

#### A worked example

Say a container has two days of history. Its weighted CPU histogram has a median of 100m, a 90th percentile of 180m and a 95th percentile of 240m. Confidence is 2.

- Target: 180m × 1.15 ≈ **207m**
- Lower bound: 100m × 1.15 × ~1.0 ≈ **115m**
- Upper bound: 240m × 1.15 × (1 + 1/2) ≈ **414m**

If the Pod currently requests 1 CPU, that is above the upper bound, so the updater will act and bring the request down to about 207m. If it requests 300m, that's inside the bounds and only 45% above the target, so nothing happens until the Pod has been running for 12 hours; after that the difference is big enough to act on.

#### Persistence and tuning

The histograms are saved regularly in `VerticalPodAutoscalerCheckpoint` objects, so restarting the recommender doesn't throw away days of history. Optionally, the recommender can load past usage from Prometheus when it starts, so a fresh install doesn't start from zero.

All the numbers above are defaults set by recommender flags, such as `--target-cpu-percentile`, `--target-memory-percentile`, `--recommendation-margin-fraction`, `--cpu-histogram-decay-half-life` and `--memory-aggregation-interval`, and they apply to the whole cluster. If different workloads need different behaviour (say, a 95th-percentile target for latency-sensitive services), you can run several recommenders with different flags and choose one per VPA with `spec.recommenders`. Newer VPA releases are also starting to expose some of these settings, such as the OOM bump-up ratio, per VPA object.

### The admission controller: applying requests to new Pods

The admission controller is a mutating webhook. Every time a Pod is created, the API server asks it whether that Pod belongs to a workload with a VPA. If it does, and the update mode allows it, the webhook rewrites the Pod's resource requests (and limits, see below) to match the current recommendation *before the Pod is stored*.

This is the key trick: VPA never edits your Deployment. Your manifest in Git still says `cpu: 500m`, but every Pod created from it gets the VPA's value. That avoids a fight with GitOps tools that would otherwise keep reverting the change, but it also means **`kubectl get deployment -o yaml` doesn't show what your Pods are actually running with**. Look at the Pods.

If the admission controller is down, Pods are still created, just with the original values from the template. The webhook is configured to fail open.

### The updater: changing Pods that are already running

The admission controller only acts when a Pod is created. Pods in a Deployment might run for weeks. The updater is what makes VPA act on running Pods.

Every minute it compares each Pod's current requests with the recommendation, and decides whether a change is worth the disruption. It won't touch a Pod just because the target moved by 2%. Roughly, a Pod becomes a candidate when:

- its request is outside the recommended range (below the lower bound or above the upper bound), or
- it has been running for a while (12 hours by default) and the request differs from the target by a meaningful amount (10% by default), or
- it was OOMKilled shortly after starting.

What the updater then does depends on the update mode. Traditionally, it could only do one thing: **evict the Pod**. The workload controller creates a replacement, the admission controller gives the replacement the new requests, and the job is done. The updater respects PodDisruptionBudgets, by default doesn't act on workloads with fewer than two replicas, and limits how many Pods of a workload it evicts at once.

Eviction works, but it means a restart: connections are dropped, caches are cold, a JVM has to warm up again. That was the main reason people were reluctant to let VPA run automatically. Which brings us to the biggest change in VPA's recent history.

## In-place resize: vertical scaling without restarts

For most of Kubernetes' life, a Pod's resources were immutable. The only way to change them was to delete the Pod and create a new one.

That changed with **in-place Pod resize** (KEP-1287), which went alpha in Kubernetes v1.27, beta (enabled by default) in v1.33, and [stable in v1.35](https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga/). CPU and memory requests and limits can now be changed on a running Pod through its `resize` subresource. The kubelet adjusts the container's cgroup, usually without restarting the container. The Pod's status shows what was actually applied, and two conditions, `PodResizePending` and `PodResizeInProgress`, report progress. If the node doesn't have room, the resize is marked `Deferred` (and retried later) or `Infeasible`.

VPA builds on this with new update modes, which brings us to the full list.

## Update modes

The mode is set in `spec.updatePolicy.updateMode`:

| Mode | New Pods | Running Pods | Notes |
| --- | --- | --- | --- |
| `Off` | Unchanged | Unchanged | Recommend only. The safe way to start. |
| `Initial` | Get the recommendation | Never changed | Good for workloads that must not be disrupted, combined with regular deploys. |
| `Recreate` | Get the recommendation | Evicted and recreated when the change is big enough | The classic behaviour, and the default if you don't set a mode. |
| `InPlaceOrRecreate` | Get the recommendation | Resized in place; evicted only if the resize can't be done | Beta. Needs in-place resize in the cluster (v1.33+). |
| `InPlace` | Get the recommendation | Resized in place only; never evicted | Alpha since VPA 1.7, behind the `InPlace` feature gate on the updater and admission controller. |
| `Auto` | | | Deprecated since VPA 1.4. Currently an alias for `Recreate`. |

`Off` deserves a closer look, because it's where most people start. In `kubectl describe vpa` it shows up as `Update Policy: Update Mode: Off`. Only the recommender does any work in this mode: it keeps collecting usage and keeps the lower bound, target and upper bound in the VPA's status up to date. The admission controller lets new Pods through with whatever requests the workload template says, and the updater never evicts or resizes a running Pod. VPA watches and advises; you read the recommendation and decide whether to copy it into your manifests. Switching to another mode later is a one-line change, and the history the recommender has built up carries over, so VPA starts acting on a well-informed recommendation straight away.

If you have older manifests using `Auto`, switch them to an explicit mode. `Auto` still works, but you'll see a deprecation warning, and it will be removed in a future API version.

`InPlaceOrRecreate` is the mode most people will want for live workloads. It tries a resize first and only falls back to eviction when it must: for example when the Pod needs more than its node has free, or when a resize stays pending for too long.

`InPlace` is for workloads where any restart is unacceptable. When a resize can't be done, it waits and retries instead of evicting. The trade-off is that a Pod on a full node may stay under-sized until room appears.

Some things to know about in-place updates:

- **Not every resize is restart-free.** Each container has a `resizePolicy` per resource. The default is `NotRequired` (no restart), but some applications only read their memory size at startup, for example a JVM with a fixed heap. Those should use `RestartContainer` for memory, so the container restarts (in place, not rescheduled) when memory changes.
- **Shrinking memory is risky.** The kubelet only lowers a memory limit if current usage is below the new limit, but this check is best effort. A container whose usage grows right after a decrease can still be killed.
- **Some setups can't resize.** In-place resize is not allowed with swap, or with the static CPU Manager or Memory Manager policies.

## Controlling what VPA is allowed to do

A `resourcePolicy` puts guard rails around the recommendations, per container:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "InPlaceOrRecreate"
    minReplicas: 2
  resourcePolicy:
    containerPolicies:
      - containerName: web
        minAllowed:
          cpu: 100m
          memory: 256Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits
      - containerName: log-shipper
        mode: "Off"
```

- **`minAllowed` / `maxAllowed`** cap the target. Always set a `maxAllowed`. Without it, a memory leak becomes a recommendation to give the leaking container more memory, forever, until the Pod no longer fits on any node.
- **`controlledResources`** limits VPA to CPU only or memory only. A common pattern is to let HPA handle CPU-driven scaling and VPA handle only memory.
- **`mode: "Off"` per container** excludes a container, such as a sidecar you size yourself. The special name `"*"` sets a default policy for all containers.
- **`controlledValues`** decides what happens to limits.

### What happens to limits

By default (`RequestsAndLimits`), VPA keeps the ratio between limit and request that you wrote in the Pod template. If you wrote `requests.memory: 512Mi` and `limits.memory: 1Gi`, and VPA sets the request to 400Mi, the limit becomes 800Mi. If you didn't set a limit, VPA doesn't invent one.

With `RequestsOnly`, VPA only changes requests and leaves limits alone. That's safer if your limits are deliberately fixed, but watch out: if the recommended request rises above your fixed limit, the request gets capped at the limit.

VPA also respects `LimitRange` objects in the namespace. If a LimitRange sets a maximum or a maximum limit-to-request ratio, VPA adjusts its values to stay inside it.

Two other per-VPA settings were added recently: you can set how soon after an OOM kill a Pod should be evicted (`evictAfterOOMThreshold`), and the memory aggregation window mentioned earlier.

## VPA and the other autoscalers

**VPA and HPA.** Don't let both act on the same resource. If HPA scales on CPU utilization and VPA changes the CPU request, they fight: utilization is measured *as a percentage of the request*, so when VPA raises the request, utilization drops, and HPA removes Pods; then usage per Pod rises, and VPA raises the request again. The usual ways out are:

- HPA scales on something else, such as requests per second or queue length from custom or external metrics, while VPA manages CPU and memory.
- VPA only controls memory (`controlledResources: ["memory"]`) and HPA scales on CPU.
- VPA runs in `Off` mode, and you use its recommendations to set requests while HPA does the live scaling.

**VPA and the cluster autoscaler (or Karpenter).** These work together well. When VPA raises requests and Pods no longer fit, they become pending and the node autoscaler adds capacity. When VPA lowers requests, nodes empty out and can be removed. The main thing to watch is `maxAllowed`: a recommendation larger than your largest node type gives you Pods that can never be scheduled.

## Getting started

You need `metrics-server` (or another implementation of the Metrics API) running in the cluster. Then install VPA, either from your provider's add-on, from the official Helm chart in the autoscaler repository, or with the install script:

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

Check that the three components are running:

```bash
kubectl get pods -n kube-system | grep vpa
```

Then adopt it gradually:

1. **Start with `Off`.** Create VPA objects for your workloads in recommend-only mode and leave them for a week or two, so the histograms see weekly patterns.
2. **Compare.** Put the recommendations next to the current requests. You'll probably find a few badly under-sized workloads and many over-sized ones. Tools like Goldilocks put a dashboard on top of VPA recommendations if you have many workloads.
3. **Apply with care.** For stateless services with several replicas and a PodDisruptionBudget, move to `InPlaceOrRecreate`. For workloads that must not restart, use `Initial` and let your normal deployments pick up new values, or try `InPlace` if you're comfortable with an alpha feature.
4. **Set bounds.** Add `minAllowed` and `maxAllowed` for every container before letting VPA apply anything.
5. **Watch.** Keep an eye on OOM kills, CPU throttling, evictions and pending Pods for the first weeks.

## Limitations and gotchas

- **VPA reacts in hours, not seconds.** It is built for right-sizing based on history, not for absorbing a sudden traffic spike. That's HPA's job. A burst of load will hit your current requests long before VPA reacts.
- **It needs history.** New workloads get wide, cautious bounds for the first days.
- **Single-replica workloads aren't evicted by default.** `minReplicas` defaults to 2 so that VPA doesn't take down your only Pod. With in-place modes, this matters less.
- **Don't point two VPAs at the same workload.** The behaviour is undefined.
- **Spiky startup is tricky.** Applications that need a lot of CPU at startup (JVMs, anything that warms caches) and little afterwards get a recommendation sized for steady state, and then start slowly. VPA has experimental CPU startup boost support for this, and in-place resize makes the pattern (start big, shrink later) much more practical.
- **Your manifests drift from reality.** Since VPA mutates Pods rather than workloads, the numbers in Git stop being the numbers in production. Make sure the people debugging your cluster know to look at Pods and VPA status.
- **Memory leaks look like demand.** VPA can't tell the difference between a service that needs more memory and one that's leaking. `maxAllowed` and OOM alerts are your protection.

## Bottom line

- **Resource requests decide your cluster's cost and stability**, and hand-picked values are almost always wrong, usually too high.
- **VPA measures real usage and turns it into requests**, using decaying histograms of CPU usage and memory peaks, percentiles, safety margins and OOM feedback.
- **Three components do the work:** the recommender computes, the admission controller applies values to new Pods, and the updater acts on running ones.
- **In-place resize changed the trade-off.** With Kubernetes v1.35+ and the `InPlaceOrRecreate` mode, VPA can usually resize running Pods without restarting them, which removes the biggest reason people kept it switched off.
- **Start in `Off` mode, set bounds, and keep VPA and HPA off each other's resources.**

## Sources

- [Vertical Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/), Kubernetes documentation: architecture, update modes and resource policies.
- [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler), project README, [quick start](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/quickstart.md) and [API reference](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs/api.md).
- [VPA release notes](https://github.com/kubernetes/autoscaler/releases), including [1.7.0](https://github.com/kubernetes/autoscaler/releases/tag/vertical-pod-autoscaler-1.7.0) (`InPlace` mode) and [1.8.0](https://github.com/kubernetes/autoscaler/releases/tag/vertical-pod-autoscaler-1.8.0).
- [Deprecate UpdateMode Auto](https://github.com/kubernetes/autoscaler/pull/8426), the pull request that deprecated `Auto` and made `Recreate` the default.
- [Kubernetes v1.35: In-Place Pod Resize Graduates to Stable](https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga/), Kubernetes blog.
- [Resize CPU and Memory Resources assigned to Containers](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/), Kubernetes documentation.
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), Kubernetes documentation: how requests and limits are used.
