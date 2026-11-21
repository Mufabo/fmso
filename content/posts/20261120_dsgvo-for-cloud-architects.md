---
title: "DSGVO for Cloud Architects: What You Need to Know in Germany"
date: 2026-11-20
draft: true
description: "What a cloud architect working in Germany needs to know about the DSGVO (GDPR): the German legal layer, controller and processor roles, third-country transfers and the US question, security measures, logs and deletion, documentation, BSI C5, and the regulations that overlap with it in late 2026."
summary: "The GDPR rarely says anything about clouds, regions or Kubernetes. But almost every requirement turns into an architecture decision. Here is how the DSGVO lands on a cloud architect's desk in Germany, and what to build so the paperwork is true."
tags: ["gdpr", "dsgvo", "cloud", "architecture", "compliance", "security", "germany", "privacy"]
categories: ["Programming"]
---

The DSGVO (*Datenschutz-Grundverordnung*) is the German name for the GDPR. If you design cloud platforms in Germany, you'll hear the German acronym far more often than the English one, and you'll find that German data protection authorities, works councils and customers take it more seriously than in most other places.

The regulation itself hardly mentions technology. It doesn't say "use region `eu-central-1`" or "encrypt your etcd". Instead, it sets principles and obligations, and leaves it to you to turn them into systems. That's why so much of GDPR compliance lands on architects: the decisions about where data lives, who can reach it, how long it stays and how fast you notice a breach are architecture decisions.

This post covers what I think a cloud architect in Germany needs to know: the legal layers, the roles and contracts, third-country transfers, security measures, the data lifecycle, the documents you'll contribute to, and the regulations that overlap with the GDPR in late 2026.

A disclaimer up front: I'm not a lawyer, and this isn't legal advice. Your data protection officer and legal team decide what's sufficient. What I can do is show how the requirements map onto architecture.

If you work in financial services, my post on Kubernetes in regulated environments (BaFin, DORA and GDPR) goes deeper on the platform side. This post stays at the cloud architecture level and focuses on data protection.

## The legal layers

### GDPR: the main act

The GDPR (Regulation (EU) 2016/679) has applied since 25 May 2018[^1]. It's an EU regulation, so it applies directly in Germany without national transposition. The articles that matter most for architects are:

- **Art. 5** (principles): lawfulness, purpose limitation, **data minimisation**, accuracy, **storage limitation**, integrity and confidentiality, and accountability. You have to be able to *demonstrate* compliance, not just claim it.
- **Art. 25** (data protection by design and by default).
- **Art. 28** (processors).
- **Art. 30** (record of processing activities).
- **Art. 32** (security of processing).
- **Art. 33 and 34** (breach notification).
- **Art. 35** (data protection impact assessment).
- **Art. 44–49** (transfers to third countries).

Fines go up to €20 million or 4% of worldwide annual turnover, whichever is higher[^1].

### The German layer

The GDPR leaves room for national rules through so-called opening clauses, and Germany uses them.

**BDSG** (*Bundesdatenschutzgesetz*)[^2] adds federal rules. Two points matter for architects:

- **§38: you need a data protection officer** once, as a rule, at least 20 people are regularly involved in automated processing of personal data. For most companies that build cloud platforms, this applies. The DPO is your first stop for every question below.
- **§26 (employee data) is weaker than it looks.** In March 2023, the CJEU ruled that a Hessian rule worded almost identically to §26(1) BDSG didn't meet the requirements of the GDPR's opening clause for employment data[^3]. In practice, processing of employee data (HR systems, access logs, monitoring tools) now has to rest directly on Art. 6 GDPR, and the Hessian data protection authority published guidance on what that means[^4]. If your platform logs what employees do, and it does, that logging needs a documented legal basis and a defined purpose.

