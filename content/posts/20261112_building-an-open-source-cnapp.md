---
title: "Building an Open Source CNAPP: What It Takes and What's Missing"
date: 2026-11-12
draft: true
description: "A capability-by-capability look at replacing a commercial CNAPP (Wiz, Cortex Cloud, CrowdStrike, Defender for Cloud, Orca) with open source tools: what Prowler, Cartography, Trivy, Falco, PMapper, Presidio and DefectDojo cover, where the real gaps are (DSPM, IAM path coverage, agentless VM scanning, correlation, threat research), and what it costs in people and infrastructure."
summary: "You can replace most of a commercial CNAPP's checks with open source. You can't download its correlation engine, its research team or its UI. Here's a gap-by-gap breakdown and what it would take to close each one."
tags: ["security", "cloud", "cnapp", "cspm", "ciem", "dspm", "open source"]
categories: ["Security"]
---

In the [previous post]({{< relref "20261112_cnapp-and-open-source-alternatives" >}}) I explained what a CNAPP is and mapped its capabilities onto open source tools. The natural follow-up question: could you skip the commercial platforms (Wiz, Palo Alto's Cortex Cloud, CrowdStrike, Microsoft Defender for Cloud, Orca and the rest) and run on open source alone? What would that take, and what would still be missing?

The short answer: you can replace most of the **checks**. What you can't download is the **product around them**: the correlation that turns millions of findings into a short ranked list, the same depth of coverage on every cloud, the research team, and the workflow. Those you'd have to build and staff yourself. Some of it is a few weeks of glue code. Some of it, like data security posture management, is a genuine gap with no good open source answer today.

## What "a CNAPP" actually does

It's easy to compare feature checklists and conclude that open source covers everything. But a commercial CNAPP does several jobs, and only the second is "run checks":

1. **Collect**: inventory every resource, identity, workload, datastore and code repository across every cloud, continuously.
2. **Detect**: run misconfiguration, vulnerability, permission, secret, sensitive-data and runtime checks.
3. **Correlate**: connect those findings so that "public VM + critical CVE + role that can read the customer database" becomes one issue instead of three unrelated alerts. Vendors describe this as joining code, cloud and runtime into one shared context.[^1]
4. **Prioritize**: rank issues by how exploitable and how damaging they are.
5. **Route and track**: assign issues to owners, open tickets, track SLAs, show progress.
6. **Stay current**: keep up with new cloud services, attack techniques and CVEs, which is what vendor research teams do full-time.

Open source is strong at 1 and 2, decent and improving at 3, and weak at 4, 5 and 6. The rest of this post goes capability by capability.

## The scorecard

| Capability | Open source status | Main tools | What's missing |
|---|---|---|---|
| Cloud posture (CSPM) and compliance | ✅ Covered | Prowler, Steampipe/Powerpipe | Little; mostly operations |
| IaC and code scanning | ✅ Covered | Checkov, Trivy | Mapping a cloud finding back to the code and owner that created it |
| Container image vulnerabilities | ✅ Covered | Trivy, Grype | Little |
| Kubernetes posture | ✅ Covered | Kubescape | Little |
| Runtime detection on hosts and containers | ✅ Mostly covered | Falco, Kubescape | Response actions, investigation UI |
| Security graph and attack paths | 🟡 Partial | Cartography, Prowler Attack Paths | Multicloud depth, curated toxic-combination logic |
| Identity and permissions (CIEM) | 🟡 Partial | PMapper, Cloudsplaining, Prowler | Many known escalation paths undetected; non-AWS depth |
| Agentless VM scanning | 🟡 Partial | `trivy vm` | AWS only, experimental, no orchestration |
| Cloud detection and response (CDR) | 🟡 Partial | Falco plugins | Correlation of cloud logs with posture; multicloud rule depth |
| Findings aggregation and workflow | 🟡 Partial | DefectDojo | Cloud-context prioritization, polished UI |
| Data security posture (DSPM) | ❌ Gap | Presidio (building block only) | The whole scanner |
| AI security posture (AI-SPM) | ❌ Mostly a gap | Early Cartography/Prowler coverage | AI inventory, model and pipeline risk |
| Threat research and intel | ❌ Gap | Community rules | A team tracking new techniques |

## The covered parts

Posture, IaC, images, Kubernetes and host runtime are well served, as covered in the previous post. The work here is operational rather than missing capability: scheduling scans across every account, managing scanner credentials, upgrading tools, and tuning noisy rules (Falco in particular needs tuning before its alerts are trustworthy).

One thing to take seriously in this category is the security of the tools themselves. Your scanners hold read access to everything. The March 2026 Trivy compromise, where attackers pushed credential-stealing code into official releases and GitHub Actions tags, shows that a self-run security stack is itself a supply chain to defend.[^2] With a commercial CNAPP, that's the vendor's problem. With open source, it's yours: pin by digest or SHA, verify signatures, and isolate scanner credentials.

## The partial gaps

### The security graph

This is the gap closest to being closed. Cartography builds a Neo4j graph of your infrastructure, and in early 2026 Prowler shipped **Attack Paths** in its open source app: after each AWS scan, it loads Cartography's inventory into a graph, adds Prowler's findings as nodes, and runs built-in openCypher queries for privilege escalation and network exposure. You can also write your own read-only queries.[^3][^4]

That's a real step toward the graph-based correlation the commercial platforms sell. What's still missing:

- **Breadth.** Prowler's Attack Paths covers AWS.[^5] Cartography ingests Azure, GCP and Kubernetes too, but the query libraries for toxic combinations on those platforms are thinner.
- **Curated correlation logic.** A commercial CNAPP's value isn't that it has a graph; it's the hundreds of maintained rules that walk the graph and decide what matters. With open source, you write and maintain those queries, and keep them working as the Cartography schema evolves.
- **Scale and operations.** Neo4j needs sizing, backups and upgrades, and full re-syncs of a large estate take time.

**To close it:** run the Prowler App with Attack Paths, plus standalone Cartography for non-AWS sources and extra integrations (Okta, GitHub, scanner results). Budget ongoing time for someone to own the query library.

### Identity and permissions (CIEM)

Working out who can actually do what in a cloud account is a computation, not a lookup. Effective access depends on identity policies, resource policies, permission boundaries, organization-level policies and role chaining.[^6] Open source tools tackle this in AWS: PMapper builds a graph of principals and escalation edges, Cloudsplaining flags risky policies, and Prowler has dedicated privilege escalation checks.

The problem is coverage. In December 2025, Datadog Security Labs published pathfinding.cloud, a catalog of AWS IAM privilege escalation paths. Of the 65 paths documented at launch, 27 (42%) weren't detected by any of the open source tools they evaluated. The author attributes this to some projects being unmaintained and others not keeping up with new research.[^7] A later benchmark across 57 paths found the tools detected anywhere from 7% to 65% of them, with Prowler at the top among established tools. That benchmark was written by the author of a competing tool, so treat its exact numbers with caution.[^8]

Two other gaps: these tools are mostly AWS-focused, and they find problems but don't do the remediation workflow (just-in-time access, automated least-privilege rollouts) that commercial CIEM sells.[^6]

**To close it:** run Prowler plus PMapper, use Datadog's free [Pathfinding Labs](https://github.com/DataDog/pathfinding-labs) environments to measure what your stack actually detects,[^9] and write custom checks for the paths you care about. For Azure and GCP, expect to write more yourself.

### Agentless VM scanning

The agentless CNAPPs' headline trick is scanning VM disks through snapshots, with no agent. The open source equivalent is Trivy's `vm` command, which can scan an AMI or EBS snapshot directly through the EBS direct APIs:[^10]

```bash
trivy vm --scanners vuln ebs:snap-0123456789abcdefg
trivy vm --scanners vuln ami:ami-0123456789abcdefg
```

The limits: it's marked experimental, the only cloud targets supported are AWS AMIs and EBS snapshots (otherwise you point it at a local disk image), and it scans one snapshot at a time.[^11] Everything a commercial platform automates around it is missing: enumerating every instance, snapshotting its volumes, scanning them, deleting the snapshots, handling encrypted volumes and cross-account access, and doing it on a schedule without running up a large EBS bill.

**To close it:** write an orchestrator, for example a Lambda or a Kubernetes job per account that snapshots, scans and cleans up, and push results into Cartography or DefectDojo. For Azure and GCP, either download disk images and scan locally, or accept agents (Falco, or a lightweight inventory agent) on long-lived VMs. Many teams simply decide that VMs must be built from scanned images in a pipeline and never patched by hand, which moves the problem left.

### Cloud detection and response

Falco covers host and container runtime well. For the cloud control plane, Falco's plugin framework includes an AWS CloudTrail plugin that applies Falco rules to CloudTrail events as a stream, without first copying them into a SIEM.[^12] The plugin system makes it possible to add other log sources.

Missing pieces: equally deep rule sets for Azure activity logs and GCP audit logs, correlation of a runtime alert with the posture graph ("this container that just spawned a shell has a role that can reach production data"), and investigation tooling. Commercial vendors are pushing hard in this direction; Palo Alto's merger of Prisma Cloud into Cortex Cloud was largely about tying CNAPP findings to its detection and response platform.[^13] With open source, you'll likely end up sending everything to a SIEM, which is either another project to run or a commercial bill.

### Findings aggregation and workflow

You need one place where findings from all these tools land, get deduplicated, assigned and tracked. The open source answer is OWASP DefectDojo, which imports output from hundreds of tools, deduplicates overlapping findings and integrates with Jira.[^14] Its maintainers are clear that the open source edition has limited UI, metrics and reporting compared with their paid edition.[^15]

DefectDojo also doesn't know your cloud graph. It can tell you a CVE was found by three scanners; it can't tell you the affected VM is internet-facing. So the prioritization step, combining graph context with findings into a risk score, is something you design yourself, typically by writing graph query results back into DefectDojo as severity adjustments or tags.

## The real gaps

### Data security posture management (DSPM)

This is the biggest hole. A DSPM tool has to find every datastore (including forgotten ones), sample its contents, classify what it contains, and connect that to who can access it.[^16] There's no mature open source project that does this end to end for cloud storage and databases.

What exists is building blocks. Microsoft Presidio is an open source SDK for detecting PII in text, images and structured data, using named-entity recognition, regular expressions, checksums and custom recognizers.[^17] It's a classifier, not a scanner: it won't crawl your buckets or connect to your databases.

**To close it, you'd build:**

1. Datastore inventory from Cartography or Steampipe (buckets, databases, warehouses, file shares).
2. A sampler that reads a statistically useful subset of objects and rows, running **inside** your accounts so data never leaves them.
3. Presidio (plus custom recognizers for your own identifiers) to classify samples.
4. Results written back into the graph as "this bucket contains credit card numbers" properties, so attack path queries can use them.
5. Controls for cost (object reads and data transfer add up) and for the scanner's own access, which by definition can read your most sensitive data.

That's a substantial internal project, and DSPM is the area where buying is most likely to make sense even for a team otherwise committed to open source.

### AI security posture

Commercial CNAPPs now inventory AI services, SDKs and models and check them for misconfiguration; Wiz and Cortex Cloud both list AI-SPM as a module.[^18][^19] Open source coverage is early: Cartography has modules for some AI platforms, and Prowler's Attack Paths added SageMaker and Bedrock data.[^20] There's no open source equivalent of an AI bill of materials across a whole estate. If you run serious AI workloads, plan for this gap.

### Threat research

This one is easy to overlook. Part of what a CNAPP subscription buys is a research team that tracks new attack techniques and turns them into detections. A vendor research team was among those that identified the Trivy supply chain attack.[^2] With open source, you depend on community rule updates, which arrive later and unevenly across projects; the pathfinding.cloud numbers above are a concrete example of the lag. You can mitigate it by following projects like pathfinding.cloud and Stratus Red Team and budgeting time for your own detection engineering, but there's no substitute for a full-time team.

## What it takes: a reference architecture

Putting it together, a full open source CNAPP looks something like this:

```text
                     ┌──────────────────────── Collectors ─────────────────────────┐
  Cloud APIs ──────► │ Prowler · Steampipe · Cartography                           │
  Code / CI  ──────► │ Checkov · Trivy (fs, image, config) · secret scanning       │
  VM disks   ──────► │ trivy vm + custom snapshot orchestrator (AWS)               │
  Clusters   ──────► │ Kubescape operator                                          │
  Datastores ──────► │ custom sampler + Presidio                                   │
                     └──────────────┬───────────────────────────┬──────────────────┘
                                    ▼                           ▼
                        Neo4j graph (Cartography,      DefectDojo (dedupe,
                        Prowler Attack Paths,          tickets, SLAs)
                        your Cypher rule library) ───► risk score adjustments
                                    ▲
  Runtime ─────────────────────────┘
  Falco (hosts, containers, CloudTrail) ──► SIEM / alerting ──► on-call
```

And the work to run it:

- **Infrastructure.** Prowler App (API, workers, Postgres), Neo4j, DefectDojo, a SIEM or log pipeline, Falco and Kubescape in every cluster, scheduled jobs in every cloud account. None of it is exotic, but it's a platform to operate, patch and back up.
- **Glue code.** Orchestrating VM snapshot scans, the DSPM sampler, writing graph context into DefectDojo, mapping findings to owners (through tags or code ownership files), and multicloud credential management.
- **People.** This is my own rough estimate rather than a sourced figure: for a mid-sized, multicloud estate, expect something like two to four engineers spending a large share of their time building and then running the stack, plus detection engineering on top. A smaller single-cloud shop running mostly Kubernetes could get by with less, because the covered parts of the scorecard are exactly the parts it needs.
- **Time.** Posture, IaC and runtime coverage take weeks. The graph and prioritization layer takes months to become trustworthy. DSPM is a project in its own right.

## Is it worth it?

On money alone, rarely. Commercial CNAPP pricing is mostly by quote, but published estimates put mid-market contracts around $75,000 a year and large enterprise deals well above $500,000.[^21][^22] The fully loaded cost of even one dedicated engineer is in the same range as the lower figure. And if you're mostly on Azure, Microsoft's foundational CSPM tier is free, which is a hard price to beat.[^23]

The reasons to build anyway are different:

- **Control and transparency.** You can read every check and every query, and fix them yourself.
- **Independence.** The market is consolidating fast, and some teams would rather not depend on a platform that may change owners or, as with Wiz joining Google, end up owned by a competing cloud provider.[^24]
- **Data residency.** Everything, including the DSPM samples, stays in your accounts.
- **Fit.** If you're mostly Kubernetes on one cloud, the parts you need are the parts open source does well.
- **Learning.** Building it teaches your team more about your cloud than any dashboard will.

There's also a middle path: the open-core vendors behind these projects (Prowler's hosted platform, SubImage for Cartography, ARMO for Kubescape, Sysdig for Falco) sell managed versions. You keep inspectable checks and avoid proprietary lock-in while paying someone else to run the infrastructure. And many teams mix the two: open source for posture, IaC and runtime, plus a commercial product for DSPM only.

