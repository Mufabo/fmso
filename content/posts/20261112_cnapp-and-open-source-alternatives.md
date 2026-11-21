---
title: "What Is a CNAPP? The Commercial Platforms and Their Open Source Alternatives"
date: 2026-11-12
draft: true
description: "What a Cloud-Native Application Protection Platform (CNAPP) is, the capabilities it bundles (CSPM, CWPP, CIEM, DSPM, CDR), how the major vendors (Wiz, Palo Alto Cortex Cloud, CrowdStrike, Microsoft Defender for Cloud, Orca, Sysdig, Aqua) differ, and which open source tools cover the same ground."
summary: "CNAPP is the category that swallowed cloud security: posture, workloads, identities, data and runtime in one platform. Here's what it means, who sells it, and how far open source tools like Prowler, Cartography, Trivy, Checkov, Kubescape and Falco can take you."
tags: ["security", "cloud", "cnapp", "cspm", "kubernetes", "open source"]
categories: ["Security"]
---

If you work anywhere near cloud infrastructure, you've probably run into the acronym CNAPP, usually in a vendor pitch, or in the headline about Google paying $32 billion for Wiz. This post explains what a CNAPP actually is, who sells them, and how far you can get with open source tools instead. A [follow-up post]({{< relref "20261112_building-an-open-source-cnapp" >}}) looks at what it would take to replace a commercial CNAPP entirely.

## What a CNAPP is

CNAPP stands for **Cloud-Native Application Protection Platform**. Gartner coined the term in 2021.[^1] The idea behind it was that securing cloud applications properly needs one integrated approach that runs from development all the way to production, rather than a pile of separate tools.[^2]

Before CNAPP, a security team might buy a posture management tool, a workload protection tool, a container scanner, an IaC scanner and an identity tool from five vendors, each with its own console and alert stream. Gartner's definition explicitly describes CNAPPs as consolidating those previously separate capabilities into one product.[^3] Part of the reason for coining the term was to push the market further toward that consolidation.[^4]

### The building blocks

| Acronym | Stands for | Question it answers |
|---|---|---|
| CSPM | Cloud Security Posture Management | Is my cloud configured safely? (public buckets, open security groups, logging off) |
| KSPM | Kubernetes Security Posture Management | Same question, for clusters |
| CWPP | Cloud Workload Protection Platform | Are my VMs, containers and functions vulnerable or compromised? |
| CIEM | Cloud Infrastructure Entitlement Management | Who can access what, and do they need to? |
| DSPM | Data Security Posture Management | Where is my sensitive data, and who can reach it? |
| IaC scanning | Infrastructure as Code scanning | Will this Terraform create a misconfiguration before it's applied? |
| CDR | Cloud Detection and Response | Is someone attacking me right now? |
| AI-SPM | AI Security Posture Management | What AI services and models do I run, and are they configured safely? |

Not every CNAPP has all of these, and Gartner itself notes that no product is best at every capability.[^5] Over time, runtime visibility and detection have moved from optional extras to expected parts of the platform.[^6]

### What makes it a platform rather than a bundle

The real value of a CNAPP isn't the individual checks; most of those are commodity. It's **correlation**. Separate tools produce separate lists: 4,000 VMs with CVEs, 300 over-privileged roles, 50 public buckets. A good CNAPP joins those findings in one data model, typically a graph of resources, identities, network paths and data. It then surfaces the handful of *toxic combinations* that matter: the internet-facing VM with a critical CVE whose role can read the bucket full of customer data.

## How the vendors differ

### Agentless vs agent-based

The biggest architectural split is how a CNAPP looks inside workloads.

- **Agentless** platforms connect to your cloud accounts with read-only API access. They read configuration through the provider's APIs and scan workloads by snapshotting their disks, so nothing runs on your machines. Orca pioneered this with what it calls SideScanning, and Wiz built its success on the same approach.[^7] The appeal is speed: full inventory in a day, no agent rollout.
- **Agent-based** platforms put a sensor on each host or node. That's slower to deploy, but it sees what's actually happening at runtime, which snapshots can't.