**TDDDG** (*Telekommunikation-Digitale-Dienste-Datenschutz-Gesetz*, formerly TTDSG)[^5] matters for anything with a frontend. Its §25 says that storing information on, or reading it from, a user's device requires consent, unless it's strictly necessary for the service the user asked for. That covers cookies, but also local storage, device fingerprinting, tracking pixels and mobile SDKs. The consent requirement sits *before* the GDPR question of whether you may process the resulting data.

### Who enforces it

Germany has no single data protection authority. Each federal state (*Land*) has its own authority for the private sector in that state, and the federal commissioner (BfDI) covers federal agencies and telecommunications. The authorities coordinate through the **Datenschutzkonferenz (DSK)**[^6], whose resolutions, guidance papers and lists (for example, the list of processing operations that always need an impact assessment) are effectively the working interpretation of the law in Germany. When your DPO says "the DSK says…", that's what they mean.

Practical consequence: which authority supervises you depends on where your company's main establishment is, and authorities differ somewhat in tone and focus. Your DPO will know yours.

## Roles and contracts

### Controller, processor, sub-processor

The GDPR assigns responsibility by role:

- The **controller** decides why and how personal data is processed. That's usually your company.
- A **processor** processes personal data on the controller's behalf. Your cloud provider, your managed database vendor, your SaaS logging tool and your email delivery service are typically processors.
- **Sub-processors** are processors' processors: the data centre operator, the support company in another country, the CDN.
- **Joint controllers** (Art. 26) decide purposes and means together, for example two companies running a shared platform. They need an arrangement between them.

### The data processing agreement (AVV)

Every processor needs a contract that meets Art. 28[^1]. In German it's the **AVV** (*Auftragsverarbeitungsvertrag*). The big cloud providers have standard ones, usually part of their online terms. You can't negotiate much with a hyperscaler, but you have to:

- make sure an AVV is in place *before* personal data flows,
- know the provider's **sub-processor list** and subscribe to change notifications, because you usually have a right to object,
- check where sub-processors sit, since a sub-processor outside the EU means a third-country transfer (see below).

### Why this is an architecture topic

Every managed service you add can mean another processor, another contract and possibly another transfer. Adding "just a small SaaS for error tracking" to an architecture diagram creates legal work. I find it helps to treat processors like any other dependency: list them in your architecture documentation, with their AVV status, data categories, location and sub-processors, and review new ones in the same process as new technology choices.

## Third-country transfers

This is where architects have the most influence, and where the most expensive mistakes happen.

### What counts as a transfer

Chapter V of the GDPR (Art. 44 ff.) restricts transfers of personal data to countries outside the EU/EEA[^1]. Two points surprise people:

1. **Remote access is a transfer.** Data stored in Frankfurt but readable by a support engineer in a non-EU country counts as a transfer. The EDPB's guidelines on the interplay between Art. 3 and Chapter V make clear that making data available to an importer in a third country, including by remote access, is a transfer[^7]. Follow-the-sun support models and global admin teams are transfers.
2. **Region selection alone doesn't solve the US question.** A provider headquartered in the US can be compelled under US law (for example, the CLOUD Act and FISA Section 702) to hand over data it controls, even if that data sits in an EU data centre. Whether that's a transfer is debated, but data protection authorities treat it as a risk you have to assess and mitigate.

### The legal tools

A transfer needs a legal basis. The main ones:

- **Adequacy decisions** (Art. 45): the Commission has decided that a country offers essentially equivalent protection. Transfers to those countries need no extra safeguards.
- **Standard Contractual Clauses (SCCs)** (Art. 46): contract templates adopted by the Commission in 2021[^8]. After the *Schrems II* judgment, you must also carry out a **transfer impact assessment** and add **supplementary measures** if the destination country's laws undermine the clauses. The EDPB's recommendations explain how[^9].
- **Binding Corporate Rules** for intra-group transfers, and narrow derogations (Art. 49) for occasional cases.

### The US: valid, but on appeal

Transfers to the US currently rest mainly on the **EU-US Data Privacy Framework (DPF)**, an adequacy decision adopted in July 2023 that covers US companies that self-certify[^10]. Most large US cloud providers are certified.

