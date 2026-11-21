---
title: "An Introduction to CI/CD with Tekton"
date: 2026-11-07
draft: true
description: "What Tekton is, how its Tasks, Pipelines and runs map onto Kubernetes Pods and containers, and how to build, run and trigger a first CI pipeline on a local cluster."
summary: "Tekton turns CI/CD into Kubernetes objects: every step is a container, every task is a Pod, and every pipeline is YAML you can apply with kubectl. Here is how it works and how to build your first pipeline."
tags: ["kubernetes", "tekton", "ci/cd", "devops", "platform engineering"]
categories: ["Programming"]
---

Most CI systems are a server with a queue of jobs and a fleet of workers. Jenkins, GitLab runners and GitHub Actions all work roughly this way. **Tekton** takes a different approach: there is no CI server. Your pipelines are Kubernetes objects, each step runs in its own container, and the Kubernetes control plane does the scheduling. Tekton describes itself as providing Kubernetes-style resources for declaring CI/CD pipelines, built to run on Kubernetes and to use containers as their building blocks[^1].

This post explains what that means in practice: the handful of concepts you need, what actually happens in the cluster when a pipeline runs, and how to build, run and trigger a small CI pipeline on a local cluster.

Everything here uses the `tekton.dev/v1` API. Tekton Pipelines reached 1.0 in May 2025[^2], and the most recent long-term-support line I could confirm at the time of writing is v1.15, released at the end of July 2026 and supported until July 2027[^3].

## CI/CD in one paragraph

**Continuous integration (CI)** means every change is built and tested automatically, so problems show up minutes after a commit instead of weeks later. **Continuous delivery/deployment (CD)** extends that: a change that passes CI is packaged and released (delivery) or pushed all the way to production (deployment) by the same automated process. A CI/CD system is the engine that runs those steps: check out code, build, test, scan, package, publish, deploy. Tekton is such an engine, built as an extension of Kubernetes.

## Why Tekton?

- **Kubernetes-native.** Pipelines are Custom Resources. You manage them with `kubectl`, store them in Git, review them like any other manifest, and secure them with the same RBAC, namespaces and quotas as the rest of your cluster.
- **Everything is a container.** Each step names an image. Need Go 1.25, Node 22 and `helm` in one pipeline? Use three images. No plugins to install on a shared agent, no "works on the CI box" drift.
- **No central server.** Scaling is the cluster's job. Idle pipelines cost nothing but a few controller Pods.
- **Reusable and vendor-neutral.** Tasks are written once and shared through catalogs. Tekton is a CD Foundation project and sits underneath several commercial products, such as Red Hat OpenShift Pipelines, which is its direct downstream[^4].

The trade-off: Tekton is a set of building blocks, not a finished product. There is no built-in UI for pull-request status, no secrets wizard, no YAML editor with autocomplete. You assemble those from other Tekton components or other tools.

## The building blocks

Tekton Pipelines has a small vocabulary. Learn these and the rest is detail.

```text
        Pipeline  ──────────── run by ────────────►  PipelineRun
  ┌──────────────────────┐                     (one execution, with
  │  Task A ──► Task B   │                      real params and storage)
  │     └────► Task C    │                             │
  └──────────────────────┘                             │ creates
          │                                            ▼
          │ each Task is                        TaskRun per Task
          ▼                                            │
        Task  ─────────────── run by ─────────►  TaskRun ──► one Pod
  ┌──────────────────────┐                             │
  │ step 1 → step 2 → …  │                             ▼
  └──────────────────────┘                   one container per step
```

| Object | What it is |
| --- | --- |
| **Step** | One container that runs one command or script. The smallest unit. |
| **Task** | An ordered list of steps that run in sequence. A reusable template, like a function definition. |
| **TaskRun** | One execution of a Task with concrete inputs. Creates exactly one Pod. |
| **Pipeline** | A graph of Tasks. Tasks run in parallel unless you say otherwise. |
| **PipelineRun** | One execution of a Pipeline. Creates a TaskRun for each Task. |

Three more ideas connect them:

- **Params** are string (or array/object) inputs, like function arguments. They are referenced as `$(params.name)`.
- **Workspaces** are shared storage, usually a PersistentVolumeClaim. A Task declares that it *needs* a workspace; the run decides *which* volume backs it. This is how source code cloned by one Task reaches the next.
- **Results** are small string outputs (a commit SHA, an image digest) that one Task writes and later Tasks read as `$(tasks.<name>.results.<result>)`.

The split between *definition* (Task, Pipeline) and *execution* (TaskRun, PipelineRun) is the key design idea. Definitions are generic and shareable; runs supply the specifics: which repository, which branch, which volume, which service account.

## What happens when a pipeline runs

When you create a PipelineRun, the Tekton controller:

1. **Resolves** the Pipeline and its Tasks. They can live in the cluster, or be fetched by a *remote resolver* from Git, an OCI bundle, a cluster-scoped catalog or Artifact Hub[^5].
2. **Builds the graph.** Tasks with no dependencies start immediately and in parallel. A Task waits if it has `runAfter`, or if it consumes another Task's result.
3. **Creates a TaskRun per Task**, and each TaskRun creates one Pod.
4. **Runs the steps.** All steps of a Task are containers in the same Pod, so they share the Pod's volumes and node. Tekton injects a small entrypoint binary into each container that waits for the previous step to finish before starting its own command. That is how containers in one Pod run in order instead of all at once.
5. **Records status** on the TaskRun and PipelineRun objects: start and end times, results, success or failure per step.

Two consequences are worth remembering. First, **a Task is the unit of scheduling**: steps in a Task always land on the same node and share local files cheaply, while separate Tasks need a workspace to pass files. Second, **logs live in Pods**. Once the Pods are deleted, the logs are gone unless you ship them elsewhere (see Tekton Results later).

## Installing Tekton

