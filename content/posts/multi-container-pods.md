---
title: "Multi-Container Pods in Kubernetes"
date: 2026-11-17
draft: true
description: "What containers in the same Pod share, when to put more than one container in a Pod, how init, sidecar and ephemeral containers work, the sidecar, ambassador and adapter patterns, and how resources, probes and shutdown behave."
summary: "A Pod is not a container. It is a small group of containers that share a network, storage and a lifecycle. Here is what they share, when grouping them makes sense, how native sidecars changed the picture, and the pitfalls to watch for."
tags: ["kubernetes", "pods", "sidecar", "init containers", "containers", "cka"]
categories: ["Programming"]
---

Most Pods you will ever write have exactly one container, and for good reason: one process, one image, one thing to scale. But the Pod was designed from the start as a *group* of containers, and several of the most useful patterns in Kubernetes, such as log shippers, service mesh proxies, config reloaders and database migrations, only work because a Pod can hold more than one.

This post covers what containers in the same Pod actually share, when it makes sense to put them together (and when it doesn't), the four kinds of containers a Pod can have, the classic sidecar, ambassador and adapter patterns, and how resources, probes and shutdown work once there is more than one container involved.

Everything here is based on Kubernetes v1.37, the current release at the time of writing.

## A Pod is a group, not a container

The Kubernetes documentation describes a Pod as a group of one or more containers with shared storage and network resources and a specification for how to run them[^1]. The containers in a Pod are always scheduled together onto the same node, start and stop as a unit, and are treated as one replica when you scale.

A useful mental model is an old-fashioned server. Before containers, you might run a web application, a log forwarder and a monitoring agent on the same machine. They were separate programs, but they shared `localhost` and the disk. A Pod is that machine, and its containers are those programs, except each one comes with its own image and its own resource limits.

```text
┌────────────────────────── Pod (10.244.1.17) ──────────────────────────┐
│                                                                       │
│   ┌──────────────┐     localhost:8080     ┌──────────────────────┐    │
│   │  app         │ ◄───────────────────── │  sidecar (proxy)     │    │
│   │  :8080       │                        │  :15001              │    │
│   └──────┬───────┘                        └──────────┬───────────┘    │
│          │ writes                                    │ reads          │
│          ▼                                           ▼                │
│   ┌─────────────────────── shared volume: /var/log/app ────────────┐  │
│   └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
│   one IP · one network namespace · one lifecycle · one node           │
└───────────────────────────────────────────────────────────────────────┘
```

## What containers in a Pod share

The details matter, because they decide what you can and can't do between containers.

| Shared | Not shared (by default) |
| --- | --- |
| **Network namespace**: one IP address, one set of ports, one `localhost` | **Filesystem**: each container has its own root filesystem from its own image |
| **IPC namespace**: System V IPC and POSIX shared memory | **Process namespace**: each container sees only its own processes, unless you turn sharing on |
| **UTS namespace**: the same hostname (the Pod name) | **Resource limits**: CPU and memory are set per container |
| **Volumes** declared in the Pod spec, if each container mounts them | **Security context**: user, capabilities and so on can differ per container |
| **Node, lifecycle and fate**: scheduled, evicted and deleted together | **Restarts**: a crashed container is restarted on its own, not the whole Pod |

Under the hood, the kubelet first starts a tiny *pause* container (the "sandbox") that does nothing but hold the network and IPC namespaces open. Every other container in the Pod joins those namespaces. That is why a container can crash and restart without the Pod losing its IP address.

Two consequences follow directly from the table:

- **Two containers in one Pod can't listen on the same port.** They share one network stack, so the second one to bind `:8080` gets `address already in use`.
- **Containers can't see each other's files** unless you give them a shared volume. Writing to `/tmp` in one container does nothing for the other.

## When to use more than one container

The rule of thumb: put containers in the same Pod only when they are **tightly coupled**, meaning they must run on the same machine, share files or `localhost`, and live and die together. Ask yourself:

- **Do they need to scale together?** If you would ever want three replicas of one and ten of the other, they belong in separate Pods.
- **Does one only exist to serve the other?** A log shipper for this app, a proxy for this app, a config reloader for this app: yes. A database the app talks to: no.
- **Do they need a shared filesystem or `localhost`?** If they only talk over the network, a Service between two Pods is cleaner.
- **Would you deploy them separately?** Changing any container in a Deployment's template replaces every Pod. If the two have unrelated release cycles, coupling them means every release of one restarts the other.

The classic mistake is the "LAMP Pod": a web server and its database in the same Pod. They scale differently, they are released differently, and a database restarted every time you ship a CSS change is not something anyone wants.

## The four kinds of containers

A Pod spec can contain four kinds of containers, each with different rules.

| Kind | Where it's declared | When it runs | Typical use |
| --- | --- | --- | --- |
| **App containers** | `spec.containers` | For the life of the Pod | Your application |
| **Init containers** | `spec.initContainers` | Before app containers, one after another, each to completion | Setup: wait for a dependency, run a migration, fetch config |
| **Sidecar containers** | `spec.initContainers` with `restartPolicy: Always` | Start before app containers, keep running alongside them, stop after them | Proxies, log shippers, config reloaders |
| **Ephemeral containers** | Added at runtime with `kubectl debug` | Until you exit | Debugging a running Pod |

### Init containers

Init containers run **in order**, and each one must exit successfully before the next starts. Only when all of them have finished do the app containers start[^3]. If one fails, the kubelet retries it according to the Pod's `restartPolicy`; with `restartPolicy: Never`, the whole Pod fails.

They are a good fit for anything that must happen *before* the application is allowed to start:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shop
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ["sh", "-c", "until nc -z postgres 5432; do echo waiting for db; sleep 2; done"]
    - name: migrate
      image: registry.example.com/shop:3.1.0
      command: ["./shop", "migrate", "--up"]
      envFrom:
        - secretRef:
            name: shop-db
  containers:
    - name: app
      image: registry.example.com/shop:3.1.0
      ports:
        - containerPort: 8080
      envFrom:
        - secretRef:
            name: shop-db
```

A few properties make init containers useful beyond ordering:

- **They can use a different image.** The setup tools (`git`, `psql`, `curl`) don't need to be in your application image, which keeps it small and gives an attacker less to work with.
- **They can have different permissions.** An init container can run with a capability or a Secret that the app container never sees.
- **They block startup by design.** Readiness, liveness and startup probes aren't supported on regular init containers; they either finish or they don't[^3].

Because init containers can run more than once (after a node restart, or when the Pod is re-created), make them **idempotent**. A migration tool that is safe to run twice is fine; a script that inserts seed data every time is not.

### Sidecar containers: the old way

For years Kubernetes had no concept of a sidecar. A sidecar was just a second entry in `spec.containers`, and the kubelet treated it exactly like the application. That caused three well-known problems:

1. **No startup order.** All app containers start at roughly the same time. If your app made a network call before the mesh proxy was ready, it failed. Istio added a `holdApplicationUntilProxyStarts` option, built on a `postStart` hook trick, purely to work around this.
2. **Jobs never finished.** A Job completes when all containers in its Pod exit. The application finished, but the log shipper or proxy kept running forever, so the Job hung. The workarounds were ugly: a shared file the app touched on exit, or the app calling a `/quitquitquit` endpoint on the proxy.
3. **No shutdown order.** On termination, all containers get `SIGTERM` at the same time. The proxy could exit first, and the app lost its network while it was still draining requests.

And because init containers ran before any sidecar, an init container could never use the mesh proxy, for example to reach a database that required mTLS.

### Sidecar containers: native sidecars

Native sidecars fix all of this. The feature arrived in v1.28 and has been stable since v1.33[^2][^5]. Kubernetes implements a sidecar as a special case of init container: an entry in `initContainers` with a container-level `restartPolicy: Always`[^2].

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  initContainers:
    - name: log-shipper
      image: fluent/fluent-bit:3.2
      restartPolicy: Always          # this is what makes it a sidecar
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true
  containers:
    - name: app
      image: registry.example.com/web:2.4.0
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

Declaring it this way gives the sidecar a defined lifecycle:

- **It starts in order.** The kubelet starts init containers and sidecars in the order they are listed. A sidecar counts as "started" once its process is running, or, if it has a `startupProbe`, once that probe succeeds. Only then does the next entry start. Put the sidecar first and add a startup probe, and your app is guaranteed not to start before the proxy is ready.
- **It keeps running.** Unlike a regular init container, it doesn't have to exit, and app containers start while it is still running.
- **It restarts on its own.** If the sidecar crashes, it is restarted regardless of the Pod's `restartPolicy`, so it works even in Job Pods with `restartPolicy: Never`.
- **It doesn't block Jobs.** A Job completes when the main containers finish; a running sidecar no longer keeps it alive[^2].
- **It stops last.** On shutdown, the app containers get `SIGTERM` first. Sidecars are terminated only after the app containers have exited, in the reverse order they were declared.
- **It supports probes and lifecycle hooks.** Unlike regular init containers, sidecars can have startup, readiness and liveness probes, and `postStart`/`preStop` hooks. A sidecar's readiness counts toward the Pod being Ready.
- **Init containers after it can use it.** A regular init container listed *after* a sidecar runs while the sidecar is up, so a migration can go through the mesh proxy.

Here is the ordering in practice: a proxy that must be ready before a migration and before the app.

```yaml
spec:
  initContainers:
    - name: proxy
      image: registry.example.com/mesh-proxy:1.9.0
      restartPolicy: Always
      startupProbe:
        httpGet:
          path: /healthz/ready
          port: 15021
        periodSeconds: 1
        failureThreshold: 30
    - name: migrate                    # runs after proxy is ready, through the proxy
      image: registry.example.com/shop:3.1.0
      command: ["./shop", "migrate", "--up"]
  containers:
    - name: app                        # starts after migrate has finished
      image: registry.example.com/shop:3.1.0
```

Startup: `proxy` → (probe passes) → `migrate` → (exits 0) → `app`. Shutdown: `app` → `proxy`.

One catch on shutdown: sidecars don't get extra time. Everything shares the Pod's `terminationGracePeriodSeconds`. If your app uses 28 of the default 30 seconds to drain, the sidecar has 2 seconds left before it is killed. Size the grace period for the whole sequence.

If you are on v1.29 or later (beta, enabled by default) there is little reason to write a sidecar the old way. Most service meshes and logging agents now inject native sidecars.

### Ephemeral containers

Ephemeral containers are for debugging. Many production images are distroless: no shell, no `ps`, no `curl`. You can't `kubectl exec` into something that has no shell. Instead, you add a temporary container with the tools you need to a running Pod[^9]:

```bash
kubectl debug -it web --image=busybox:1.36 --target=app
```

`--target` puts the debug container into the process namespace of the `app` container, so `ps` shows the app's processes and `/proc/<pid>/root` shows its filesystem. Ephemeral containers can't have ports, probes or resource guarantees, they are never restarted, and once added they can't be removed: they stay in the Pod spec until the Pod is deleted.

## The classic patterns

In 2016, Brendan Burns and David Oppenheimer described a set of design patterns for container-based systems, and three of them are about multi-container Pods[^10]. The names have stuck.

### Sidecar: extend the main container

A sidecar adds a capability to the main container without changing it. The main container does its job; the sidecar does something alongside.

Common examples:

- A **log shipper** that tails files the app writes to a shared volume and forwards them.
- A **config reloader** that watches a ConfigMap volume and tells the app to reload.
- **git-sync**, which keeps a directory in sync with a Git repository for a web server to serve.
- A **service mesh proxy** that handles mTLS, retries and metrics for all traffic in and out.

Here is the shared-volume version, the simplest sidecar there is:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-site
spec:
  initContainers:
    - name: git-sync
      image: registry.k8s.io/git-sync/git-sync:v4.4.0
      restartPolicy: Always
      args:
        - --repo=https://github.com/example/site.git
        - --root=/git
        - --link=current
        - --period=60s
      volumeMounts:
        - name: content
          mountPath: /git
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
          subPath: current
          readOnly: true
  volumes:
    - name: content
      emptyDir: {}
```

Nginx knows nothing about Git. git-sync knows nothing about HTTP. The shared `emptyDir` is the only contract between them[^11].

### Ambassador: proxy the outside world

An ambassador sits between the main container and an *external* service, so the app can talk to `localhost` and leave the hard parts to someone else. The app connects to `localhost:5432`; the ambassador handles authentication, TLS, discovery or sharding.

The best-known example is a cloud database proxy. The application thinks it is talking to a local Postgres; the ambassador opens an authenticated, encrypted connection to the managed database, using the Pod's workload identity:

```yaml
spec:
  serviceAccountName: shop
  initContainers:
    - name: db-proxy
      image: registry.example.com/cloud-sql-proxy:2.14
      restartPolicy: Always
      args: ["--port=5432", "my-project:europe-west3:shop-db"]
      securityContext:
        runAsNonRoot: true
  containers:
    - name: app
      image: registry.example.com/shop:3.1.0
      env:
        - name: DB_HOST
          value: "127.0.0.1"
        - name: DB_PORT
          value: "5432"
```

The same shape works for a local Redis proxy that hides a sharded cluster, or for an outbound proxy that adds credentials to API calls. The app stays simple, and the same image works in development (with a real local database) and in production (with the ambassador).

### Adapter: present a standard interface

An adapter goes the other way: it translates the main container's output into a format the rest of the world expects. The most common case is monitoring. Your cluster scrapes Prometheus metrics, but the app exposes its status in its own format. An adapter reads the app's format over `localhost` and serves `/metrics`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  annotations:
    kubectl.kubernetes.io/default-container: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      volumeMounts:
        - name: conf
          mountPath: /etc/nginx/conf.d
    - name: exporter
      image: nginx/nginx-prometheus-exporter:1.4
      args: ["--nginx.scrape-uri=http://127.0.0.1:8081/stub_status"]
      ports:
        - name: metrics
          containerPort: 9113
  volumes:
    - name: conf
      configMap:
        name: nginx-conf          # enables stub_status on 127.0.0.1:8081
```

Here the exporter is a plain second app container rather than a native sidecar. That's a reasonable choice when start order doesn't matter and the adapter has nothing to do with Jobs; either works.

The three patterns are really the same mechanism, a second container that shares the network and volumes, seen from three angles:

| Pattern | Direction | Example |
| --- | --- | --- |
| Sidecar | Adds to the app | Log shipper, config reloader, mesh proxy |
| Ambassador | App → outside world | Database proxy, outbound auth proxy |
| Adapter | Outside world → app | Metrics exporter, log format converter |

## How containers talk to each other

There are three channels, from most to least common.

### `localhost`

Everything in the Pod shares one network namespace, so one container reaches another at `127.0.0.1:<port>`. No Service, no DNS, no network policy in between. That's convenient, and also a security point: every container in the Pod can reach every port that any other container listens on, including admin and debug ports bound to `127.0.0.1`. A third-party sidecar you didn't write gets the same access to your app as your app has to itself.

### Shared volumes

An `emptyDir` volume is created when the Pod is assigned to a node and deleted when the Pod goes away. Mount it into several containers and they share a directory. Use `medium: Memory` for a tmpfs when the data is small, hot or sensitive, and remember that memory-backed `emptyDir` counts against the containers' memory limits.

```yaml
volumes:
  - name: scratch
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
```

Mount it read-only (`readOnly: true`) into containers that only consume the data, as in the examples above.

### A shared process namespace

By default each container has its own PID namespace. Set `shareProcessNamespace: true` and all containers in the Pod share one[^6]:

```yaml
spec:
  shareProcessNamespace: true
  containers:
    - name: nginx
      image: nginx:1.27
    - name: reloader
      image: registry.example.com/config-reloader:0.9
      securityContext:
        capabilities:
          add: ["SYS_PTRACE"]
```

Now the reloader can see nginx's processes and send `kill -HUP <pid>` when the config changes. Be aware of what changes along with it[^6]:

- **Your process is no longer PID 1.** The pause container is. Applications that rely on being PID 1 (for signal handling or reaping zombies) may behave differently.
- **Processes are visible to every container**, along with their command lines and environment through `/proc`. If you pass a password as a command-line argument, every container can read it.
- **Filesystems are reachable** through `/proc/<pid>/root`, subject to permissions. Sharing the process namespace weakens the filesystem isolation between containers too.

Use it when you need signals or inspection between containers, not by default.

## Resources

### Requests and limits are per container

Each container has its own `resources` block, and the kubelet enforces limits per container. If the sidecar leaks memory and hits its limit, the sidecar is OOM-killed and restarted; the app keeps running. That isolation is one of the main reasons to use separate containers rather than running two processes in one.

### How the scheduler adds it up

The scheduler doesn't place containers, it places Pods, so it needs one number per resource for the whole Pod. With init containers and sidecars in the picture, that number is not a simple sum[^2][^7]:

- **Steady state** = sum of all app containers + all sidecars.
- **Each init step** = that init container + every sidecar declared *before* it (they are already running).
- **The Pod's effective request** = the larger of the steady state and the biggest init step, plus any Pod overhead from the RuntimeClass.

An example, with the proxy from earlier:

| Container | Kind | CPU | Memory |
| --- | --- | --- | --- |
| `proxy` | sidecar (declared first) | 100m | 128Mi |
| `migrate` | init (after `proxy`) | 1000m | 256Mi |
| `app` | app | 500m | 512Mi |

- Steady state: 100m + 500m = **600m** CPU; 128Mi + 512Mi = **640Mi**.
- Init step `migrate`: 1000m + 100m = **1100m** CPU; 256Mi + 128Mi = 384Mi.
- Effective request: **1100m** CPU (from the init step) and **640Mi** memory (from steady state).

So a migration that runs for five seconds can decide which nodes the Pod fits on. If that's a problem, give the init container a smaller request; it's allowed to be slow.

Two more things to keep in mind:

- **QoS class is decided by all containers.** For a Pod to be `Guaranteed`, *every* container, sidecars and init containers included, must have requests equal to limits for CPU and memory. One injected sidecar without limits silently drops the whole Pod to `Burstable`.
- **Sidecars are a tax on every replica.** A proxy asking for 100m CPU and 128Mi in a Deployment with 50 replicas reserves 5 CPUs and 6.4Gi of memory, whether it uses them or not. Size sidecar requests from real usage, not from the default in someone's Helm chart.

### Pod-level resources

Setting requests for each container separately is awkward when what you actually care about is the budget for the Pod as a whole. Pod-level resources let you set requests and limits for CPU, memory and hugepages in `spec.resources`; the feature has been beta and enabled by default since v1.34[^8]:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: budgeted
spec:
  resources:
    requests:
      cpu: "1"
      memory: 512Mi
    limits:
      cpu: "1"
      memory: 512Mi
  containers:
    - name: app
      image: registry.example.com/web:2.4.0
    - name: exporter
      image: registry.example.com/exporter:1.0
```

The containers share the Pod's budget, so idle CPU in one is available to the other. You can still set container-level values inside it to protect one container from another. For multi-container Pods where you know the total but not the split, this is much simpler than guessing a number for every container.

## Probes, readiness and restarts

- **A Pod is Ready only when all of its containers are Ready.** A Service sends traffic to the Pod only when every app container *and* every sidecar with a readiness probe passes. A broken metrics exporter can take a healthy app out of rotation, so think about whether the adapter really needs a readiness probe.
- **Liveness failures restart a single container**, not the Pod. The Pod keeps its IP, its volumes and its other containers.
- **`kubectl get pods` shows `READY 1/2`** when one of two containers isn't ready, and the `RESTARTS` count adds up restarts across containers. `kubectl describe pod` shows each container separately.
- **Pod phase reflects all containers.** In a Pod with `restartPolicy: Never`, the Pod is `Failed` if any app container fails. Native sidecars are the exception: their exit codes don't decide the phase.

## Working with multi-container Pods

Most `kubectl` commands assume one container. With several, you have to say which one:

```bash
# Logs from one container, or all of them
kubectl logs web -c log-shipper
kubectl logs web --all-containers --prefix

# The previous instance of a container that crashed
kubectl logs web -c app --previous

# Exec into a specific container
kubectl exec -it web -c app -- sh

# List the containers in a Pod
kubectl get pod web -o jsonpath='{range .spec.initContainers[*]}{.name}{" (init)\n"}{end}{range .spec.containers[*]}{.name}{"\n"}{end}'
```

To save typing, mark the container people usually want with an annotation. `kubectl logs`, `exec` and `attach` will use it when you leave out `-c`:

```yaml
metadata:
  annotations:
    kubectl.kubernetes.io/default-container: app
```

Without it, `kubectl` picks the first container in `spec.containers` and prints a hint about the others.

## Pitfalls

- **Port conflicts.** Two containers in one Pod can't bind the same port. Injected sidecars sometimes collide with an app that uses a common port such as `15000` or `9090`.
- **Coupled releases.** Any change to any container in the template replaces the whole Pod. Bumping a sidecar version restarts every replica of every app that uses it.
- **Coupled scaling.** Replicas scale the whole Pod. A sidecar that needs one instance per node, not per replica, should be a DaemonSet instead.
- **Shutdown time.** Everything shares one `terminationGracePeriodSeconds`. Leave room for sidecars to stop after the app.
- **QoS surprises.** A sidecar without limits turns a `Guaranteed` Pod into `Burstable`. Check what your injector adds.
- **Security blast radius.** Containers share `localhost` and, optionally, processes. A compromised sidecar is inside your app's network boundary. Give each container its own minimal `securityContext`, and remember that the Pod's service account token is mounted into every container unless you limit it.
- **Non-idempotent init containers.** They can run again after a node reboot or Pod re-creation. Make them safe to repeat.
- **Old-style sidecars in Jobs.** If a Job hangs with `1/2` containers completed, a sidecar in `spec.containers` is keeping it alive. Move it to `initContainers` with `restartPolicy: Always`.

It's also worth asking whether you need a per-Pod sidecar at all. Some service meshes now offer node-level proxies instead (Istio's ambient mode, for example), trading the per-Pod resource tax for shared infrastructure on each node. For log collection, a DaemonSet that reads container logs from the node is usually cheaper than a log shipper in every Pod, as long as your app writes to stdout.

