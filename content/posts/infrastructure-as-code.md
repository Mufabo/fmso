---
title: "Infrastructure as Code: Principles, Tools and Practice in 2026"
date: 2026-11-16
draft: true
description: "What Infrastructure as Code actually is, why it matters, how declarative tools like Terraform and OpenTofu work under the hood (state, plans, drift), how to structure, test and secure IaC, and what the tool landscape looks like after the Terraform license change and the IBM acquisition."
summary: "Infrastructure as Code means your servers, networks and cloud accounts are defined in version-controlled files and changed through the same review process as application code. Here are the core ideas, a worked OpenTofu example, the practices that keep it sane, and where the tools stand in 2026."
tags: ["infrastructure as code", "terraform", "opentofu", "pulumi", "crossplane", "devops", "platform engineering"]
categories: ["Programming"]
---

Somewhere in most organisations there is a cloud account that was built by clicking. Someone created a VPC in the console, added a few subnets, attached a database, opened a port "just for testing", and moved on. Two years later nobody knows why that security group allows `0.0.0.0/0` on port 5432, whether staging matches production, or how to rebuild any of it if the region goes down.

Infrastructure as Code (IaC) is the answer the industry converged on. Martin Fowler's short definition is still the best one: IaC is the approach of defining computing and network infrastructure through source code that can be treated just like any software system.[^1] The files live in Git, changes go through pull requests, and a tool turns the files into real infrastructure.

This post covers the ideas behind IaC, a worked example with OpenTofu, the practices that keep an IaC codebase healthy, and the state of the tools at the end of 2026.

## What "as code" buys you

Writing infrastructure down as code sounds like bureaucracy until you list what it gives you:

- **Reproducibility.** The same files produce the same environment. Building a second region, a new staging environment or a test copy for a single feature branch becomes a command, not a project.
- **Review.** A change to a firewall rule is a diff that a colleague reads before it ships, exactly like a code change.
- **History.** `git log` tells you who opened that port, when, and (if the commit message is any good) why.
- **Disaster recovery.** If you can recreate your infrastructure from a repository and a backup of your data, you have a recovery plan you can actually test.
- **Automation.** Once infrastructure is text, CI can lint it, test it, scan it for security problems and apply it.
- **Documentation that can't go stale.** The code *is* the description of what exists, as long as nobody changes things by hand (more on that below).

## Core concepts

### Declarative vs imperative

There are two ways to tell a machine what you want.

**Imperative** code lists steps: create a bucket, then enable versioning, then add a policy. A shell script calling the AWS CLI is imperative. It works the first time; the second run fails because the bucket already exists, so you add checks, and soon half the script is `if` statements.

**Declarative** code describes the end state: *there should be a bucket with versioning enabled and this policy*. The tool figures out the steps by comparing what you asked for with what exists. Run it twice and the second run does nothing.

Most modern IaC tools are declarative at the core. Even Pulumi, where you write TypeScript or Python with real loops and functions, uses the program to *build* a desired-state graph which an engine then reconciles. The language is imperative; the model is declarative.

### Idempotence and convergence

The property that makes declarative tools pleasant is **idempotence**: applying the same configuration twice has the same effect as applying it once. That makes it safe to run on every merge, from CI, without anyone wondering whether it will create duplicates.

### Provisioning vs configuration management

IaC tools roughly fall into two families:

| | Provisioning | Configuration management |
|---|---|---|
| Manages | Cloud resources: networks, VMs, databases, DNS, IAM | What's *inside* a machine: packages, files, services |
| Examples | Terraform, OpenTofu, Pulumi, CloudFormation, Crossplane | Ansible, Puppet, Chef, Salt |
| Typical question | "Does this database exist with these settings?" | "Is nginx installed and is this config file in place?" |

The line has blurred. With immutable infrastructure (bake an image with Packer, or ship a container, and replace instead of patch) and Kubernetes, there is much less to configure inside a machine than there used to be. But for bare metal, long-lived VMs and network gear, Ansible and friends are still very much alive.

### State