In practice most vendors now do both. Wiz, Orca and Palo Alto are agentless-first with optional agents; CrowdStrike started agent-first and added agentless posture scanning.[^8]

### The main players

As of late 2026, the commercial shortlist usually includes these:[^9]

- **Wiz** (now part of Google Cloud). Agentless, built around its Security Graph. Google closed its $32 billion acquisition on March 11, 2026, the largest in Google's history; Wiz keeps its brand and continues to support AWS, Azure and Oracle Cloud as well as GCP.[^10]
- **Palo Alto Networks Cortex Cloud**. This is what Prisma Cloud became: in February 2025 Palo Alto merged Prisma Cloud with its Cortex CDR detection engine and folded it into its Cortex security operations platform.[^11] Prisma Cloud itself was assembled from several acquisitions (Evident.io, RedLock, Twistlock, PureSec), and it has the broadest feature list of the group.[^12]
- **CrowdStrike Falcon Cloud Security**. Agent-first, building on CrowdStrike's endpoint agent; a natural fit for existing Falcon customers.
- **Microsoft Defender for Cloud**. The default for Azure-heavy shops. Its foundational CSPM tier is free, with paid plans priced per resource, and it also covers AWS and GCP.[^13]
- **Orca Security**. Agentless, independent, often positioned for mid-market and data security.
- **Sysdig**. Runtime-first, built on the open source Falco engine.
- **Aqua Security**. Container-first, and the company behind the open source scanner Trivy.
- Others: SentinelOne, Check Point, Tenable, Fortinet (which absorbed Lacework), Qualys, Upwind and more.

### Consolidation

The market has been consolidating hard. Gartner's 2023 market guide predicted that most enterprises would consolidate posture and workload protection onto a single vendor.[^5] On the vendor side, Google bought Wiz, Palo Alto rebuilt Prisma Cloud into Cortex Cloud, and SentinelOne, Tenable and CrowdStrike all bought cloud security startups to fill gaps.[^14] Not every story ends well: Fortinet picked up Lacework in 2024 for a reported ~$200 million, a fraction of its earlier $8.3 billion valuation.[^14]

For buyers, consolidation cuts both ways. Fewer consoles is good; less choice and more lock-in is not. The Cloud Security Alliance suggested enterprises treat the Wiz acquisition as a vendor risk event worth reviewing, given that a hyperscaler now owns a leading tool that has read access into rival clouds.[^15]

### What it costs

Commercial CNAPPs are priced by workloads, accounts and modules, and few publish list prices. One vendor review puts Wiz at around $75,000 a year for mid-market, with enterprise contracts often over $500,000.[^16] Another puts Cortex Cloud anywhere from $100,000 to more than $500,000 a year.[^17] Microsoft's free foundational tier is the exception at the low end.

Price, lock-in, and wanting to see what the tool actually checks are the usual reasons people look at open source.

## Is there an open source CNAPP?

Not as a single product. No open source project replicates a full commercial CNAPP, and anyone telling you otherwise is usually selling the commercial edition of their open source project (which is fine, but worth knowing).

What does exist is a set of mature, focused tools, each covering one or two of the categories above. Combined, they cover most of what a commercial CNAPP checks. What you don't get out of the box is the correlation layer and the single UI; you build those yourself, or live with several dashboards.

| CNAPP capability | Open source option |
|---|---|
| CSPM and compliance benchmarks | Prowler, Steampipe + Powerpipe, CloudSploit |
| Security graph, attack paths, "who can access what" | Cartography |
| Vulnerability scanning (images, filesystems, SBOMs) | Trivy, Grype |
| IaC scanning | Checkov (also Trivy) |
| Kubernetes posture | Kubescape |
| Runtime detection (CDR) | Falco, Kubescape runtime |
| DSPM | No strong open source equivalent |