## Bottom line

- **A Pod is a group of containers** that share a network namespace, IPC, a hostname, volumes and a lifecycle, but not a filesystem, process namespace or resource limits.
- **Group containers only when they are tightly coupled**: they must share files or `localhost`, and scale and deploy together.
- **Init containers** run in order, to completion, before the app. **Native sidecars** (`initContainers` with `restartPolicy: Always`, stable since v1.33) start before the app, run alongside it, stop after it, and don't block Jobs.
- **Sidecar, ambassador and adapter** are three uses of the same mechanism: extending the app, proxying the outside world, and adapting the app's interface.
- **Resources are per container, scheduling is per Pod.** Learn the effective request formula, watch QoS, and consider Pod-level resources for multi-container Pods.

## References

[^1]: [Pods](https://kubernetes.io/docs/concepts/workloads/pods/), Kubernetes documentation: what a Pod is, shared resources, and how Pods manage multiple containers.
[^2]: [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/), Kubernetes documentation: native sidecars as init containers with `restartPolicy: Always`, startup and termination order, Jobs, and resource calculation.
[^3]: [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/), Kubernetes documentation: ordering, restart behaviour, unsupported probes, and idempotency.
[^4]: [KEP-753: Sidecar Containers](https://www.kubernetes.dev/resources/keps/753/), Kubernetes Enhancement Proposal.
[^5]: [Kubernetes v1.33: Octarine](https://kubernetes.io/blog/2025/04/23/kubernetes-v1-33-release/), Kubernetes blog: sidecar containers graduate to stable.
[^6]: [Share Process Namespace between Containers in a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/), Kubernetes documentation.
[^7]: [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), Kubernetes documentation.
[^8]: [Kubernetes v1.34: Pod Level Resources Graduated to Beta](https://kubernetes.io/blog/2025/09/22/kubernetes-v1-34-pod-level-resources/), Kubernetes blog.
[^9]: [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/), Kubernetes documentation: ephemeral containers and `kubectl debug`.
[^10]: Brendan Burns and David Oppenheimer, [Design patterns for container-based distributed systems](https://www.usenix.org/conference/hotcloud16/workshop-program/presentation/burns), USENIX HotCloud 2016.
[^11]: [Communicate Between Containers in the Same Pod Using a Shared Volume](https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/), Kubernetes documentation.
