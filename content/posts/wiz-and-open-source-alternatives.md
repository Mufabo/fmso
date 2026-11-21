---
title: "What Is Wiz, and What Are the Open Source Alternatives?"
date: 2026-11-11
draft: true
description: "What Wiz actually does (agentless scanning, the Security Graph, CSPM, CWPP, CIEM, DSPM), what Google's $32 billion acquisition changed, and how to assemble an open source alternative from Prowler, Cartography, Steampipe/Powerpipe, Trivy, Checkov, Kubescape and Falco."
summary: "Wiz is the cloud security platform Google bought for $32 billion. There is no single open source Wiz, but you can get surprisingly close by combining a handful of mature projects. Here is what each piece does and where the gaps are."
tags: ["security", "cloud", "cnapp", "cspm", "kubernetes", "open source"]
categories: ["Security"]
---

If you work anywhere near cloud infrastructure, you have probably seen Wiz in a security team's toolbox, or seen the headline about Google paying $32 billion for it. This post covers what the product actually does, explains the alphabet soup it lives in, and then looks at how far you can get with open source tools instead.

## What Wiz is

Wiz is a **CNAPP**: a Cloud-Native Application Protection Platform. That term is an umbrella for several security categories that used to be sold as separate products. Wiz was founded in 2020 by four former Microsoft employees: Assaf Rappaport, Ami Luttwak, Yinon Costica and Roy Reznik.[^1]

Its pitch rests on two ideas.

**Agentless scanning.** Instead of installing an agent on every VM and container host, Wiz connects to your cloud accounts with read-only API access. It reads configuration through the cloud provider's APIs and inspects workloads by taking snapshots of their disks and scanning those, so nothing runs inside your machines.[^2] That is why it can show a full inventory within a day or so of connecting an account, where agent rollouts can take months.

**The Security Graph.** Every resource, identity, network path, vulnerability and piece of sensitive data goes into one graph. Instead of producing separate lists ("these 4,000 VMs have CVEs", "these 300 roles are over-privileged", "these buckets are public"), Wiz looks for *toxic combinations*: the one VM that is reachable from the internet **and** has a critical CVE **and** runs with a role that can read the bucket full of customer data.[^3] That correlation is the real product. Most of the individual checks are things open source tools can also do.

### The acronyms

| Acronym | Stands for | Question it answers |
|---|---|---|
| CSPM | Cloud Security Posture Management | Is my cloud configured safely? (public buckets, open security groups, logging off) |
| KSPM | Kubernetes Security Posture Management | Same question, for clusters |
| CWPP | Cloud Workload Protection Platform | Are my VMs, containers and functions vulnerable or compromised? |
| CIEM | Cloud Infrastructure Entitlement Management | Who can access what, and do they need to? |
| DSPM | Data Security Posture Management | Where is my sensitive data, and who can reach it? |
| IaC scanning | Infrastructure as Code scanning | Will this Terraform create a misconfiguration before it's applied? |
| CDR | Cloud Detection and Response | Is someone attacking me right now? |

Wiz covers all of these, plus AI security posture management for model and AI service inventories.[^4]

### Product lines

Wiz sells the platform as three main modules:[^2]

- **Wiz Cloud**: the agentless part. CSPM, vulnerability management, CIEM, DSPM and IaC checks.
- **Wiz Code**: scanning of repositories, CI/CD pipelines, container registries and images, with automated fix pull requests.
- **Wiz Defend**: runtime threat detection using an eBPF-based sensor deployed on workloads.

That last point is worth noting: agentless scanning sees posture and vulnerabilities, but not live attacks. For runtime detection, Wiz does need something running on the host.[^5]

### Who uses it, and what it costs

Wiz says half of the Fortune 100 are customers.[^6] It passed $1 billion in annual recurring revenue in 2025.[^7] Pricing isn't published; it's quoted based on workloads, accounts and modules. One vendor review puts mid-market deals at roughly $75,000 per year, with enterprise contracts often exceeding $500,000 annually.[^5]

## The Google acquisition

