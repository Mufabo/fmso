---
title: "Version Control on Autopilot: semantic-release and Renovate"
date: 2026-11-21
draft: true
description: "How to automate versioning, changelogs and releases with semantic-release and Conventional Commits, keep dependencies current with Renovate, and make the two work together so dependency updates turn into the right releases without anyone touching a version number."
summary: "Two bots, one loop: semantic-release turns commit messages into version numbers, changelogs and published releases, and Renovate keeps every dependency up to date with pull requests that speak the same commit language. Here is how to set up both and make them cooperate."
tags: ["semantic-release", "renovate", "conventional commits", "semver", "git", "ci/cd", "github actions", "supply chain security", "devops"]
categories: ["Programming"]
---

In the [CI/CD for Infrastructure as Code post]({{< relref "20261116_cicd-for-infrastructure-as-code" >}}) I said modules should be released with semantic versions and that "dependency bots such as Renovate can open those bump PRs automatically". This post covers both halves of that sentence:

1. **semantic-release** decides the next version number from your commit history, writes the changelog, tags the release and publishes it.
2. **Renovate** watches every dependency you have (npm packages, container images, Helm charts, Terraform modules, GitHub Actions…) and opens a pull request whenever there's something newer.

Each tool is useful alone. Together they form a loop: your library is released by semantic-release, Renovate notices the new version in every repository that uses it, opens an update PR there, and once that PR is merged, semantic-release in *that* repository cuts its own release. Nobody types a version number.

## Semantic versioning in one paragraph

A version `MAJOR.MINOR.PATCH` is a promise to whoever depends on you. **Patch** (`1.4.2 → 1.4.3`): bug fixes, nothing else changes. **Minor** (`1.4.3 → 1.5.0`): new features, everything that worked still works. **Major** (`1.5.0 → 2.0.0`): something that worked may now break. Versions starting with `0.` make no promises at all.

The promise is only worth something if it's kept, and humans are bad at keeping it. Someone bumps the minor version for a change that removed a function, or forgets to bump at all, or ships a "quick fix" as `1.4.2` a second time. Automating the decision removes the human from the part humans get wrong.

## semantic-release

### The idea

semantic-release reads the commit messages since the last release, follows formalized conventions to work out the next semantic version, generates a changelog and publishes the release.[^1] It's meant to run in CI after every successful build on the release branch, so that no human is directly involved in the release process.[^2]

The consequence is a bit of a mindset shift: **you no longer decide when to release or what number it gets**. You decide what kind of change each commit is, and the tool does the rest. Merging a `fix` to `main` *is* the release.

### Conventional Commits

