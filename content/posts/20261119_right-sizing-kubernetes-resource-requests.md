---
title: "How to Figure Out Resource Requests for Kubernetes Pods"
date: 2026-11-19
draft: true
description: "A practical method for choosing CPU and memory requests and limits for Kubernetes Pods: measure real usage, read the right percentiles, handle runtime quirks, and keep the numbers current with VPA and in-place resize."
summary: "Most resource requests are guesses copied from another service. Here is a repeatable way to replace them with numbers based on what your containers actually use, and to keep them right as the workload changes."
tags: ["kubernetes", "resource management", "prometheus", "autoscaling", "platform engineering"]
categories: ["Programming"]
---

Every container in Kubernetes should declare how much CPU and memory it needs. In practice those numbers are usually picked once, when the service is first deployed, by copying another service or guessing. Then nobody touches them again, except to raise them after an outage.

This post is a practical method for setting them properly. It doesn't need special tools: a metrics pipeline and a bit of PromQL get you most of the way. In short:

1. Deploy with a sensible guess.
2. Measure real usage under real load.
3. Turn percentiles into requests and limits.
4. Watch for the signs that the numbers are wrong.
5. Revisit them regularly, ideally with help from automation.

My post on the Vertical Pod Autoscaler covers how VPA automates most of this. This post is about how to do it yourself, and how to understand the numbers VPA hands you.

## What requests and limits do

A quick recap, because the method only makes sense if you know what each number controls[^1]:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    memory: 512Mi
```

- **Requests** are used by the scheduler. A Pod only goes on a node with enough unrequested capacity, and that capacity stays reserved for it whether it uses it or not. The CPU request also sets the container's share of CPU when the node is busy.
- **Limits** are enforced at runtime. A container that tries to use more CPU than its limit is **throttled**. A container that uses more memory than its limit is **killed** (OOMKilled).

Requests and limits together also decide the Pod's **quality-of-service (QoS) class**[^2]:

| QoS class | When | Under node memory pressure |
| --- | --- | --- |
| Guaranteed | Every container has CPU and memory requests equal to its limits | Evicted last |
| Burstable | At least one request or limit set, but not Guaranteed | Evicted after BestEffort |
| BestEffort | No requests or limits at all | Evicted first |

So getting requests wrong costs you in both directions. Too high, and you pay for nodes that sit mostly idle. Too low, and Pods get packed too tightly, fight over CPU, and get evicted or OOMKilled when things get busy.

## Step 1: Start with a sensible guess

You can't measure a workload that isn't running, so the first numbers are a guess. Make it an informed one:

- Run the service locally or in a test environment under a load test and check `docker stats` or `kubectl top`.
- Look at similar services already running in the cluster.
- Know your runtime's baseline. A JVM service rarely runs in less than a few hundred MiB; a small Go service may need 30Mi.

Err slightly on the generous side for memory, since running out of memory kills the container while running out of CPU only slows it down. And make sure every namespace has a `LimitRange` with default requests[^3], so a container nobody thought about doesn't end up as BestEffort:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: shop
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        memory: 256Mi
```

## Step 2: Measure real usage

### Which metrics to use

The kubelet exposes per-container metrics through cAdvisor[^4]. Two matter here:

- **CPU:** `container_cpu_usage_seconds_total`, a counter of CPU seconds used. Its `rate()` is usage in cores.
- **Memory:** `container_memory_working_set_bytes`. Use the working set, not `container_memory_usage_bytes`, because usage includes page cache the kernel can reclaim. The working set is what the kubelet uses for eviction decisions[^5], and it is the closest thing to what the OOM killer cares about.

`kubectl top pod --containers` is fine for a quick look, but it only shows the current value. Requests should be based on history, so you want Prometheus (or another time-series store) with at least a week of data.

### Over what period

Your measuring window needs to include your peaks: the Monday morning rush, the nightly batch run, the end-of-month report. A week is a reasonable minimum. Two weeks is better if traffic varies by week. If the service is new, run a load test that looks like production traffic, but don't fully trust the numbers until you've seen real traffic.

### Use percentiles, not averages

An average is a bad basis for a request: the container would be using more than its request about half the time. Look at high percentiles over the whole window instead.

P95 of CPU usage over seven days, sampled every 5 minutes:

```promql
quantile_over_time(0.95,
  rate(container_cpu_usage_seconds_total{namespace="shop", container="web"}[5m])[7d:5m]
)
```

Memory is different. What matters is the highest point, because going over the limit kills the container. Look at the maximum and a high percentile:

```promql
max_over_time(container_memory_working_set_bytes{namespace="shop", container="web"}[7d])

quantile_over_time(0.99,
  container_memory_working_set_bytes{namespace="shop", container="web"}[7d]
)
```