A declarative tool needs to know what it created last time, so it can tell "this resource should be updated" apart from "this is a new resource" and "this was removed from the code and should be destroyed". Terraform and OpenTofu keep that mapping in a **state file**: a JSON document linking each resource in your code (`aws_s3_bucket.logs`) to a real object (`arn:aws:s3:::acme-logs-prod`) along with its last known attributes.

State is the source of most IaC pain, so it's worth understanding early:

- It must be **shared**, or two people will create two copies of everything. Store it in a remote backend (S3, GCS, Azure Blob, a Kubernetes Secret, HCP Terraform, Pulumi Cloud and others), never in Git.
- It must be **locked** during changes, or two simultaneous applies will corrupt it.
- It often contains **secrets**, because providers record every attribute, including generated database passwords. Treat the state backend like a secrets store.

Kubernetes-native tools such as Crossplane take a different approach: the "state" is the set of custom resources stored in the cluster, and controllers reconcile continuously instead of on demand.

#### Is state ACID?

A natural question for anyone with a database background: can you make all this ACID? The honest answer is that you can make the *state file* behave close to ACID, but not the *apply*. Cloud APIs have no transactions. There is no way to create a VPC, a database and a DNS record as one unit that rolls back if the third call fails.

- **Atomicity.** Writing the state is effectively atomic on common backends: an S3 `PUT` replaces the whole object, so readers see either the old state or the new one. The apply is not atomic. If it fails on resource 7 of 10, resources 1 to 6 exist and are recorded in state; nothing rolls back. If the tool can't reach the backend at all, it writes the state to a local `errored.tfstate` so you can push it later.
- **Consistency.** Every state has a `lineage` (an ID for its history) and a `serial` (incremented on each write). The tool refuses to overwrite a state with a different lineage or an older serial, and a saved plan is rejected as stale if the state changed after it was created.
- **Isolation.** State locking (`use_lockfile = true` on S3, a DynamoDB table, the `pg` backend's advisory locks, or a run queue in HCP Terraform, Spacelift and similar) serialises runs against the same state. The gap is out-of-band change: a console click doesn't take the lock. That's drift, and only a scheduled plan or a continuously reconciling tool catches it.
- **Durability.** Turn on bucket versioning so every past state is recoverable, add replication and delete protection, and restrict who can write. If you'd rather have a real database underneath, the `pg` backend stores state in PostgreSQL with transactional writes and the usual Postgres backup and replication options.

Instead of atomicity, IaC relies on **convergence**. Because applies are idempotent, the recovery for a half-finished apply is to fix the cause and run `apply` again; the tool continues from what the state records. A few habits make that safe: keep each state small so a failure has a small blast radius, use `create_before_destroy` for replacements, put `prevent_destroy` on resources that must never go away, apply saved plans so the stale-plan check protects you, and run applies from a single pipeline so nobody bypasses the lock.

### Plan, then apply

The workflow that made Terraform popular is the **plan**: before changing anything, the tool shows you exactly what it will create, change and destroy. You read it (or a reviewer reads it in the pull request), and only then apply. A plan that says `1 to destroy` on your production database is the cheapest outage you will ever prevent.

### Drift

**Drift** is the gap between what the code says and what actually exists, usually because someone fixed something by hand at 3 a.m. Declarative tools detect drift on the next plan and will try to revert it. The fix is cultural as much as technical: either the console becomes read-only for production, or every manual change is followed by a commit that codifies it.

## A worked example with OpenTofu

The examples use OpenTofu, but the HCL is identical in Terraform: replace `tofu` with `terraform`. We'll create a versioned, private S3 bucket for application logs in two environments.

### Layout

```text
infra/
  modules/
    log-bucket/
      main.tf
      variables.tf
      outputs.tf
      tests/
        log_bucket.tftest.hcl
  envs/
    staging/
      main.tf
      backend.tf
    prod/
      main.tf
      backend.tf
```

Reusable pieces go in `modules/`; each environment is a small root module that calls them with different inputs and has its own state.

### A module

```hcl
# modules/log-bucket/variables.tf
variable "name" {
  description = "Globally unique bucket name."
  type        = string
}

variable "retention_days" {
  description = "Days to keep log objects before they expire."
  type        = number
  default     = 30

  validation {
    condition     = var.retention_days >= 1 && var.retention_days <= 3650
    error_message = "retention_days must be between 1 and 3650."
  }
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/log-bucket/main.tf
resource "aws_s3_bucket" "this" {
  bucket = var.name
  tags   = var.tags
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    id     = "expire-logs"
    status = "Enabled"
    filter {}
    expiration {
      days = var.retention_days
    }
  }
}
```

```hcl
# modules/log-bucket/outputs.tf
output "bucket_arn" {
  value = aws_s3_bucket.this.arn
}
```

Notice the references: `aws_s3_bucket.this.id` tells the tool that the versioning resource depends on the bucket, so it builds a dependency graph and creates things in the right order, in parallel where it can. You never write the order down yourself.

### An environment

```hcl
# envs/prod/backend.tf
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  backend "s3" {
    bucket       = "acme-tofu-state"
    key          = "prod/logging.tfstate"
    region       = "eu-central-1"
    encrypt      = true
    use_lockfile = true   # S3-native locking, no DynamoDB table needed
  }
}
```

```hcl
# envs/prod/main.tf
provider "aws" {
  region = "eu-central-1"
}

module "app_logs" {
  source         = "../../modules/log-bucket"
  name           = "acme-app-logs-prod"
  retention_days = 365
  tags = {
    env   = "prod"
    owner = "platform"
  }
}
```

The staging root looks the same with a different bucket name, a shorter retention and its own state key. The `terraform { }` block keeps that name in OpenTofu for compatibility.

### The workflow

```bash
cd infra/envs/prod
tofu init        # download providers, configure the backend
tofu fmt -check  # formatting
tofu validate    # syntax and type checks
tofu plan -out=plan.bin
tofu apply plan.bin
```

Saving the plan to a file and applying *that file* guarantees that what gets applied is exactly what was reviewed, even if someone merged another change in between.

The plan output is the heart of the review:

```text
  # module.app_logs.aws_s3_bucket.this will be created
  + resource "aws_s3_bucket" "this" {
      + bucket = "acme-app-logs-prod"
      ...
    }

Plan: 4 to add, 0 to change, 0 to destroy.
```

### Refactoring without destroying

Renaming a resource in code looks to the tool like "delete the old one, create a new one", which for a bucket full of logs is a disaster. `moved` blocks tell it that the object simply has a new address:

```hcl
moved {
  from = aws_s3_bucket.logs
  to   = module.app_logs.aws_s3_bucket.this
}
```

Similarly, `import` blocks bring resources that were created by hand under management, and `removed` blocks stop managing something without deleting it. All three are declarative, reviewed in a pull request, and show up in the plan.

## Practices that keep IaC healthy

### Keep state small

One giant state file for "all of production" means every plan is slow, every apply has a huge blast radius, and everyone waits on the same lock. Split state along lines of ownership and change frequency: networking, shared data stores, each application. Wire them together with outputs, data sources or remote state lookups. Tools like Terragrunt exist largely to manage many small root modules without repeating yourself.

### Modules: useful, not universal

Modules are functions for infrastructure. A good module encodes a decision your organisation made once ("every bucket blocks public access and expires its logs") so nobody has to remember it. A bad module is a thin wrapper that re-exposes every argument of the underlying resource and adds nothing but indirection. Pin module versions the same way you pin provider versions.

### Test it

IaC testing has matured a lot. The layers, from cheap to expensive:

1. **Static checks**: `fmt`, `validate`, linters like TFLint.
2. **Security and policy scanners** such as Checkov, Trivy or KICS, which flag things like unencrypted volumes or public buckets before anything is created.
3. **Unit-style tests** with the built-in test framework (`terraform test` / `tofu test`). Tests can run against a plan only, so they don't create anything:

```hcl
# modules/log-bucket/tests/log_bucket.tftest.hcl
variables {
  name           = "test-bucket-example"
  retention_days = 7
}

run "blocks_public_access" {
  command = plan

  assert {
    condition     = aws_s3_bucket_public_access_block.this.block_public_policy == true
    error_message = "Bucket must block public policies."
  }
}

run "rejects_silly_retention" {
  command = plan
  variables {
    retention_days = 0
  }
  expect_failures = [var.retention_days]
}
```

4. **Integration tests** that apply into a throwaway account and verify the result (with `command = apply` in the same framework, or Terratest in Go), then destroy everything.

### Policy as code

Tests check that *your* module behaves. Policy checks that *every* change follows the organisation's rules: no resources outside approved regions, mandatory cost-centre tags, no instance types above a certain size. Open Policy Agent (OPA) with Rego, or Conftest on top of it, evaluates the JSON form of a plan (`tofu show -json plan.bin`) and can block a merge. HashiCorp's commercial equivalent is Sentinel, and at its TechXchange 2026 conference IBM announced a new HCL-based policy framework for HCP Terraform intended to be simpler than existing policy-as-code options.[^2]

### Secrets

Three rules:

1. **Never put secrets in `.tf` files or `.tfvars` committed to Git.** Read them at runtime from a secrets manager (Vault, AWS Secrets Manager, SOPS-encrypted files) or have the cloud generate them.
2. **Lock down the state backend.** Encrypt it at rest, restrict who can read it, and log access.
3. **Use the newer features that keep secrets out of state entirely.** Terraform added ephemeral resources and values in 1.10 and OpenTofu followed in 1.11; they let a value flow through a run without ever being written to plan or state.[^3] OpenTofu also offers client-side encryption of state and plan files, which Terraform does not:[^3]

```hcl
terraform {
  encryption {
    key_provider "pbkdf2" "main" {
      passphrase = var.state_passphrase
    }
    method "aes_gcm" "main" {
      keys = key_provider.pbkdf2.main
    }
    state {
      method = method.aes_gcm.main
    }
    plan {
      method = method.aes_gcm.main
    }
  }
}
```

In real setups you would use a KMS key provider (AWS KMS, GCP KMS, OpenBao) instead of a passphrase.

### Run it from a pipeline, not a laptop

Once a team is bigger than one person, `apply` should happen in one controlled place:

```text
  pull request ──► fmt / validate / lint / scan / test ──► plan ──► plan posted as PR comment
                                                                         │
                                                         review + approval
                                                                         │
                         merge ──► apply the saved plan ──► state updated in backend
```

Atlantis (open source), HCP Terraform, Spacelift, env0, Scalr and Terragrunt's own platform all implement variants of this loop. If you already run Tekton or GitHub Actions, a plain pipeline does the job too. Give the pipeline its cloud credentials through OIDC federation rather than long-lived access keys, and give humans read-only access to production consoles. The follow-up post on [CI/CD for Infrastructure as Code]({{< relref "cicd-for-infrastructure-as-code" >}}) builds this pipeline end to end, with multiple environments and version control.

### Drift detection

Schedule a nightly `plan` against every environment. An empty plan is the expected result; anything else means someone changed infrastructure outside of code, and you want to hear about it the next morning rather than during the next unrelated deploy.

## The tool landscape in 2026

### Terraform and OpenTofu

Terraform defined the category, but two events reshaped it. In August 2023 HashiCorp moved Terraform from the open-source Mozilla Public License to the Business Source License (BSL 1.1), which restricts use that competes with HashiCorp's own products; each release converts back to MPL after four years.[^3] The community forked the last MPL version as **OpenTofu**, now stewarded by the Linux Foundation under MPL 2.0.[^3] Then IBM closed its $6.4 billion acquisition of HashiCorp in February 2025.[^4][^5] The acquisition did not change Terraform's license, but it raised questions about long-term stewardship.[^3]

For two years the tools were near-identical. In 2026 they genuinely diverged. As of late October 2026 the current versions were OpenTofu 1.13.2 and Terraform 1.17.0.[^3] Some differences:

| Feature | OpenTofu | Terraform |
|---|---|---|
| License | MPL 2.0 (OSI-approved) | BSL 1.1 |
| State and plan encryption | Since 1.7 | Not available |
| Ephemeral resources and values | Since 1.11 | Since 1.10 |
| `enabled` meta-argument | Since 1.11 | Not available |
| Provider-defined functions | Since 1.7 | Since 1.8 |
| Modules and providers from OCI registries | Since 1.10 | Not available |
| Dynamic `prevent_destroy` | Since 1.12 (May 2026) | Static only |

Sources: [^3][^6]

Terraform, in turn, has things OpenTofu doesn't, mainly on the commercial side: HCP Terraform Stacks, Sentinel and the tight integration with the rest of IBM's portfolio, including Red Hat Ansible.[^7] The HCL syntax and the state format remain compatible,[^7] and migrating from Terraform to OpenTofu is usually a matter of checking that your providers work, running `tofu init` against existing state, confirming that `tofu plan` shows no unexpected changes and swapping the binary in CI.[^3] Teams that depend on Stacks or Sentinel have more work to do.[^3]

On the commercial side, HashiCorp retired the legacy HCP Terraform Free plan on March 31, 2026 and moved remaining organisations to the "enhanced" free tier.[^8] That tier is capped at 500 managed resources, and paid plans are billed per managed resource per month.[^9]

My take: for a new project with no attachment to HCP Terraform, OpenTofu is the default choice. It's open source, compatible, and currently shipping more user-facing features. For an existing Terraform estate that's working, there's no urgency to move unless the license, pricing or a missing feature actually bites.

### Pulumi

Pulumi lets you write infrastructure in TypeScript, Python, Go, C#, Java or YAML, with real loops, functions, types, package managers and unit-testing frameworks. It's open source under Apache 2.0, and many of its providers are bridged from Terraform providers, so coverage is broad.[^10] In 2026 Pulumi leaned into interoperability: in August it made Terraform state, cross-language Terraform modules and HCL support generally available on its platform.[^11] It suits teams of software engineers who'd rather not learn a configuration language, and it's the most obvious landing spot for former CDKTF users.

### CDK for Terraform: gone

HashiCorp's own "write Terraform in a programming language" project, CDKTF, was deprecated and its repository archived on December 10, 2025; HashiCorp's guidance is to migrate to plain HCL.[^12] A community fork called CDK Terrain exists,[^13] but for new work treat CDKTF as legacy.

### Cloud-native tools

AWS CloudFormation and CDK, Azure Bicep and Google's Infrastructure Manager are good choices if you are firmly committed to one cloud. They need no separate state backend (the cloud keeps the state) and get new services on day one. The price is lock-in, and you will still need something else for DNS providers, monitoring SaaS and everything outside that cloud.

### Crossplane

Crossplane turns a Kubernetes cluster into a control plane for infrastructure. Cloud resources become Kubernetes custom resources, controllers reconcile them continuously (drift is corrected automatically, not on the next plan), and platform teams define their own higher-level APIs through compositions. Crossplane 2.0, released in August 2025, made composite and managed resources namespaced, allowed compositions to include any Kubernetes resource, and removed the old patch-and-transform composition in favour of composition functions.[^14][^15] If your organisation already runs everything through Kubernetes and GitOps with Argo CD or Flux, Crossplane fits that model naturally. If not, it's a lot of machinery to adopt for infrastructure alone.

### Configuration management

Ansible remains the default for configuring machines and network devices, and is increasingly used alongside Terraform rather than instead of it: provision with one, configure with the other. With both now under the IBM umbrella, tighter integration between the two is an explicit goal.[^7]

## Common pitfalls

- **Clicking "just this once" in production.** It's always just this once. Then the next plan wants to revert it.
- **Committing state or `.tfvars` with secrets.** Add `*.tfstate*` and secret variable files to `.gitignore` on day one.
- **Unpinned providers and modules.** A provider's major version can rename arguments or change defaults. Pin versions and commit the `.terraform.lock.hcl` file.
- **Not reading the plan.** "4 to add, 1 to change, **1 to destroy**" deserves a second look every time. Add `prevent_destroy` to resources that must never disappear.
- **One state to rule them all.** Slow, scary and lock-contended. Split early.
- **Over-abstraction.** Three layers of modules wrapping one resource make the code harder to read than the resources themselves.
- **Ignoring `count` vs `for_each`.** With `count`, removing an element from the middle of a list shifts every index after it and the tool will want to destroy and recreate those resources. Use `for_each` over a map or set keyed by a stable name.

## Getting started

If you're introducing IaC to a team that doesn't have it yet:

1. Pick one tool. OpenTofu or Terraform if you want the biggest ecosystem; Pulumi if your team prefers a general-purpose language.
2. Set up a remote, locked, encrypted state backend first.
3. Start with something new and low risk, not the production database.
4. Import existing resources gradually with `import` blocks, verifying that each import produces an empty plan.
5. Put a pipeline in front of `apply` as soon as a second person is involved.
6. Add scanning, tests and policy as the codebase grows.

Infrastructure as Code isn't really about any particular tool. It's the decision that your infrastructure should be described in text, reviewed by people, applied by machines and reproducible from a repository. The tools have changed a lot in the last three years, and they will change again. That idea hasn't.

## References

[^1]: [Infrastructure As Code](https://martinfowler.com/bliki/InfrastructureAsCode.html), Martin Fowler.
[^2]: [IBM Extends Scope and Reach of DevSecOps Workflows in the AI Era](https://devops.com/ibm-extends-scope-and-reach-of-devsecops-workflows-in-the-ai-era), DevOps.com, 2026.
[^3]: [OpenTofu vs Terraform in 2026: License, Features, and Migration](https://encore.dev/articles/opentofu-vs-terraform-2026), Encore, updated November 2026.
[^4]: [IBM closes $6.4B HashiCorp acquisition](https://techcrunch.com/?p=2972557), TechCrunch, February 2025.
[^5]: [Terraform Cloud alternatives](https://www.env0.com/blog/terraform-cloud-alternative), env0.
[^6]: [Terraform vs OpenTofu in 2026: the fork has diverged](https://ecorpit.com/terraform-vs-opentofu-2026-standardize-migration/), eCorp IT, August 2026.
[^7]: [OpenTofu vs. Terraform: Which IaC Tool Really Delivers](https://www.cloudmagazin.com/en/2026/06/05/opentofu-vs-terraform-which-iac-tool-really-delivers/), cloudmagazin, June 2026; [IBM acquires HashiCorp: Terraform and Ansible to work together](https://heise.de/-10423804), heise.
[^8]: [End of Life Notice for HCP Terraform Legacy Free Plan](https://support.hashicorp.com/hc/en-us/articles/47520801288083-End-of-Life-Notice-for-HCP-Terraform-Legacy-Free-Plan), HashiCorp Support.
[^9]: [HCP Terraform Free Tier in 2026: What Survived the EOL](https://scalr.com/learning-center/hcp-terraform-free-tier-is-being-discontinued-what-you-need-to-know), Scalr, October 2026.
[^10]: [Pulumi vs. CDK for Terraform](https://pulumi.com/docs/iac/comparisons/cdktf/), Pulumi documentation; [CDKTF Alternatives in 2026](https://encore.dev/articles/cdktf-alternatives), Encore.
[^11]: [Pulumi Releases](https://www.pulumi.com/releases/), Pulumi, August 2026.
[^12]: [hashicorp/terraform-cdk](https://github.com/hashicorp/terraform-cdk), GitHub (archived December 10, 2025); [CDK for Terraform: A Practical Guide for 2026](https://encore.dev/articles/cdktf-guide), Encore.
[^13]: [CDK Terrain](https://www.thoughtworks.com/radar/tools/cdk-terrain), Thoughtworks Technology Radar.
[^14]: [What's New in Crossplane v2](https://docs.crossplane.io/v2.2/whats-new/), Crossplane documentation.
[^15]: [Crossplane Tackles Applications alongside Cloud Infrastructure with v2.0 Release](https://www.infoq.com/news/2025/08/crossplane-applications-v2), InfoQ, August 2025.