The DPF has survived its first legal challenge. On 3 September 2025, the General Court dismissed the annulment action brought by French MP Philippe Latombe. He appealed to the Court of Justice on 31 October 2025 (Case C-703/25 P)[^11], and the appeal is pending. Microsoft has intervened in support of the Commission[^12].

History is the reason to stay cautious: the CJEU struck down both predecessors, Safe Harbor in 2015 (*Schrems I*) and the Privacy Shield in 2020 (*Schrems II*). Each time, companies that relied only on the framework had to scramble.

### What to do as an architect

Design so that losing the DPF would be an inconvenience, not a crisis:

- **Default to EU regions** for all personal data, including backups, logs, analytics and disaster recovery. Enforce it technically (see below), not just by convention.
- **Know which services are global.** Some managed services (identity, DNS, CDN, some AI and support tooling) process data outside the selected region. Check the provider's documentation for each service, not just the platform.
- **Control the keys.** Encryption is the most effective supplementary measure the EDPB describes, but only if the provider can't access the keys[^9]. Customer-managed keys in the provider's KMS help against some threats; keys held outside the provider (an external key manager or an HSM you operate) help against more. Be honest in your assessment about which threats each option actually addresses: a provider that processes your data in clear text in memory can still be compelled to access it.
- **Keep SCCs and a transfer impact assessment ready** for every US processor, even if you rely on the DPF today.
- **Consider sovereign options for sensitive data.** European providers (for example STACKIT, IONOS or Open Telekom Cloud) and the EU-operated "sovereign" offerings of hyperscalers reduce the third-country problem. They differ in how far they go: an EU subsidiary with EU staff isn't the same as a provider outside US jurisdiction altogether. Ask precise questions.
- **Map support access.** Find out from where the provider's support staff can access your environment, and whether you can restrict or approve that access.

