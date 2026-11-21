---
title: "Docker Best Practices"
date: 2026-11-12
draft: true
tags: ["docker", "containers", "devops", "security", "go"]
summary: "A practical checklist for writing smaller, faster, and safer Docker images and running containers sensibly, with examples in Go."
---

Docker makes it easy to package an application, but it's just as easy to end up with a 2 GB image that runs as root, rebuilds from scratch on every commit, and leaks a `.env` file into a registry. This post collects the practices I keep coming back to, split into three parts: **building images**, **securing images and containers**, and **running them day to day**. The examples use a small Go HTTP service, but the ideas carry over to any stack.

## Building images

### Use multi-stage builds

A single-stage build drags compilers, package managers, and build caches into the final image. Multi-stage builds let you use several `FROM` statements in one Dockerfile, compile in one stage, and copy only the finished artifacts into a lean runtime stage[^1]. Docker's own guidance recommends them for all types of applications[^2], and Go is the ideal case: the whole toolchain stays in the build stage and the runtime stage gets a single static binary.

```dockerfile
# syntax=docker/dockerfile:1

# --- build stage ---
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -tags timetzdata \
    -ldflags="-s -w" -o /out/server ./cmd/server

# --- runtime stage ---
FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /out/server /server
EXPOSE 8000
USER 65532:65532
ENTRYPOINT ["/server"]
```

`CGO_ENABLED=0` produces a statically linked binary that doesn't need libc in the runtime image. `-trimpath` and `-ldflags="-s -w"` strip local paths and debug symbols to shrink it further. The runtime stage starts from `scratch`, an empty image, so the final image is the binary plus a CA certificate bundle and nothing else. More on that choice below.

A bonus with BuildKit: when you build a specific `--target`, it only builds the stages that target actually depends on[^1], and independent stages can run in parallel[^3].

### Order instructions for the build cache

Docker walks the Dockerfile top to bottom and reuses a cached layer for each instruction until something changes; from that point on, every subsequent layer is rebuilt[^4]. So put things that change rarely first and things that change constantly last:

1. Base image and system packages
2. Dependency manifests (`go.mod`, `go.sum`)
3. Dependency download (`go mod download`)
4. Application source and the build itself

Copying only `go.mod` and `go.sum` before downloading modules (as in the example above) means a code change doesn't re-download every dependency. Cache mounts go one step further by keeping Go's module and build caches between builds, so even a cache miss only recompiles the packages that changed:

```dockerfile
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server ./cmd/server
```

### Keep the build context small with `.dockerignore`

Everything in the build directory is sent to the builder as context. A `.dockerignore` file keeps logs, `.git`, build output, and local secrets out of the context, which speeds up builds and prevents sensitive files from accidentally ending up in an image[^5].

```text
.git
bin/
tmp/
*.log
*.test
coverage.out
.env
.env.*
```

### Choose a small, trusted base image

Start from official or verified images, and pick the smallest one that works. Fewer packages means a smaller download, a faster pull, and less attack surface. For interpreted languages that usually means a `-slim` or Alpine variant. For a static Go binary you can go much further and ship an image with no operating system userland at all.

### Start `FROM scratch`

My default for Go services is `FROM scratch`. It is not really an image: `scratch` is a reserved name that tells the builder to start from nothing, so your first instruction becomes the first filesystem layer, and you can't pull it or run it[^11]. The result contains exactly what you copy in. There is no shell, no package manager, no third-party base to track or verify, and nothing for a scanner to flag except your own binary.

The price is that you supply everything the binary quietly depends on. The Dockerfile at the top of this post already does, and each line is there for a reason:

- **CA certificates.** Without them every outbound HTTPS call fails with an unknown-authority error. That is the `COPY --from=build /etc/ssl/certs/ca-certificates.crt` line.
- **Timezone data.** `time.LoadLocation` fails without it. Building with `-tags timetzdata` embeds it in the binary, so there is no file to copy.
- **Users.** There is no `/etc/passwd`, so `USER` must be numeric, hence `USER 65532:65532`.
- **`/tmp`.** It doesn't exist. If the service writes temp files, run it with `--tmpfs /tmp`.
- **A truly static binary.** If the binary is dynamically linked it fails at startup with a misleading "no such file or directory", because the dynamic loader is missing. `CGO_ENABLED=0` takes care of that.

