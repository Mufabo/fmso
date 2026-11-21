---
title: "The Twelve-Factor App: Fifteen Years On"
date: 2026-11-14
draft: true
description: "A practical walk through the twelve factors for building cloud-native applications: what each one means, how it maps to containers and Kubernetes today, where it has aged, and what the open source revival of the methodology is trying to fix."
summary: "Written in 2011 by Heroku's co-founder, the Twelve-Factor App is still the checklist cloud-native services are quietly judged against. Here is what each factor means, how it looks on Kubernetes, and where it shows its age."
tags: ["twelve-factor", "cloud native", "kubernetes", "architecture", "devops", "platform engineering"]
categories: ["Programming"]
---

In 2011 Adam Wiggins, a co-founder of Heroku, published a short website called *The Twelve-Factor App*.[^1] It distilled what the Heroku team had learned from watching hundreds of thousands of apps get built, deployed and broken on their platform into twelve rules for writing software-as-a-service applications.

Docker didn't exist yet. Kubernetes was three years away. Yet if you read the factors today, most of them describe exactly how a well-behaved container on Kubernetes is supposed to work. That is not a coincidence: the platforms that came later were built by people who had read it, and they baked many of the factors into their defaults.

In November 2024 Heroku handed the methodology over to an open source project, with the stated aim of updating the parts that have aged.[^2] This post goes through the twelve factors one at a time: what each says, what it looks like in practice now, and where it falls short.

## What problem it solves

The factors are about the contract between an **application** and the **platform** that runs it. If the app keeps its side of the contract, the platform can do useful things without knowing anything about the app's internals:

- build it the same way every time,
- run any number of copies of it,
- kill and restart those copies whenever it likes,
- move it between environments by changing configuration only,
- collect its logs and wire it up to databases and queues.

Every factor is in service of one of those capabilities. Keep that in mind and the rules stop looking arbitrary.

## The twelve factors

### I. Codebase: one codebase tracked in revision control, many deploys

One app, one repository (or one directory in a monorepo with its own build), deployed many times: a developer's laptop, staging, production. If two apps share code, that code becomes a library pulled in as a dependency. If one repository produces several independently deployed things, those are several apps.[^3]

The point is a one-to-one mapping between "what is running" and "what commit it came from". Every deploy should be traceable to a revision.

**Today:** this is the least controversial factor. Monorepos are fine as long as each deployable unit has its own build and release pipeline. Your image tag or a label like `org.opencontainers.image.revision` is how you trace a running Pod back to its commit.

### II. Dependencies: explicitly declare and isolate dependencies

Never rely on something happening to be installed on the machine. Declare every dependency in a manifest (`go.mod`, `package.json`, `requirements.txt`, `Cargo.toml`) and isolate the app from the system so nothing leaks in implicitly. The original text goes as far as saying you shouldn't shell out to system tools like `curl` or ImageMagick unless you vendor them.[^4]

**Today:** container images solved the isolation half for the whole process tree, including system libraries. The declaration half is still on you. Use lock files, pin base images by digest, and generate an SBOM so you can answer "are we affected by this CVE?" without guessing. A `FROM ubuntu:latest` followed by `apt-get install` without versions quietly breaks this factor.

### III. Config: store config in the environment

Config is everything that changes between deploys: database URLs, credentials for external services, per-environment hostnames. It must be strictly separated from code. The test the original gives is a good one: could you open-source the codebase right now without leaking a credential?[^5]

Twelve-factor's specific answer is **environment variables**, because they are language-agnostic and hard to commit by accident. It also argues against grouping config into named environments ("development", "staging", "production") inside the code, because that doesn't scale as you add more deploys. Each variable should be an independent knob.

```go
dbURL := os.Getenv("DATABASE_URL")
if dbURL == "" {
    log.Fatal("DATABASE_URL is required")
}
```

**Today:** the principle holds, but "store it in env vars" has aged worse than any other line in the document.

- Environment variables are a poor place for **secrets**. They are inherited by child processes, show up in crash dumps and `/proc/<pid>/environ`, and get printed by well-meaning debug endpoints. Mounting secrets as files (a Kubernetes Secret volume, or a secrets store CSI driver) and reading them at startup is generally safer.
- Env vars are read once at start. Changing one on Kubernetes means rolling the Pods. That's often what you want, but it's worth knowing.
- Large structured config (feature flags, routing tables) doesn't fit comfortably in flat strings.

A reasonable modern reading: **config comes from outside the build artifact, and the same image runs in every environment.** Env vars, mounted files and a config service all satisfy that. Baking `config.prod.yaml` into the image does not.