The commit format is what carries the information. By default semantic-release uses the Angular commit message conventions.[^1] In practice most teams use the slightly broader [Conventional Commits](https://www.conventionalcommits.org/) spec, which is the same thing plus the `!` shorthand for breaking changes:

```text
<type>(<optional scope>): <description>

<optional body>

<optional footer(s)>
```

How each type maps to a release:

| Commit | Release |
|---|---|
| `fix(parser): handle empty input` | Patch |
| `feat(api): add pagination` | Minor |
| `feat(api)!: drop v1 endpoints` or any commit with a `BREAKING CHANGE:` footer | Major |
| `docs:`, `chore:`, `ci:`, `test:`, `refactor:`, `style:`, `build:` | No release |

The `BREAKING CHANGE:` token must be in the commit footer, not the subject.[^3] The `!` shorthand is only understood by the `conventionalcommits` preset, not by the default `angular` one, which is one more reason to switch presets (see the configuration below).

A full example:

```text
feat(auth)!: replace session cookies with OIDC tokens

Sessions are now validated against the identity provider on every
request. The /login form is gone.

BREAKING CHANGE: clients must send a bearer token instead of the
`sid` cookie.
Refs: #412
```

Two practical tips:

- **Squash-merge pull requests and lint the PR title.** Then only the PR title has to follow the convention, not every "wip" commit on the branch. A PR-title check (for example `amannn/action-semantic-pull-request`) is much less friction than enforcing it on every local commit.
- If you do want local enforcement, tools such as commitlint (with a Git hook) or commitizen (an interactive prompt) help contributors write valid messages.[^2]

### What a release run does

When `semantic-release` runs, it goes through a fixed sequence of steps: verify conditions, get the last release, analyze commits, verify the release, generate notes, create the Git tag, prepare, publish and notify.[^3] Each step is implemented by plugins:

| Plugin | Does |
|---|---|
| `@semantic-release/commit-analyzer` | Works out patch/minor/major from the commits |
| `@semantic-release/release-notes-generator` | Writes the release notes |
| `@semantic-release/changelog` | Writes those notes to `CHANGELOG.md` (optional) |
| `@semantic-release/npm` | Bumps `package.json` and publishes to npm |
| `@semantic-release/git` | Commits changed files (changelog, version) back to the repo (optional) |
| `@semantic-release/github` / `@semantic-release/gitlab` | Creates the GitHub/GitLab release, comments on fixed issues and PRs |
| `@semantic-release/exec` | Runs arbitrary shell commands in any step |

The "last release" is simply the most recent Git tag that looks like a version. That has two consequences: the CI checkout must include the full history and tags, and you must never delete or move release tags by hand.

### Configuration

A `.releaserc.json` for a typical npm library:

```json
{
  "branches": [
    "main",
    { "name": "next", "prerelease": true },
    { "name": "1.x", "range": "1.x", "channel": "1.x" }
  ],
  "plugins": [
    ["@semantic-release/commit-analyzer", { "preset": "conventionalcommits" }],
    ["@semantic-release/release-notes-generator", { "preset": "conventionalcommits" }],
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    "@semantic-release/npm",
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json", "package-lock.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
    }],
    "@semantic-release/github"
  ]
}
```

The `branches` section is where semantic-release gets interesting:

- `main` produces normal releases (`2.3.0`).
- `next` produces prereleases (`3.0.0-next.1`, `3.0.0-next.2`, …) published to the `next` dist-tag, so you can ship a big breaking change in stages without users getting it by accident.
- `1.x` is a maintenance branch: fixes there are released as `1.8.4` even though `main` is at `2.x`, and semantic-release refuses to release anything from it that would leave the `1.x` range.

Whether to commit `CHANGELOG.md` and the version bump back to the repository (`@semantic-release/git`) is a matter of taste. It needs a push to a protected branch, which complicates branch protection. Many projects skip it: the GitHub release *is* the changelog, and the version in `package.json` in Git stays at `0.0.0-development`; only the published artifact carries the real number.

### Running it in GitHub Actions

semantic-release v25 needs Node.js 22.14 or later (or 24.10+ in the 24 line); support for Node 20, 21 and 23 was dropped.[^4] v25 also upgraded the npm plugin to one that adds support for npm **trusted publishing**,[^5] which lets the workflow publish with a short-lived OIDC token instead of a long-lived `NPM_TOKEN`. The npm plugin's docs now recommend trusted publishing whenever you publish from a supported CI provider, and provenance attestations are generated automatically when you do.[^6]

```yaml
# .github/workflows/release.yml
name: release
on:
  push:
    branches: [main, next, "*.x"]

permissions:
  contents: read

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write       # create tags and GitHub releases
      issues: write         # comment on released issues
      pull-requests: write  # comment on released PRs
      id-token: write       # OIDC for npm trusted publishing
    steps:
      - uses: actions/checkout@<pinned-sha>
        with:
          fetch-depth: 0          # full history and tags
          persist-credentials: false
      - uses: actions/setup-node@<pinned-sha>
        with:
          node-version: lts/*
          # note: no registry-url here
      - run: npm ci
      - run: npm test
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Gotchas worth knowing:

- **`fetch-depth: 0`.** The default shallow clone has no tags, so semantic-release thinks there was never a release and tries to publish `1.0.0`.
- **Don't set `registry-url` in `setup-node`.** It writes an `.npmrc` that can conflict with semantic-release's own npm authentication and cause `EINVALIDNPMTOKEN` errors even when the token is fine.[^7]
- **Install semantic-release as a pinned devDependency** rather than running `npx semantic-release@latest`. Then a new major version of the tool can't break your release pipeline overnight; it arrives as a Renovate PR instead.
- **Protected branches.** If you use `@semantic-release/git`, the default `GITHUB_TOKEN` usually can't push to a protected `main`. Use a GitHub App token that's allowed to bypass the rule, or drop the plugin.

### Not just for npm

Despite its npm roots, semantic-release works for anything that has a Git repository. Configure it without the npm plugin and let `@semantic-release/exec` do the publishing:

```json
{
  "branches": ["main"],
  "plugins": [
    ["@semantic-release/commit-analyzer", { "preset": "conventionalcommits" }],
    ["@semantic-release/release-notes-generator", { "preset": "conventionalcommits" }],
    ["@semantic-release/exec", {
      "prepareCmd": "docker build -t ghcr.io/acme/app:${nextRelease.version} .",
      "publishCmd": "docker push ghcr.io/acme/app:${nextRelease.version}"
    }],
    "@semantic-release/github"
  ]
}
```

The same pattern works for Helm charts (`sed` the version into `Chart.yaml`, then `helm package` and `helm push`), Go modules (the tag *is* the release, so `@semantic-release/github` alone is enough) or Terraform modules. There are community plugins for Maven, Gradle, PyPI, Cargo and others.

### Alternatives

- **release-please** (Google) uses the same Conventional Commits but opens a "release PR" that accumulates changes; the release happens when you merge that PR. Good if you want a human to decide *when*, while the tool still decides *what*.
- **Changesets** is popular in JavaScript monorepos. Contributors add a small markdown file describing each change and its bump type, instead of encoding it in the commit message.
- **GoReleaser**, **cargo-release** and similar tools focus on building and publishing for one ecosystem; they're often combined with one of the above for the version decision.

## Renovate

### The idea

Every project depends on other projects, and those move on whether you follow or not. Falling behind is comfortable for a while and then very expensive: a security fix lands in a version three majors ahead, and the upgrade becomes a project.

Renovate scans a repository, finds every dependency declaration it understands (it supports dozens of package managers and file formats: `package.json`, `go.mod`, `Dockerfile`, Helm `values.yaml`, Terraform, GitHub Actions workflows, Kubernetes manifests, and more), checks for newer versions and opens one pull request per update, with release notes in the PR body. It runs as the hosted Mend Renovate app on GitHub, or self-hosted as a CLI on any CI or cron. On its own, the CLI runs once and exits, so it runs only as often as whoever administers it schedules it.[^8]

Compared with GitHub's Dependabot, Renovate supports more ecosystems and platforms (GitLab, Bitbucket, Gitea/Forgejo, Azure DevOps) and is far more configurable: grouping, schedules, automerge policies, shared presets and a Dependency Dashboard issue that lists every pending update in one place.

### A sensible starting config

Renovate's onboarding PR adds a `renovate.json` that extends `config:recommended`, the preset meant for most users regardless of language.[^9] For a project you care about, I'd start a step further, at `config:best-practices`, which adds the maintainers' best practices on top: pinning Docker image digests, treating packages without a release for a year as abandoned, weekly lock file maintenance and more.[^9]

```json5
// renovate.json5
{
  $schema: "https://docs.renovatebot.com/renovate-schema.json",
  extends: [
    "config:best-practices",
    ":semanticCommits",
    "helpers:pinGitHubActionDigests"
  ],
  timezone: "Europe/Berlin",
  schedule: ["before 6am on monday"],
  prConcurrentLimit: 10,
  packageRules: [
    {
      description: "Automerge non-major updates of stable packages",
      matchUpdateTypes: ["minor", "patch", "digest"],
      matchCurrentVersion: "!/^0/",
      automerge: true
    },
    {
      description: "Group all ESLint packages into one PR",
      matchPackageNames: ["eslint", "@eslint/**", "eslint-plugin-**"],
      groupName: "eslint"
    },
    {
      description: "Majors need an explicit OK on the Dependency Dashboard",
      matchUpdateTypes: ["major"],
      dependencyDashboardApproval: true
    }
  ]
}
```

What the parts do:

- **`schedule`** keeps PRs to one window a week, so they don't trickle in all day and interrupt everyone.
- **`packageRules`** are the core of Renovate configuration. Each rule matches some updates (by package name, update type, dependency type, file path, datasource…) and changes how they're handled.
- **`dependencyDashboardApproval`** keeps majors out of the PR list until someone ticks a box in the Dependency Dashboard issue.
- **Grouping** turns twelve ESLint PRs into one. Monorepos such as Angular, React or the AWS SDK are grouped automatically by built-in presets.

### Automerge

Automerge is what makes Renovate scale. Without it, a repository with a few hundred dependencies generates more PRs than anyone will review, people start ignoring them, and you're back to falling behind, just with more noise.

The rule in the config above, automerging non-major updates for packages that have left `0.x`, is exactly what Renovate's own `:automergeStableNonMajor` preset does.[^10] By default Renovate uses the platform's native automerge (GitHub's "auto-merge" button, GitLab's "merge when pipeline succeeds"), so the merge only happens once the required checks pass.[^11] Two things follow:

- **Automerge is only as safe as your tests.** If CI doesn't really exercise the code, automerge merges broken updates. Start with devDependencies and linters, watch for a few weeks, then widen.
- **Branch protection must require status checks.** Otherwise "native automerge" merges immediately. Also, Renovate won't automerge on GitHub while a PR has an outstanding negative review.[^12]

You can check that Renovate understood your intent: the PR body says "Automerge: Enabled", or "Automerge: Disabled by config" if a rule is missing.[^11]

### Supply-chain safety: wait before you update

Automatically pulling in the newest version of everything has an obvious downside: if a maintainer account is compromised and a malicious version is published, you get it within the hour. Renovate's answer is `minimumReleaseAge`, a "dependency cooldown" that holds an update back until the release is a certain age; the point is to reduce supply-chain risk, not to slow things down. With 14 days, for example, security researchers and automated scanners get time to catch malicious releases first.[^13]

After the wave of npm supply-chain attacks in 2025, the Renovate maintainers made a 3-day minimum release age the default for npm packages in `config:best-practices`, starting with Renovate 42.[^14] For anything you automerge, I'd go longer:

```json5
{
  packageRules: [
    {
      description: "Cool down third-party updates before automerging",
      matchUpdateTypes: ["minor", "patch", "digest"],
      minimumReleaseAge: "14 days"
    },
    {
      description: "Our own packages: no waiting",
      matchPackageNames: ["@acme/**"],
      minimumReleaseAge: null
    }
  ],
  vulnerabilityAlerts: {
    labels: ["security"],
    minimumReleaseAge: null   // security fixes go out immediately
  }
}
```

Other measures in the same spirit: pin GitHub Actions to commit SHAs (`helpers:pinGitHubActionDigests`) and Docker images to digests, so a moved tag can't change what runs; Renovate keeps the pins current, so pinning no longer means "frozen".

### Shared presets

Once you have more than a handful of repositories, put the config in one place. Create a repository (say `acme/renovate-config`) with a `default.json`, and have every repository extend it:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>acme/renovate-config"]
}
```

Renovate ships with over 100 built-in presets of its own,[^15] and organization presets work the same way: you change the policy once, and every repository follows on its next run.

## Making them work together

This is where the two tools meet: **Renovate's commits are the input to semantic-release.** If they don't agree on what a dependency update means, you either get a release for every devDependency bump or never release a security fix.

### What Renovate writes

Renovate detects whether a repository uses semantic commits and, if so, uses them.[^16] With `config:recommended`, nearly all updates use the `chore` type, with exceptions;[^16] the main one comes from the `:semanticPrefixFixDepsChoreOthers` preset, which uses `fix` for production dependencies and `chore` for everything else.[^17] So:

- `fix(deps): update dependency express to v4.21.2` → **patch release**
- `chore(deps): update dependency eslint to v9.20.0` → **no release**

For a library, that default is usually right. A runtime dependency fix changes what your users install, so they should get a new version; a new linter doesn't change anything they receive.

### When the default is wrong

Adjust it with `semanticCommitType` in `packageRules`:

```json5
{
  extends: ["config:best-practices", ":semanticCommits"],
  packageRules: [
    {
      description: "Base image updates change the shipped artifact: release",
      matchDatasources: ["docker"],
      matchFileNames: ["Dockerfile"],
      semanticCommitType: "fix"
    },
    {
      description: "CI tooling never warrants a release",
      matchManagers: ["github-actions"],
      semanticCommitType: "ci"
    },
    {
      description: "Release tooling itself: no release",
      matchPackageNames: ["semantic-release", "@semantic-release/**"],
      semanticCommitType: "build"
    }
  ]
}
```

Renovate's own repository does something very similar: it marks production dependencies and `semantic-release` itself as `build`, uses `fix` for patch updates of its base image and `feat` for minor and major ones.[^18]

Two things are worth thinking about explicitly:

- **Applications vs libraries.** For a container image, a base-image update *does* change what users run, so it should produce a release even though it's not an npm "production dependency". For a library, that same Dockerfile might only be used for tests.
- **A major dependency upgrade is not automatically a major release.** Going from `express` 4 to 5 internally is a `fix` (or `feat`) for your users if your API stays the same. Only if the upgrade leaks into your public API — a peer dependency, re-exported types — should it become a breaking change, and that needs a human to add `BREAKING CHANGE:` when merging. This is a good reason to keep majors behind dashboard approval.

### The full loop

```text
  library repo                                    app repo
  ────────────                                    ────────
  feat(api): add pagination  ─► merge
                │
     semantic-release: 1.5.0 ─► npm / GitHub release
                                    │
                                    ▼
                              Renovate sees 1.5.0
                                    │
                                    ▼
                    fix(deps): update @acme/lib to v1.5.0  (release notes in PR body)
                                    │
                         CI green ─► automerge
                                    │
                                    ▼
                         semantic-release: 3.8.1 ─► image pushed
```

The release notes semantic-release wrote in the library end up in the body of the Renovate PR in the app, so whoever reviews (or audits an automerged) update can see exactly what changed. A good commit message in one repository becomes useful documentation in every repository downstream.

### Batching releases

With automerge on, a busy Monday morning can produce ten Renovate merges and therefore ten patch releases. That's correct, but noisy for consumers. Options:

- **Group** non-major updates (`group:allNonMajor`) so one PR, one merge, one release.
- **Run semantic-release on a schedule** (a nightly `workflow_dispatch`/cron) instead of on every push. It still releases everything since the last tag, just once.
- **Use release-please** if you want releases to accumulate in a PR until someone merges it.

## Common pitfalls

- **Shallow clones.** `fetch-depth: 0`, always.
- **Unconventional merges.** One "Merge branch 'main'" commit is harmless, but if merges with non-conventional titles reach `main`, their changes are released silently in the next version without a changelog entry. Squash and lint PR titles.
- **Forgetting that `chore` means "no release".** If a Renovate update fixes a CVE in a runtime dependency but is typed `chore`, your users don't get the fix. Check the `semanticCommitType` rules, especially for non-npm ecosystems.
- **Automerge without tests or branch protection.** It merges what it's allowed to merge.
- **Release tooling on `@latest`.** Pin semantic-release and its plugins; let Renovate update them like anything else.
- **Hand-made tags.** A manually pushed `v2.0.0` tag tells semantic-release that 2.0.0 was released, whether it was or not.
- **Ignoring the Dependency Dashboard.** Majors waiting for approval don't go away. Look at the dashboard regularly, or the backlog of breaking upgrades quietly grows.

## Summary

- Write commits (or at least squash-merge PR titles) as Conventional Commits; `fix` → patch, `feat` → minor, `!`/`BREAKING CHANGE` → major.
- Let semantic-release run in CI on every push to the release branch: it decides the version, writes the notes, tags and publishes. Use OIDC trusted publishing instead of long-lived tokens.
- Let Renovate open update PRs for every dependency type you have, starting from `config:best-practices`, on a schedule, with grouping.
- Automerge non-major updates once your tests deserve that trust, and hold third-party updates back with `minimumReleaseAge`.
- Align the two: make sure Renovate's commit types match what you consider release-worthy, so runtime fixes produce releases and tooling updates don't.

## References

[^1]: [semantic-release README (v24.2.3)](https://unpkg.com/semantic-release@24.2.3/README.md), semantic-release project.
[^2]: [semantic-release on npm](https://npmjs.com/semantic-release), semantic-release project.
[^3]: [semantic-release README (v25.0.9)](https://cdn.jsdelivr.net/npm/semantic-release@25.0.9/README.md), semantic-release project.
[^4]: [semantic-release v25 release notes](https://gitlab.com/gitlab-formatters/vfile-reporter-gitlab/-/merge_requests/32/pipelines), as included in a Renovate update MR; see also the [upstream release notes](https://gitlab.com/dclint/ci-component/-/merge_requests/17).
[^5]: [semantic-release v25.0.1 release notes](https://gitlab.com/el-capitano/tools/ci-cd-templates/commit/e4d1fe3ba2cfc7081832fc2d5b785e2c2554455d), as included in a Renovate update commit.
[^6]: [@semantic-release/npm README (v13.2.0)](https://unpkg.com/@semantic-release/npm@13.2.0/README.md), semantic-release project.
[^7]: [semantic-release 25.0.3 documentation diff](https://socket.dev/npm/package/semantic-release/diff/25.0.3) (GitHub Actions recipe), Socket.
[^8]: [Noise Reduction](https://docs.renovatebot.com/noise-reduction), Renovate documentation.
[^9]: [Full Config Presets](https://docs.renovatebot.com/presets-config/), Renovate documentation.
[^10]: [Default Presets](https://docs.renovatebot.com/presets-default/), Renovate documentation.
[^11]: [Automerge configuration and troubleshooting](https://github.com/renovatebot/renovate/blob/f70f0c82beda4712f11f93a4a111ca4f56d4ea5a/docs/usage/key-concepts/automerge.md), Renovate documentation.
[^12]: [Configuration Options: automerge](https://github.com/renovatebot/renovate/blob/3096f340c8cdb7524e7573df20b2e481bfa797d1/docs/usage/configuration-options.md), Renovate documentation.
[^13]: [Minimum Release Age](https://docs.renovatebot.com/key-concepts/minimum-release-age/), Renovate documentation.
[^14]: [Building a more secure npm ecosystem with Mend Renovate](https://www.mend.io/blog/secure-npm-ecosystem-with-mend-renovate/), Mend.io blog.
[^15]: [Use Cases](https://docs.renovatebot.com/getting-started/use-cases), Renovate documentation.
[^16]: [Semantic Commit messages](https://docs.renovatebot.com/semantic-commits), Renovate documentation.
[^17]: [Full Config Presets: config:recommended](https://docs.renovatebot.com/presets-config/), Renovate documentation.
[^18]: [renovate.json in the Renovate repository](https://redirect.github.com/renovatebot/renovate/blob/main/renovate.json), Renovate project.