### Certificates: the trust bundle vs. your own

"Certificates in the container" covers two very different things, and they get opposite treatment.

**The CA trust bundle** is the file copied into the `scratch` image above. It holds public root certificates that your binary uses to *verify* the servers it connects to. Nothing in it is secret, so baking it into the image is fine. You only need it if the service makes outbound TLS calls; a service that only serves plain HTTP behind a proxy can drop that `COPY` line.

The baked-in bundle is frozen at build time and rotates when you rebuild with `--pull`, which picks up the build image's current `ca-certificates` package. Roots change rarely, but they do get added and distrusted, so this is one more reason to rebuild on a schedule. If you'd rather decouple the bundle from your release cycle, there are two alternatives:

- **Mount it at runtime.** Leave it out of the image and bind-mount a bundle read-only, for example the host's own, which updates with the host's packages:

  ```bash
  docker run \
    -v /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.crt:ro \
    myapp:1.4.2
  ```

  On Linux, Go also reads the `SSL_CERT_FILE` and `SSL_CERT_DIR` environment variables, which override its default search paths[^13], so you can mount the bundle anywhere and point to it. The host path varies by distribution (`/etc/pki/tls/certs/ca-bundle.crt` on RHEL-family systems). This is also the right way to add a private or internal CA, which changes on your PKI's schedule rather than your release schedule. Go loads the system roots once, so a changed bundle takes effect on the next container restart.

- **Embed it in the binary.** A blank import of `golang.org/x/crypto/x509roots/fallback` compiles a root set into the binary, used only when the system provides none[^14]:

  ```go
  import _ "golang.org/x/crypto/x509roots/fallback"
  ```

  That removes the `COPY` line entirely. The trade-off is that rotation now means bumping a Go module: the package documentation says it must be kept up to date and recommends `govulncheck` to flag new versions[^14].

**Your service's own certificate and private key** are the opposite case. If the service terminates TLS itself, the key is a secret and must never be in the image: anyone who can pull the image would have it, and every renewal would mean a rebuild. Mount it at runtime instead, from Docker or Compose secrets[^7], a Kubernetes Secret, or a volume that Vault or cert-manager keeps current.

To rotate without a restart, load the certificate through `tls.Config.GetCertificate` and reload it when the file changes:

```go
type certReloader struct {
    certFile, keyFile string

    mu   sync.Mutex
    cert *tls.Certificate
    mod  time.Time
}

func (r *certReloader) GetCertificate(*tls.ClientHelloInfo) (*tls.Certificate, error) {
    info, err := os.Stat(r.certFile)
    if err != nil {
        return nil, err
    }
    r.mu.Lock()
    defer r.mu.Unlock()
    if r.cert == nil || info.ModTime().After(r.mod) {
        c, err := tls.LoadX509KeyPair(r.certFile, r.keyFile)
        if err != nil {
            if r.cert != nil {
                return r.cert, nil // keep serving the old cert if the new pair is incomplete
            }
            return nil, err
        }
        r.cert, r.mod = &c, info.ModTime()
    }
    return r.cert, nil
}
```

```go
r := &certReloader{certFile: "/run/secrets/tls.crt", keyFile: "/run/secrets/tls.key"}
srv := &http.Server{Addr: ":8443", TLSConfig: &tls.Config{GetCertificate: r.GetCertificate}}
log.Fatal(srv.ListenAndServeTLS("", ""))
```

The simpler option is to restart the container after renewal, which is fine if you run more than one replica. Simpler still: let a reverse proxy or ingress (Caddy, Traefik, nginx, a cloud load balancer) terminate TLS and renew certificates automatically, and have the Go service speak plain HTTP on an internal network. Then the container holds no private key at all.

### When to reach for distroless instead

Distroless images are `scratch` with those gaps already filled. `gcr.io/distroless/static` adds exactly four things: CA certificates, an `/etc/passwd`, a `/tmp` directory, and timezone data[^12]. It is still only around 2 MiB[^6]. On top of that you get a `:nonroot` tag, `:debug` tags that include a BusyBox shell, and images signed with cosign[^6].