### IV. Backing services: treat backing services as attached resources

A backing service is anything the app talks to over the network: a database, a cache, a message broker, an SMTP server, an object store, a third-party API. The app should make **no distinction between local and third-party services**. Swapping a self-hosted PostgreSQL for a managed one should be a config change, not a code change.[^6]

**Today:** this is how most teams already think. The factor says less than it could about the fact that these resources need credentials, and those credentials need to rotate. Workload identity (an IAM role bound to a Kubernetes ServiceAccount, for example) is often a better answer than a password in a URL.

### V. Build, release, run: strictly separate build and run stages

Turning code into a running deploy has three stages:[^7]

1. **Build**: compile the code and fetch dependencies into an executable bundle.
2. **Release**: combine that build with a specific deploy's config. A release has a unique ID and is immutable; changes mean a new release.
3. **Run**: launch processes from a release.

You never change code at runtime. Rollback means picking an earlier release.

```text
  commit ──► build ──► image@sha256:… ──┐
                                        ├──► release v42 ──► run (N processes)
  config for prod ──────────────────────┘
```

**Today:** this maps neatly onto modern tooling. The build is your CI producing an image; the release is the image digest plus a set of manifests or Helm values, ideally committed to Git; the run is Kubernetes reconciling it. GitOps tools like Argo CD and Flux are an almost literal implementation of this factor. `kubectl exec` into a production Pod to patch a file is the thing it forbids.

### VI. Processes: execute the app as one or more stateless processes

Twelve-factor processes are **stateless and share nothing**. Anything that has to survive must live in a backing service. The local disk and memory are a scratchpad for a single request or job, never a store.[^8]

The factor calls out **sticky sessions** specifically as a violation: if a user's session lives in one process's memory, you can't freely kill that process or route the next request elsewhere. Put sessions in Redis, a database, or a signed cookie.

**Today:** this is the factor that makes horizontal scaling and rolling updates possible, and it's the one most often broken by accident. Common offenders: uploads written to the container filesystem, in-memory caches that the app assumes are complete, and background jobs tracked in a local variable.

Not everything is stateless, and that's fine. Databases, brokers and search clusters are stateful by nature; on Kubernetes they're StatefulSets or, more often, managed services. The factor is a rule for *your app*, not a claim that state doesn't exist.

### VII. Port binding: export services via port binding

The app is self-contained and serves HTTP (or another protocol) by binding to a port itself. It doesn't rely on being injected into an external web server like Apache with `mod_php` or a Java application server.[^9]

```go
port := os.Getenv("PORT")
if port == "" {
    port = "8080"
}
log.Fatal(http.ListenAndServe(":"+port, handler))
```

Because each app exposes a port, one app can become another app's backing service just by having its URL in config.

**Today:** essentially universal. Every container exposes a port, and Services, Ingresses and service meshes route to it. The factor's original concern (deploying WAR files into a shared Tomcat) is mostly history.

### VIII. Concurrency: scale out via the process model

Scale by running **more processes**, not by making one process bigger. Different kinds of work get different **process types**: `web` for HTTP, `worker` for background jobs, `clock` for scheduled tasks. Each type is scaled independently.[^10]

Internally a process may use threads, goroutines or an event loop; the factor doesn't care. What it cares about is that you never daemonize or write PID files. Let the platform's process manager handle restarts, output and crashes.

**Today:** Heroku's `Procfile` process types became separate Deployments, each with its own replica count and HorizontalPodAutoscaler. Running a web server and a job worker in the same container under `supervisord` is the pattern this factor steers you away from.

### IX. Disposability: maximize robustness with fast startup and graceful shutdown

Processes can be started or stopped at any moment. That only works if:[^11]

- **Startup is fast**, ideally seconds. Fast startup makes scaling and releases quick and lets processes move between machines cheaply.
- **Shutdown is graceful.** On `SIGTERM`, a web process stops accepting new requests, finishes the ones in flight and exits. A worker returns its current job to the queue.
- **Sudden death is survivable.** Hardware fails and processes get `SIGKILL`ed. Jobs should be idempotent or wrapped in transactions so a crash halfway through doesn't corrupt anything.

```go
srv := &http.Server{Addr: ":8080", Handler: handler}
go srv.ListenAndServe()

stop := make(chan os.Signal, 1)
signal.Notify(stop, syscall.SIGTERM, os.Interrupt)
<-stop

ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
defer cancel()
srv.Shutdown(ctx) // stop accepting, drain in-flight requests
```