Google first tried to buy Wiz in 2024 for $23 billion; Wiz turned it down.[^7] Talks restarted in early 2025, and Google announced a $32 billion all-cash deal on March 18, 2025.[^8] After roughly a year of antitrust review, with unconditional clearances from the US, the European Commission and several other jurisdictions, the deal closed on **March 11, 2026**.[^9] It is the largest acquisition in Google's history, well past the $12.5 billion it paid for Motorola Mobility in 2012.[^7]

Wiz now operates as a unit inside Google Cloud but keeps its brand. Google has committed to keeping it multicloud: it continues to support AWS, Azure and Oracle Cloud as well as GCP.[^10]

For customers on other clouds, that promise matters. Some teams are nevertheless uneasy about having their AWS or Azure security posture, with full read access to their accounts, sitting in a product owned by a competing cloud provider. That, plus price, is a common reason people go looking for open source options.

## Is there an open source Wiz?

Not really. No single open source project replicates the whole platform, and anyone telling you otherwise is selling something (often the commercial edition of their open source project, which is fine, but worth knowing).

What does exist is a set of mature, focused tools that each cover one or two of the categories above. You can combine them into a stack that covers most of what Wiz checks. What you won't get out of the box is the polished correlation layer and the single UI. You build that yourself, or accept separate dashboards.

Here is the map:

| Wiz capability | Open source option |
|---|---|
| CSPM and compliance benchmarks | Prowler, Steampipe + Powerpipe, CloudSploit |
| Security graph, attack paths, "who can access what" | Cartography |
| Vulnerability scanning (images, filesystems, SBOMs) | Trivy, Grype |
| IaC scanning | Checkov (also Trivy) |
| Kubernetes posture | Kubescape |
| Runtime detection (CDR) | Falco, Kubescape runtime |
| DSPM | No strong open source equivalent |

## The tools

### Prowler: CSPM and compliance

Prowler is the closest thing to a drop-in replacement for the posture-management half of Wiz, and it is often cited as the leading free alternative.[^11] It is Apache 2.0 licensed, so every check is readable and modifiable. It started as an AWS tool and now covers Azure, GCP, Kubernetes and a range of SaaS platforms; its maintainers claim 16 cloud and SaaS providers and more than 70 compliance frameworks (CIS, PCI DSS, HIPAA, NIST and so on).[^12] Those figures come from the vendor's own comparison page, so treat the head-to-head marketing with appropriate skepticism, but the CLI really is free with no feature gates.

```bash
pip install prowler
prowler aws                                   # all checks, default credentials
prowler aws --compliance cis_3.0_aws          # just one framework
prowler kubernetes                            # in-cluster or via kubeconfig
```

Output goes to CSV, JSON (including OCSF) and HTML. There is also a self-hostable web app with dashboards, and a paid hosted version.

### Steampipe + Powerpipe: SQL over your cloud

Steampipe, from Turbot, takes a different approach: it exposes cloud APIs as Postgres tables, so you can query your infrastructure with SQL. Powerpipe sits on top and runs dashboards and benchmarks. Turbot publishes more than 5,000 open source controls covering CIS, NIST, PCI, HIPAA, FedRAMP and others, as "mods" for AWS, Azure, GCP, GitHub, Kubernetes, Terraform, Microsoft 365 and more.[^13]

```bash
steampipe plugin install aws
steampipe query "select name, region from aws_s3_bucket where bucket_policy_is_public"

powerpipe mod init
powerpipe mod install github.com/turbot/steampipe-mod-aws-compliance
steampipe service start
powerpipe benchmark list
powerpipe server                              # dashboards on localhost:9033
```

The SQL interface is the selling point: anything the benchmarks don't cover, you can ask for yourself, and you can join across accounts and providers in one query. Note the licensing: the mods are Apache 2.0, but the Steampipe and Powerpipe products themselves are distributed by Turbot under its own terms, so read their open source FAQ if you plan to redistribute.[^14]

CloudSploit is a third, older option in this space, covering AWS, Azure, GCP and Oracle Cloud misconfiguration checks.[^15]

### Cartography: the security graph

If the Security Graph is the heart of Wiz, Cartography is the closest open source equivalent. It's a Python tool that pulls assets and their relationships from your infrastructure into a Neo4j graph database.[^16] It was created at Lyft and accepted into the CNCF as a Sandbox project on August 23, 2024.[^17]