I switch to it in two cases. The first is cgo: a binary that links against glibc can't run on `scratch`, and `distroless/base` adds glibc and libssl (`base-nossl` adds glibc alone)[^12]. The second is when a team wants the `:debug` variant on hand, or would rather have someone else maintain the certificate bundle than own those extra lines.

| | `scratch` | `distroless/static` |
|---|---|---|
| Size before your binary | 0 | ~2 MiB |
| CA certificates | Copy them in | Included |
| Timezone data | Copy or embed | Included |
| `/etc/passwd`, non-root user | Numeric `USER` only | `:nonroot` tag |
| `/tmp` | Missing | Included |
| Shell for debugging | None | `:debug` tag |
| cgo / glibc | Not possible | `base` variants |
| Base image updates | Nothing to update | Rebuild to pick up new certs and tzdata |

Either way there is no shell, so `docker exec -it api sh` won't work. To poke at a running container, attach a throwaway one to its namespaces:

```bash
docker run --rm -it --pid=container:api --network=container:api busybox
```

### Pin base image versions — deliberately

`FROM golang:latest` is a moving target. Pinning to a minor tag like `golang:1.26` still floats to the latest patch release, which Docker describes as a reasonable default since you pick up fixes automatically[^3]. If you need fully reproducible builds, pin by digest (`FROM golang:1.26@sha256:...`) and automate the bumps so you don't miss security patches. Docker's docs point to Docker Scout for this[^3]; Renovate and Dependabot do the same job.

### Rebuild images often

An image is a snapshot: the base image and every dependency are frozen at build time. Rebuild regularly, not only when your code changes, so patches actually reach production[^3]. Two flags matter here, and they do different things:

```bash
# Check for a newer base image even if one is cached locally
docker build --pull -t myapp:1.4.2 .

# Also ignore the layer cache and re-run every step
docker build --pull --no-cache -t myapp:1.4.2 .
```

`--pull` refreshes the base image; `--no-cache` re-runs the build steps but does not pull a fresh base on its own[^3]. A scheduled weekly CI build with `--pull` is a cheap way to stay current. This matters even with a `scratch` runtime stage: the rebuild is what picks up Go toolchain and standard library patches and a fresh CA bundle from the build image.

### Combine and clean up package installs