**Today:** on Kubernetes this factor is non-negotiable, because the scheduler moves Pods during node drains, autoscaling and upgrades. The practical details: Kubernetes sends `SIGTERM`, waits `terminationGracePeriodSeconds` (30 by default), then sends `SIGKILL`. Make sure your app is PID 1 or behind an init that forwards signals (a shell-form `CMD` swallows them). Readiness probes stop new traffic, and a short `preStop` sleep covers the gap while endpoints update.

### X. Dev/prod parity: keep development, staging and production as similar as possible

The original identifies three gaps between development and production:[^12]

| Gap | Traditional | Twelve-factor |
|---|---|---|
| Time | Weeks between writing code and deploying it | Hours or minutes |
| Personnel | Developers write, ops deploy | Developers deploy and watch their code in production |
| Tools | SQLite and nginx locally, PostgreSQL and Apache in prod | The same services everywhere |

The third gap is where it's most specific: resist using a "lightweight" substitute backing service locally, because small incompatibilities pile up into production bugs that never showed in testing.

**Today:** containers made the tools gap much cheaper to close. Docker Compose, Testcontainers and local Kubernetes clusters (kind, k3d) give you the real PostgreSQL in a few seconds. The time and personnel gaps are what continuous delivery and "you build it, you run it" set out to close, and that's arguably the factor's most lasting cultural influence.

### XI. Logs: treat logs as event streams

An app should never manage its own log files. It writes an unbuffered stream of events to **stdout**, and the execution environment collects, routes and stores it.[^13]

**Today:** this is exactly how container runtimes work: stdout and stderr are captured, and an agent such as Fluent Bit, Vector or the OpenTelemetry Collector ships them on. Two updates are worth making to the 2011 version:

- **Structure your logs.** One JSON object per line with consistent fields (level, timestamp, request ID, trace ID) is far more useful than free text.
- **Logs are one signal of three.** Metrics and traces matter as much, and the factor says nothing about them. This is one of the gaps the open source maintainers have said they want to fill.[^14]

### XII. Admin processes: run admin/management tasks as one-off processes

Database migrations, one-off scripts and REPL sessions run as **one-off processes** in an environment identical to the app's regular processes: same release, same code, same config. Admin code ships with the app so it can't drift out of sync.[^15]

**Today:** a Kubernetes Job (or a Helm hook, or an Argo CD sync hook) that runs the same image with a different command is the textbook implementation. Running migrations from a developer's laptop against production with whatever version of the code they happen to have checked out is the anti-pattern.

## Quick reference: the factors on Kubernetes

| # | Factor | Typical Kubernetes-era implementation |
|---|---|---|
| I | Codebase | One repo or monorepo path per deployable; image labelled with commit SHA |
| II | Dependencies | Lock files; container image pinned by digest; SBOM |
| III | Config | ConfigMaps and env vars; Secrets mounted as files; same image everywhere |
| IV | Backing services | Connection details in config; managed services; workload identity |
| V | Build, release, run | CI builds image → manifests/values in Git → GitOps controller applies |
| VI | Processes | Deployments with no local state; sessions in Redis or cookies |
| VII | Port binding | `containerPort` + Service |
| VIII | Concurrency | One Deployment per process type; HPA per Deployment |
| IX | Disposability | `SIGTERM` handling, readiness probes, `preStop`, grace period |
| X | Dev/prod parity | Compose, Testcontainers, kind; same images in every environment |
| XI | Logs | JSON to stdout; collector DaemonSet; OpenTelemetry |
| XII | Admin processes | Kubernetes Jobs or deploy hooks running the release image |

## Where it has aged

The methodology is a product of its moment: one app, a few backing services, deployed on a PaaS. Several things are missing or awkward now.

**Secrets.** Discussed under factor III. "Put credentials in environment variables" is now widely seen as the weakest piece of advice in the document.

**Observability.** Logs alone don't cut it for distributed systems. Metrics, traces and health endpoints are expected parts of the app–platform contract.

**Security and identity.** The original barely mentions authentication or authorization between services. In a world of zero trust networking and short-lived workload credentials, that's a big omission.

**API contracts.** With dozens of services calling each other, how an app versions and documents its interface matters as much as how it's deployed.

**More than one app.** As one of the open source maintainers put it in an interview, the original is clear about one app with backing services and a data store, which no longer describes most cloud systems.[^14]

Kevin Hoffman's *Beyond the Twelve-Factor App* (O'Reilly, 2016) was an early attempt to patch these holes. It revisits the original twelve and adds three more, **API first**, **telemetry** and **authentication and authorization**, which is why you'll sometimes see people refer to "fifteen-factor apps".[^16]

