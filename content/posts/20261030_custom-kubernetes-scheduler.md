---
title: "Writing a Custom Kubernetes Scheduler: Why and How"
date: 2026-10-30
draft: true
description: "When the default kube-scheduler stops being enough, what to try before writing code, and how to build, deploy and test your own scheduler plugin in Go."
summary: "Most clusters never need a custom scheduler, but some do. How kube-scheduler makes decisions, when to extend it, and a complete walkthrough of building and running your own scheduler plugin."
tags: ["kubernetes", "scheduler", "go", "platform engineering"]
categories: ["Programming"]
---

Every Pod in Kubernetes waits in line until a scheduler picks a node for it. Most of the time the default scheduler, `kube-scheduler`, does that well enough that nobody thinks about it. But it optimises for a general-purpose cluster: spread the load, respect the constraints, don't overcommit. If your cluster has a more specific goal, such as packing nodes tightly to save money, starting all workers of a training job at once, or placing work next to its data, you may want the scheduler to think differently.

This post covers what the scheduler actually does, the real reasons to customise it, the options from cheapest to most involved, and a complete example of building, deploying and testing your own scheduler.

Everything here is based on Kubernetes v1.37, the current release at the time of writing.

## What a scheduler actually does

A scheduler has one job: find Pods that have no node assigned, choose a node for each one, and record the choice by creating a **Binding**. That's it. The kubelet on the chosen node notices the Pod and starts it.

Every Pod has a `spec.schedulerName` field. If you don't set it, the API server fills in `default-scheduler`. Each scheduler only handles Pods carrying its own name, which is what makes it possible to run several schedulers side by side.

To see how little is fundamentally required, here is a complete (and terrible) scheduler. It finds unscheduled Pods that asked for `toy-scheduler` and binds each one to a random node:

```go
package main

import (
	"context"
	"log"
	"math/rand/v2"
	"time"

	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)

func main() {
	cfg, err := rest.InClusterConfig()
	if err != nil {
		log.Fatal(err)
	}
	cs := kubernetes.NewForConfigOrDie(cfg)
	ctx := context.Background()

	for {
		pods, err := cs.CoreV1().Pods("").List(ctx, metav1.ListOptions{
			// Pods that asked for us and have no node yet.
			FieldSelector: "spec.schedulerName=toy-scheduler,spec.nodeName=",
		})
		if err != nil {
			log.Println(err)
			time.Sleep(5 * time.Second)
			continue
		}
		nodes, err := cs.CoreV1().Nodes().List(ctx, metav1.ListOptions{})
		if err != nil || len(nodes.Items) == 0 {
			time.Sleep(5 * time.Second)
			continue
		}

		for _, p := range pods.Items {
			node := nodes.Items[rand.IntN(len(nodes.Items))]
			binding := &v1.Binding{
				ObjectMeta: metav1.ObjectMeta{Name: p.Name, Namespace: p.Namespace},
				Target:     v1.ObjectReference{Kind: "Node", Name: node.Name},
			}
			if err := cs.CoreV1().Pods(p.Namespace).Bind(ctx, binding, metav1.CreateOptions{}); err != nil {
				log.Printf("binding %s/%s: %v", p.Namespace, p.Name, err)
				continue
			}
			log.Printf("bound %s/%s to %s", p.Namespace, p.Name, node.Name)
		}
		time.Sleep(2 * time.Second)
	}
}
```

It works, in the sense that Pods start. But look at what it ignores: CPU and memory requests, taints, node selectors and affinity, volumes that only exist in one zone, ports, Pod priority and preemption, and nodes that are cordoned or not ready. It also polls the API server instead of watching, and has no retry queue for Pods that can't be placed.

All of that is what `kube-scheduler` gives you. That is the main lesson of this post: **almost nobody should write a scheduler from scratch. You extend the existing one.**

## How kube-scheduler makes decisions

Since v1.19, `kube-scheduler` is built on the **scheduling framework**: a fixed pipeline of extension points, with every piece of scheduling logic implemented as a plugin that hooks into one or more of them.[^1] Even the built-in behaviour, such as resource fitting, node affinity and taints, is just a set of default plugins.[^2]

For each Pod, scheduling runs in two phases:

- The **scheduling cycle** picks a node. It runs one Pod at a time, so decisions never conflict with each other.
- The **binding cycle** writes the decision to the API server. It runs asynchronously, so a slow bind doesn't block the next Pod.

The extension points you'll use most:

| Extension point | What it does | Typical use |
| --- | --- | --- |
| `queueSort` | Orders the pending-Pod queue. Only one plugin allowed. | Priority, or keeping a group of Pods together |
| `preFilter` | Pre-computes data, or rejects the Pod outright | Check a quota before looking at any node |
| `filter` | Removes nodes that can't run the Pod | "Not enough GPU memory", "wrong zone" |
| `postFilter` | Runs only if no node passed the filters | Preemption: evict lower-priority Pods |
| `preScore` / `score` | Gives each remaining node a score from 0 to 100 | Bin packing, cost, data locality |
| `reserve` | Tells stateful plugins a node was chosen (with `Unreserve` to roll back) | Track resources the API doesn't know about |
| `permit` | Approves, denies or delays the binding | Wait until all members of a group are placed |
| `preBind` / `bind` / `postBind` | Do work before, during and after binding | Provision a volume, then bind |

Final placement is the node with the highest **weighted** sum of scores, where each score plugin has a configurable weight. Newer releases add extension points for scheduling whole PodGroups at once (`placementGenerate`, `placementScore`, `podGroupPostFilter`), which matter if you work on the gang scheduling features covered below.

Two built-in features matter a lot when you customise:

- **Profiles.** One `kube-scheduler` process can run several profiles. Each has its own `schedulerName` and its own set of plugins and weights, but they share one queue and one view of the cluster.
- **The configuration file.** You change plugins and weights with a `KubeSchedulerConfiguration` file (API version `kubescheduler.config.k8s.io/v1`) passed with `--config`. No code required.

## Why you might want your own scheduler

These are the situations where teams usually outgrow the defaults.

**Cost and bin packing.** By default the `NodeResourcesFit` plugin uses the `LeastAllocated` strategy, which spreads Pods across nodes. That is good for resilience but bad for cost, because half-empty nodes never become empty enough for the autoscaler to remove them. A cost-focused cluster wants the opposite: fill nodes up, prefer cheaper instance types, prefer spot capacity.

**Batch, AI and HPC jobs that need gang scheduling.** A distributed training job with 8 workers is useless with 7. The default scheduler places Pods one at a time, so two big jobs can each grab half the GPUs and deadlock, both waiting forever for the rest. These jobs need all-or-nothing placement, often with network topology in mind (all workers in the same rack or switch domain).

**Real utilisation instead of requests.** The default scheduler only looks at what Pods *request*, not what they actually use. If requests are badly sized, nodes can look full while sitting at 20% CPU, or look empty while running hot. Load-aware scheduling uses live metrics.

**Data and hardware locality.** You may want Pods near a cache, a dataset, a licence server or a particular accelerator type, using information that doesn't fit neatly into node labels.

**Fairness and quotas between teams.** Who gets the GPUs when demand exceeds supply? Plain `ResourceQuota` objects cap usage but don't queue, borrow or share.

**Business rules.** Licence limits per node, compliance zones, maintenance windows, "never put these two customers on the same host".

**Research and learning.** Writing a scheduler plugin is one of the best ways to understand how Kubernetes works internally.

## Check what already exists first

Before writing code, check whether the problem is already solved. Kubernetes has grown many scheduling features in the last few releases, and the ecosystem fills most of the remaining gaps.