Enforcing region restrictions is easy to do and easy to forget. On AWS, a service control policy at the organisation level denies API calls outside approved regions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonEURegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "sts:*",
        "support:*",
        "cloudfront:*",
        "route53:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["eu-central-1", "eu-central-2", "eu-west-1"]
        }
      }
    }
  ]
}
```

The `NotAction` list exempts global services that only work through `us-east-1` endpoints. Every service on that list is one you have to assess separately. On Google Cloud, the organisation policy constraint `gcp.resourceLocations` with the value group `in:eu-locations` does the same job; Azure Policy has an "Allowed locations" built-in policy.

## Security of processing: Art. 25 and Art. 32

### What the law asks for

Art. 32 requires "appropriate technical and organisational measures" (in German: *TOMs*) for a level of security appropriate to the risk. It names examples: pseudonymisation and encryption; ongoing confidentiality, integrity, availability and resilience; the ability to restore availability and access after an incident; and a process for regularly testing the measures[^1].

Art. 25 adds **data protection by design and by default**: build privacy into the system from the start, and make the most privacy-friendly setting the default[^1].

Neither article prescribes specific controls, so you need to show that your controls fit the risk. In practice, that means a short, current description of your TOMs that matches what's actually deployed.

### What that means in a cloud architecture

- **Encryption at rest and in transit, everywhere.** Including internal traffic between services, backups, snapshots and replicas.
- **Key management with separation of duties.** The people who administer data stores shouldn't automatically administer the keys.
- **Least privilege and strong identity.** Federated identity with MFA, no long-lived access keys for humans, short-lived credentials for workloads, regular access reviews, and break-glass accounts that are monitored.
- **Network segmentation.** Private endpoints for data services, no public database endpoints, controlled egress.
- **Pseudonymisation where you can.** Analytics and test environments rarely need real names or e-mail addresses. Tokenise or hash identifiers before data leaves the production system.
- **No production personal data in test environments** unless you have a documented reason and the same protection as in production. This is one of the most common findings in audits.
- **Tested backups and restores**, because availability and the ability to restore are part of Art. 32.
- **Defaults that collect less.** Logging levels, analytics, telemetry from SDKs, retention periods: the default should be the minimal setting, and more should need a decision.

### BSI C5: the German cloud attestation

The **C5** (*Cloud Computing Compliance Criteria Catalogue*) from the Federal Office for Information Security (BSI) is the German reference standard for cloud security. An independent auditor attests a provider's compliance; a **Type 2** attestation also covers whether the controls were effective over a period of time.

C5 isn't required by the GDPR, but it's how many German customers, public-sector tenders and auditors check a provider's security, and in some sectors it's law. Under §393 SGB V, providers of statutory health insurance services (doctors, hospitals, pharmacies and others) and their processors may only process health and social data in the cloud if there's a current C5 attestation[^13]. The BSI published a new version of the catalogue in 2026, and for attestations under §393 SGB V the new standard becomes mandatory from June 2027[^14].

As an architect: ask your providers for their C5 report (Type 2, current), read the section on *complementary customer controls* (the things the provider expects *you* to do), and make sure your architecture actually does them.

## The data lifecycle

### Logs, metrics and traces are personal data

This is the point engineers most often underestimate. The CJEU decided in *Breyer* (2016) that even dynamic IP addresses can be personal data for a website operator[^15]. Logs usually contain IP addresses, user IDs, e-mail addresses in URLs or error messages, and sometimes full request bodies. Traces and metrics with user-level labels do too.

So your observability pipeline is a processing activity, with all that follows:

- **A defined purpose** (security, troubleshooting, capacity planning) and a legal basis, usually legitimate interest (Art. 6(1)(f)).
- **Minimisation at the source.** Don't log request bodies or tokens; shorten or hash IP addresses where full addresses aren't needed.
- **Retention per log type**, with automatic deletion.
- **Access control** for who can search logs.
- **Processors**: if logs go to a SaaS observability tool, that's a processor and possibly a transfer.

Minimisation is easiest in the log shipper, before data leaves the node. A Fluent Bit example that drops sensitive fields and truncates IPv4 addresses to /24:

```ini
[FILTER]
    Name    modify
    Match   app.*
    Remove  authorization
    Remove  request_body
    Remove  set_cookie

[FILTER]
    Name    lua
    Match   app.*
    script  /fluent-bit/scripts/anonymize.lua
    call    truncate_ip
```

```lua
-- /fluent-bit/scripts/anonymize.lua
function truncate_ip(tag, timestamp, record)
  local ip = record["client_ip"]
  if ip ~= nil then
    local a, b, c = string.match(ip, "^(%d+)%.(%d+)%.(%d+)%.%d+$")
    if a ~= nil then
      record["client_ip"] = a .. "." .. b .. "." .. c .. ".0"
      return 1, timestamp, record
    end
  end
  return 0, timestamp, record
end
```

Whether truncated IPs are still personal data depends on what else you hold, so treat this as risk reduction, not as anonymisation. Security logs, where you need the full address, can go to a separate, more tightly controlled stream with a shorter retention period.

### Retention and deletion

Storage limitation (Art. 5(1)(e)) means personal data must be deleted when it's no longer needed for its purpose. In Germany, the usual tool is a **deletion concept** (*Löschkonzept*), often structured along DIN 66398. It defines, for each data category, a retention period, the trigger that starts it, and how deletion happens.

Retention also has a floor: commercial and tax law (§257 HGB, §147 AO) require some records, such as invoices and accounting documents, to be kept for six to ten years. The deletion concept reconciles both.

For architects, the hard part is the copies:

- **Backups and snapshots.** You usually can't delete one person's record from an immutable backup. The common approach: backups expire after a fixed, documented period; restored data is checked against an erasure list before it goes live; backups are used only for recovery.
- **Replicas, caches, search indexes, data lakes, ML training sets and exports.** Every copy needs a retention rule.
- **Object lock and WORM storage.** Great for tamper-proof security logs and ransomware protection, but a retention lock that outlasts your retention period is a compliance problem you can't fix. Set lock periods deliberately.

Automate expiry wherever possible. An S3 lifecycle rule in Terraform:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "app_logs" {
  bucket = aws_s3_bucket.app_logs.id

  rule {
    id     = "expire-app-logs"
    status = "Enabled"

    filter {
      prefix = "app/"
    }

    expiration {
      days = 30
    }

    noncurrent_version_expiration {
      noncurrent_days = 7
    }
  }
}
```

