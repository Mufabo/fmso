---
title: "Best Practices When Building Images for Kubernetes Clusters"
date: 2026-11-26
draft: false
tags: ["kubernetes", "docker", "containers", "devops", "security", "go"]
summary: "A practical checklist for building small, safe container images that run well on Kubernetes, from the Dockerfile to the pod spec, with examples in Go."
---

You build the image with Docker, but on a cluster the image is run by Kubernetes, through containerd or CRI-O, not by Docker (Kubernetes removed its built-in Docker support in 1.24[^1]). That changes what "a good image" means. The size, the build cache, and the supply chain matter exactly as much as before. But everything that a `docker run` command line would normally provide, like the user, the capabilities, the health check, the restart behaviour, and the secrets, now comes from the pod spec. The image has to be built so that it works under those settings, ideally under the strictest ones Kubernetes offers.

This post collects the practices I keep coming back to, in four parts: **building the image**, **securing it**, **handling certificates**, and **running it on the cluster**. The examples use a small Go HTTP service, but the ideas carry over to any stack.

## Building the image

### Use multi-stage builds

A single-stage build drags compilers, package managers, and build caches into the final image. Multi-stage builds let you use several `FROM` statements in one Dockerfile, compile in one stage, and copy only the finished artifacts into a lean runtime stage[^2]. Docker's own guidance recommends them for all types of applications[^3], and Go is the ideal case: the whole toolchain stays in the build stage and the runtime stage gets a single static binary.