It's also worth remembering that the factors were written for stateless web services. They are a poor fit for databases, desktop software, embedded systems, batch pipelines with large local working sets, or anything that has to keep durable local state. Applying them blindly to those leads to strange architecture.

## The open source revival

On 12 November 2024, Heroku (now part of Salesforce) announced that Twelve-Factor had become an open source project, saying the movement was bigger than one company.[^17] The maintainers' own announcement describes the concepts as still valuable but many of the details as outdated, and sets out the goal of turning it into a living document updated through ordinary pull requests and reviews.[^2] Engineers from AWS, Google Cloud and other companies joined as early maintainers. The source lives on GitHub,[^18] and discussion happens on the project's Discord.

The stated plans include updating the details for containers and cloud-native platforms, clarifying what makes an app twelve-factor (balancing restrictions on the app against what the platform gains in return), and covering telemetry properly.[^2][^14] An early blog post framed twelve-factor as an **application–platform interface**: a narrow contract between the people writing apps and the people building platforms, centred on how the platform hands configuration to application code rather than on the details of any particular orchestrator.[^19] Seen that way, the factors were always about keeping the channels between app and platform few and well defined (config in, logs out, a port, signals), and that framing is arguably the clearest statement of what they're for.

Progress has been slow. As of this writing, the project's December 2024 update reported mostly formatting fixes and organizational groundwork, its blog has been quiet since April 2025, and the factor pages on 12factor.net still read as they did before the project was opened up.[^20] If you want to influence the next version, the repository's issues are where the discussion is.

## Is it still worth reading?

Yes. It's short (you can read the whole thing in half an hour), and most of it describes behaviour the platforms you already use assume. When a service misbehaves on Kubernetes, the cause is very often a broken factor: state on local disk (VI), an ignored `SIGTERM` (IX), config baked into the image (III), or a migration run by hand (XII).

Treat it as a set of questions rather than a checklist:

- Can I build this exact artifact again from a commit?
- Can I run the same artifact in every environment, changing only config?
- Can the platform kill any instance at any time without losing data?
- Can I scale by adding instances?
- Does the platform get everything it needs (logs, health, signals) through a few well-defined channels?

If the answer to all five is yes, you have the substance of a twelve-factor app, whether or not you followed every line of the 2011 text.

## References

[^1]: [The Twelve-Factor App](https://12factor.net/), Adam Wiggins.
[^2]: [Twelve-Factor App Methodology is now Open Source](https://12factor.net/blog/open-source-announcement), Yehuda Katz, 12factor.net blog, November 2024.
[^3]: [I. Codebase](https://12factor.net/codebase), The Twelve-Factor App.
[^4]: [II. Dependencies](https://12factor.net/dependencies), The Twelve-Factor App.
[^5]: [III. Config](https://12factor.net/config), The Twelve-Factor App.
[^6]: [IV. Backing services](https://12factor.net/backing-services), The Twelve-Factor App.
[^7]: [V. Build, release, run](https://12factor.net/build-release-run), The Twelve-Factor App.
[^8]: [VI. Processes](https://12factor.net/processes), The Twelve-Factor App.
[^9]: [VII. Port binding](https://12factor.net/port-binding), The Twelve-Factor App.
[^10]: [VIII. Concurrency](https://12factor.net/concurrency), The Twelve-Factor App.
[^11]: [IX. Disposability](https://12factor.net/disposability), The Twelve-Factor App.
[^12]: [X. Dev/prod parity](https://12factor.net/dev-prod-parity), The Twelve-Factor App.
[^13]: [XI. Logs](https://12factor.net/logs), The Twelve-Factor App.
[^14]: [Heroku Moved Twelve-Factor Apps to Open Source. What's Next?](https://thenewstack.io/heroku-moved-twelve-factor-apps-to-open-source-whats-next/), The New Stack.
[^15]: [XII. Admin processes](https://12factor.net/admin-processes), The Twelve-Factor App.
[^16]: [Beyond the Twelve-Factor App](https://www.oreilly.com/library/view/beyond-the-twelve-factor/9781492042631/), Kevin Hoffman, O'Reilly, 2016.
[^17]: [Heroku Open Sources the Twelve-Factor App Definition](https://www.heroku.com/blog/heroku-open-sources-twelve-factor-app-definition/), Heroku blog, November 2024.
[^18]: [twelve-factor/twelve-factor](https://github.com/twelve-factor/twelve-factor), GitHub.
[^19]: [Narrow Conduits and the Application-Platform Interface](https://12factor.net/blog/narrow-conduits), Vish Abrams, 12factor.net blog, November 2024.
[^20]: [Twelve-Factor blog](https://12factor.net/blog), 12factor.net.