You need a Kubernetes cluster and `kubectl`. For experimenting, a local [kind](https://kind.sigs.k8s.io/) or minikube cluster is enough:

```bash
kind create cluster --name tekton
```

Install Tekton Pipelines from the latest release manifest[^6]:

```bash
kubectl apply --filename \
  https://infra.tekton.dev/tekton-releases/pipeline/latest/release.yaml
```

This creates the `tekton-pipelines` namespace with the controller and webhook, plus the `tekton-pipelines-resolvers` namespace for remote resolution. Wait for everything to be ready:

```bash
kubectl get pods --namespace tekton-pipelines --watch
```

When `tekton-pipelines-controller` and `tekton-pipelines-webhook` show `1/1`, you're ready. Older tutorials use a `storage.googleapis.com/tekton-releases/...` URL; recent releases are published under `infra.tekton.dev`[^7]. For production clusters, the Tekton docs recommend the Tekton Operator rather than raw manifests, since it handles upgrades and configuration for you[^6].

Next, install the `tkn` CLI. On macOS it's `brew install tektoncd-cli`; for other platforms grab a binary from the [tektoncd/cli releases](https://github.com/tektoncd/cli/releases). `kubectl` alone is enough to run everything, but `tkn` makes starting runs and following logs much nicer.

Optionally, add the Dashboard, a web UI for browsing runs and logs. It installs read-only by default[^8]:

```bash
kubectl apply --filename \
  https://infra.tekton.dev/tekton-releases/dashboard/latest/release.yaml
kubectl port-forward -n tekton-pipelines svc/tekton-dashboard 9097:9097
```

Then open <http://localhost:9097>.

## Your first Task

Save this as `hello-task.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: hello
spec:
  params:
    - name: name
      type: string
      default: world
  steps:
    - name: greet
      image: alpine:3.20
      script: |
        #!/bin/sh
        echo "Hello, $(params.name)!"
```

Apply it and run it:

```bash
kubectl apply -f hello-task.yaml
tkn task start hello -p name=Tekton --showlog
```

```text
TaskRun started: hello-run-x7k2p
Waiting for logs to be available...
[greet] Hello, Tekton!
```

`tkn` created a TaskRun for you. You can do the same with plain YAML, which is what automation will do:

```yaml
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  generateName: hello-run-
spec:
  taskRef:
    name: hello
  params:
    - name: name
      value: Kubernetes
```

Note `generateName`: every run needs a unique name, so create runs with `kubectl create -f`, not `kubectl apply`.

Look at what happened in the cluster:

```bash
kubectl get taskruns
kubectl get pods
```

You'll see one Pod per TaskRun, in `Completed` state, with one container per step. That's the whole model.

## A real CI pipeline

Now something closer to real CI: clone a Go repository, then run `go test` and `go vet` **in parallel**, and finally print a summary whether the checks passed or not.

### A reusable Go Task

First, a small Task that runs any `go` subcommand in a source workspace. Save as `go-task.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: go
spec:
  description: Run a go command in the source workspace.
  params:
    - name: args
      type: string
      description: Arguments to pass to the go command
    - name: image
      type: string
      default: golang:1.25
  workspaces:
    - name: source
      description: The Go module to work on
  steps:
    - name: run
      image: $(params.image)
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/sh
        set -eu
        go version
        go $(params.args)
```

Because the arguments are a parameter, the same Task serves for testing, vetting or building. That's the reuse model: write small generic Tasks, compose them in Pipelines.

### The Pipeline

For cloning, there's no need to write anything. The community `git-clone` Task from the Tekton catalog does it, and the **hub resolver** fetches it from Artifact Hub at run time[^5]. Save as `go-ci-pipeline.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: go-ci
spec:
  params:
    - name: repo-url
      type: string
    - name: revision
      type: string
      default: main
  workspaces:
    - name: source
  tasks:
    - name: fetch
      taskRef:
        resolver: hub
        params:
          - name: name
            value: git-clone
          - name: version
            value: "0.9"
      params:
        - name: url
          value: $(params.repo-url)
        - name: revision
          value: $(params.revision)
      workspaces:
        - name: output
          workspace: source

    - name: test
      runAfter: [fetch]
      taskRef:
        name: go
      params:
        - name: args
          value: test ./...
      workspaces:
        - name: source
          workspace: source

    - name: vet
      runAfter: [fetch]
      taskRef:
        name: go
      params:
        - name: args
          value: vet ./...
      workspaces:
        - name: source
          workspace: source

  finally:
    - name: report
      params:
        - name: commit
          value: $(tasks.fetch.results.commit)
        - name: status
          value: $(tasks.status)
      taskSpec:
        params:
          - name: commit
          - name: status
        steps:
          - name: print
            image: alpine:3.20
            script: |
              #!/bin/sh
              echo "Commit $(params.commit) finished with status: $(params.status)"
```

A few things to notice:

- **The workspace is declared, not provided.** The Pipeline says it needs a workspace called `source` and wires it into each Task (`git-clone` calls its workspace `output`; our Task calls it `source`). Which volume backs it is decided by the run.
- **`runAfter` creates the graph.** `test` and `vet` both wait for `fetch`, but not for each other, so they run at the same time.
- **`finally` tasks always run**, after everything else, whether the pipeline succeeded or failed. They're the place for notifications and cleanup. `$(tasks.status)` gives the aggregate status (`Succeeded`, `Failed`, `Completed` or `None`). One catch: if a `finally` Task reads a result from a Task that never produced it (here, if the clone fails), that `finally` Task is skipped.
- **`taskSpec` embeds a Task inline**, handy for one-off glue that isn't worth a separate object.

### Running it

The PipelineRun supplies the specifics. `volumeClaimTemplate` asks Tekton to create a fresh PVC for this run and delete it along with the run. Save as `go-ci-run.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: go-ci-run-
spec:
  pipelineRef:
    name: go-ci
  params:
    - name: repo-url
      value: https://github.com/google/go-cmp
    - name: revision
      value: master
  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 1Gi
```

```bash
kubectl apply -f go-task.yaml -f go-ci-pipeline.yaml
kubectl create -f go-ci-run.yaml
tkn pipelinerun logs --last -f
```

You'll see the clone, then the interleaved output of `test` and `vet` running side by side, then the report. `tkn pipelinerun describe --last` shows each Task's status and duration.

Because the three main Tasks share a `ReadWriteOnce` volume, Tekton's *affinity assistant* makes sure their Pods are scheduled on the same node, since a node-local volume can't be mounted on two nodes at once. On a real multi-node cluster, keep this in mind when you wonder why parallel Tasks aren't spreading out.

### From CI to CD

This pipeline only checks code. To deliver something, you would add Tasks to build and push a container image (the catalog has Tasks for tools such as Buildah and ko) and then deploy it, for example with a `kubectl` or `helm` Task. The image push needs registry credentials, which Tekton takes from Secrets attached to the run's ServiceAccount.

Many teams stop Tekton at "image built, signed and pushed" and let a GitOps tool such as Argo CD or Flux handle deployment: Tekton's last Task updates the image tag in a Git repository, and the GitOps controller syncs the cluster. That keeps CI (Tekton) and the source of truth for what's deployed (Git) cleanly separated.

## Triggering pipelines from Git

So far, every run was started by hand. Real CI starts a run when someone pushes. That's the job of **Tekton Triggers**, a separate component with three main objects[^9]:

- **EventListener**: a Deployment and Service that receives HTTP webhooks.
- **TriggerBinding**: extracts fields from the event payload (the commit SHA, the clone URL) into parameters.
- **TriggerTemplate**: a template for the resources to create, usually a PipelineRun, filled in with those parameters.

Optional **interceptors** sit in front: the built-in GitHub, GitLab and Bitbucket interceptors validate webhook signatures and filter by event type, and the CEL interceptor can filter on any expression, such as "only pushes to `main`".

Install Triggers and its interceptors:

```bash
kubectl apply --filename \
  https://infra.tekton.dev/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename \
  https://infra.tekton.dev/tekton-releases/triggers/latest/interceptors.yaml
```

Then wire GitHub push events to the pipeline above:

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-push
spec:
  params:
    - name: repo-url
      value: $(body.repository.clone_url)
    - name: revision
      value: $(body.after)
---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: go-ci-template
spec:
  params:
    - name: repo-url
    - name: revision
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: go-ci-run-
      spec:
        pipelineRef:
          name: go-ci
        params:
          - name: repo-url
            value: $(tt.params.repo-url)
          - name: revision
            value: $(tt.params.revision)
        workspaces:
          - name: source
            volumeClaimTemplate:
              spec:
                accessModes: [ReadWriteOnce]
                resources:
                  requests:
                    storage: 1Gi
---
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - name: on-push
      interceptors:
        - ref:
            name: github
          params:
            - name: secretRef
              value:
                secretName: github-webhook-secret
                secretKey: token
            - name: eventTypes
              value: ["push"]
      bindings:
        - ref: github-push
      template:
        ref: go-ci-template
```

The EventListener's ServiceAccount needs RBAC permissions to read Triggers objects and create PipelineRuns; the Triggers getting-started guide has the exact Role and ClusterRole bindings[^9]. Expose the listener's Service (`el-github-listener`) through an Ingress, point a GitHub webhook at it with the same secret, and every push now starts a PipelineRun.

If wiring this up by hand sounds like a lot, it is. That's why **Pipelines-as-Code** exists: it handles the GitHub/GitLab app integration, reads PipelineRun definitions from a `.tekton/` folder in the repository itself, and reports status back to pull requests, much like GitHub Actions workflows but executed by Tekton.

## The wider Tekton ecosystem

Tekton is a family of projects. The ones you'll meet first:

| Component | What it does |
| --- | --- |
| **Pipelines** | The core: Tasks, Pipelines and runs. |
| **Triggers** | Starts runs from webhooks and other events. |
| **CLI (`tkn`)** | Start, list, describe and follow runs. |
| **Dashboard** | Web UI for runs and logs. |
| **Chains** | Watches finished runs, generates provenance (SLSA attestations) for the artifacts they built, and signs them[^2]. |
| **Results** | Stores run history and logs outside the cluster, so you can clean up old runs without losing their records. |
| **Operator** | Installs, configures and upgrades all of the above. |

For finding ready-made Tasks, use **Artifact Hub**. The old Tekton Hub is deprecated, and the hub resolver now defaults to Artifact Hub[^10].

## Tekton vs a traditional CI server

| | Tekton | Jenkins / GitHub Actions style |
| --- | --- | --- |
| Where it runs | Your Kubernetes cluster | A CI server, hosted runners or agents |
| Pipeline definition | Kubernetes Custom Resources | Jenkinsfile, workflow YAML |
| Isolation | Pod per Task, container per step | Job per agent or runner |
| Scaling | Kubernetes scheduler and node autoscaler | Agent pools |
| Ready-made UX | Minimal; assemble Dashboard, Pipelines-as-Code, Results | Rich out of the box |
| Best fit | Platform teams building a shared CI/CD platform on Kubernetes | Teams that want CI working today with little setup |

Tekton shines when you're building a platform for others: you control where builds run, what they can access, and how they're audited, all with Kubernetes primitives. For a single small project, a hosted CI service is usually less work.

## Gotchas

- **Old tutorials are everywhere.** Use `tekton.dev/v1`. If you see `v1beta1` Tasks or `PipelineResource` objects, the material predates the 1.0 API; PipelineResources were removed entirely.
- **Runs pile up.** Every run leaves TaskRun objects and completed Pods behind, and PVCs from `volumeClaimTemplate` live until the run is deleted. Set up pruning (the Operator has a pruner) or you'll fill your namespace and storage.
- **Workspaces cost time.** Each Task that touches a PVC waits for it to attach. Put closely related steps in one Task when you can, and don't split a pipeline into dozens of tiny Tasks.
- **Parallel Tasks may share a node.** The affinity assistant keeps Tasks with a shared `ReadWriteOnce` workspace on one node, which limits parallelism.
- **Results are small.** They're meant for short strings like digests and SHAs, not files. Pass files through workspaces.
- **Keep it patched.** Tekton runs arbitrary code from your repositories with cluster credentials. 2026 alone saw several high-severity fixes in the resolvers, including a Git resolver bug that could leak the configured API token[^11]. Track an LTS line and apply patch releases.

## Bottom line

- **Tekton is CI/CD as Kubernetes objects.** Steps are containers, Tasks are Pods, and Pipelines are graphs of Tasks you manage with `kubectl`.
- **Definitions are separate from runs.** Tasks and Pipelines are reusable templates; TaskRuns and PipelineRuns supply the parameters, storage and credentials.
- **Workspaces move files, results move values, `runAfter` orders Tasks**, and everything else runs in parallel.
- **Triggers (or Pipelines-as-Code) turn webhooks into runs**, and Chains, Results and the Dashboard fill in the supply-chain security, history and UI.
- **It rewards platform teams.** Expect to assemble pieces, and in return get CI/CD that scales, isolates and is governed like the rest of your cluster.

[^1]: [Tekton Pipelines README](https://pkg.go.dev/github.com/tektoncd/pipeline@v1.9.5), tektoncd/pipeline on pkg.go.dev.
[^2]: [Tekton blog](https://tekton.dev/blog): "Tekton Pipelines Reaches 1.0" (May 2025), the v1.9.0 LTS announcement and the Tekton Chains overview.
[^3]: [Tekton Pipeline releases and support windows](https://github.com/tektoncd/pipeline/blob/main/releases.md), tektoncd/pipeline.
[^4]: [Tekton Pipelines 1.0 and OpenShift for CI/CD](https://www.redhat.com/zh-tw/tekton-pipelines-1.0-and-openshift-for-cicd), Red Hat.
[^5]: [Hub Resolver](https://tekton.dev/docs/pipelines/hub-resolver/), Tekton documentation.
[^6]: [Install Tekton Pipelines](https://tekton.dev/docs/pipelines/install/), Tekton documentation.
[^7]: [Tekton Pipeline release v1.14.0](https://github.com/tektoncd/pipeline/releases/tag/v1.14.0), release notes and installation one-liner.
[^8]: [Installing Tekton Dashboard](https://tekton.dev/docs/dashboard/install), Tekton documentation.
[^9]: [Getting Started with Triggers](https://tekton.dev/docs/getting-started/triggers/), Tekton documentation.
[^10]: [Hub Resolver (v1.6.x LTS)](https://tekton.dev/vault/Pipelines-v1.6.x-LTS/hub-resolver/), Tekton documentation, including the Tekton Hub deprecation notice.
[^11]: [Tekton Pipeline release v1.9.3 security fixes](https://newreleases.io/project/github/tektoncd/pipeline/release/v1.9.3), NewReleases summary of the GitHub release notes.
