---
title: "CI/CD for Infrastructure as Code: Version Control, Pipelines and Multiple Environments"
date: 2026-11-16
draft: true
description: "How to run Infrastructure as Code like software: structuring the Git repository, versioning modules, separating dev, staging and production, promoting changes between environments, and building a plan-review-apply pipeline with GitHub Actions, OIDC and approval gates."
summary: "Infrastructure as Code only pays off when changes flow through version control and an automated pipeline. Here is how to lay out the repository, keep environments separate but alike, promote changes from dev to prod, and build a pipeline that applies exactly what was reviewed."
tags: ["infrastructure as code", "ci/cd", "opentofu", "terraform", "github actions", "gitops", "devops", "platform engineering"]
categories: ["Programming"]
---

In the [Infrastructure as Code post]({{< relref "infrastructure-as-code" >}}) I ended with "run it from a pipeline, not a laptop" and a small diagram. This post fills that in. It covers three things that are tightly linked:

1. **Version control**: how the repository is laid out, how modules are versioned, and which branching model works for infrastructure.
2. **Multiple environments**: how dev, staging and production stay separate but similar.
3. **CI/CD**: a pipeline that checks every change, shows the plan in review, and promotes the change through the environments with an approval gate before production.

The examples use OpenTofu and GitHub Actions on AWS, but the ideas carry over to Terraform, GitLab CI, Azure DevOps or Tekton without much change.

## Version control

### Git is the source of truth

The goal is simple to state: **the main branch describes what exists**. If it's in `main`, it's deployed (or about to be); if it's not in `main`, it shouldn't exist. Every other practice in this post serves that sentence.

Two things live in Git and two things don't:

| In Git | Not in Git |
|---|---|
| Configuration (`.tf`, `.tofu`, `.tfvars` without secrets) | State files (they belong in the remote backend) |
| The dependency lock file `.terraform.lock.hcl` | Secrets, credentials, plan files |

A starting `.gitignore`:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
plan.bin
crash.log
*.auto.tfvars    # if you use these for local secrets
```

### Repository layout

A layout that scales well separates **modules** (reusable building blocks) from **live** configuration (what is actually deployed, per environment):

```text
infra/
  modules/
    network/
    log-bucket/
    app-service/
  live/
    dev/
      network/        # one root module = one state
      app/
    staging/
      network/
      app/
    prod/
      network/
      app/
```

Each leaf under `live/` is its own root module with its own state file. That keeps states small, lets the pipeline plan only what changed, and lets you grant different people different rights per environment.

Whether `modules/` and `live/` share a repository is a matter of team size. One repository (a monorepo) is simpler: a module change and its use land in a single pull request. Separate repositories make sense when a platform team publishes modules for many consuming teams, because then modules need real releases.

### Versioning modules

When modules live in their own repository, tag releases with semantic versions and reference a tag, never a branch:

```hcl
module "logs" {
  source = "git::https://github.com/acme/tf-modules.git//log-bucket?ref=log-bucket/v1.4.0"