The `noncurrent_version_expiration` block matters on versioned buckets: without it, "deleted" objects live on as old versions.

### Data subject rights

People have the right to access their data (Art. 15), to have it corrected (Art. 16) or erased (Art. 17), to restrict processing (Art. 18) and to receive it in a portable format (Art. 20). Requests must normally be answered within one month[^1].

You can only do that if you know where someone's data is. That's an architecture property: a **data inventory** that maps data categories to systems, consistent identifiers across systems, and ideally lineage for data that flows into analytics and machine learning. Systems that copy personal data without anyone recording it are the ones that make access requests painful.

## Documentation you'll contribute to

Accountability (Art. 5(2)) means you must be able to demonstrate compliance. Architects usually contribute to four documents:

- **Record of processing activities** (Art. 30, German: *VVT*). For each processing activity: purposes, categories of data and people, recipients, transfers, retention periods, and a general description of the TOMs.
- **Description of technical and organisational measures (TOMs).** It has to match reality. An outdated TOM document is worse than a short, accurate one.
- **Data protection impact assessment** (Art. 35, German: *DSFA*). Required when processing is likely to result in a high risk, for example large-scale processing of sensitive data, systematic monitoring, or new technologies such as AI applied to personal data. The DSK publishes a list of processing types that always need one[^6]. Start it during design, while you can still change the design.
- **Transfer impact assessments** for third-country transfers (see above).

## Breach readiness: 72 hours

A personal data breach must be reported to the supervisory authority within **72 hours** of becoming aware of it, unless it's unlikely to result in a risk to people; high-risk breaches must also be communicated to the affected people (Art. 33 and 34)[^1]. You also have to document every breach internally, reported or not.

Seventy-two hours is short if the first day goes on figuring out what happened. The architecture determines how fast you can answer the questions the authority will ask: what happened, which data and how many people were affected, what the likely consequences are, and what you've done about it. That needs:

- **Central, tamper-resistant logging** of access to personal data, admin actions and network flows, retained long enough to investigate.
- **Detection and alerting** that reaches someone on call.
- **An asset and data inventory**, so "which data was on that system?" is a query, not a meeting.
- **A tested incident process** that includes the DPO from the start, because the DPO decides about notification, not the engineers.

## Working with the data protection officer

Involve the DPO in design reviews, not in the week before go-live. A five-minute conversation about a new processor or a new data flow early on is much cheaper than a redesign after an objection. Good DPOs appreciate architects who arrive with a data flow diagram that shows which personal data goes where, which processors are involved, which regions, and how long data is kept. That diagram is also most of what goes into the record of processing activities.

## Neighbouring regulations

The GDPR is no longer the only EU digital regulation a cloud architect deals with. In late 2026, these overlap with it most:

- **NIS2.** Germany's NIS2 implementation act was published on 5 December 2025 and has been in force since 6 December 2025, replacing the old BSI Act (BSIG)[^16]. It imposes cybersecurity risk management, registration and incident reporting obligations on roughly 30,000 companies, with no transition period. Many Art. 32 controls do double duty here.
- **DORA** for financial entities, with its own requirements on ICT risk, incident reporting and third-party risk. See my post on Kubernetes in regulated environments.
- **EU Data Act.** Applicable since 12 September 2025, it includes obligations for cloud providers to make switching between providers easier[^17]. Useful leverage for your exit strategy.
- **AI Act.** If personal data flows into AI systems, you'll need to consider both regimes, and probably a DPIA.
- **Digital Omnibus.** In November 2025, the Commission proposed a package of targeted GDPR amendments, including a narrower definition of personal data and changes to the cookie rules. The EDPB and EDPS adopted a joint opinion in February 2026[^18], and civil society groups report that a first Council compromise removed several of the most controversial GDPR changes[^19]. Adoption is expected around the end of 2026, with effects possibly from 2027–2028[^20]. Watch it, but don't design around proposals that may not survive.