It has more than 30 integrations: the major clouds, Kubernetes, Okta, GitHub, CrowdStrike, and even AI platforms. Crucially, it can also ingest findings from other scanners (Trivy results for ECR images, for example), which is what lets you start building Wiz-style toxic-combination queries yourself.[^18]

```bash
pip install cartography
cartography --neo4j-uri bolt://localhost:7687 --selected-modules aws
cartography-rules list                        # pre-built security queries
```

Then open Neo4j's browser on port 7474 and ask questions in Cypher: which EC2 instances are internet-facing and run with an IAM role that can read S3; which GitHub users still have admin rights to repos after leaving the Okta directory; and so on. Cartography Rules adds a library of pre-defined queries so you don't start from an empty prompt.[^19]

The main operational cost is Neo4j itself, which you'll need to run and size appropriately.[^20] The original Cartography team has since founded SubImage, a commercial open-core platform built on it that explicitly positions itself as an alternative to Wiz.[^21]

### Trivy and Grype: vulnerability scanning

Trivy, from Aqua Security, is the Swiss Army knife of open source scanners: container images, filesystems, Git repositories, Kubernetes clusters, IaC files, secrets and SBOMs, all from one binary. Grype (from Anchore, usually paired with Syft for SBOM generation) is a lighter alternative focused purely on vulnerability matching.

```bash
trivy image nginx:1.27
trivy fs --scanners vuln,secret,misconfig .
trivy k8s --report summary cluster
```

A warning that applies to every tool in this post: on **March 19, 2026**, attackers compromised Trivy's release pipeline and injected credential-stealing malware into official releases and into the `trivy-action` and `setup-trivy` GitHub Actions.[^22] The attackers force-pushed malicious code to 75 of the 76 version tags of the GitHub Action, so workflows that referenced a tag like `@v0.28.0` silently ran the malicious version.[^23] The attack was traced to credentials that hadn't been fully rotated after an earlier breach in late February.[^24]

The lesson isn't "don't use Trivy". It's that security scanners run with exactly the credentials attackers want, so they are prime supply chain targets. Pin GitHub Actions to a full commit SHA rather than a tag, verify release signatures, and give scanners the narrowest credentials that work.

### Checkov: IaC scanning

Checkov scans Terraform, CloudFormation, Kubernetes manifests, Helm charts, Dockerfiles and more for misconfigurations before they are deployed. It was built by Bridgecrew, which Palo Alto Networks acquired, and it is the open source engine behind Prisma Cloud's IaC security.[^25] It's Apache 2.0, fully usable on its own, and has been downloaded more than 80 million times.[^26] Since version 2.0 it builds a graph of resources so checks can reason about relationships, not just single resources.[^27]

```bash
pip install checkov
checkov -d ./terraform
checkov -d . --framework kubernetes,helm
```

This covers the "shift-left" part of Wiz Code: catching the public bucket in the pull request instead of in production.

### Kubescape: Kubernetes posture and runtime

Kubescape, created by ARMO, scans clusters, manifests, Helm charts, repositories and container images against the NSA-CISA hardening guide, CIS Kubernetes benchmark and MITRE ATT&CK.[^28] It joined the CNCF Sandbox in December 2022 and moved to Incubating on January 13, 2025.[^29] Version 4.0, released in early 2026, brought runtime threat detection to general availability, using eBPF on the nodes.[^30]

```bash
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
kubescape scan                                # current kube context
kubescape scan framework nsa
```

For the in-cluster operator (continuous scanning plus runtime), install it with Helm.

### Falco: runtime detection