  name           = "acme-app-logs-prod"
  retention_days = 365
}
```

Major version for breaking input changes or anything that replaces resources, minor for new optional inputs, patch for fixes. Keep a changelog; the people upgrading a module want to know whether `v2` will recreate their database.

Tags also give you a clean promotion mechanism (more on that below): dev can run `v1.5.0` while prod stays on `v1.4.0` until someone opens a pull request bumping it. Dependency bots such as Renovate can open those bump PRs automatically.

### Pin and lock providers

Pin provider versions with a constraint (`version = "~> 6.0"`) and **commit the lock file**. The lock file records the exact provider version and its checksums, so CI and every laptop use identical binaries. Because the checksums differ per platform, generate them for every platform your team and runners use:

```bash
tofu providers lock -platform=linux_amd64 -platform=linux_arm64 -platform=darwin_arm64
```

Upgrading a provider then becomes an explicit pull request that changes the lock file, which is exactly the kind of change you want reviewed.

### Branching: trunk-based, not branch-per-environment

A tempting pattern is one long-lived branch per environment: `dev`, `staging`, `prod`, with changes promoted by merging from one to the next. It looks tidy and goes wrong quickly:

- Hotfixes applied to `prod` must be merged back, and sometimes aren't.
- The branches drift apart, so "promote staging to prod" carries along whatever else happened to be on `staging`.
- Merge conflicts in infrastructure code are a poor place to discover that environments differ.

The pattern that works is **trunk-based development**: short-lived feature branches, pull requests into `main`, and environments represented as **directories** (or module versions), not branches. Promotion happens in the pipeline or through a version bump, never through a merge between environment branches.

### Protect the trunk

- **Branch protection** on `main`: no direct pushes, required status checks (format, validate, scan, plan), at least one approving review.
- **CODEOWNERS** to require review from the right people for the right paths, for example the platform team for `live/prod/` and `modules/network/`:

```text
# .github/CODEOWNERS
/infra/live/prod/        @acme/platform-leads
/infra/modules/network/  @acme/network-team
```

- **pre-commit hooks** (for example `pre-commit-terraform`) to run `fmt`, `validate` and a linter before code even leaves a laptop.
- **Pin CI actions** to commit SHAs. The pipeline that applies to production has very powerful credentials, so treat its dependencies as carefully as your own code.

### Rollback is a revert, not an undo

`git revert` followed by an apply returns the *configuration* to its previous version, and the tool converges infrastructure towards it. That's usually what you want, but it isn't a true undo: a deleted database stays deleted, and data written in between stays written. Version control gives you a record and a way back for configuration. It doesn't give you transactions, which is why `prevent_destroy` and backups still matter.

## Multiple environments

### Workspaces or directories?

Terraform and OpenTofu have **CLI workspaces**, which give one configuration several state files. They look like a natural fit for dev/staging/prod, but HashiCorp's own documentation advises against it: deployments like staging and production often need different credentials and access controls, and workspaces in one working directory share a single backend, so they aren't a suitable isolation mechanism.[^1] The recommended alternative is a separate configuration per instance, each with its own backend, with the shared parts in reusable modules.[^1]

That's the `live/<env>/` layout above. Workspaces remain useful for many near-identical, equally-trusted copies, like ephemeral test environments, but not as the wall between production and everything else.

(HCP Terraform also has "workspaces", but they are a different concept and behave much more like separate directories.)

### One cloud account per environment

Directories separate state. **Accounts** (AWS accounts, Azure subscriptions, GCP projects) separate everything else: IAM, quotas, billing, blast radius. A misconfigured dev pipeline simply has no credentials that work in the prod account. With AWS Organizations or the equivalent, an account per environment is cheap and is the single strongest isolation you can buy.

### Same code, different inputs

Environments should differ in **size and settings**, not in **shape**. Every root module calls the same modules; only the inputs change:

```hcl
# live/dev/app/main.tf
module "app" {
  source        = "../../../modules/app-service"
  environment   = "dev"
  instance_type = "t4g.small"
  min_replicas  = 1
  multi_az      = false
}
```

```hcl
# live/prod/app/main.tf
module "app" {
  source        = "../../../modules/app-service"
  environment   = "prod"
  instance_type = "m7g.large"
  min_replicas  = 3
  multi_az      = true
}
```

If you find yourself writing `count = var.environment == "prod" ? 1 : 0` all over a module, the environments are diverging in shape, and staging is no longer testing what prod will run. Some differences are legitimate (prod may have a WAF that dev doesn't need), but keep them few and explicit.

The backend configuration differs per environment too. Partial backend configuration keeps the code identical and puts the differences in a small file:

```hcl
# backend.tf (identical in every environment)
terraform {
  backend "s3" {}
}
```

```hcl
# live/prod/app/backend.hcl
bucket       = "acme-tofu-state-prod"
key          = "app/terraform.tfstate"
region       = "eu-central-1"
use_lockfile = true
encrypt      = true
```

```bash
tofu init -backend-config=backend.hcl
```

### Reducing repetition

With many components and environments, the `live/` tree accumulates near-identical boilerplate: provider blocks, backend blocks, the same module call with slightly different inputs. Two common ways out:

- **Terragrunt** generates backends and providers and wires units together. It reached 1.0 in 2026, with Gruntwork committing to no breaking changes in minor releases from then on,[^2] and its "stacks" let you define a group of units once and stamp it out per environment.[^3]
- **Plain modules plus a little convention**: a thin root module per environment that calls one "environment" module. Less tooling, a bit more copy-paste.

Either is fine. What matters is that the difference between two environments is visible in a short diff.

### Promotion models

There are two clean ways to move a change from dev to prod:

**1. Same commit, sequential applies.** A merge to `main` triggers the pipeline, which applies `dev`, then `staging`, then (after approval) `prod`, all from the same commit. Simple, and every environment converges on `main` quickly. This works best in a monorepo where the live configuration references local modules (`source = "../../modules/..."`).

**2. Version promotion.** Modules are released with tags. Dev references the newest version; a pull request bumps staging; once staging is healthy, another pull request bumps prod. Each environment's pull request has its own plan and its own review. More ceremony, but the state of each environment is visible right in its code, and a bad module version never reaches prod by accident.

Many teams combine them: version promotion for shared modules owned by a platform team, sequential applies for an application's own infrastructure.

## The pipeline

### Merge-then-apply vs apply-then-merge

There are two schools:

- **Merge-then-apply**: the pull request shows a plan; after review and merge, the pipeline on `main` applies. `main` always reflects what was intended, and this is often recommended as the safer approach.[^4]
- **Apply-then-merge**: you apply from the pull request (typically by commenting `atlantis apply`), confirm it worked, and merge afterwards. This is Atlantis's model: plan when the PR is opened, apply on comment, release the lock on merge.[^5] Atlantis locks the directory and workspace on plan until the pull request is merged or closed,[^6] so two PRs can't fight over the same state. The advantage is that a failed apply is discovered *before* the change lands in `main`.

Both work. The example below uses merge-then-apply because it fits naturally with GitHub's environment protection rules.

### Stages

```text
Pull request                                   Merge to main
───────────────                                ──────────────────────────────────────────────
fmt ─ validate ─ lint ─ scan ─ test            dev:     plan ─► apply
           │                                                  │
           ▼                                   staging: plan ─► apply