These give one series per Pod. Wrap them in `max by (container) (...)` to get a single number for all replicas.

One thing to keep in mind: if the container already has a tight CPU limit, its measured usage is capped by that limit. Throttled containers look like they need exactly their limit, when they would use more if allowed. Check for throttling (see step 4) before trusting CPU numbers.

## Step 3: Turn the measurements into numbers

A starting rule of thumb that works for most services:

| | Request | Limit |
| --- | --- | --- |
| **CPU** | P90 to P95 of usage | Often none (see below) |
| **Memory** | P99 or max of working set, plus 10 to 30% headroom | Equal to the request, or slightly above |

### Memory

Memory can't be compressed. When a container needs more than its limit, it dies. So base the request on the peak, add headroom for growth and for the peak you didn't happen to see, and set the limit equal to the request (or a little above).

Setting the memory limit equal to the request is the safe default. With a limit much higher than the request, the scheduler places the Pod based on the small number, and then the Pod uses the big number. When several Pods on a node do that at once, the node runs out of memory and the kubelet starts evicting Pods. A limit equal to the request means the Pod gets exactly what was reserved for it.

### CPU

CPU is compressible. A container that wants more than it can get just runs slower. That changes the advice:

- **Set the request around typical busy usage**, P90 to P95. This guarantees the container that much CPU when the node is under contention.
- **Think hard before setting a CPU limit.** CPU limits are enforced by the CFS quota in 100ms periods. A container that uses its whole quota early in a period is throttled for the rest of it, even if the node has idle CPUs. For latency-sensitive services that shows up as unexplained tail latency. Many teams set only a CPU request and no limit, so containers can use spare CPU when it's there and fall back to their fair share when it isn't.

There are good reasons to keep CPU limits: multi-tenant clusters where one team shouldn't be able to soak up spare CPU, performance testing where you want predictable numbers, or when you need Guaranteed QoS (which requires CPU limit = request). If you do set them, give generous headroom above the request and watch throttling.

### A worked example

Say a web service shows this over two weeks:

- CPU: P50 = 120m, P95 = 310m, max = 900m (a few spikes at deploy time)
- Memory working set: P99 = 410Mi, max = 450Mi

Reasonable settings:

```yaml
resources:
  requests:
    cpu: 300m        # around P95
    memory: 540Mi    # max 450Mi + 20% headroom
  limits:
    memory: 540Mi    # same as the request
    # no CPU limit: the startup spikes can use spare CPU
```

If the service is critical and you want Guaranteed QoS, add `cpu: 300m` (or a bit more) as a limit, and accept slower starts.

## Step 4: Watch for signs the numbers are wrong

Once the numbers are applied, these signals tell you something needs adjusting.

**CPU throttling.** The share of CFS periods in which the container was throttled:

```promql
sum by (namespace, pod, container) (rate(container_cpu_cfs_throttled_periods_total[5m]))
/
sum by (namespace, pod, container) (rate(container_cpu_cfs_periods_total[5m]))
```

A few percent is normal. If it's consistently above 10 to 25% on a latency-sensitive service, the CPU limit is too low or shouldn't be there.

**OOM kills.** With kube-state-metrics:

```promql
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
```

Also watch restart counts. Repeated OOM kills mean memory is too tight, or the application is leaking. Raising the limit fixes the first problem and only delays the second.

**Pending Pods.** If Pods stay `Pending` with `Insufficient cpu` or `Insufficient memory` events, the requests don't fit on any node with free capacity. Either the requests are too big, or the cluster really needs more nodes.

**Waste across the cluster.** Compare what's requested with what's used, per namespace:

```promql
sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"})
/
sum by (namespace) (rate(container_cpu_usage_seconds_total{container!="", container!="POD"}[5m]))
```

A ratio of 4 means the namespace reserves four times the CPU it uses. That's where the savings are. Ratios close to or below 1 point to under-provisioned workloads that are probably being squeezed.

## Step 5: Know your runtime

Some runtimes don't look at container limits by default, or look at the wrong ones. Check these before blaming your numbers.

**JVM.** Modern JVMs read the container's memory limit, but by default the heap gets only 25% of it. Set the heap as a percentage instead of a fixed `-Xmx`, so it follows the limit:

```text
-XX:MaxRAMPercentage=75.0
```

Leave the rest for metaspace, thread stacks, direct buffers and the JIT. A JVM service with a 75% heap and a few hundred threads can still go over its limit, so watch the working set, not just heap metrics.

**Go.** Since Go 1.25, `GOMAXPROCS` defaults to the container's CPU limit when there is one[^6]. Without a CPU limit, it uses the number of CPUs on the node, which can be far more than the container's request. On older Go versions, Uber's `automaxprocs`[^7] does the same job. For memory, set `GOMEMLIMIT` a bit below the container memory limit[^8], so the garbage collector works harder before the container gets killed:

```yaml
env:
  - name: GOMEMLIMIT
    valueFrom:
      resourceFieldRef:
        resource: limits.memory
        divisor: "1"
```

Using the full limit works, but setting it to around 90% of it (via a fixed value) leaves room for memory the Go runtime doesn't manage.

**Node.js.** Set `--max-old-space-size` (in MiB) below the memory limit, so V8 collects garbage before the kernel kills the process.

**Python, Ruby and others with worker processes.** Memory scales with the number of workers. If you run four Gunicorn workers, the request has to cover all four. Pick the worker count first, then measure.

## Step 6: Let tools do the measuring

Doing the above by hand is fine for a few services. Past that, let a tool do the measuring:

- **VPA in recommendation mode.** Create a `VerticalPodAutoscaler` with `updateMode: "Off"`[^9]. It builds a history of usage and writes a lower bound, target and upper bound into its status, without changing any Pods. Its target is close to what this post recommends: the 90th percentile of CPU usage and of daily memory peaks, weighted toward recent days, plus a 15% margin.
- **Goldilocks.** Fairwinds' Goldilocks[^10] creates VPA objects in recommendation mode for every workload in labelled namespaces and shows the results in a dashboard. Handy for a cluster-wide review.
- **Commercial tools** such as Kubecost, CAST AI or ScaleOps do the same across the whole cluster, add cost figures, and can apply changes for you.

Treat recommendations as a starting point, not a verdict. VPA doesn't know about the quarterly batch job that hasn't run yet, or that your service is about to get twice the traffic.

## Step 7: Mind the Horizontal Pod Autoscaler

If a workload uses HPA with CPU or memory utilization targets, requests matter twice. HPA calculates utilization as usage divided by the request[^11]:

- **Requests too high:** utilization looks low, so HPA scales out late, or not at all.
- **Requests too low:** utilization looks high, so HPA scales out early and runs more Pods than needed.

So fix requests first, then tune the HPA target. And don't let VPA (in an active mode) and HPA both act on CPU for the same workload: when VPA raises the request, utilization drops, HPA removes Pods, usage per Pod rises, and VPA raises the request again. Let HPA scale on CPU and VPA manage only memory, or let HPA scale on a custom metric like requests per second.

## Step 8: Keep the numbers current

The right requests change over time. Traffic grows, a new feature uses more memory, a library upgrade makes something faster. Build a habit of checking:

- After big releases, compare usage before and after.
- Every month or quarter, review the namespaces with the worst request-to-usage ratio.
- Alert on OOM kills and on sustained throttling, so under-provisioning finds you instead of the other way round.

Changing requests used to mean restarting Pods, which made people reluctant to tweak them. That's no longer necessary: **in-place Pod resize** went alpha in Kubernetes v1.27, beta in v1.33, and stable in v1.35[^12]. CPU and memory can now be changed on a running Pod, usually without restarting the container. Combined with VPA's `InPlaceOrRecreate` mode, requests can follow actual usage with far less disruption than before.

## Checklist

- Every container has requests. Namespaces have a `LimitRange` with defaults.
- Requests are based on at least a week of data that includes peak traffic, not on averages.
- **CPU:** request around P90 to P95. No limit, unless you have a reason for one.
- **Memory:** request at P99 or max working set plus 10 to 30% headroom. Limit equal to the request.
- Runtimes are told about their limits: `MaxRAMPercentage`, `GOMEMLIMIT`, `--max-old-space-size`.
- Dashboards or alerts for throttling, OOM kills, pending Pods and request-to-usage ratio.
- VPA in `Off` mode (or Goldilocks) running as a second opinion.
- Requests fixed before HPA targets are tuned.
- Numbers reviewed regularly, and adjusted in place where the cluster supports it.

[^1]: [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), Kubernetes documentation.
[^2]: [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/), Kubernetes documentation.
[^3]: [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/), Kubernetes documentation.
[^4]: [cAdvisor Prometheus metrics](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md), cAdvisor documentation.
[^5]: [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/), Kubernetes documentation.
[^6]: [Container-aware GOMAXPROCS](https://go.dev/blog/container-aware-gomaxprocs), The Go Blog.
[^7]: [uber-go/automaxprocs](https://github.com/uber-go/automaxprocs), GitHub.
[^8]: [A Guide to the Go Garbage Collector](https://go.dev/doc/gc-guide), Go documentation (section on the memory limit).
[^9]: [Vertical Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/), Kubernetes documentation.
[^10]: [FairwindsOps/goldilocks](https://github.com/FairwindsOps/goldilocks), GitHub.
[^11]: [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/), Kubernetes documentation.
[^12]: [Kubernetes 1.35: In-Place Pod Resize Graduates to Stable](https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga/), Kubernetes blog.