Interestingly, several commercial CNAPPs are built on these projects: Sysdig on Falco, Palo Alto's IaC scanning on Checkov, ARMO's platform on Kubescape. The open source tools are not toy versions; they're often the engines.

## The tools

### Prowler: CSPM and compliance

Prowler is the closest thing to a drop-in for the posture-management half of a CNAPP, and it's frequently cited as the strongest free alternative in the category.[^18] It's Apache 2.0 licensed, so every check is readable and modifiable. It started as an AWS tool and now covers Azure, GCP, Kubernetes and a range of SaaS platforms; its maintainers claim 16 cloud and SaaS providers and more than 70 compliance frameworks (CIS, PCI DSS, HIPAA, NIST and so on).[^19] Those numbers come from the vendor's own comparison page, so read the head-to-head marketing with appropriate skepticism, but the CLI really is free with no feature gates.

```bash
pip install prowler
prowler aws                                   # all checks, default credentials
prowler aws --compliance cis_3.0_aws          # just one framework
prowler kubernetes                            # in-cluster or via kubeconfig
```

Output goes to CSV, JSON (including OCSF) and HTML. There's also a self-hostable web app with dashboards, and a paid hosted version.

### Steampipe + Powerpipe: SQL over your cloud

Steampipe, from Turbot, exposes cloud APIs as Postgres tables, so you can query your infrastructure with SQL. Powerpipe sits on top and runs dashboards and benchmarks. Turbot publishes more than 5,000 open source controls covering CIS, NIST, PCI, HIPAA, FedRAMP and others, as "mods" for AWS, Azure, GCP, GitHub, Kubernetes, Terraform, Microsoft 365 and more.[^20]

```bash
steampipe plugin install aws
steampipe query "select name, region from aws_s3_bucket where bucket_policy_is_public"

powerpipe mod init
powerpipe mod install github.com/turbot/steampipe-mod-aws-compliance
steampipe service start
powerpipe benchmark list
powerpipe server                              # dashboards on localhost:9033
```

The SQL interface is the selling point: anything the benchmarks don't cover, you can ask yourself, and you can join across accounts and providers in one query. On licensing, the mods are Apache 2.0, but the Steampipe and Powerpipe products themselves are distributed by Turbot under its own terms, so read their open source FAQ if you plan to redistribute.[^21]

CloudSploit is a third, older option in this space, covering AWS, Azure, GCP and Oracle Cloud misconfiguration checks.[^22]

### Cartography: the security graph

If correlation is the heart of a CNAPP, Cartography is the closest open source equivalent. It's a Python tool that pulls assets and their relationships into a Neo4j graph database.[^23] It was created at Lyft and accepted into the CNCF as a Sandbox project on August 23, 2024.[^24]

It has more than 30 integrations: the major clouds, Kubernetes, Okta, GitHub, CrowdStrike, and even AI platforms. It can also ingest findings from other scanners (Trivy results for ECR images, for example), which is what lets you start building toxic-combination queries yourself.[^25]

```bash
pip install cartography
cartography --neo4j-uri bolt://localhost:7687 --selected-modules aws
cartography-rules list                        # pre-built security queries
```

Then open Neo4j's browser on port 7474 and ask questions in Cypher: which EC2 instances are internet-facing and run with an IAM role that can read S3; which GitHub users still have admin rights after leaving the Okta directory. Cartography Rules adds a library of pre-defined queries so you don't start from a blank prompt.[^26]

The main operational cost is Neo4j itself, which you'll need to run and size.[^27] The original Cartography team has since founded SubImage, a commercial open-core platform built on it.[^28]

### Trivy and Grype: vulnerability scanning

Trivy, from Aqua Security, is the Swiss Army knife of open source scanners: container images, filesystems, Git repositories, Kubernetes clusters, IaC files, secrets and SBOMs, all from one binary. Grype (from Anchore, usually paired with Syft for SBOM generation) is a lighter alternative focused on vulnerability matching.

```bash
trivy image nginx:1.27
trivy fs --scanners vuln,secret,misconfig .
trivy k8s --report summary cluster
```