plan (dev, staging, prod) ─► summary on PR                    │
           │                                   prod:    plan ─► ⏸ approval ─► apply saved plan
     review + approve
```

On the pull request, everything is read-only: static checks, security scanning, tests, and a plan for every environment the change affects. Reviewers read the code *and* the plans. After merge, each environment is planned and applied in order. Production pauses for a human, who approves the *specific* saved plan that will be applied.

### Credentials: OIDC, separate roles, least privilege

Don't store cloud access keys in CI secrets. GitHub Actions, GitLab CI and most other CI systems can issue a short-lived OpenID Connect token that the cloud exchanges for temporary credentials.[^7] On AWS that means an IAM role per purpose and environment:

| Role | Used by | Permissions |
|---|---|---|
| `tofu-plan` (per account) | PR plans, pre-apply plans | Read-only on resources; read state; write the lock file |
| `tofu-apply` (per account) | Apply jobs | What the infrastructure needs, nothing more |

The trust policy is where the real protection is. GitHub's OIDC token includes a `sub` claim that identifies what's running, for example `repo:acme/infra:pull_request` for a PR, or `repo:acme/infra:environment:prod` for a job bound to the `prod` environment. Let the prod apply role trust **only** `repo:acme/infra:environment:prod`, and restrict the `prod` GitHub environment to the `main` branch with required reviewers. Then nothing — not a PR from a fork, not a feature branch, not a modified workflow file on a branch — can get prod write credentials without passing the approval gate.

### A GitHub Actions example

The pipeline has three files: one for pull requests, one for `main`, and a reusable workflow that plans and applies a single environment.

**Pull requests: checks and plans.**

```yaml
# .github/workflows/infra-pr.yml
name: infra-pr
on:
  pull_request:
    paths: ["infra/**"]

permissions:
  id-token: write
  contents: read
  pull-requests: write