## Summary

Building a CNAPP from open source is feasible for the checks and hard for everything that makes the checks useful at scale. Posture, IaC, image, Kubernetes and runtime scanning are covered. The graph and attack path layer is now within reach thanks to Cartography and Prowler's Attack Paths, at least on AWS. Identity analysis, agentless VM scanning and cloud detection are partially covered and need custom work. DSPM, AI security posture and threat research are genuine gaps.

So the honest answer to "what's missing" is: one product (DSPM), a lot of glue, and the people to own it.

## References

[^1]: [Google Completes Acquisition of Wiz](https://www.nasdaq.com/press-release/google-completes-acquisition-wiz-2026-03-11), Google press release via Nasdaq, March 11, 2026.
[^2]: [Trivy Compromised: Everything You Need to Know about the Latest Supply Chain Attack](https://wiz.io/blog/trivy-compromised-teampcp-supply-chain-attack), Wiz Research, March 2026.
[^3]: [Prowler 5.17.0 release notes](https://newreleases.io/project/github/prowler-cloud/prowler/release/5.17.0), Prowler on GitHub.
[^4]: [Attack Paths](https://docs.prowler.com/user-guide/tutorials/prowler-app-attack-paths), Prowler documentation.
[^5]: [Prowler README: Attack Paths](https://github.com/prowler-cloud/prowler), Prowler on GitHub.
[^6]: [Top 7 Open-Source Cloud Entitlement and CIEM Tools](https://startwithidentity.com/articles/top-7-open-source-cloud-entitlement-ciem-tools/), Start with Identity, August 2026.
[^7]: [Introducing Pathfinding.cloud](https://securitylabs.datadoghq.com/articles/introducing-pathfinding.cloud/), Datadog Security Labs, December 17, 2025.
[^8]: [5 AWS IAM Privesc Scanners vs 57 Paths: Coverage from 7% to 93%](https://haitmg.pl/blog/aws-iam-privesc-scanners-benchmark/), April 2026 (author also maintains one of the tools tested).
[^9]: [Introducing Pathfinding Labs](https://securitylabs.datadoghq.com/articles/introducing-pathfinding-labs.md), Datadog Security Labs, 2026.
[^10]: [trivy vm](https://trivy.dev/docs/latest/guide/references/configuration/cli/trivy_vm/), Trivy CLI reference.
[^11]: [Virtual Machine Image](https://trivy.dev/docs/latest/target/vm), Trivy documentation.
[^12]: [Falco Open Source Adds AWS Cloud Security Monitoring](https://sysdig.com/press-releases/falco-open-source-adds-aws-cloud-security-monitoring), Sysdig, October 2021.
[^13]: [Palo Alto Networks Reboots CNAPP: The Shift from Prisma Cloud to Cortex Cloud](https://www.delloro.com/palo-alto-networks-reboots-cnapp-the-shift-from-prisma-cloud-to-cortex-cloud/), Dell'Oro Group.
[^14]: [DefectDojo](https://appsecsanta.com/defectdojo), AppSec Santa, July 2026.
[^15]: [About DefectDojo](https://docs.defectdojo.com/get_started/about/about_defectdojo/), DefectDojo documentation.
[^16]: [Choosing a DSPM tool](https://orca.security/?p=73359), Orca Security.
[^17]: [Presidio: Data Protection and De-identification SDK](https://microsoft.github.io/presidio), Microsoft.
[^18]: [How to Evaluate Wiz: Common FAQs](https://wiz.io/es-es/academy/cloud-security/how-to-evaluate-wiz-faq), Wiz Academy.
[^19]: [Introducing Cortex Cloud](https://www.paloaltonetworks.com/blog/?p=334105), Palo Alto Networks blog, 2025.
[^20]: [Prowler 5.18.0 release notes](https://newreleases.io/project/github/prowler-cloud/prowler/release/5.18.0), Prowler on GitHub.
[^21]: [Wiz review: the graph-based CNAPP](https://ssojet.com/security-vendors/cnapp/wiz/), SSOJet, 2026.
[^22]: [Best Cloud Security CNAPP Platforms in 2026](https://cipherssecurity.com/?p=15453), Ciphers Security, August 2026.
[^23]: [Top 10 CNAPP tools for cloud security management in 2026](https://cloudaware.com/blog/cnapp-tools/), Cloudaware.
[^24]: [Wiz Joins Google: CNAPP Market Consolidation Risks](https://labs.cloudsecurityalliance.org/research/csa-research-note-wiz-google-cnapp-market-consolidation-2026/), Cloud Security Alliance, March 13, 2026.