## A checklist

- [ ] An AVV with every processor; sub-processor lists reviewed and change notifications on.
- [ ] Personal data in EU regions by default, enforced by organisation policy, including backups, logs and DR.
- [ ] Global and non-regional services identified and assessed.
- [ ] SCCs and a transfer impact assessment ready for every US processor, even under the DPF.
- [ ] Encryption at rest and in transit everywhere; key ownership decided per data class.
- [ ] Federated identity with MFA, least privilege, short-lived credentials, access reviews.
- [ ] No production personal data in test environments without a documented reason.
- [ ] Logs minimised at the source, retention per log type, access controlled.
- [ ] A deletion concept that covers backups, replicas, caches and analytics copies.
- [ ] A data inventory good enough to answer access and erasure requests within a month.
- [ ] TOMs documented and matching what's deployed.
- [ ] DPIA started during design for high-risk processing.
- [ ] Breach detection, central logging and an incident process that involves the DPO.
- [ ] Current C5 Type 2 reports from your providers, and their customer controls implemented.

## Bottom line

- **The DSGVO is the GDPR plus a German layer:** BDSG (DPO requirement, employee data), TDDDG §25 for device access, and sixteen state authorities coordinated by the DSK.
- **Every managed service is a potential processor and a potential transfer.** Treat processors as dependencies with documentation.
- **Region selection isn't enough.** Remote access is a transfer, and US providers raise questions that encryption with your own keys only partly answers. The DPF is valid but on appeal; keep a fallback ready.
- **Logs are personal data.** Minimise at the source and delete on schedule.
- **Deletion is an architecture property.** Every copy needs a retention rule.
- **Breach readiness is mostly logging and inventory,** built before you need it.
- **Bring in the DPO early,** with a data flow diagram.

## References

