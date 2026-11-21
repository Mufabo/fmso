---
title: 'Open source alternatives to GitHub and GitHub Actions'
date: 2026-10-24
draft: true
tags: [git, ci/cd, self-hosting, open source]
description: 'Open source forges (Forgejo, Gitea, GitLab CE, SourceHut, Radicle and more) and CI systems that can replace GitHub and GitHub Actions, with a comparison and recommendations.'
---

This blog doesn't live on GitHub. The source sits on
[Codeberg](https://codeberg.org/), a Forgejo Actions job triggered by a
push turns it into HTML, and Codeberg Pages serves the result. Nothing about
that was hard to set up, and that's the point of this post: GitHub is a
very good default, but it isn't the only option for hosting code or
running CI, and the open source options have become quite good.

Here is the landscape as I see it, in two parts: where your repositories
live, and what runs your pipelines.

## Why look elsewhere at all

GitHub is a proprietary platform owned by Microsoft. That's fine for a lot
of people. You might still want an alternative if:

- you want to **self-host**, because of compliance, data residency, or an
  air-gapped network;
- you'd like your projects' home to be run by a **non-profit or a
  community** rather than a company that can change its terms and pricing;
- you'd rather not feed your code into someone else's product decisions;
- or you just like knowing that the tools you depend on could be forked
  if their stewards went wrong.

None of these is a moral obligation. They are trade-offs, and it helps to
know what's on the other side.

## Code forges

A forge is the whole package: Git hosting, issues, pull requests, wikis,
releases, and usually CI. These are the open source ones worth knowing.

### Forgejo

[Forgejo](https://forgejo.org/) is a lightweight forge written in Go. It
started in 2022 as a soft fork of Gitea, after Gitea's trademarks and
domains moved to a for-profit company, and became a hard fork in
February 2024.[^1] It is developed under the umbrella of Codeberg e.V., a
German non-profit. Since v9.0 it has been licensed under the GPL v3 or
later; earlier releases were MIT, like Gitea.[^2]

It's a single binary, runs happily on a small VPS or a Raspberry Pi, and
feels familiar to anyone who has used GitHub: same repository layout,
pull requests, issues, labels, milestones, a package registry. Work on
**federation** (letting instances talk to each other via ActivityPub) is
ongoing, with the long-term idea that you could star or follow a project
on another server without an account there.

If you don't want to run it yourself, [Codeberg](https://codeberg.org/)
is the flagship public instance, free for open source projects and
funded by donations and memberships.

### Gitea

[Gitea](https://about.gitea.com/) is Forgejo's parent, and the two are
still similar day to day. Gitea is MIT-licensed and backed by a company,
Gitea Ltd., which also sells an enterprise edition and hosted service.
If you want a vendor you can sign a support contract with, that's a
point in its favour; if community governance matters more to you,
Forgejo is the one to pick.

### GitLab Community Edition

[GitLab](https://about.gitlab.com/) is the heavyweight. The Community
Edition is MIT-licensed and self-hostable, and it covers far more than
code hosting: built-in CI/CD, a container registry, issue boards,
environments, and more. GitLab is *open core*, though: a lot of features
live only in the paid tiers, and the boundary moves over time.

It is also big. Expect to give it several gigabytes of RAM and to spend
real time on upgrades. For a team that wants one integrated platform,
it's the closest like-for-like replacement for GitHub plus Actions. For
a personal server, it's usually more than you need.

### SourceHut

[SourceHut](https://sourcehut.org/) takes a different philosophy:
email-driven development, no JavaScript in the interface, and a set of
small, separate services (`git.sr.ht`, `todo.sr.ht`, `lists.sr.ht`,
`builds.sr.ht`) instead of one monolith. Patches are sent to mailing
lists, the way the Linux kernel works. Everything is AGPL and
self-hostable; the hosted version at sr.ht is paid.

If you like Git's original workflow, you'll feel at home. If your
contributors expect a "Create pull request" button, expect friction.

### The rest of the list

- **[OneDev](https://onedev.io/)** — a Java-based, MIT-licensed forge
  with built-in CI/CD, a visual job editor, and good code search.
- **[Gogs](https://gogs.io/)** — the ancestor of Gitea. Still around,
  minimal, and slower-moving.
- **[Radicle](https://radicle.xyz/)** — a peer-to-peer forge. There's no
  central server at all: repositories, issues and patches are replicated
  between nodes over a gossip protocol. Experimental in feel, but the
  most radical answer to "who controls my project?".

## CI/CD: replacing GitHub Actions

The other half of the lock-in is CI. A `.github/workflows/` folder full
of YAML is a real migration cost, so compatibility matters.

### Forgejo Actions and Gitea Actions

Both forges ship a CI system that deliberately mimics GitHub Actions.
Workflows live in `.forgejo/workflows/` (or `.gitea/workflows/`), use
the same YAML schema, and are executed by a separate runner process
derived from [act](https://github.com/nektos/act). Many existing actions
work unchanged.

This site's own deployment is a good example. `actions/checkout` works
by its short name, and two more actions, one from GitHub and one from
Codeberg, are pulled in by full URL:

```yaml
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: codeberg-tiny
    steps:
      - uses: actions/checkout@v4
      - uses: https://github.com/peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.154.5'
          extended: true
      - run: hugo --minify
      - uses: https://codeberg.org/git-pages/action@v2
        with:
          site: 'https://fatih.macht.sachen.online/'
          server: 'codeberg.page'
          token: ${{ forge.token }}
          source: public/
```

`forge.token` is injected automatically, so there's no secret to manage.
The `server:` line is only needed because this site uses a custom domain;
with a plain `*.codeberg.page` address you'd leave it out.

The differences are in the edges: `runs-on` names *your* runners'
labels rather than GitHub's hosted images, some GitHub-specific
contexts and APIs don't exist, and anything that calls the GitHub API
directly will need rework. Plan for a few hours of fixes, not a rewrite.

### GitLab CI

If you move to GitLab, its built-in CI (`.gitlab-ci.yml`) is mature and
powerful: pipelines with stages, merge request pipelines, environments,
caching, and self-hosted runners that can use Docker, Kubernetes or a
plain shell. The syntax is different from Actions, so this is a
rewrite — but a well-documented one.

### Woodpecker CI

[Woodpecker](https://woodpecker-ci.org/) is a community fork of Drone
CI from before Drone changed its licensing. It is Apache-licensed,
container-first (every step runs in its own image), and pleasantly
small. It integrates with Forgejo, Gitea, GitLab, GitHub and Bitbucket,
and Codeberg offers a hosted instance for its users. Pipelines are YAML
but simpler than Actions: a list of steps, each with an image and
commands.

### The general-purpose engines

- **[Jenkins](https://www.jenkins.io/)** — the old workhorse. Endlessly
  extensible through plugins, which is both its strength and the source
  of most Jenkins pain.
- **[Concourse](https://concourse-ci.org/)** — pipelines built from
  resources, jobs and tasks, with an emphasis on reproducibility.
- **[Tekton](https://tekton.dev/)** and
  **[Argo Workflows](https://argoproj.github.io/workflows/)** —
  Kubernetes-native building blocks, for teams that already think in
  CRDs.
- **[Buildbot](https://buildbot.net/)** — CI configured in Python.
- **[builds.sr.ht](https://man.sr.ht/builds.sr.ht/)** — SourceHut's
  build service: a manifest, a fresh VM, and a shell.

### Making pipelines portable

Whatever you choose, the best protection against lock-in is keeping the
CI file thin. Put the real logic in a `Makefile`, a shell script, or a
tool like [Dagger](https://dagger.io/) that runs pipelines as code in
containers, and let the YAML just call it. Then the YAML is the only
thing that changes when you switch platforms, and you can run the same
steps on your laptop. For existing GitHub workflows, `act` will run them
locally, which is handy while you migrate.

## Quick comparison

| Tool       | Type        | License    | Hosted option     | Actions-compatible |
| ---------- | ----------- | ---------- | ----------------- | ------------------ |
| Forgejo    | Forge + CI  | GPL v3+    | Codeberg          | Mostly             |
| Gitea      | Forge + CI  | MIT        | Gitea Cloud       | Mostly             |
| GitLab CE  | Forge + CI  | MIT (core) | GitLab.com (EE)   | No                 |
| SourceHut  | Forge + CI  | AGPL       | sr.ht (paid)      | No                 |
| OneDev     | Forge + CI  | MIT        | —                 | No                 |
| Radicle    | P2P forge   | MIT/Apache | — (by design)     | No                 |
| Woodpecker | CI          | Apache 2.0 | Codeberg CI       | No                 |
| Jenkins    | CI          | MIT        | —                 | No                 |

## What I'd pick

For a personal project or a small team that likes the GitHub workflow,
**Forgejo** is the easy answer: on Codeberg if you want zero
maintenance, on a small server if you want full control. Your Actions
knowledge transfers almost directly.

For an organisation that wants a single integrated platform with
planning, security scanning and deployments, **GitLab** is the realistic
choice, as long as you check which features you actually need are in the
free edition.

And if you like mailing lists and minimal interfaces, give
**SourceHut** an afternoon. It might change how you think about code
review.

Moving off GitHub is mostly a matter of moving your CI and telling
people where to find you. Your Git history comes with you, and you can
leave a mirror behind for anyone who doesn't follow.

[^1]: [Forgejo forks its own path forward](https://forgejo.org/2024-02-forking-forward/), Forgejo, 15 February 2024.
[^2]: [Forgejo is now copyleft, just like Git](https://forgejo.org/2024-08-gpl/), Forgejo, 22 August 2024.
