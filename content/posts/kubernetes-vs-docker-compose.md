---
title: "Kubernetes vs Docker Compose: Which One Do You Actually Need?"
date: 2026-11-20
draft: true
description: "A practical comparison of Docker Compose and Kubernetes: what each one is for, how the same app looks in both, where Compose runs out of road, and how to move from one to the other with Compose Bridge or Kompose."
summary: "Docker Compose runs a group of containers on one machine. Kubernetes runs workloads across a cluster and keeps them in the state you asked for. Here is how they differ in practice, the same app written for both, and how to choose."
tags: ["kubernetes", "docker", "docker compose", "containers", "orchestration"]
categories: ["Programming"]
---

"Should we use Kubernetes or Docker Compose?" comes up in almost every team that ships containers. It sounds like a choice between two competing products, but the two tools solve problems of very different sizes. Compose describes a handful of containers that run together on **one** machine. Kubernetes is a distributed system that runs workloads across **many** machines and keeps repairing them so they match what you declared.

This post compares them on the things that matter in practice, shows the same small application written for both, and ends with a rule of thumb and the tools for moving from one to the other. Versions at the time of writing: Docker Compose 5.6 and Kubernetes 1.37.[^1][^2]

## What each tool is

The three names that come up, Docker, Docker Compose and Kubernetes, sit at different layers. Each one builds on the layer below it.

### Docker

Docker packages an application with everything it needs (runtime, libraries, default config) into an **image**, and runs that image as an isolated process called a **container**. You build images from a `Dockerfile`, run them with `docker run`, and share them through a registry with `docker push` and `docker pull`.[^10]

Docker answers the question "how do I make this app run the same on my laptop, in CI and on a server?" But it works one container at a time. If your app needs an API, a database and a cache, you end up typing three long `docker run` commands with matching networks, volumes, ports and environment variables, and you have to remember all of them:

```bash
docker network create app
docker volume create db-data
docker run -d --name db --network app -v db-data:/var/lib/postgresql/data \
  -e POSTGRES_USER=app -e POSTGRES_DB=app -e POSTGRES_PASSWORD=secret postgres:17
docker run -d --name api --network app -p 8080:8080 \
  -e DATABASE_URL=postgres://app:secret@db:5432/app ghcr.io/example/api:1.4.0
```

Both Compose and Kubernetes exist to replace this with a file that describes the whole application.

One point causes a lot of confusion: Kubernetes doesn't need Docker. It runs containers through runtimes that implement its Container Runtime Interface, usually containerd or CRI-O, and it removed its built-in Docker integration (dockershim) in version 1.24.[^11] Images built with Docker follow the OCI image standard, so they run on Kubernetes unchanged. In practice, Docker stays the tool you build images with even when Kubernetes runs them.

### Docker Compose

Compose is a tool for defining and running multi-container applications. You write a `compose.yaml` that lists your services, networks and volumes, then run `docker compose up`. Compose talks to the local Docker Engine, creates the containers, puts them on a shared network where they can reach each other by service name, and that's it.[^3]

A few things worth knowing about modern Compose:

- **The file format is the Compose Specification.** The old `version: "3.8"` line at the top of the file is obsolete and ignored. There is one format now, maintained as an open spec.[^4]
- **The CLI is a Docker plugin.** It's `docker compose` (with a space), not the old Python `docker-compose` binary.
- **Version numbers jumped from 2 to 5.** Docker released Compose v5 in late 2025, skipping 3 and 4 on purpose so nobody confuses the tool version with the legacy `2.x`/`3.x` file formats. v5 delegates builds to Docker Bake and can be embedded in other software as a Go SDK. Your `compose.yaml` files don't need to change.[^1]

### Kubernetes

Kubernetes is a container orchestrator. You describe the desired state of your system as API objects (Deployments, Services, ConfigMaps, PersistentVolumeClaims and so on), submit them to the API server, and a set of controllers works continuously to make the cluster match. If a node dies, its Pods are rescheduled elsewhere. If a container crashes, it's restarted. If you change the image tag, the Deployment controller rolls Pods over gradually.[^5]

That **reconciliation loop** is the core difference. Compose does what you tell it when you run a command, then steps away. Kubernetes never stops checking.

## The same app in both

Take a typical small web app: a stateless API, a PostgreSQL database with persistent storage, and a password.

### Compose

```yaml
# compose.yaml
services:
  api:
    image: ghcr.io/example/api:1.4.0
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://app@db:5432/app
    env_file: .env            # holds DB_PASSWORD
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 10

volumes:
  db-data:

secrets:
  db_password:
    file: ./db_password.txt
```

About 40 lines, readable by anyone, and `docker compose up -d` brings the whole thing up.

### Kubernetes