| You want | Try first |
| --- | --- |
| Pin Pods to certain nodes | `nodeSelector`, node affinity, taints and tolerations |
| Spread across zones or nodes | Pod topology spread constraints |
| Pack nodes tightly | `NodeResourcesFit` with the `MostAllocated` strategy (a config change, shown below) |
| Different rules for different workloads | Multiple profiles in one `kube-scheduler` |
| Gang scheduling | The native Workload / PodGroup API (beta in v1.37, but off by default behind the `GenericWorkload` feature gate), or the Coscheduling plugin from `scheduler-plugins` |
| Job queueing, quotas and fair sharing | [Kueue](https://kueue.sigs.k8s.io/), which admits jobs and works with the default scheduler |
| Load-aware scheduling | The Trimaran plugins in [`scheduler-plugins`](https://github.com/kubernetes-sigs/scheduler-plugins) |
| GPUs and other devices | Dynamic Resource Allocation (DRA) |
| A full batch/HPC scheduler | [Volcano](https://volcano.sh/) or [Apache YuniKorn](https://yunikorn.apache.org/) |

The gang scheduling row deserves a note. Kubernetes v1.35 introduced a `Workload` API in alpha that lets you describe a group of Pods and give it a `gang` policy with a minimum count; the scheduler then holds the Pods until the whole group can be placed.[^3] In v1.37 the Workload and PodGroup APIs moved to beta (`scheduling.k8s.io/v1beta1`), along with workload-aware preemption.[^4] They are still disabled by default, so check whether your provider lets you turn them on. Many of the custom gang schedulers people built over the years can eventually be replaced by this.

## Your options, from least to most effort

1. **Reconfigure kube-scheduler** with a `KubeSchedulerConfiguration`: change weights, swap strategies, disable plugins, add profiles. No code.
2. **Use an existing plugin** from [`kubernetes-sigs/scheduler-plugins`](https://github.com/kubernetes-sigs/scheduler-plugins). The project ships a ready-built `kube-scheduler` image that includes extra plugins such as Coscheduling, Capacity Scheduling, Node Resource Topology and Trimaran.[^5]
3. **Write your own plugin** in Go and compile it into a custom `kube-scheduler` binary. This is the recommended way to add new logic.
4. **Write a scheduler extender**: an HTTP webhook that the scheduler calls during filtering, scoring, preemption or binding. It works with any language and doesn't require a rebuild, but every call is an HTTP round trip, it only reaches a few extension points, and it can't see the scheduler's internal state. The plugin framework was created largely to replace this approach.
5. **Adopt or write a different scheduler** entirely. Volcano and YuniKorn are full alternatives. Writing your own from scratch is only reasonable for research.

The rest of the post walks through options 1 and 3.

## Option 1: a bin-packing profile, no code required

This configuration keeps the normal behaviour for ordinary Pods and adds a second profile, `bin-packing-scheduler`, that fills nodes before using new ones:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: bin-packing-scheduler
    pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: MostAllocated
            resources:
              - name: cpu
                weight: 1
              - name: memory
                weight: 1
```

A Pod opts in with one line:

```yaml
spec:
  schedulerName: bin-packing-scheduler
```

Because both profiles live in the same process, they share one cache and never make conflicting decisions. This is the cleanest form of customisation, and it's worth trying before anything else.

The catch: you need to control the `kube-scheduler` flags. On self-managed clusters (kubeadm, k3s, bare metal) you edit the static Pod manifest. On managed services such as EKS, GKE and AKS, the control plane belongs to the provider, so you usually can't. In that case you run the same configuration as a **second scheduler** in your own Deployment, which is exactly how the plugin example below is deployed.

## Option 3: writing a scheduler plugin

The example plugin, `NodeCost`, prefers cheaper nodes. Each node carries a label with its hourly price, for example from your autoscaler or a small controller that reads your cloud provider's pricing:

```bash
kubectl label node worker-1 example.com/hourly-cost-cents=42
kubectl label node worker-2 example.com/hourly-cost-cents=118
```

The plugin implements two extension points:

- **Filter** rejects nodes labelled `example.com/draining=true`, for nodes you're about to retire. (You could do this with taints too; it's here to show the shape of a filter.)
- **Score** reads the price, and **NormalizeScore** turns prices into scores so that the cheapest node gets 100 and the most expensive gets 0. Nodes without a price label get 0.

All the default plugins still run, so resource fitting, affinity and taints behave as usual. `NodeCost` only adds an opinion on top.

### Project layout

```text
cost-scheduler/
├── go.mod
├── cmd/scheduler/main.go
├── pkg/nodecost/nodecost.go
├── pkg/nodecost/nodecost_test.go
└── Dockerfile
```

### The plugin

In recent releases (v1.35 onwards) the plugin interfaces live in the published module `k8s.io/kube-scheduler/framework`, usually imported as `fwk`.[^6] Older tutorials import `k8s.io/kubernetes/pkg/scheduler/framework` and use slightly different signatures; this is the most common reason copied example code doesn't compile.

```go
// pkg/nodecost/nodecost.go
package nodecost

import (
	"context"
	"strconv"

	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	fwk "k8s.io/kube-scheduler/framework"
)

const (
	// Name is the name used in the scheduler configuration.
	Name = "NodeCost"

	costLabel     = "example.com/hourly-cost-cents"
	drainingLabel = "example.com/draining"

	// Marks nodes without a valid price label. NormalizeScore
	// gives them the lowest score.
	unknownCost = -1
)

type NodeCost struct {
	handle fwk.Handle
}

// Compile-time checks that we implement the interfaces we think we do.
var (
	_ fwk.FilterPlugin = &NodeCost{}
	_ fwk.ScorePlugin  = &NodeCost{}
	_ fwk.SignPlugin   = &NodeCost{}
)

// New is the factory the scheduler calls once per profile.
func New(_ context.Context, _ runtime.Object, h fwk.Handle) (fwk.Plugin, error) {
	return &NodeCost{handle: h}, nil
}

func (pl *NodeCost) Name() string { return Name }

// Filter runs for every node that passed the earlier filters.
// Returning nil means "this node is fine".
func (pl *NodeCost) Filter(ctx context.Context, state fwk.CycleState, pod *v1.Pod, nodeInfo fwk.NodeInfo) *fwk.Status {
	if nodeInfo.Node().Labels[drainingLabel] == "true" {
		return fwk.NewStatus(fwk.UnschedulableAndUnresolvable, "node is being drained")
	}
	return nil
}

// Score returns the node's raw hourly cost. NormalizeScore turns
// the raw costs into the 0–100 range the framework expects.
func (pl *NodeCost) Score(ctx context.Context, state fwk.CycleState, pod *v1.Pod, nodeInfo fwk.NodeInfo) (int64, *fwk.Status) {
	return costOf(nodeInfo.Node()), nil
}

func (pl *NodeCost) ScoreExtensions() fwk.ScoreExtensions { return pl }

// NormalizeScore maps the cheapest labelled node to 100 and the most
// expensive to 0. Unlabelled nodes always get 0, and are left out of the
// range so they don't squash the differences between labelled nodes.
func (pl *NodeCost) NormalizeScore(ctx context.Context, state fwk.CycleState, pod *v1.Pod, scores fwk.NodeScoreList) *fwk.Status {
	lo, hi := int64(-1), int64(-1)
	for _, s := range scores {
		if s.Score == unknownCost {
			continue
		}
		if lo == -1 || s.Score < lo {
			lo = s.Score
		}
		hi = max(hi, s.Score)
	}
	for i := range scores {
		switch {
		case scores[i].Score == unknownCost:
			scores[i].Score = 0
		case hi == lo:
			scores[i].Score = fwk.MaxNodeScore
		default:
			scores[i].Score = (hi - scores[i].Score) * fwk.MaxNodeScore / (hi - lo)
		}
	}
	return nil
}

// SignPod lets the scheduler batch similar Pods (KEP-5598). Our result
// depends only on the node, never on the Pod, so there is nothing to sign.
func (pl *NodeCost) SignPod(ctx context.Context, pod *v1.Pod) ([]fwk.SignFragment, *fwk.Status) {
	return nil, nil
}

func costOf(n *v1.Node) int64 {
	v, ok := n.Labels[costLabel]
	if !ok {
		return unknownCost
	}
	c, err := strconv.ParseInt(v, 10, 64)
	if err != nil || c < 0 {
		return unknownCost
	}
	return c
}
```

A few things are worth knowing about this code:

- **Statuses carry meaning.** `Unschedulable` tells the scheduler that preemption might help (evicting a lower-priority Pod could free the node). `UnschedulableAndUnresolvable` says preemption is pointless, which is right for a draining node.
- **Score sees one node at a time.** That's why the plugin needs `NormalizeScore`: only there can it see every node's raw score and rescale them. Without normalisation, a raw score above 100 makes the scheduler fail the Pod. Raw scores outside 0–100 (like the `-1` for unlabelled nodes) are fine, as long as `NormalizeScore` brings them back into range.
- **Keep sentinel values out of the arithmetic.** It's tempting to give unlabelled nodes a huge fake price, say 1,000,000, and let them fall to the bottom naturally. But then a single unlabelled node becomes the maximum, and nodes costing 42 and 118 cents score 100 and 99: the outlier squashes every real difference. That's why unlabelled nodes are handled separately.
- **Batching needs `SignPod`.** Recent releases let the scheduler reuse one scheduling decision for a run of identical Pods ("opportunistic batching").[^9] The scheduler does that only if every filter and score plugin in the profile implements `SignPlugin` and describes which parts of the Pod its result depends on. A plugin that doesn't turns batching off for the whole profile, and the scheduler logs "Disabling signatures for profile because plugins do not support it". `NodeCost` never looks at the Pod, so returning no fragments is correct.
- **Keep Filter and Score fast.** They run for every candidate Pod and node pair, potentially thousands of times a second. Never call the API server from them. Read cluster state through the handle's informers (`h.SharedInformerFactory()`), and compute anything expensive once per Pod in `PreFilter` or `PreScore`, storing it in `CycleState` for later stages to read.
- **Requeueing.** When a Pod is rejected, the scheduler parks it until something in the cluster changes. A plugin can implement `EnqueueExtensions` to say which events (a node label changing, say) could make a rejected Pod schedulable. If you don't implement it, the scheduler conservatively retries on all events, which is correct but wasteful in big clusters.

### A unit test

The scoring logic is plain Go, so it's easy to test without a cluster:

```go
// pkg/nodecost/nodecost_test.go
package nodecost

import (
	"context"
	"testing"

	fwk "k8s.io/kube-scheduler/framework"
)

func TestNormalizeScore(t *testing.T) {
	pl := &NodeCost{}
	scores := fwk.NodeScoreList{
		{Name: "cheap", Score: 40},
		{Name: "mid", Score: 70},
		{Name: "pricey", Score: 100},
	}
	if st := pl.NormalizeScore(context.Background(), nil, nil, scores); st != nil {
		t.Fatalf("unexpected status: %v", st)
	}
	want := map[string]int64{"cheap": 100, "mid": 50, "pricey": 0}
	for _, s := range scores {
		if s.Score != want[s.Name] {
			t.Errorf("%s: got %d, want %d", s.Name, s.Score, want[s.Name])
		}
	}
}
```

### Building the scheduler binary

You don't write a scheduler `main` yourself. You reuse the real `kube-scheduler` command and register your plugin in its registry:

```go
// cmd/scheduler/main.go
package main

import (
	"os"

	"k8s.io/component-base/cli"
	"k8s.io/kubernetes/cmd/kube-scheduler/app"

	"example.com/cost-scheduler/pkg/nodecost"
)

func main() {
	cmd := app.NewSchedulerCommand(
		app.WithPlugin(nodecost.Name, nodecost.New),
	)
	os.Exit(cli.Run(cmd))
}
```

The result is a full `kube-scheduler` with every default plugin plus `NodeCost`, and the same flags, metrics and health endpoints.

**The `go.mod` trap.** Importing `k8s.io/kubernetes` is awkward because it isn't designed to be used as a library. Its `go.mod` refers to the `k8s.io/*` staging modules (`k8s.io/api`, `k8s.io/client-go`, `k8s.io/kube-scheduler` and two dozen others) at version `v0.0.0`, which doesn't exist. You need a `replace` directive for every one of them, pinned to the matching release: Kubernetes v1.37.x pairs with `v0.37.x` of the staging modules.

```text
module example.com/cost-scheduler

go 1.26

require k8s.io/kubernetes v1.37.x

replace (
	k8s.io/api => k8s.io/api v0.37.x
	k8s.io/apimachinery => k8s.io/apimachinery v0.37.x
	k8s.io/client-go => k8s.io/client-go v0.37.x
	k8s.io/kube-scheduler => k8s.io/kube-scheduler v0.37.x
	// ...and so on for every k8s.io staging module
)
```

Replace `x` with the patch release you target. The easiest way to get the full list right is to copy the `replace` block from the `go.mod` of `kubernetes-sigs/scheduler-plugins` at the matching release, which has to solve the same problem.

Then a two-stage Docker build:

```dockerfile
# Use the Go version from the go.mod of the Kubernetes release you build against.
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /kube-scheduler ./cmd/scheduler

FROM gcr.io/distroless/static:nonroot
COPY --from=build /kube-scheduler /usr/local/bin/kube-scheduler
ENTRYPOINT ["/usr/local/bin/kube-scheduler"]
```

### Configuring the plugin

The configuration creates a profile named `cost-scheduler` and enables `NodeCost` through `multiPoint`, which turns it on at every extension point it implements:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: true
  resourceNamespace: kube-system
  resourceName: cost-scheduler
profiles:
  - schedulerName: cost-scheduler
    plugins:
      multiPoint:
        enabled:
          - name: NodeCost
            weight: 5
```

The weight controls how much cost matters compared with the default scoring plugins, which mostly have a weight of 1 or 2. At 5, cost dominates but doesn't completely override things like topology spreading. Tune it to taste.

Leader election lets you run two replicas safely: only the leader schedules, and the other takes over if it dies. The lock name must differ from the default scheduler's `kube-scheduler`.

### Deploying it as a second scheduler

The scheduler needs the same permissions as `kube-scheduler`. The simplest way is to bind your ServiceAccount to the built-in roles, as the official multiple schedulers guide does:[^7]

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cost-scheduler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cost-scheduler-as-kube-scheduler
subjects:
  - kind: ServiceAccount
    name: cost-scheduler
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:kube-scheduler
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cost-scheduler-as-volume-scheduler
subjects:
  - kind: ServiceAccount
    name: cost-scheduler
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:volume-scheduler
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cost-scheduler-extension-apiserver-authentication-reader
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: cost-scheduler
    namespace: kube-system
roleRef:
  kind: Role
  name: extension-apiserver-authentication-reader
  apiGroup: rbac.authorization.k8s.io
---
# system:kube-scheduler may create leases, but may only update the ones
# named kube-scheduler. This grants access to our own lock.
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cost-scheduler-leader-election
  namespace: kube-system
rules:
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    resourceNames: ["cost-scheduler"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cost-scheduler-leader-election
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: cost-scheduler
    namespace: kube-system
roleRef:
  kind: Role
  name: cost-scheduler-leader-election
  apiGroup: rbac.authorization.k8s.io
```

The official guide instead edits the `system:kube-scheduler` ClusterRole to add the new lock name.[^7] A separate Role does the same job without touching a built-in object that the API server may reconcile.

Then the ConfigMap and Deployment:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cost-scheduler-config
  namespace: kube-system
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    leaderElection:
      leaderElect: true
      resourceNamespace: kube-system
      resourceName: cost-scheduler
    profiles:
      - schedulerName: cost-scheduler
        plugins:
          multiPoint:
            enabled:
              - name: NodeCost
                weight: 5
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cost-scheduler
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cost-scheduler
  template:
    metadata:
      labels:
        app: cost-scheduler
    spec:
      serviceAccountName: cost-scheduler
      priorityClassName: system-cluster-critical
      containers:
        - name: scheduler
          image: registry.example.com/cost-scheduler:v1
          args:
            - --config=/etc/cost-scheduler/config.yaml
          livenessProbe:
            httpGet:
              path: /healthz
              port: 10259
              scheme: HTTPS
            initialDelaySeconds: 15
          readinessProbe:
            httpGet:
              path: /healthz
              port: 10259
              scheme: HTTPS
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
          volumeMounts:
            - name: config
              mountPath: /etc/cost-scheduler
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: cost-scheduler-config
```

Note the irony: the scheduler's own Pods are placed by the default scheduler. Your scheduler can't schedule itself.

### Using and verifying it

Pods opt in by name:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-worker
spec:
  replicas: 4
  selector:
    matchLabels:
      app: batch-worker
  template:
    metadata:
      labels:
        app: batch-worker
    spec:
      schedulerName: cost-scheduler
      containers:
        - name: worker
          image: busybox
          command: ["sleep", "3600"]
          resources:
            requests:
              cpu: 100m
```

To check which scheduler placed a Pod, look at its events. The `Scheduled` event is reported by the scheduler that bound it:

```bash
kubectl get pods -l app=batch-worker -o wide
kubectl get events --field-selector reason=Scheduled -o custom-columns=POD:.involvedObject.name,BY:.reportingComponent,MESSAGE:.message
```

If the Pods sit in `Pending` with no events at all, the scheduler isn't picking them up: check that the Deployment is running, that the `schedulerName` matches the profile exactly, and the scheduler logs (`kubectl logs -n kube-system deploy/cost-scheduler`). If you see `FailedScheduling` events, the message lists how many nodes each filter rejected, including yours ("node is being drained").

### Testing beyond unit tests

- **kind.** A local [kind](https://kind.sigs.k8s.io/) cluster with a few worker nodes is enough to test labels, filters and scores end to end. Load your image with `kind load docker-image`.
- **The scheduler simulator.** [`kube-scheduler-simulator`](https://github.com/kubernetes-sigs/kube-scheduler-simulator) runs the scheduler against a fake cluster and shows each plugin's filter result and score for every node.[^8] It's the fastest way to understand why a Pod landed where it did.
- **Load.** Before production, test with as many nodes and pending Pods as your real cluster. A plugin that adds a millisecond per node is invisible in kind and painful at 5,000 nodes.

## What can go wrong

**Two schedulers, one cluster.** A second scheduler has its own cache and doesn't know what the default scheduler is deciding at the same moment. Both can pick the last free slot on the same node; the kubelet then rejects one Pod (with a status such as `OutOfcpu`) and a controller has to recreate it. It's rare at low churn and common at high churn. The fixes, in order of preference: run your plugin as a profile inside the *only* scheduler, or give each scheduler its own node pool with taints and node selectors so they never compete.

**The autoscaler doesn't know your rules.** The Cluster Autoscaler decides whether a new node would help by simulating the standard scheduling rules. It doesn't run your plugin. If your filter rejects nodes for reasons the autoscaler can't see, it may add nodes your scheduler then refuses to use, or never add the node you need. Express hard constraints as labels, taints or affinity where possible, so the autoscaler can see them too.

**Your scheduler is now critical infrastructure.** If it crashes, its Pods stay `Pending` forever. There is no fallback to the default scheduler. Run two replicas with leader election, give it a high priority class, monitor the `scheduler_pending_pods` and `scheduler_schedule_attempts_total` metrics, and alert on Pods pending too long.

**Upgrades are on you.** Your binary *is* `kube-scheduler`, compiled at a specific version. Every Kubernetes minor upgrade means bumping dependencies, fixing any interface changes, rebuilding and re-testing. The scheduler supports a skew of one minor version behind the API server at most, so don't let it fall behind. Budget this work before you start.

**Managed clusters limit you.** On most managed services you can't change the default scheduler, so you're running a second one, with the race described above.

## Bottom line

- **Most clusters don't need a custom scheduler.** Affinity, taints, topology spread and scheduler profiles cover a lot.
- **Check the ecosystem next.** Native gang scheduling is maturing fast, Kueue handles queueing and quotas, and `scheduler-plugins` covers co-scheduling, capacity sharing and load-aware placement.
- **If you need new logic, write a plugin, not a scheduler.** You keep every built-in behaviour and add only your opinion, usually as a Filter or Score plugin in under 200 lines of Go.
- **Prefer profiles in one scheduler over a second scheduler** whenever you control the control plane.
- **Plan for operations**: high availability, metrics, autoscaler interaction and an upgrade for every Kubernetes release.

[^1]: [KEP-624: Scheduling Framework](https://www.kubernetes.dev/resources/keps/624/), the design of the plugin framework.
[^2]: [Scheduler Configuration](https://kubernetes.io/docs/reference/scheduling/config/), Kubernetes documentation: extension points, default plugins, profiles and `multiPoint`.
[^3]: [Kubernetes v1.35: Introducing Workload Aware Scheduling](https://kubernetes.io/blog/2025/12/29/kubernetes-v1-35-introducing-workload-aware-scheduling/), Kubernetes blog.
[^4]: [Kubernetes v1.37: Advancing Workload-Aware Scheduling](https://kubernetes.io/blog/2026/09/08/kubernetes-v1-37-advancing-workload-aware-scheduling/), Kubernetes blog.
[^5]: [kubernetes-sigs/scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins), out-of-tree plugins and ready-built images.
[^6]: [kubernetes/kube-scheduler](https://github.com/kubernetes/kube-scheduler), the published module containing the configuration types and plugin framework interfaces.
[^7]: [Configure Multiple Schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/), Kubernetes documentation.
[^8]: [kube-scheduler-simulator](https://github.com/kubernetes-sigs/kube-scheduler-simulator), for visualising plugin decisions.
[^9]: [KEP-5598: Opportunistic Batching](https://github.com/kubernetes/enhancements/issues/5598), Kubernetes enhancement tracking issue.