```dockerfile
# syntax=docker/dockerfile:1

# --- build stage ---
FROM golang:1.27 AS build
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

`CGO_ENABLED=0` produces a statically linked binary that doesn't need libc in the runtime image. `-trimpath` and `-ldflags="-s -w"` strip local paths and debug symbols to shrink it further. The runtime stage starts from `scratch`, an empty image, so the final image is the binary plus a CA certificate bundle and nothing else. Small images matter more on a cluster than on a laptop: every node that schedules a pod has to pull the image first, so image size shows up directly in how fast pods start and how fast a Deployment can scale out.

A bonus with BuildKit: when you build a specific `--target`, it only builds the stages that target actually depends on[^2], and independent stages can run in parallel[^4].

### Order instructions for the build cache

Docker walks the Dockerfile top to bottom and reuses a cached layer for each instruction until something changes; from that point on, every subsequent layer is rebuilt[^5]. So put things that change rarely first and things that change constantly last:

1. Base image and system packages
2. Dependency manifests (`go.mod`, `go.sum`)
3. Dependency download (`go mod download`)
4. Application source and the build itself

Copying only `go.mod` and `go.sum` before downloading modules (as in the example above) means a code change doesn't re-download every dependency. Cache mounts go one step further: they keep Go's module and build caches between builds, so even when a layer has to be rebuilt, only the packages that changed get recompiled. Replace the download and build steps with this block, which shares the same module cache between both commands:

```dockerfile
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -tags timetzdata -ldflags="-s -w" -o /out/server ./cmd/server
```

Cluster images are usually built in CI, and CI runners are often thrown away after each job, taking the local cache with them. To keep the layer cache between runs, export it to a registry or your CI provider's cache with `--cache-to` and read it back with `--cache-from`[^6]. With the default `docker` build driver, the `registry` and `gha` (GitHub Actions) backends only work if the containerd image store is enabled[^6].

### Keep the build context small with `.dockerignore`

Everything in the build directory is sent to the builder as context. A `.dockerignore` file keeps logs, `.git`, build output, and local secrets out of the context, which speeds up builds and prevents sensitive files from accidentally ending up in an image[^7]. In a Kubernetes repo, this includes your manifests and Helm charts: they have no business inside the image.

```text
.git
bin/
tmp/
deploy/
charts/
*.log
*.test
coverage.out
.env
.env.*
```

### Start `FROM scratch`

Start from official or verified images, and pick the smallest one that works. For interpreted languages that usually means a `-slim` or Alpine variant. For a static Go binary you can go much further and ship an image with no operating system userland at all.

My default for Go services is `FROM scratch`. It is not really an image: `scratch` is a reserved name that tells the builder to start from nothing, so your first instruction becomes the first filesystem layer, and you can't pull it or run it[^8]. The result contains exactly what you copy in. There is no shell, no package manager, and no operating system distribution with its own package inventory to track, verify, and patch. What remains is your binary, its Go dependencies, and whatever files you copy in, such as the CA bundle. Scan the final image anyway, and check the binary's dependencies with `govulncheck` (see [Scan and lint](#scan-and-lint)).

The price is that you supply everything the binary quietly depends on. The Dockerfile at the top of this post already does, and each line is there for a reason:

- **CA certificates.** Without them every outbound HTTPS call fails with an unknown-authority error. That is the `COPY --from=build /etc/ssl/certs/ca-certificates.crt` line. On Kubernetes you can also let the cluster provide them; see [Certificates](#certificates) below.
- **Timezone data.** `time.LoadLocation` fails without it. Building with `-tags timetzdata` embeds it in the binary, so there is no file to copy.
- **Users.** There is no `/etc/passwd`, so `USER` must be numeric, hence `USER 65532:65532`. As it turns out, that is exactly what Kubernetes wants anyway (more below).
- **`/tmp`.** It doesn't exist. If the service writes temp files, mount an `emptyDir` volume at `/tmp`, as the example Deployment at the end does.
- **A truly static binary.** If the binary is dynamically linked it fails at startup with a misleading "no such file or directory", because the dynamic loader is missing. `CGO_ENABLED=0` takes care of that.

### When to reach for distroless instead

Distroless images are `scratch` with those gaps already filled. `gcr.io/distroless/static` adds exactly four things: CA certificates, an `/etc/passwd`, a `/tmp` directory, and timezone data[^9]. It is still only around 2 MiB[^10]. On top of that you get a `:nonroot` tag, `:debug` tags that include a BusyBox shell, and images signed with cosign[^10].

I switch to it in two cases. The first is cgo: a binary that links against glibc can't run on `scratch`, and `distroless/base` adds glibc and libssl (`base-nossl` adds glibc alone)[^9]. The second is when a team would rather have someone else maintain the certificate bundle than own those extra lines.

| | `scratch` | `distroless/static` |
|---|---|---|
| Size before your binary | 0 | ~2 MiB |
| CA certificates | Copy them in | Included |
| Timezone data | Copy or embed | Included |
| `/etc/passwd`, non-root user | Numeric `USER` only | `:nonroot` tag (set a numeric `runAsUser` too, see below) |
| `/tmp` | Missing | Included |
| Shell for debugging | None | `:debug` tag |
| cgo / glibc | Not possible | `base` variants |
| Staying current | Rebuild to refresh the copied certs and the Go toolchain | Rebuild to pick up new certs and tzdata |

Either way there is no shell, so `kubectl exec -it pod/api-... -- sh` won't work. On Kubernetes you don't need one in the image: `kubectl debug` attaches a temporary container with its own tools to the running pod, and `--target` puts it in the same process namespace as your container:

```bash
kubectl debug -it pod/<name> --image=busybox --target=api --profile=restricted
```

Two prerequisites. First, Pod Security admission checks ephemeral containers too, so in a namespace that enforces Restricted, a debug container without matching security settings is rejected. `--profile=restricted` applies a set of security settings compatible with the Restricted policy[^11]. In this post's manifest, the debug container also inherits the pod-level `runAsUser: 65532`, so BusyBox runs as the same user as the service. Second, `--target` has to be supported by the container runtime. Without that support, the debug container may not start at all, or it starts in its own process namespace and can't see the service's processes[^11].

That makes the `:debug` tags much less important on a cluster than they would be on a plain Docker host.

### Pin base image versions — deliberately

`FROM golang:latest` is a moving target. Pinning to a minor tag like `golang:1.27` still floats to the latest patch release, so you pick up fixes automatically. To fix the exact base image used by the build, pin by digest (`FROM golang:1.27@sha256:...`) and automate the bumps so you don't miss security patches. Docker's docs point to Docker Scout for base image updates[^4]; Renovate and Dependabot do the same job. For fully reproducible builds you also need to control the Dockerfile frontend: the `# syntax=docker/dockerfile:1` line in the example floats across `1.x` releases, so pin it to a specific version or digest too.

### Rebuild images often