env:
  TOFU_VERSION: "1.13.2"

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: ${{ env.TOFU_VERSION }}
      - run: tofu fmt -check -recursive infra
      - name: Validate and test modules
        run: |
          for m in infra/modules/*/; do
            echo "::group::$m"
            (cd "$m" && tofu init -backend=false -input=false && tofu validate && tofu test)
            echo "::endgroup::"
          done
      - name: Scan for misconfigurations
        uses: aquasecurity/trivy-action@<pinned-sha>
        with:
          scan-type: config
          scan-ref: infra
          exit-code: "1"
          severity: HIGH,CRITICAL

  plan:
    needs: checks
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - { env: dev,     account: "111111111111" }
          - { env: staging, account: "222222222222" }
          - { env: prod,    account: "333333333333" }
    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: ${{ env.TOFU_VERSION }}
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ matrix.account }}:role/tofu-plan
          aws-region: eu-central-1
      - name: Plan
        working-directory: infra/live/${{ matrix.env }}/app
        run: |
          tofu init -input=false -backend-config=backend.hcl
          tofu plan -input=false -lock=false -no-color -out=plan.bin
          {
            echo "### Plan for \`${{ matrix.env }}\`"
            echo '```'
            tofu show -no-color plan.bin
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"
```

`-lock=false` is acceptable here because a PR plan is speculative and never applied; it also means PR plans don't block real applies. To post the plan as a PR comment instead of a job summary, use `actions/github-script` or a purpose-built tool such as `tfcmt`. For a real repository with several components, add a step that works out which `live/` directories changed (and which are affected by a changed module) and feed that into the matrix instead of hard-coding it.

**Reusable workflow: plan and apply one environment.**

```yaml
# .github/workflows/deploy-env.yml
name: deploy-env
on:
  workflow_call:
    inputs:
      environment: { type: string, required: true }
      account:     { type: string, required: true }

env:
  TOFU_VERSION: "1.13.2"
  DIR: infra/live/${{ inputs.environment }}/app

jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      changes: ${{ steps.plan.outputs.changes }}
    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: ${{ env.TOFU_VERSION }}
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ inputs.account }}:role/tofu-plan
          aws-region: eu-central-1
      - id: plan
        working-directory: ${{ env.DIR }}
        run: |
          tofu init -input=false -backend-config=backend.hcl
          set +e
          tofu plan -input=false -detailed-exitcode -out=plan.bin
          code=$?
          set -e
          [ "$code" -eq 1 ] && exit 1
          echo "changes=$([ "$code" -eq 2 ] && echo true || echo false)" >> "$GITHUB_OUTPUT"
          { echo '```'; tofu show -no-color plan.bin; echo '```'; } >> "$GITHUB_STEP_SUMMARY"
      - uses: actions/upload-artifact@v4
        if: steps.plan.outputs.changes == 'true'
        with:
          name: plan-${{ inputs.environment }}
          path: ${{ env.DIR }}/plan.bin
          retention-days: 1

  apply:
    needs: plan
    if: needs.plan.outputs.changes == 'true'
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}   # prod has required reviewers
    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: ${{ env.TOFU_VERSION }}
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ inputs.account }}:role/tofu-apply
          aws-region: eu-central-1
      - uses: actions/download-artifact@v4
        with:
          name: plan-${{ inputs.environment }}
          path: ${{ env.DIR }}
      - working-directory: ${{ env.DIR }}
        run: |
          tofu init -input=false -backend-config=backend.hcl
          tofu apply -input=false plan.bin
```

A few details make this safe:

- `-detailed-exitcode` returns 0 for "no changes", 2 for "changes" and 1 for errors, so an environment with nothing to do skips the apply job entirely and nobody is asked to approve an empty change.
- The plan job runs **before** the approval gate, so the reviewer can open the run, read the plan in the summary, and approve exactly that.
- The apply job applies the **saved plan file**, not a fresh plan. If someone else changed the state in the meantime, the apply fails with a stale-plan error instead of doing something nobody reviewed.
- Plan files contain the full configuration and can contain secret values. Keep artifact retention short, restrict who can download artifacts, and consider OpenTofu's plan encryption.

**Main: promote through environments.**

```yaml
# .github/workflows/infra-main.yml
name: infra-main
on:
  push:
    branches: [main]
    paths: ["infra/**"]

permissions:
  id-token: write
  contents: read

concurrency:
  group: infra-main
  cancel-in-progress: false   # never cancel an apply halfway

jobs:
  dev:
    uses: ./.github/workflows/deploy-env.yml
    with: { environment: dev, account: "111111111111" }

  staging:
    needs: dev
    uses: ./.github/workflows/deploy-env.yml
    with: { environment: staging, account: "222222222222" }

  prod:
    needs: staging
    uses: ./.github/workflows/deploy-env.yml
    with: { environment: prod, account: "333333333333" }
```

The `concurrency` group serialises runs on `main`, and `cancel-in-progress: false` matters: cancelling a job in the middle of an apply leaves you with a half-applied change and possibly a stale lock.

To finish the setup, create GitHub environments `dev`, `staging` and `prod`; give `prod` required reviewers and a deployment branch rule that only allows `main`; and configure the IAM trust policies as described above.

### Between staging and prod: verify

A green apply means the cloud accepted the API calls, not that the system works. Before the prod gate, run something that checks the result: smoke tests against the staging endpoint, a health check, or integration tests (`tofu test` with `command = apply` in a throwaway account, or Terratest). Put it as a job between `staging` and `prod` so a broken staging blocks promotion automatically.

### Scheduled drift detection

Add a scheduled workflow that runs `tofu plan -detailed-exitcode` against every environment each night and alerts (Slack, an issue, a failed run) when the exit code is 2. A non-empty plan on an untouched `main` means someone changed infrastructure outside the pipeline.

```yaml
on:
  schedule:
    - cron: "17 3 * * *"
```

### Preview environments (optional)

For application infrastructure, some teams create a full environment per pull request (in a sandbox account, using a workspace or a generated directory) and destroy it when the PR closes. It's the most realistic test you can get, and the most expensive. If you try it, put a hard time limit and a cleanup job on it; orphaned preview environments are a classic source of surprise cloud bills.

## Tools that do this for you

Hand-written pipelines like the one above are perfectly viable, and they're the best way to understand what's going on. When the number of root modules grows, a dedicated tool saves a lot of glue code:

- **Atlantis**: open source (Apache 2.0), self-hosted, runs Terraform, OpenTofu and Terragrunt from pull request comments; v0.42.0 was released in April 2026.[^5]
- **Digger**: open source, runs inside your existing CI (GitHub Actions) instead of as a separate server, with per-project apply requirements such as "approved and mergeable" for prod directories.[^8]
- **Terragrunt** and Gruntwork's commercial pipelines product, which launched a free tier alongside Terragrunt 1.0.[^9]
- **Commercial platforms**: HCP Terraform, Spacelift, env0 and Scalr handle runs, state, policy, drift detection and approvals as a service.

## Common pitfalls

- **Environment branches.** Covered above; use directories and trunk-based development.
- **One pipeline credential for everything.** If the PR job can assume the prod apply role, branch protection is decoration.
- **Approving without a plan.** An approval gate placed *before* the plan has been generated asks reviewers to approve blind.
- **Applying a fresh plan after approval.** Always apply the saved plan that was reviewed.
- **Cancelling applies.** Use a concurrency group and never cancel in progress.
- **Unbounded matrix.** Planning every environment on every PR gets slow. Detect what changed, but remember that a module change affects every environment that uses it.
- **Manual "quick fixes" in prod.** They show up as drift the next morning and get reverted by the next apply. Make the console read-only and route fixes, even urgent ones, through a fast-track PR.

## Summary

- Keep configuration, modules and the lock file in Git; keep state, secrets and plans out of it.
- Use trunk-based development. Represent environments as directories or module versions, never as branches.
- Separate environments by state, by credentials and ideally by cloud account. Keep them the same shape.
- Promote changes either by applying the same commit environment by environment, or by bumping module versions per environment.
- On every PR: format, validate, lint, scan, test and plan. After merge: plan, approve for prod, and apply the saved plan.
- Give CI short-lived OIDC credentials, with prod write access tied to a protected environment on `main`.
- Check for drift every night.

## References

[^1]: [Manage Workspaces Overview](https://developer.hashicorp.com/terraform/cli/v1.16.x/workspaces), HashiCorp Terraform documentation.
[^2]: [Terragrunt v1.0.0 release notes](https://newreleases.io/project/github/gruntwork-io/terragrunt/release/v1.0.0), Gruntwork; [Terragrunt 1.0 Released](https://www.gruntwork.io/blog/terragrunt-1-0-released), Gruntwork blog.
[^3]: [The Road to 1.0: Terragrunt Stacks](https://www.gruntwork.io/blog/the-road-to-terragrunt-1-0-stacks), Gruntwork blog.
[^4]: [Terraform Pull Request Automation: A Complete Guide](https://scalr.com/learning-center/terraform-pull-request-automation-101/), Scalr, March 2026.
[^5]: [Atlantis Terraform Guide](https://env0.com/blog/atlantis-terraform-guide), env0, 2026.
[^6]: [Atlantis: Terraform Pull Request Automation](https://www.runatlantis.io/), Atlantis documentation (locking).
[^7]: [How to Use AWS OIDC Authentication with OpenTofu](https://oneuptime.com/blog/post/2026-03-20-aws-oidc-authentication-opentofu/view), OneUptime, March 2026.
[^8]: [How to Use OpenTofu with Digger for PR Automation](https://oneuptime.com/blog/post/2026-03-20-opentofu-with-digger-pr-automation/view), OneUptime, March 2026.
[^9]: [Gruntwork Newsletter: Spring 2026](https://www.gruntwork.io/blog/gruntwork-newsletter-spring-2026), Gruntwork blog.