A warning that applies to every tool in this post: on **March 19, 2026**, attackers compromised Trivy's release pipeline and injected credential-stealing malware into official releases and into the `trivy-action` and `setup-trivy` GitHub Actions.[^29] They force-pushed malicious code to 75 of the 76 version tags of the GitHub Action, so workflows that referenced a tag silently ran the malicious version.[^30] The attack was traced to credentials that hadn't been fully rotated after an earlier breach in late February.[^31]

The lesson isn't "don't use Trivy". It's that security scanners run with exactly the credentials attackers want, so they're prime supply chain targets. Pin GitHub Actions to a full commit SHA rather than a tag, verify release signatures, and give scanners the narrowest credentials that work.

### Checkov: IaC scanning

Checkov scans Terraform, CloudFormation, Kubernetes manifests, Helm charts, Dockerfiles and more for misconfigurations before deployment. It was built by Bridgecrew, which Palo Alto acquired, and it's the open source engine behind Palo Alto's IaC security.[^32] It's Apache 2.0, fully usable on its own, and has been downloaded more than 80 million times.[^33] Since version 2.0 it builds a graph of resources so checks can reason about relationships between them.[^34]

```bash
pip install checkov
checkov -d ./terraform
checkov -d . --framework kubernetes,helm
```

This covers the "shift-left" part of a CNAPP: catching the public bucket in the pull request rather than in production.

### Kubescape: Kubernetes posture and runtime

Kubescape, created by ARMO, scans clusters, manifests, Helm charts, repositories and images against the NSA-CISA hardening guide, the CIS Kubernetes benchmark and MITRE ATT&CK.[^35] It joined the CNCF Sandbox in December 2022 and moved to Incubating on January 13, 2025.[^36] Version 4.0, released in early 2026, brought eBPF-based runtime threat detection to general availability.[^37]

```bash
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
kubescape scan                                # current kube context
kubescape scan framework nsa
```

For continuous scanning plus runtime, install the in-cluster operator with Helm.

### Falco: runtime detection