If a stage needs OS packages (for example, a cgo build that links against a C library), install them in a single `RUN`, skip recommended extras, and clean up in the same layer so the cleanup actually shrinks the image:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      libsqlite3-dev \
    && rm -rf /var/lib/apt/lists/*
```

Sorting multi-line package lists alphabetically makes diffs and duplicate-spotting easier[^3].

### Prefer `COPY` over `ADD`, and use exec-form `CMD`/`ENTRYPOINT`

`COPY` does exactly one obvious thing. Reach for `ADD` only when you need its extra behaviour (remote URLs, auto-extracting tarballs). Write `ENTRYPOINT ["/server"]` rather than `ENTRYPOINT /server` so your process is PID 1 and receives signals like `SIGTERM` directly instead of having a shell swallow them. On `scratch` the exec form is the only one that works anyway, because there is no shell.

Your app still has to handle `SIGTERM` to shut down gracefully. In Go that is a few lines:

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, os.Interrupt)
defer stop()

go func() { _ = srv.ListenAndServe() }()

<-ctx.Done()
shutdownCtx, cancel := context.WithTimeout(context.Background(), 8*time.Second)
defer cancel()
_ = srv.Shutdown(shutdownCtx)
```

`docker stop` waits 10 seconds by default before sending `SIGKILL`, so keep the shutdown timeout below that or raise it with `--stop-timeout`.

## Security

### Don't run as root

By default a container runs as root. Configuring an unprivileged user is the most direct defence against privilege escalation[^7]. On `scratch` that is a single line, `USER 65532:65532`, with a numeric ID because there is no `/etc/passwd` to look names up in. On a full distro image, create the user yourself:

```dockerfile
RUN addgroup --system app && adduser --system --ingroup app app
USER app
```

That syntax is for Debian/Ubuntu-based images; on Alpine use `addgroup -S app && adduser -S -G app app`.

### Never bake secrets into images

`ENV API_KEY=...` or `COPY .env .` puts the secret into a layer, where anyone with the image can read it — even if a later layer deletes the file. Use BuildKit secret mounts for build-time secrets[^8] and inject runtime secrets via your orchestrator or environment. The common Go case is fetching private modules:

```dockerfile
RUN --mount=type=secret,id=netrc,target=/root/.netrc \
    GOPRIVATE=github.com/yourorg/* go mod download
```

```bash
docker build --secret id=netrc,src=$HOME/.netrc .
```

The `.netrc` file is available only during that one `RUN` step and is never written to a layer.

### Drop privileges at runtime

The OWASP Docker Security Cheat Sheet is the best single checklist here[^7]. The highlights:

- **Drop capabilities** you don't need: `--cap-drop all --cap-add <only-what-you-need>`.
- **Block privilege escalation** with `--security-opt=no-new-privileges`.
- **Never use `--privileged`** unless you truly know why.
- **Make the filesystem read-only** with `--read-only`, plus a `tmpfs` for any scratch paths.
- **Keep the default seccomp/AppArmor/SELinux profiles** enabled.
- **Don't mount `/var/run/docker.sock`** into containers — it's effectively root on the host.

A static Go service on `scratch` needs almost none of what Docker grants by default, so it can run fully locked down (add `--tmpfs /tmp` if it writes temp files):

```bash
docker run -d --name api \
  --read-only \
  --cap-drop all \
  --security-opt=no-new-privileges \
  -p 127.0.0.1:8000:8000 \
  myapp:1.4.2
```

### Consider rootless mode

The settings above restrict the container, but the Docker daemon itself normally runs as root. Rootless mode runs both the daemon and the containers as an unprivileged user, which limits the damage if the daemon or runtime is compromised[^7]. It is not a complete answer: containers still share the host kernel, so kernel vulnerabilities remain in scope, and rootless mode comes with its own known limitations[^7]. Docker's rootless documentation covers setup and what doesn't work[^9]. It is worth it for hosts where security matters more than convenience. Note that `userns-remap` is a different feature and leaves the daemon running as root[^7].

### Bind ports to localhost when they don't need to be public

Docker's published ports bypass host firewalls like `ufw`, so `-p 8000:8000` may be reachable from the internet even if you think the firewall blocks it. If a service only needs local access, bind it explicitly[^7]:

```yaml
services:
  api:
    image: myapp:1.4.2
    ports:
      - "127.0.0.1:8000:8000"
```

### Scan and lint

- Add a Dockerfile linter to your CI pipeline[^7]. [Hadolint](https://github.com/hadolint/hadolint) is the usual choice.
- Scan images for known CVEs with Docker Scout, Trivy, or Grype. For Go code, [`govulncheck`](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck) reports only the vulnerabilities your code actually calls.
- Keep the Docker Engine and host OS up to date — it's rule #0 in the OWASP list for good reason[^7].

### Secure the supply chain

Scanning tells you what is in an image. Supply-chain measures let you prove where it came from. OWASP groups these under provenance, SBOMs, image signing, and a trusted registry[^7].

- **SBOM and provenance.** BuildKit can attach both as attestations at build time[^10]:

  ```bash
  docker buildx build --sbom=true --provenance=mode=max \
    -t registry.example.com/myapp:1.4.2 --push .
  ```

  The classic local image store drops attestations, so push straight to a registry or enable the containerd image store[^10].
- **Signing.** Sign the pushed image by digest with a tool such as [cosign](https://github.com/sigstore/cosign), and verify signatures before deploying. With a `scratch` runtime stage the only upstream image left to trust is the build image, so pin that by digest.
- **Trusted registry.** Store images in a registry with access controls, and deploy by digest rather than by mutable tag.

## Running containers

### One concern per container

Containers should be ephemeral: stoppable, destroyable, and replaceable with minimal setup[^3]. Run one main process per container and connect services over Docker networks rather than cramming a web server, database, and cron daemon into one image.

### Persist data in volumes, not the container layer

Anything written to the container's writable layer disappears when the container is removed. Use named volumes (or bind mounts in development) for databases, uploads, and anything else that must survive a redeploy.

### Set resource limits

An unbounded container can starve the host. OWASP recommends limiting memory, CPU, and the number of processes to blunt denial-of-service problems[^7]. With `docker run` that looks like this (`--pids-limit` caps the process count and stops fork bombs):

```bash
docker run --memory=256m --cpus=1 --pids-limit=100 myapp:1.4.2
```

Since Go 1.25 the runtime sets `GOMAXPROCS` from the container's CPU limit automatically. Memory is still on you: set `GOMEMLIMIT` a little below the container limit (for example `-e GOMEMLIMIT=230MiB`) so the garbage collector works harder before the kernel's OOM killer steps in.

### Add a health check and a restart policy

The check runs inside the container, so it can only use what the image contains. A `scratch` image has no `curl`, `wget`, or shell, so the cleanest option is to make the binary check itself:

```go
if len(os.Args) > 1 && os.Args[1] == "healthcheck" {
    resp, err := http.Get("http://127.0.0.1:8000/health")
    if err != nil || resp.StatusCode != http.StatusOK {
        os.Exit(1)
    }
    os.Exit(0)
}
```

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD ["/server", "healthcheck"]
```

Combine with `--restart unless-stopped` (or the equivalent in Compose) so a crashed service comes back on its own.

### Log to stdout/stderr

Let Docker (or your orchestrator) collect logs from standard output instead of writing log files inside the container. It keeps the container stateless, works with `docker logs` and any log driver, and is compatible with a read-only filesystem. Go's `log/slog` with a JSON handler on `os.Stdout` is all you need.

### Tag images meaningfully

`latest` tells you nothing about what's running. Tag with a version and/or the git commit SHA (`myapp:1.4.2`, `myapp:3f9c2ab`) so deployments are traceable and rollbacks are trivial. You can stamp the same value into the binary so the service can report it:

```dockerfile
ARG VERSION=dev
RUN CGO_ENABLED=0 go build -trimpath \
    -ldflags="-s -w -X main.version=${VERSION}" -o /out/server ./cmd/server
```

```bash
docker build --build-arg VERSION=1.4.2 -t myapp:1.4.2 .
```

## Running on Kubernetes

Everything above about the image still applies on Kubernetes: the same `scratch` image runs unchanged. What changes is where the runtime settings live. Kubernetes doesn't read `docker run` flags, so each one moves into the pod spec, and a few Docker features are ignored or replaced.

### The numeric user pays off

Kubernetes' `runAsNonRoot: true` makes the kubelet check that the container won't run as UID 0. It can only check a numeric `USER`: an image with a named user such as `nonroot` is refused with a "cannot verify user is non-root" error unless you also set `runAsUser`[^15]. The `USER 65532:65532` line in the `scratch` image passes as is.

### From `docker run` flags to the pod spec

| Docker | Kubernetes |
|---|---|
| `--cap-drop all` | `securityContext.capabilities.drop: ["ALL"]` |
| `--security-opt=no-new-privileges` | `securityContext.allowPrivilegeEscalation: false` |
| `--read-only` | `securityContext.readOnlyRootFilesystem: true` |
| Default seccomp profile (automatic) | `seccompProfile.type: RuntimeDefault` (not automatic, see below) |
| `--tmpfs /tmp` | `emptyDir` volume mounted at `/tmp` |
| `--memory`, `--cpus` | `resources.requests` and `resources.limits` |
| `--pids-limit` | Set per node by the cluster admin, not per pod |
| `--restart unless-stopped` | Handled by the Deployment |
| `-p 127.0.0.1:8000:8000` | ClusterIP Service plus a NetworkPolicy |
| `HEALTHCHECK` | Readiness and liveness probes |

Three of these rows need a closer look.

**Seccomp is not on by default.** Docker applies its default seccomp profile to every container. Kubernetes runs pods `Unconfined` unless the pod sets a profile or the cluster admin turns on the kubelet's seccomp-default setting[^16]. Set `seccompProfile.type: RuntimeDefault` explicitly. OWASP recommends enforcing the Restricted Pod Security level cluster-wide[^7], and that level requires this setting along with most of the others in the table.

**`HEALTHCHECK` is ignored.** Kubernetes does not read the image's `HEALTHCHECK`; it only runs the probes declared in the pod spec[^17]. That works in favour of `scratch`: an `httpGet` probe is sent by the kubelet from outside the container, so the image needs no tools and the `healthcheck` subcommand from earlier is only needed for plain Docker or Compose.

**Port binding doesn't translate.** A ClusterIP Service is reachable only inside the cluster, and a NetworkPolicy restricts which pods can reach it, as OWASP suggests for inter-container traffic[^7].

### Shutdown without a shell

When a pod is deleted, Kubernetes sends `SIGTERM` and waits `terminationGracePeriodSeconds`, 30 seconds by default, before killing it. The catch is that the pod is removed from the Service endpoints at the same time. For a few seconds, traffic can still arrive after the server has stopped accepting connections.

The usual fix is a `preStop` hook that sleeps, but `sh -c "sleep 5"` doesn't exist in a `scratch` image. Kubernetes now has a built-in sleep action that the kubelet runs itself, so it works with no binaries in the image[^18][^19]. It reached beta in 1.30 and stable in 1.34[^18]. On older clusters, have the Go service keep serving for a few seconds after `SIGTERM` before calling `Shutdown`.

### Certificates on Kubernetes

The trust bundle can stay baked in, but on Kubernetes you can also leave it out of the image entirely and let the platform provide it. Then the image is just the binary.

- **No outbound TLS, no bundle.** If the service only serves plain HTTP behind an Ingress, and any outbound TLS is handled by a service mesh sidecar, the binary never verifies a certificate and needs no bundle at all.
- **Mount it from a ConfigMap.** Publish the bundle as a ConfigMap, which cert-manager's trust-manager can keep in sync across namespaces, mount it as a volume, and point `SSL_CERT_FILE` at the file[^13]. This is the natural fit for an internal CA, and updating the bundle no longer needs a rebuild. Kubernetes also has a native `clusterTrustBundle` projected volume for this, but it is still beta and disabled by default[^20].

```yaml
          env:
            - name: SSL_CERT_FILE
              value: /etc/ca/ca-certificates.crt
          volumeMounts:
            - { name: ca-bundle, mountPath: /etc/ca, readOnly: true }
      volumes:
        - name: ca-bundle
          configMap:
            name: ca-bundle   # key: ca-certificates.crt
```

The trade-off is that the image is no longer self-contained: without the mount, every outbound TLS call fails, and it won't run correctly under plain `docker run`. Importing `x509roots/fallback` covers that gap: Go uses the embedded roots only when the system provides none[^14], so the mounted bundle wins on Kubernetes and the binary still works anywhere else.

For the service's own certificate, cert-manager can issue it into a Secret that you mount as a volume. Don't mount it with `subPath`: a `subPath` mount never receives updates, which silently breaks rotation[^20]. A normal Secret volume is updated in place after a short delay, and the `GetCertificate` reloader from earlier picks up the change without a restart. Remember that Secrets are stored unencrypted unless the cluster enables etcd encryption[^7].

Often the simplest answer is to keep TLS out of the container entirely: an Ingress controller terminates external TLS, and a service mesh can handle mTLS between pods.

### Other changes

- **Service account token.** Every pod gets a Kubernetes API credential by default. Set `automountServiceAccountToken: false` unless the service calls the API.
- **Debugging.** `kubectl debug -it pod/<name> --image=busybox --target=api` attaches an ephemeral container with tools to the running pod, replacing the `docker run --pid=container:` trick.
- **Image references.** Deploy by digest, and consider an admission policy that verifies image signatures before pods start.

### Example Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        runAsGroup: 65532
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: api
          image: registry.example.com/myapp@sha256:...   # deploy by digest
          ports:
            - containerPort: 8000
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits:   { cpu: "1", memory: 256Mi }
          env:
            - name: GOMEMLIMIT
              value: 230MiB
          readinessProbe:
            httpGet: { path: /health, port: 8000 }
          livenessProbe:
            httpGet: { path: /health, port: 8000 }
          lifecycle:
            preStop:
              sleep: { seconds: 5 }   # no shell needed; stable since 1.34
          volumeMounts:
            - { name: tmp, mountPath: /tmp }
      volumes:
        - name: tmp
          emptyDir: { medium: Memory }
```

Setting `runAsUser` here duplicates the image's `USER`, but it means the pod stays non-root even if someone later builds the image without it.

## Quick checklist

- [ ] Multi-stage build; final stage contains only the binary
- [ ] `go.mod`/`go.sum` copied and modules downloaded before source code
- [ ] `.dockerignore` excludes `.git`, build output, `.env`, logs
- [ ] Runtime stage is `FROM scratch` (distroless if you need cgo); build image pinned to a version or digest
- [ ] Images rebuilt on a schedule with `--pull`
- [ ] Runs as a non-root `USER`
- [ ] No secrets in `ENV`, `ARG`, or copied files
- [ ] CA bundle only if the service makes outbound TLS calls, baked in or mounted at runtime; TLS private keys mounted at runtime, never baked in
- [ ] Exec-form `ENTRYPOINT`; app handles `SIGTERM`
- [ ] Dockerfile linted and image scanned in CI
- [ ] SBOM and provenance attached; images signed
- [ ] Runtime: dropped capabilities, `no-new-privileges`, read-only filesystem
- [ ] Rootless mode considered for security-sensitive hosts
- [ ] Memory/CPU/PID limits, health check, restart policy
- [ ] Meaningful image tags
- [ ] On Kubernetes: `runAsNonRoot` with a numeric UID, `RuntimeDefault` seccomp, probes instead of `HEALTHCHECK`, `preStop` sleep, no `subPath` for rotating Secrets

## References

[^1]: [Multi-stage builds — Docker Docs](https://docs.docker.com/build/building/multi-stage/)
[^2]: [Multi-stage builds (Docker concepts) — Docker Docs](https://docs.docker.com/get-started/docker-concepts/building-images/multi-stage-builds/)
[^3]: [Building best practices — Docker Docs](https://docs.docker.com/build/building/best-practices/)
[^4]: [Build cache invalidation — Docker Docs](https://docs.docker.com/build/cache/invalidation/)
[^5]: [Best Practices for Building Docker Images — Better Stack](https://betterstack.com/community/guides/scaling-docker/docker-build-best-practices/)
[^6]: [Distroless container images — GoogleContainerTools](https://github.com/GoogleContainerTools/distroless)
[^7]: [Docker Security Cheat Sheet — OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
[^8]: [Build secrets — Docker Docs](https://docs.docker.com/build/building/secrets/)
[^9]: [Rootless mode — Docker Docs](https://docs.docker.com/engine/security/rootless/)
[^10]: [Build attestations — Docker Docs](https://docs.docker.com/build/metadata/attestations/)
[^11]: [Base images — Docker Docs](https://docs.docker.com/build/building/base-images/)
[^12]: [distroless/base and distroless/static image contents — GoogleContainerTools](https://github.com/GoogleContainerTools/distroless/blob/master/base/README.md)
[^13]: [crypto/x509 root_unix.go — Go source](https://go.dev/src/crypto/x509/root_unix.go)
[^14]: [x509roots/fallback package — Go Packages](https://pkg.go.dev/golang.org/x/crypto/x509roots/fallback)
[^15]: [MustRunAsNonRoot should reject a pod if it has non-numeric USER — kubernetes/kubernetes #56503](https://github.com/kubernetes/kubernetes/pull/56503)
[^16]: [Enable seccomp for all workloads with a new v1.22 alpha feature — Kubernetes Blog](https://kubernetes.io/blog/2021/08/25/seccomp-default/)
[^17]: [browncoat: Examples with Kubernetes — Bret Fisher](https://github.com/BretFisher/browncoat/blob/master/README.md)
[^18]: [KEP-3960: Pod lifecycle sleep action — Kubernetes](https://www.k8s.dev/resources/keps/3960/)
[^19]: [Container Lifecycle Hooks — Kubernetes Docs](https://v1-34.docs.kubernetes.io/docs/concepts/containers/container-lifecycle-hooks)
[^20]: [Projected Volumes — Kubernetes Docs](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