An image is a snapshot: the base image and every dependency are frozen at build time. Rebuild regularly, not only when your code changes, so patches actually reach production[^4]. Two flags matter here, and they do different things:

```bash
# Check for a newer base image even if one is cached locally
docker build --pull -t registry.example.com/myapp:1.4.2 .

# Also ignore the layer cache and re-run every step
docker build --pull --no-cache -t registry.example.com/myapp:1.4.2 .
```

`--pull` refreshes the base image; `--no-cache` re-runs the build steps but does not pull a fresh base on its own[^4]. This matters even with a `scratch` runtime stage: a newer build image is what brings Go toolchain and standard library patches and a fresh CA bundle.

How you stay current depends on how you pin the build image:

- **Floating tag** (`golang:1.27`): a scheduled weekly CI build with `--pull` picks up the latest patch release on its own.
- **Pinned digest** (`golang:1.27@sha256:...`): `--pull` fetches exactly the same image every time, so a scheduled rebuild alone changes nothing. The update arrives as a digest bump from Renovate or Dependabot. Merge the bump, rebuild, then deploy the new image digest.

On a cluster a rebuild alone changes nothing: pods keep running the image they started with. The new image only reaches production when the Deployment points at it, which is one more reason to deploy by digest. A new digest in the manifest is an explicit, reviewable rollout; re-pushing a tag is not.

### Combine and clean up package installs