[^1]: [Regulation (EU) 2016/679 (GDPR)](https://eur-lex.europa.eu/eli/reg/2016/679/oj), EUR-Lex.
[^2]: [Bundesdatenschutzgesetz (BDSG)](https://www.gesetze-im-internet.de/bdsg_2018/), gesetze-im-internet.de.
[^3]: [CJEU, judgment of 30 March 2023, C-34/21 (Hauptpersonalrat der Lehrerinnen und Lehrer)](https://curia.europa.eu/juris/liste.jsf?language=de&num=C-34/21), Curia. For a summary of the consequences for §26 BDSG, see [EuGH: Keine Generalklauseln im Beschäftigtendatenschutz](https://cms.law/de/deu/legal-updates/eugh-keine-generalklauseln-im-beschaeftigtendatenschutz), CMS.
[^4]: [Handreichung zur Verarbeitung personenbezogener Daten von Beschäftigten im Lichte des EuGH-Urteils vom 30. März 2023 (PDF)](https://www.datenschutz.Hessen.de/sites/datenschutz.hessen.de/files/2023-05/handreichung_beschaeftigtendatenschutz_eugh-urteil.pdf), Hessian Commissioner for Data Protection and Freedom of Information.
[^5]: [Telekommunikation-Digitale-Dienste-Datenschutz-Gesetz (TDDDG)](https://www.gesetze-im-internet.de/ttdsg/), gesetze-im-internet.de.
[^6]: [Datenschutzkonferenz (DSK)](https://www.datenschutzkonferenz-online.de/), resolutions, guidance and the list of processing operations requiring a DPIA.
[^7]: [Guidelines 05/2021 on the interplay between the application of Article 3 and the provisions on international transfers as per Chapter V of the GDPR](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-052021-interplay-between-application-article-3_en), EDPB.
[^8]: [Commission Implementing Decision (EU) 2021/914 on standard contractual clauses](https://eur-lex.europa.eu/eli/dec_impl/2021/914/oj), EUR-Lex.
[^9]: [Recommendations 01/2020 on measures that supplement transfer tools](https://www.edpb.europa.eu/our-work-tools/our-documents/recommendations/recommendations-012020-measures-supplement-transfer_en), EDPB.
[^10]: [Commission Implementing Decision (EU) 2023/1795 (EU-US Data Privacy Framework)](https://eur-lex.europa.eu/eli/dec_impl/2023/1795/oj), EUR-Lex.
[^11]: [Latombe filed appeal against General Court dismissal of challenge to EU-US Data Privacy Framework adequacy decision](https://digitalpolicyalert.org/event/35459-latombe-filed-appeal-against-general-court-dismissal-of-challenge-to-european-unionunited-states-data-protection-framework-adequacy-decision-in-latombe-v-commission), Digital Policy Alert. See also [Framework Intact, For Now: EU-US Data Flows Dodge Another Bullet](https://www.kslaw.com/news-and-insights/framework-intact-for-now-eu-us-data-flows-dodge-another-bullet), King & Spalding.
[^12]: [Microsoft backs EU Commission in EU court case on transatlantic data flows](https://ieu-monitoring.com/editorial/microsoft-backs-eu-commission-in-eu-court-case-on-transatlantic-data-flows-and-privacy/1244467/), IEU Monitoring, 28 June 2026.
[^13]: [§393 SGB V: Cloud-Einsatz im Gesundheitswesen](https://www.gesetze-im-internet.de/sgb_5/__393.html), gesetze-im-internet.de.
[^14]: [Der neue C5-Katalog ist da – was Sie jetzt beachten müssen](https://www.roedl.com/insights/der-neue-c5-katalog-ist-da-was-sie-jetzt-beachten-muessen/), Rödl & Partner, 29 April 2026; and [C5-Pflicht und neues C3A-Framework](https://www.roedl.com/insights/c5-pflicht-neues-c3a-framework/), Rödl & Partner, 29 May 2026.
[^15]: [CJEU, judgment of 19 October 2016, C-582/14 (Breyer)](https://curia.europa.eu/juris/liste.jsf?num=C-582/14), Curia.
[^16]: [Das NIS-2-Umsetzungsgesetz ist in Kraft getreten](https://www.datenschutz-notizen.de/das-nis-2-umsetzungsgesetz-ist-in-kraft-getreten-welche-schritte-sind-nun-erforderlich-4457509/), Datenschutz-Notizen; [Germany's implementation of NIS2](https://www.taylorwessing.com/de/insights-and-events/insights/2025/11/germanys-implementation-of-nis2), Taylor Wessing.
[^17]: [Regulation (EU) 2023/2854 (Data Act)](https://eur-lex.europa.eu/eli/reg/2023/2854/oj), EUR-Lex.
[^18]: [The Digital Omnibus: GDPR Update](https://www.lexgo.be/en/news-and-articles/14776-the-digital-omnibus-gdpr-update), Lexgo, 29 April 2026, on EDPB-EDPS Joint Opinion 2/2026.
[^19]: [The Digital Omnibus: A step back from the brink, but the risks remain](https://edri.org/our-work/the-digital-omnibus-a-step-back-from-the-brink-but-the-risks-remain/), EDRi, 17 March 2026.
[^20]: [The EU Digital Omnibus: A Reset for Europe's Digital Rulebook](https://www.grantthornton.nl/en/insights-en/legal-services/the-eu-digital-omnibus-a-reset-for-europes-digital-rulebook/), Grant Thornton.