Falco watches system calls on Linux hosts and in Kubernetes, enriches them with container and pod metadata, and alerts when they match rules: a shell spawned inside a container, a read of `/etc/shadow`, an unexpected outbound connection. Sysdig open sourced it in 2016, donated it to the CNCF in 2018, and it graduated in February 2024.[^38] It was an early pioneer of using eBPF for security.[^39]

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco -n falco --create-namespace
```

The default modern eBPF driver needs kernel 5.8 or newer, and expect to spend time tuning rules before the alerts are signal rather than noise.[^40] Pair it with Falcosidekick to forward alerts to Slack, a SIEM or a webhook. This is the open source counterpart to a CNAPP's runtime module, and it's the engine inside Sysdig's commercial product.

## A sample stack

For a small to mid-sized team on one or two clouds plus Kubernetes, a reasonable starting point:

1. **Prowler** on a schedule (a nightly CronJob or CI job) for posture and compliance across every account.
2. **Checkov** and **Trivy** (pinned by SHA) in CI, failing pull requests on high-severity IaC findings and image vulnerabilities.
3. **Kubescape** operator in each cluster for posture.
4. **Falco** as a DaemonSet in each cluster for runtime alerts.
5. **Cartography** syncing nightly into Neo4j, ingesting scanner results, with a growing set of saved Cypher queries for the attack paths that matter to you.
6. Everything shipped into whatever you already use for alerting and tickets.

Steampipe and Powerpipe can replace or complement Prowler if your team is happier writing SQL than Python.

## Where the DIY stack falls short

- **Correlation is on you.** The tools produce findings; Cartography gives you a place to join them. But the curated logic that turns millions of findings into a short ranked list is what you're paying a CNAPP vendor for, and writing and maintaining those queries is ongoing work.
- **Agentless workload scanning.** Commercial agentless CNAPPs scan VM disks through snapshots without touching the host. The open source tools mostly scan images in registries or need access to the running system.
- **DSPM.** Finding and classifying sensitive data across buckets and databases is a weak spot for open source.
- **One UI.** You'll have several dashboards, or you'll build one.
- **Maintenance and supply chain risk.** Every tool you run is something to patch, upgrade and secure. The Trivy incident is a reminder that security tooling is part of your attack surface.

The [follow-up post]({{< relref "20261112_building-an-open-source-cnapp" >}}) goes through each of these gaps in detail.

## Summary

A CNAPP bundles posture management, workload and container scanning, identity and data risk, IaC scanning and runtime detection into one platform, and its real value is correlating those findings into a short list of real risks. The commercial market is consolidating fast around a handful of platforms: Wiz (now Google), Palo Alto's Cortex Cloud, CrowdStrike, Microsoft and a few independents. There's no single open source CNAPP, but Prowler, Steampipe/Powerpipe, Cartography, Trivy, Checkov, Kubescape and Falco together cover most of the checks. What you'd be building yourself is the correlation and the time it saves.

## References

[^1]: [What is CNAPP?](https://dynatrace.com/knowledge-base/cnapp), Dynatrace.
[^2]: [What Is a Cloud-Native Application Protection Platform (CNAPP)?](https://old.rapid7.com/fundamentals/what-is-cloud-native-application-protection-platform-cnapp), Rapid7.
[^3]: [Gartner CNAPP Market Guide](https://resources.trendmicro.com/Gartner-CNAPP-Market-Guide.html), Trend Micro (quoting the March 2023 Gartner Market Guide).
[^4]: [Why an emerging cloud security trend offers good news to businesses](https://venturebeat.com/2021/11/23/why-an-emerging-cloud-security-trend-offers-good-news-to-businesses/amp/), VentureBeat, November 2021.
[^5]: [How Does Gartner Define CNAPP?](https://www.aquasec.com/cloud-native-academy/cnapp/cnapp-gartner/), Aqua Security.
[^6]: [2024 Gartner CNAPP Market Guide: Key Takeaways](https://www.crowdstrike.com/blog/2024-gartner-cnapp-market-guide-key-takeaways), CrowdStrike.
[^7]: [Top Cloud Security Platforms 2026](https://uinat.pages.dev/rankings/top-cloud-security-platforms-2026).
[^8]: [Best CNAPP Platforms 2026](https://www.decryptiondigest.com/blog/best-cnapp-platforms-2026), Decryption Digest.
[^9]: [Top 10 CNAPP tools for cloud security management in 2026](https://cloudaware.com/blog/cnapp-tools/), Cloudaware.
[^10]: [Google Completes Acquisition of Wiz](https://www.nasdaq.com/press-release/google-completes-acquisition-wiz-2026-03-11), Google press release via Nasdaq, March 11, 2026.
[^11]: [Palo Alto Networks Introduces Cortex Cloud](https://www.paloaltonetworks.com/company/press/2025/palo-alto-networks-introduces-cortex-cloud--the-future-of-real-time-cloud-security), Palo Alto Networks, February 13, 2025.
[^12]: [CNAPP vendors in 2026: directory and buyer overview](https://ssojet.com/security-vendors/cnapp/), SSOJet.
[^13]: [Top 10 CNAPP tools for cloud security management in 2026](https://cloudaware.com/blog/cnapp-tools/) (pricing table), Cloudaware.
[^14]: [Market Consolidation & M&A Trends](https://cysec-5q3.pages.dev/analysis/consolidation).
[^15]: [Wiz Joins Google: CNAPP Market Consolidation Risks](https://labs.cloudsecurityalliance.org/research/csa-research-note-wiz-google-cnapp-market-consolidation-2026/), Cloud Security Alliance, March 13, 2026.
[^16]: [Wiz review: the graph-based CNAPP](https://ssojet.com/security-vendors/cnapp/wiz/), SSOJet, 2026.
[^17]: [Best Cloud Security CNAPP Platforms in 2026](https://cipherssecurity.com/?p=15453), Ciphers Security, August 2026.
[^18]: [Top 10 Wiz Cloud Security Alternatives in 2026](https://ethicalhacking.ai/alternatives/wiz-cloud-alternatives), EthicalHacking.ai, May 2026.
[^19]: [Prowler vs Wiz](https://prowler.com/compare/prowler-vs-wiz), Prowler (vendor comparison).
[^20]: [Powerpipe README](https://github.com/sbldevnet/powerpipe), Turbot Powerpipe on GitHub.
[^21]: [Steampipe Microsoft 365 Compliance Mod](https://github.com/turbot/steampipe-mod-microsoft365-compliance), Turbot on GitHub.
[^22]: [8 Wiz Alternatives Ranked by Price, Capability, and Best Fit](https://wizpricing.pages.dev/alternatives), 2026.
[^23]: [cartography-cncf/cartography](https://github.com/cartography-cncf/cartography), GitHub.
[^24]: [Cartography](https://www.cncf.io/projects/cartography), CNCF.
[^25]: [Cartography: Mapping Multi-Cloud Attack Surfaces with Neo4j Graph Queries](https://starlog.pages.dev/articles/data-knowledge/cartography-cncf-cartography), Starlog.
[^26]: [Cartography Rules](https://cartography-cncf.github.io/cartography/usage/rules.html), Cartography documentation.
[^27]: [Cartography: Mapping Multi-Cloud Attack Surfaces with Neo4j Graph Queries](https://starlog.pages.dev/articles/data-knowledge/cartography-cncf-cartography) (see the "Gotcha" section), Starlog.
[^28]: [SubImage: Software that maps your infrastructure](https://www.ycombinator.com/launches/MrX-subimage-software-that-maps-your-infrastructure), Launch YC.
[^29]: [Trivy Compromised: Everything You Need to Know about the Latest Supply Chain Attack](https://wiz.io/blog/trivy-compromised-teampcp-supply-chain-attack), Wiz Research, March 2026.
[^30]: [Trivy Vulnerability Scanner Compromised in Supply Chain Attack](https://threatcluster.io/cluster/trivy-vulnerability-scanner-compromised-in-supply-chain-atta-93fcccf4), ThreatCluster.
[^31]: [Critical Supply Chain Compromise in Trivy Ecosystem](https://assets.adgm.com/download/assets/432318667+-+Critical+Supply+Chain+Compromise+in+Trivy+Ecosystem.pdf/cd6aab602c2411f1899b9a2533a583b3), UAE Cyber Security Council advisory, March 27, 2026.
[^32]: [Prisma Cloud Infrastructure as Code Security](https://Www.paloaltonetworks.com/prisma/cloud/infrastructure-as-code-security), Palo Alto Networks.
[^33]: [Checkov 3.0](https://origin-researchcenter.paloaltonetworks.com/blog/?p=307181), Palo Alto Networks blog.
[^34]: [Announcing Checkov 2.0: Deepening Open Source IaC Security](https://www.paloaltonetworks.com/blog/cloud-security/checkov-2-deepening-open-source-iac-security/), Palo Alto Networks blog, April 2021.
[^35]: [CNCF Accepts Kubescape as an Incubating Project](https://www.infoq.com/news/2025/03/cncf-kubescape-incubating), InfoQ, March 2025.
[^36]: [Kubescape](https://presentations.cncf.io/projects/kubescape/), CNCF.
[^37]: [Kubescape 4.0 Brings Runtime Security and AI Agent Scanning to Kubernetes](https://infoq.com/news/2026/03/kubescape-40/), InfoQ, March 2026.
[^38]: [Falco's Journey to CNCF graduation](https://sysdig.com/blog/falco-cncf-graduation), Sysdig, November 2024.
[^39]: [Cloud Native Computing Foundation Announces Falco Graduation](https://cncf.io/announcements/2024/02/29/cloud-native-computing-foundation-announces-falco-graduation), CNCF, February 2024.
[^40]: [Falco](https://appsecsanta.com/falco), AppSec Santa.