The same application in Kubernetes needs, at minimum, a Deployment and Service for the API, a StatefulSet (or Deployment) plus Service and PersistentVolumeClaim for Postgres, and a Secret. Here is just the API half:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: ghcr.io/example/api:1.4.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              value: postgres://app@db:5432/app
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db
                  key: password
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
```

Add the database objects and an Ingress or Gateway API HTTPRoute to expose it, and you're well past 120 lines. The verbosity isn't wasted, though. Look at what the Kubernetes version says that the Compose version can't:

- **Three replicas**, spread across whatever nodes the scheduler picks.
- **Rolling updates with zero unavailability**: one new Pod comes up and must pass its readiness probe before an old one is removed.
- **Resource requests and limits**, which the scheduler uses for placement and the kubelet enforces.
- **Readiness gating on traffic**: the Service only sends requests to Pods whose probe passes.

That's the trade in a nutshell. Kubernetes asks you to be explicit about things that Compose simply doesn't model.

## Side-by-side

| | Docker Compose | Kubernetes |
|---|---|---|
| Scope | One host | A cluster of nodes |
| Model | Imperative commands over a declarative file | Continuous reconciliation of desired state |
| Setup | Docker Engine or Desktop | A cluster: managed (EKS, GKE, AKS), kubeadm, k3s, kind… |
| Config size | Small, one file | Large, many objects; usually Helm or Kustomize on top |
| Scaling | `deploy.replicas` / `--scale` on the same host | Replicas across nodes, plus autoscaling (HPA, VPA, cluster autoscaler) |
| Self-healing | `restart:` policy on the same host | Rescheduling to healthy nodes, liveness/readiness probes |
| Updates | Recreates changed containers | Rolling updates, rollback with `kubectl rollout undo` |
| Networking | One bridge network per project, DNS by service name | Cluster-wide Pod network, Services, Ingress/Gateway API, NetworkPolicy |
| Storage | Named volumes and bind mounts on the host | PersistentVolumes provisioned by StorageClasses, CSI drivers |
| Secrets | File-mounted secrets from local files | Secret objects, RBAC, external secret stores |
| Access control | Whoever can reach the Docker socket is effectively root | RBAC, namespaces, admission control, Pod Security |
| Ecosystem | Compact | Huge: operators, service meshes, GitOps, policy engines |
| Learning curve | An afternoon | Weeks to be productive, longer to run it well |

A few rows deserve more detail.

### Scaling and high availability

Compose can run multiple copies of a service with `deploy.replicas` or `docker compose up --scale api=3`, but they all run on the same machine. If that machine goes down, everything goes down. There's no built-in load balancing across the replicas beyond Docker's DNS round robin, and you can't publish the same fixed host port from three containers.

Kubernetes treats nodes as replaceable. A Deployment with three replicas survives a node failure because the controller notices missing Pods and schedules new ones elsewhere. Add a HorizontalPodAutoscaler and the replica count follows CPU, memory or custom metrics.

### Updates

When you change an image tag and run `docker compose up -d`, Compose stops the old container and starts the new one. For that window, the service is down. You can work around it with a reverse proxy and manual blue-green tricks, but nothing in Compose does it for you.

A Kubernetes Deployment does rolling updates by default and keeps a revision history, so `kubectl rollout undo deployment/api` takes you back.

### Health

Compose healthchecks mostly serve `depends_on: condition: service_healthy`, which orders startup. An unhealthy container is marked unhealthy, but Compose doesn't restart it or route around it.

Kubernetes splits health into **liveness** (restart me if this fails), **readiness** (don't send me traffic while this fails) and **startup** probes, and acts on each.

### Security

The Docker socket is the weak spot of a Compose setup: anyone who can talk to it can start a privileged container and own the host. There's no notion of users, namespaces or per-team permissions.

Kubernetes has RBAC, namespaces, admission controllers, Pod Security Standards and NetworkPolicies. These take work to configure, and a misconfigured cluster can be worse than a single well-locked host, but the controls exist.

## When Compose is the right answer

Compose is the better tool more often than the industry conversation suggests:

- **Local development.** Even teams that deploy to Kubernetes usually use Compose on laptops to run the app and its dependencies (database, cache, message broker). `docker compose watch` syncs code changes into running containers or rebuilds them automatically.[^6]
- **Integration tests in CI.** Spin up the stack, run the tests, `docker compose down -v`.
- **Single-server production.** A small SaaS, an internal tool, a self-hosted app on a VPS or a home server. If one machine is enough and a few seconds of downtime during deploys is acceptable, Compose plus a reverse proxy like Caddy or Traefik is simple, cheap and easy to debug.
- **Small teams without platform engineers.** Kubernetes is a system you have to operate: upgrades every few months, certificates, CNI, storage, monitoring. That's a real cost.

## When Kubernetes is worth it

Kubernetes earns its complexity when you need one or more of these:

- **High availability**: the service must survive losing a machine.
- **Horizontal scale**: load is more than one host can handle, or varies enough that autoscaling pays off.
- **Zero-downtime deploys** as a routine, not a project.
- **Many services and many teams** sharing infrastructure, with isolation and access control between them.
- **The ecosystem**: operators for databases and brokers, GitOps with Argo CD or Flux, cert-manager, policy engines, service meshes.
- **Portability across clouds**: the same manifests run on EKS, GKE, AKS or your own hardware.

If you're on a cloud provider, a managed control plane removes the hardest part of running Kubernetes. It doesn't remove the need to understand it.

## The middle ground

The choice isn't strictly binary:

- **k3s** is a lightweight, single-binary Kubernetes distribution. It runs well on one small VM or a Raspberry Pi, so you can use Kubernetes APIs without much infrastructure.
- **Docker Swarm** uses Compose-format files and adds multi-node scheduling and rolling updates. It still works, but its ecosystem is small and few new projects choose it.
- **HashiCorp Nomad** is a simpler orchestrator that also handles non-container workloads.
- **Podman** can run Kubernetes Pod YAML locally with `podman kube play`, which narrows the gap from the other direction.

## Moving from Compose to Kubernetes

Two tools can translate a `compose.yaml` into Kubernetes manifests.

**Compose Bridge** is built into Docker Desktop (4.43 and later). `docker compose bridge convert` reads your Compose file and generates Kubernetes manifests plus a Kustomize overlay.[^7] It works through *transformations*: Docker images that receive the fully resolved Compose model and write out whatever format they're programmed to produce. The default transformation targets Kubernetes, and you can extract and edit its templates or build your own to match your organization's conventions.[^8]

```bash
docker compose bridge convert -o k8s/
kubectl apply -k k8s/overlays/desktop
```

**Kompose** is the older, standalone CNCF tool for the same job:[^9]

```bash
kompose convert -f compose.yaml -o k8s/
```

Treat the output of either as a first draft, not a finished deployment. A Compose file doesn't contain the information Kubernetes needs to run well, so the generated manifests won't have sensible resource requests, real readiness probes, PodDisruptionBudgets, an Ingress or HTTPRoute matching your setup, or a proper StatefulSet for your database. Use the generated YAML to learn how the concepts map, then fill in what's missing, and consider packaging the result as a Helm chart once it settles.

The rough mapping:

| Compose | Kubernetes |
|---|---|
| service | Deployment (or StatefulSet) + Service |
| `ports` | Service (+ Ingress/HTTPRoute for external traffic) |
| `environment`, `env_file` | `env`, ConfigMap |
| `secrets` | Secret |
| named volume | PersistentVolumeClaim |
| `healthcheck` | liveness/readiness/startup probes |
| `deploy.replicas` | `spec.replicas` |
| `depends_on` | No direct equivalent: init containers, readiness probes, or app-side retries |

That last row trips people up. Kubernetes deliberately has no startup ordering between Deployments. Applications are expected to retry connections until their dependencies are reachable, which is good practice anyway and in the spirit of the twelve-factor app's "backing services" principle.

## Rule of thumb

- **One machine is enough, and brief downtime on deploy is fine** → Compose.
- **You need to survive a machine failure, scale out, or deploy without downtime** → Kubernetes (or a lighter orchestrator).
- **You're deploying to Kubernetes** → still use Compose for local development.
- **Whichever you choose** → you'll most likely still build your images with Docker.

Most teams don't outgrow Compose as quickly as they fear, and the ones that do can carry their containers, images and much of their configuration with them. Start with the simplest tool that meets your availability requirements, and switch when a concrete requirement forces it, not before.

## References

[^1]: [docker/compose releases](https://github.com/docker/compose/releases), GitHub (v5.0.0 release notes and v5.6.0, October 2026).
[^2]: [Kubernetes Releases](https://kubernetes.io/releases/), Kubernetes documentation.
[^3]: [Docker Compose overview](https://docs.docker.com/compose/), Docker documentation.
[^4]: [Compose Specification](https://github.com/compose-spec/compose-spec), compose-spec on GitHub.
[^5]: [Kubernetes Overview](https://kubernetes.io/docs/concepts/overview/), Kubernetes documentation.
[^6]: [Use Compose Watch](https://docs.docker.com/compose/how-tos/file-watch/), Docker documentation.
[^7]: [Overview of Compose Bridge](https://docs.docker.com/compose/bridge/), Docker documentation.
[^8]: [Customize Compose Bridge](https://docs.docker.com/compose/bridge/customize/), Docker documentation.
[^9]: [Kompose](https://kompose.io/), Kubernetes project.
[^10]: [What is Docker?](https://docs.docker.com/get-started/docker-overview/), Docker documentation.
[^11]: [Dockershim Removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/), Kubernetes blog, February 2022.