If a stage needs OS packages (for example, a cgo build that links against a C library), install them in a single `RUN`, skip recommended extras, and clean up in the same layer so the cleanup actually shrinks the image:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      libsqlite3-dev \
    && rm -rf /var/lib/apt/lists/*
```

Sorting multi-line package lists alphabetically makes diffs and duplicate-spotting easier[^4].

### Prefer `COPY` over `ADD`, and use exec-form `ENTRYPOINT`

`COPY` does exactly one obvious thing. Reach for `ADD` only when you need its extra behaviour (remote URLs, auto-extracting tarballs). Write `ENTRYPOINT ["/server"]` rather than `ENTRYPOINT /server` so your process is PID 1 and receives the `SIGTERM` that Kubernetes sends on shutdown directly, instead of having a shell swallow it. On `scratch` the exec form is the only one that works anyway, because there is no shell. How the app should react to that signal is covered under [graceful shutdown](#shut-down-gracefully) below.

### Tag images meaningfully, deploy by digest

`latest` tells you nothing about what's running, and on Kubernetes it has a side effect: an image with the `:latest` tag (or no tag) gets `imagePullPolicy: Always` by default, so every pod start goes to the registry[^12]. Tag with a version and/or the git commit SHA (`myapp:1.4.2`, `myapp:3f9c2ab`) so builds are traceable, and reference the image by digest in the manifest so the cluster runs exactly what you tested. You can stamp the version into the binary so the service can report it:

```dockerfile
ARG VERSION=dev
RUN CGO_ENABLED=0 go build -trimpath -tags timetzdata \
    -ldflags="-s -w -X main.version=${VERSION}" -o /out/server ./cmd/server
```

```bash
docker build --build-arg VERSION=1.4.2 -t registry.example.com/myapp:1.4.2 .
```

## Securing the image

The goal of this part is an image that runs under Kubernetes' **Restricted** Pod Security Standard, the strictest of the three built-in levels[^13]. OWASP recommends enforcing it cluster-wide[^14]. Restricted requires, among other things, a non-root user, no privilege escalation, all capabilities dropped, and an explicit seccomp profile, either the runtime's default (`RuntimeDefault`) or a custom one from the node (`Localhost`)[^13]. Those are pod spec settings, but whether your container still works under them is decided when you build the image.

### Run as a numeric, non-root user

By default a container runs as whatever user the image says, which is root unless you say otherwise. Configuring an unprivileged user is the most direct defence against privilege escalation[^14].

Kubernetes adds a twist. The pod setting `runAsNonRoot: true` makes the kubelet check that the container won't run as UID 0, but it can only check a numeric `USER`: an image with a named user such as `nonroot` is refused with a "cannot verify user is non-root" error unless the pod also sets `runAsUser`[^15]. So always put a number in `USER`. On `scratch` the line `USER 65532:65532` already does that. On a full distro image, create the user with a fixed ID and refer to it by number:

```dockerfile
RUN addgroup --system --gid 10001 app \
 && adduser --system --uid 10001 --ingroup app app
USER 10001:10001
```

That syntax is for Debian/Ubuntu-based images; on Alpine use `addgroup -S -g 10001 app && adduser -S -u 10001 -G app app`. Make sure the files the process needs are readable by that user, and listen on an unprivileged port like 8000 rather than 80, so the container needs no capabilities at all.

### Write nothing outside `/tmp`

The Restricted level doesn't require a read-only root filesystem, but it is a cheap win and worth designing for: with `readOnlyRootFilesystem: true`, an attacker who gets code execution can't modify the image's files or plant new ones in it. The volumes you mount, such as `/tmp`, stay writable, so keep those to the minimum. The image has to cooperate. The process should write only to paths that the pod mounts as volumes, typically an `emptyDir` at `/tmp`, and log to stdout instead of files. A `scratch` image is a natural fit, since there is nothing in it that expects to write anywhere.

### Never bake secrets into images

`ENV API_KEY=...` or `COPY .env .` puts the secret into a layer, where anyone who can pull the image can read it, even if a later layer deletes the file. `ARG` is no better: build arguments end up in the image history and in provenance attestations. Use BuildKit secret mounts for build-time secrets[^16]. The common Go case is fetching private modules. Combined with the cache mounts from earlier, the build stage looks like this:

```dockerfile
ENV GOPRIVATE=github.com/yourorg/*

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=secret,id=netrc,target=/root/.netrc \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=secret,id=netrc,target=/root/.netrc \
    CGO_ENABLED=0 go build -trimpath -tags timetzdata -ldflags="-s -w" -o /out/server ./cmd/server
```

```bash
docker build --secret id=netrc,src=$HOME/.netrc .
```

The `.netrc` file is available only during the `RUN` steps that mount it and is never written to a layer. `GOPRIVATE` isn't secret, so it can be a plain `ENV` in the build stage; it tells Go to fetch those modules directly instead of through the public proxy and checksum database. Both settings belong on the build step too, not only on the download. With cache mounts, the downloaded modules live in the cache mount rather than in a layer, and a fresh CI runner starts with an empty cache mount even when it imports the layer cache. Then the download step is skipped as cached, and `go build` has to fetch the modules itself.

Runtime secrets come from Kubernetes Secrets, mounted as files or injected as environment variables. Remember that Secrets are stored unencrypted in etcd unless the cluster enables encryption at rest[^14].

### Scan and lint

- Add a Dockerfile linter to your CI pipeline[^14]. [Hadolint](https://github.com/hadolint/hadolint) is the usual choice.
- Scan images for known CVEs with Docker Scout, Trivy, or Grype. For Go code, [`govulncheck`](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck) uses static analysis of your code's call graph to report the vulnerabilities in functions your code can actually reach, rather than everything in your dependency list.
- Keep the build hosts and the cluster nodes patched. A perfect image doesn't help if the kernel underneath it is vulnerable, because every container on a node shares that kernel[^14].

### Secure the supply chain

Scanning tells you what is in an image. Supply-chain measures let you prove where it came from. OWASP groups these under provenance, SBOMs, image signing, and a trusted registry[^14].

- **SBOM and provenance.** BuildKit can attach both as attestations at build time[^17]:

  ```bash
  docker buildx build --sbom=true --provenance=mode=max \
    -t registry.example.com/myapp:1.4.2 --push .
  ```

  The classic local image store drops attestations, so push straight to a registry or enable the containerd image store[^17].
- **Signing.** Sign the pushed image by digest with a tool such as [cosign](https://github.com/sigstore/cosign). With a `scratch` runtime stage the only upstream image left to trust is the build image, so pin that by digest.
- **Verification in the cluster.** Signing only helps if something checks the signature. An admission controller such as Sigstore's policy-controller or Kyverno can reject pods whose images aren't signed by your CI.
- **Trusted registry.** Store images in a registry with access controls, and deploy by digest rather than by mutable tag.

## Certificates

"Certificates in the container" covers two very different things, and they get opposite treatment.

### The CA trust bundle

The trust bundle is the file copied into the `scratch` image above. It holds public root certificates that your binary uses to *verify* the servers it connects to. Nothing in it is secret, so baking it into the image is fine. The baked-in bundle is frozen at build time and only changes when the build image does: on the next `--pull` rebuild with a floating tag, or with the next digest bump if you pin. Roots change rarely, but they do get added and distrusted, so this is one more reason to rebuild on a schedule.

On Kubernetes you have three alternatives, and they can make the image even smaller:

- **No outbound TLS, no bundle.** If the service only serves plain HTTP behind an Ingress, and any outbound TLS is handled by a service mesh sidecar, the binary never verifies a certificate and you can drop the `COPY` line.
- **Mount it from a ConfigMap.** Publish the bundle as a ConfigMap, which cert-manager's trust-manager can keep in sync across namespaces, mount it as a volume, and point `SSL_CERT_FILE` at the file. On Linux, Go reads `SSL_CERT_FILE` and `SSL_CERT_DIR` to override its default search paths[^18]. This is the natural fit for an internal CA, which changes on your PKI's schedule rather than your release schedule, and updating the bundle no longer needs a rebuild. Be careful with what the bundle contains. `SSL_CERT_FILE` replaces only the default certificate *file*[^19]; Go still reads every file in its default certificate directories (`/etc/ssl/certs` and `/etc/pki/tls/certs`), unless `SSL_CERT_DIR` points it elsewhere[^18]. In the `scratch` example, the baked-in bundle sits in `/etc/ssl/certs`, so as long as you keep that `COPY` line, a mounted internal CA adds to the public roots. Once you drop the `COPY` line to stop rebuilding for certificate updates, the mounted file is the only root source left, and a bundle that holds only your internal CA leaves every public HTTPS endpoint untrusted. If the service calls both, publish a combined bundle with the public roots plus your internal CA; trust-manager can include the public roots for you with a `useDefaultCAs: true` source next to your own CA[^20]. Go loads the system roots once per process, so a changed bundle takes effect on the next pod restart. Kubernetes also has a native `clusterTrustBundle` projected volume for this, which became stable in 1.37[^21][^22]. On 1.37 and later it can replace the ConfigMap; many clusters, especially managed ones, are still on older versions, so the ConfigMap remains the portable option.

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

- **Embed it in the binary.** A blank import of `golang.org/x/crypto/x509roots/fallback` compiles a root set into the binary, used only when the system provides none[^23]:

  ```go
  import _ "golang.org/x/crypto/x509roots/fallback"
  ```

  Rotation then means bumping a Go module: the package documentation says it must be kept up to date and recommends `govulncheck` to flag new versions[^23].

The two last options combine well. A ConfigMap-only image is no longer self-contained: without the mount, every outbound TLS call fails, including when someone runs it locally with `docker run`. With `x509roots/fallback` imported as well, the mounted bundle wins on the cluster, because Go uses the embedded roots only when the system provides none[^23], and the binary still works anywhere else. The fallback is all-or-nothing, though: when a bundle is mounted, the embedded roots are not merged into it, so it doesn't fix an internal-only bundle.

### The service's own certificate and key

If the service terminates TLS itself, the private key is a secret and must never be in the image: anyone who can pull the image would have it, and every renewal would mean a rebuild. cert-manager can issue the certificate into a Secret that you mount as a volume.

Don't mount it with `subPath`: a `subPath` mount never receives updates, which silently breaks rotation[^22]. A normal Secret volume is updated in place after a short delay. Go doesn't reload the files on its own, so restart the pods once the new pair is in place, with `kubectl rollout restart deployment/api`. How much capacity you keep during the rollout depends on the Deployment's update strategy. By default up to 25% of the pods may be unavailable at once[^24], so old pods can be removed before their replacements are ready. To keep full capacity throughout, set `maxUnavailable: 0` and allow a surge (`maxSurge` can't also be 0)[^24], as the example Deployment below does. Then an old pod is removed only after a new one passes its readiness probe. A readiness probe on plain HTTP doesn't check the certificate, though, so after the rollout confirm that the pods serve the new one, for example with `openssl s_client`. If an ingress sits in front, a check through it may only show the ingress's own certificate, so connect to the pods directly, for example through `kubectl port-forward`.

Often the simplest answer is to keep TLS out of the container entirely: an Ingress controller terminates external TLS and renews certificates automatically, a service mesh can handle mTLS between pods, and the Go service speaks plain HTTP. Then the container holds no private key at all.

## Running it on the cluster

Everything above produces the image. What changes compared with running it on a Docker host is where the runtime settings live. Kubernetes doesn't read `docker run` flags, so each one moves into the pod spec, and a few Docker features are ignored or replaced. If you test locally with `docker run`, this table maps one to the other:

| Docker | Kubernetes |
|---|---|
| `--user` / image `USER` | `securityContext.runAsUser`, `runAsNonRoot: true` |
| `--cap-drop all` | `securityContext.capabilities.drop: ["ALL"]` |
| `--security-opt=no-new-privileges` | `securityContext.allowPrivilegeEscalation: false` |
| `--read-only` | `securityContext.readOnlyRootFilesystem: true` |
| Default seccomp profile (automatic) | `seccompProfile.type: RuntimeDefault` (not automatic, see below) |
| `--tmpfs /tmp` | `emptyDir` volume mounted at `/tmp` |
| `--memory`, `--cpus` | `resources.requests` and `resources.limits` |
| `--pids-limit` | Set per node by the cluster admin, not per pod |
| `--restart unless-stopped` | Handled by the Deployment |
| `-p 127.0.0.1:8000:8000` | ClusterIP Service for internal access, with a NetworkPolicy to limit callers |
| `HEALTHCHECK` | Readiness and liveness probes |

### Turn on seccomp explicitly

Docker applies its default seccomp profile to every container. Kubernetes runs pods `Unconfined` unless the pod sets a profile or the cluster admin turns on the kubelet's seccomp-default setting[^25]. The Restricted Pod Security level forbids both `Unconfined` and a missing profile, and accepts `RuntimeDefault` or `Localhost`[^13]. This post's manifest sets `RuntimeDefault` once at the pod level, and every container inherits it; a `Localhost` profile tailored to your binary is stricter but has to be installed on every node. A static Go binary on `scratch` uses only ordinary system calls and runs fine under it.

### Set requests and limits

An unbounded container can starve the node. Set CPU and memory requests so the scheduler knows what the pod needs, and limits so a misbehaving pod can't take over the node[^14].

Since Go 1.25 the runtime sets `GOMAXPROCS` from the container's CPU limit automatically. It only looks at the CPU *limit*, not the request[^26], so in a pod without a CPU limit, `GOMAXPROCS` still defaults to all CPUs the process may run on, which is usually every CPU on the node. Memory is still worth a thought: `GOMEMLIMIT` is a soft limit on memory managed by the Go runtime, which makes the garbage collector work harder as the heap approaches it. For a 256 MiB limit, `GOMEMLIMIT=230MiB` is a starting point. Check total memory under load and leave room for memory outside the Go runtime, because exceeding the container limit gets the pod OOM-killed.

### Use probes, not `HEALTHCHECK`

Kubernetes does not read the image's `HEALTHCHECK`; it only runs the probes declared in the pod spec[^27]. That works in favour of `scratch`: an `httpGet` probe is sent by the kubelet from outside the container, so the image needs no `curl`, `wget`, or shell. The two probes answer different questions, so give them separate endpoints:

- **Readiness** (`/readyz`): can this pod accept traffic right now? It may check dependencies the service can't work without, such as its database. A failing readiness probe only removes the pod from the Service endpoints; nothing restarts.
- **Liveness** (`/livez`): is the process itself broken in a way that a restart fixes, such as a deadlock? Keep it local and cheap, and never check dependencies here. If a database outage failed the liveness probe, Kubernetes would restart every API pod at once, which helps nobody and turns a database problem into an API outage.

A single `/health` endpoint is fine only if it checks nothing beyond the process answering HTTP. Then it works as a liveness probe, but as a readiness probe it can send traffic to a pod that isn't ready yet.

### Shut down gracefully

When a pod is deleted, Kubernetes gives it `terminationGracePeriodSeconds`, 30 seconds by default, before killing it. The countdown starts as soon as termination begins: a `preStop` hook, if there is one, runs first and uses part of that time, and only then does the container get `SIGTERM`[^28]. Your app has to handle that signal and drain in-flight requests. In Go that is a few lines:

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, os.Interrupt)
defer stop()

serveErr := make(chan error, 1)
go func() { serveErr <- srv.ListenAndServe() }()

select {
case err := <-serveErr:
    if !errors.Is(err, http.ErrServerClosed) {
        log.Fatal(err)
    }
case <-ctx.Done():
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 8*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Fatal(err)
    }
    if err := <-serveErr; !errors.Is(err, http.ErrServerClosed) {
        log.Fatal(err)
    }
}
```

There is a catch. The pod is removed from the Service endpoints at the same time as it gets `SIGTERM`, and that removal takes a moment to reach every node and load balancer. For a few seconds, traffic can still arrive after the server has stopped accepting connections.

The usual fix is a `preStop` hook that sleeps before the signal is sent, but `sh -c "sleep 5"` doesn't exist in a `scratch` image. Kubernetes now has a built-in sleep action that the kubelet runs itself, so it works with no binaries in the image[^29][^28]. It reached beta in 1.30 and stable in 1.34[^29]. On older clusters, have the Go service keep serving for a few seconds after `SIGTERM` before calling `Shutdown`. Either way, the sleep plus the shutdown timeout has to fit inside the grace period: 5 + 8 seconds against 30 in the example below.

### Log to stdout/stderr

Write logs to standard output instead of files inside the container. The container runtime collects them, `kubectl logs` shows them, and whatever log shipper the cluster runs picks them up without extra configuration. It also keeps the container compatible with a read-only filesystem. Go's `log/slog` with a JSON handler on `os.Stdout` is all you need.

### One concern per container, state outside it

Pods are meant to be disposable: stoppable, destroyable, and replaceable with minimal setup[^4]. Run one main process per container and put helpers in separate containers of the same pod rather than baking a cron daemon or proxy into the image. Anything written to the container's filesystem disappears when the pod is replaced, so databases, uploads, and anything else that must survive go into a PersistentVolumeClaim or, better, a managed service outside the pod.

### Limit network access

A ClusterIP Service gives the API an internal endpoint, but by default every pod in the cluster can reach it. To limit access to intended callers, such as a frontend or the ingress controller, add an ingress NetworkPolicy that selects the API pods and allows traffic only from those callers. This requires a network plugin that enforces NetworkPolicy; without one the policy is silently ignored[^14].

### Don't mount an API token you don't use

Every pod gets a Kubernetes API credential by default. A web service that never talks to the Kubernetes API doesn't need it, and an attacker who gets in would. Set `automountServiceAccountToken: false` unless the service calls the API.

### Example manifests

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0   # keep two replicas available during a healthy rollout
      maxSurge: 1
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
            httpGet: { path: /readyz, port: 8000 }
          livenessProbe:
            httpGet: { path: /livez, port: 8000 }
          lifecycle:
            preStop:
              sleep: { seconds: 5 }   # no shell needed; stable since 1.34
          volumeMounts:
            - { name: tmp, mountPath: /tmp }
      volumes:
        - name: tmp
          emptyDir: { medium: Memory }
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
      targetPort: 8000
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-frontend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8000
```

Setting `runAsUser` in the Deployment duplicates the image's `USER`, but it means the pod stays non-root even if someone later builds the image without it. Note that the NetworkPolicy port is the container port (8000), not the Service port, because the policy applies to traffic arriving at the pods.

## Quick checklist

**Image**

- [ ] Multi-stage build; final stage contains only the binary
- [ ] `go.mod`/`go.sum` copied and modules downloaded before source code; build cache kept between CI runs
- [ ] `.dockerignore` excludes `.git`, build output, manifests, `.env`, logs
- [ ] Runtime stage is `FROM scratch` (distroless if you need cgo); build image and frontend pinned
- [ ] Floating build tag: rebuilt on a schedule with `--pull`; pinned digest: automated bumps, each followed by a rebuild and deploy
- [ ] Numeric non-root `USER`, unprivileged port, writes only to `/tmp`
- [ ] No secrets in `ENV`, `ARG`, or copied files
- [ ] Exec-form `ENTRYPOINT`; app handles `SIGTERM`
- [ ] Dockerfile linted and image scanned in CI
- [ ] SBOM and provenance attached; images signed; tagged with version or commit SHA

**Cluster**

- [ ] Deployed by digest, signatures verified at admission
- [ ] Restricted Pod Security: `runAsNonRoot`, no privilege escalation, all capabilities dropped, `RuntimeDefault` (or `Localhost`) seccomp
- [ ] Read-only root filesystem with an `emptyDir` at `/tmp`
- [ ] CPU/memory requests and limits; `GOMEMLIMIT` below the memory limit
- [ ] Separate readiness (`/readyz`) and liveness (`/livez`) probes instead of `HEALTHCHECK`; liveness checks no dependencies
- [ ] `preStop` sleep, shutdown timeout inside the grace period
- [ ] CA bundle baked in, mounted from a ConfigMap, or not needed; TLS keys only from Secrets, never via `subPath`
- [ ] NetworkPolicy limits callers; service account token not mounted

## References

[^1]: [Updated: Dockershim Removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/), Kubernetes Blog.
[^2]: [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/), Docker Docs.
[^3]: [Multi-stage builds (Docker concepts)](https://docs.docker.com/get-started/docker-concepts/building-images/multi-stage-builds/), Docker Docs.
[^4]: [Building best practices](https://docs.docker.com/build/building/best-practices/), Docker Docs.
[^5]: [Build cache invalidation](https://docs.docker.com/build/cache/invalidation/), Docker Docs.
[^6]: [Cache storage backends](https://docs.docker.com/build/cache/backends/), Docker Docs.
[^7]: [Best Practices for Building Docker Images](https://betterstack.com/community/guides/scaling-docker/docker-build-best-practices/), Better Stack.
[^8]: [Base images](https://docs.docker.com/build/building/base-images/), Docker Docs.
[^9]: [distroless/base and distroless/static image contents](https://github.com/GoogleContainerTools/distroless/blob/master/base/README.md), GoogleContainerTools.
[^10]: [Distroless container images](https://github.com/GoogleContainerTools/distroless), GoogleContainerTools.
[^11]: [Debug Running Pods: Debugging profiles](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#debugging-profiles), Kubernetes Docs.
[^12]: [Images](https://kubernetes.io/docs/concepts/containers/images/), Kubernetes Docs.
[^13]: [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/), Kubernetes Docs.
[^14]: [Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html), OWASP.
[^15]: [MustRunAsNonRoot should reject a pod if it has non-numeric USER (#56503)](https://github.com/kubernetes/kubernetes/pull/56503), kubernetes/kubernetes on GitHub.
[^16]: [Build secrets](https://docs.docker.com/build/building/secrets/), Docker Docs.
[^17]: [Build attestations](https://docs.docker.com/build/metadata/attestations/), Docker Docs.
[^18]: [crypto/x509 root_unix.go](https://go.dev/src/crypto/x509/root_unix.go), Go source.
[^19]: [crypto/x509: SystemCertPool](https://pkg.go.dev/crypto/x509#SystemCertPool), Go Packages.
[^20]: [trust-manager: Sources](https://cert-manager.io/docs/trust/trust-manager/#sources), cert-manager Docs.
[^21]: [Kubernetes v1.37: Garhwal](https://kubernetes.io/blog/2026/08/26/kubernetes-v1-37-release/), Kubernetes Blog, 26 August 2026.
[^22]: [Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/), Kubernetes Docs.
[^23]: [x509roots/fallback package](https://pkg.go.dev/golang.org/x/crypto/x509roots/fallback), Go Packages.
[^24]: [Deployments: Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy), Kubernetes Docs.
[^25]: [Restrict a Container's Syscalls with seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/), Kubernetes Docs.
[^26]: [Go 1.25 Release Notes](https://go.dev/doc/go1.25), The Go Authors.
[^27]: [browncoat: Examples with Kubernetes](https://github.com/BretFisher/browncoat/blob/master/README.md), Bret Fisher.
[^28]: [Container Lifecycle Hooks](https://v1-34.docs.kubernetes.io/docs/concepts/containers/container-lifecycle-hooks), Kubernetes Docs.
[^29]: [KEP-3960: Pod lifecycle sleep action](https://www.k8s.dev/resources/keps/3960/), Kubernetes.