Falco watches system calls on Linux hosts and in Kubernetes, enriches them with container and pod metadata, and raises alerts when they match rules: a shell spawned inside a container, a read of `/etc/shadow`, an unexpected outbound connection. Sysdig open sourced it in 2016, donated it to the CNCF in 2018, and it graduated in February 2024.[^31] It was an early pioneer of using eBPF for security.[^32]

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco -n falco --create-namespace
```

The default modern eBPF driver needs kernel 5.8 or newer, and expect to spend time tuning rules before the alerts are signal rather than noise.[^33] Pair it with Falcosidekick to forward alerts to Slack, a SIEM or a webhook.

This is the open source counterpart to Wiz Defend, and it's arguably more mature than the commercial module: Falco is the engine inside Sysdig's own commercial product.

## A sample stack

For a small to mid-sized team running on one or two clouds plus Kubernetes, a reasonable starting point:

1. **Prowler** on a schedule (a nightly CronJob or CI job) for posture and compliance across every account.
2. **Checkov** and **Trivy** (pinned by SHA) in CI, failing pull requests on high-severity IaC findings and image vulnerabilities.
3. **Kubescape** operator in each cluster for posture.
4. **Falco** as a DaemonSet in each cluster for runtime alerts.
5. **Cartography** syncing nightly into Neo4j, ingesting the scanner results, with a growing set of saved Cypher queries for the attack paths that matter to you.
6. Ship everything into whatever you already use for alerting and tickets.

Steampipe and Powerpipe can replace or complement Prowler if your team is happier writing SQL than Python.

## Where the DIY stack falls short

Being honest about the trade-offs:

- **Correlation is on you.** The tools above produce findings; Cartography gives you a place to join them. But Wiz's real value is the curated logic that turns millions of findings into a short, ranked list. Writing and maintaining those queries is ongoing engineering work.
- **Agentless workload scanning.** Wiz scans VM disks via snapshots without touching the host. The open source tools mostly scan images in registries or need access to the running system, so long-lived VMs that never pass through a pipeline are harder to cover.
- **DSPM.** Finding and classifying sensitive data across buckets and databases is a weak spot for open source. Expect to write your own checks or buy something.
- **One UI.** You'll have several dashboards, or you'll build one.
- **Maintenance and supply chain risk.** Every tool you run is something to patch, upgrade and secure. The Trivy incident is a reminder that your security tooling is part of your attack surface.

The usual rule of thumb from people who've done this: open source works well if you have engineering time to treat security tooling as a product. If you don't, a commercial platform (Wiz or one of its competitors) is buying you that engineering time.

## Summary

Wiz is an agentless, graph-based CNAPP that bundles posture management, vulnerability scanning, identity and data risk, code scanning and runtime detection into one platform. Since March 2026 it has been part of Google Cloud. There's no single open source replacement, but Prowler, Steampipe/Powerpipe, Cartography, Trivy, Checkov, Kubescape and Falco together cover most of its checks. The thing you're really paying Wiz for is the correlation layer and the time it saves, and that is exactly what you'd be building yourself.

## References

[^1]: [Google closes record $32 billion deal for Israeli cybersecurity firm Wiz](https://www.jns.org/google-closes-record-32-billion-deal-for-israeli-cybersecurity-firm-wiz/), JNS, March 2026.
[^2]: [Wiz](https://appsecsanta.com/wiz), AppSec Santa.
[^3]: [What Is Wiz? Wiz Alternatives](https://www.wiz.io/academy/cloud-security/wiz-alternatives), Wiz Academy.
[^4]: [How to Evaluate Wiz: Common FAQs](https://wiz.io/academy/cloud-security/how-to-evaluate-wiz-faq), Wiz Academy.
[^5]: [Wiz review: the graph-based CNAPP](https://ssojet.com/security-vendors/cnapp/wiz/), SSOJet, 2026.
[^6]: [Google Completes Acquisition of Wiz](https://www.nasdaq.com/press-release/google-completes-acquisition-wiz-2026-03-11), Google press release via Nasdaq, March 11, 2026.
[^7]: [Google Completes Acquisition of Wiz in Historic $32 Billion Deal](https://cybersecuritynews.com/google-acquires-wiz/amp/), Cyber Security News, March 2026.
[^8]: [Biggest-ever exit: Google completes Wiz acquisition](https://en.globes.co.il/en/article-1001537249), Globes, March 11, 2026.
[^9]: [Google Completes $32 Billion Acquisition of Wiz](https://www.clearygottlieb.com/news-and-insights/news-listing/google-completes-32-billion-acquisition-of-wiz), Cleary Gottlieb, March 11, 2026.
[^10]: [Google completes acquisition of Wiz](https://blog.google/innovation-and-ai/infrastructure-and-cloud/google-cloud/wiz-acquisition/), Google blog, March 11, 2026.
[^11]: [Top 10 Wiz Cloud Security Alternatives in 2026](https://ethicalhacking.ai/alternatives/wiz-cloud-alternatives), EthicalHacking.ai, May 2026.
[^12]: [Prowler vs Wiz](https://prowler.com/compare/prowler-vs-wiz), Prowler (vendor comparison).
[^13]: [Powerpipe README](https://github.com/sbldevnet/powerpipe), Turbot Powerpipe on GitHub.
[^14]: [Steampipe Microsoft 365 Compliance Mod](https://github.com/turbot/steampipe-mod-microsoft365-compliance), Turbot on GitHub.
[^15]: [8 Wiz Alternatives Ranked by Price, Capability, and Best Fit](https://wizpricing.pages.dev/alternatives), 2026.
[^16]: [cartography-cncf/cartography](https://github.com/cartography-cncf/cartography), GitHub.
[^17]: [Cartography](https://www.cncf.io/projects/cartography), CNCF.
[^18]: [Cartography: Mapping Multi-Cloud Attack Surfaces with Neo4j Graph Queries](https://starlog.pages.dev/articles/data-knowledge/cartography-cncf-cartography), Starlog.
[^19]: [Cartography Rules](https://cartography-cncf.github.io/cartography/usage/rules.html), Cartography documentation.
[^20]: [Cartography: Mapping Multi-Cloud Attack Surfaces with Neo4j Graph Queries](https://starlog.pages.dev/articles/data-knowledge/cartography-cncf-cartography) (see the "Gotcha" section), Starlog.
[^21]: [SubImage: Software that maps your infrastructure](https://www.ycombinator.com/launches/MrX-subimage-software-that-maps-your-infrastructure), Launch YC.
[^22]: [Trivy Compromised: Everything You Need to Know about the Latest Supply Chain Attack](https://wiz.io/blog/trivy-compromised-teampcp-supply-chain-attack), Wiz Research, March 2026.
[^23]: [Trivy Vulnerability Scanner Compromised in Supply Chain Attack](https://threatcluster.io/cluster/trivy-vulnerability-scanner-compromised-in-supply-chain-atta-93fcccf4), ThreatCluster.
[^24]: [Critical Supply Chain Compromise in Trivy Ecosystem](https://assets.adgm.com/download/assets/432318667+-+Critical+Supply+Chain+Compromise+in+Trivy+Ecosystem.pdf/cd6aab602c2411f1899b9a2533a583b3), UAE Cyber Security Council advisory, March 27, 2026.
[^25]: [Prisma Cloud Infrastructure as Code Security](https://Www.paloaltonetworks.com/prisma/cloud/infrastructure-as-code-security), Palo Alto Networks.
[^26]: [Checkov 3.0](https://origin-researchcenter.paloaltonetworks.com/blog/?p=307181), Palo Alto Networks blog.
[^27]: [Announcing Checkov 2.0: Deepening Open Source IaC Security](https://www.paloaltonetworks.com/blog/cloud-security/checkov-2-deepening-open-source-iac-security/), Palo Alto Networks blog, April 2021.
[^28]: [CNCF Accepts Kubescape as an Incubating Project](https://www.infoq.com/news/2025/03/cncf-kubescape-incubating), InfoQ, March 2025.
[^29]: [Kubescape](https://presentations.cncf.io/projects/kubescape/), CNCF.
[^30]: [Kubescape 4.0 Brings Runtime Security and AI Agent Scanning to Kubernetes](https://infoq.com/news/2026/03/kubescape-40/), InfoQ, March 2026.
[^31]: [Falco's Journey to CNCF graduation](https://sysdig.com/blog/falco-cncf-graduation), Sysdig, November 2024.
[^32]: [Cloud Native Computing Foundation Announces Falco Graduation](https://cncf.io/announcements/2024/02/29/cloud-native-computing-foundation-announces-falco-graduation), CNCF, February 2024.
[^33]: [Falco](https://appsecsanta.com/falco), AppSec Santa.
