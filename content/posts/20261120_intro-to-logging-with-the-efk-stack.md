---
title: "An Introduction to Logging with the EFK Stack"
date: 2026-11-20
draft: true
description: "How container logging works in Kubernetes, what Elasticsearch, Fluent Bit (or Fluentd) and Kibana each do, and how to build a working EFK pipeline with the ECK operator and Fluent Bit's YAML configuration, plus retention, structured logs and the pitfalls that bite first."
summary: "kubectl logs is fine until a Pod is gone, a node dies, or the bug spans twelve services. The EFK stack (Elasticsearch, Fluent Bit and Kibana) gives you one searchable place for all of it. Here is how the pieces fit together and how to set them up."
tags: ["kubernetes", "logging", "observability", "elasticsearch", "fluent-bit", "fluentd", "kibana", "efk"]
categories: ["Programming"]
---

Every application writes logs, and for a while `kubectl logs` is all you need. Then a Pod gets evicted and its logs go with it. Or a node dies. Or a request fails somewhere across six microservices and you're running `kubectl logs` in six terminals, trying to line up timestamps by eye.

Centralized logging fixes this. An agent on every node collects all the logs, adds context like the Pod name and namespace, and ships them to a central store where you can search everything at once and keep it after the Pod is long gone. The **EFK stack** is one of the most common ways to build that:

- **E**lasticsearch stores and indexes the logs.
- **F**luent Bit (or its older sibling **F**luentd) collects, enriches and forwards them.
- **K**ibana is the web UI for searching, filtering and visualizing them.

This post explains how logging works in Kubernetes, what each component does, and then builds a working pipeline step by step. Everything is based on Kubernetes v1.37, Elasticsearch and Kibana 9.5, and Fluent Bit 5.1, the current releases at the time of writing[^6][^9].

## EFK, ELK, what's the difference?

You'll also see **ELK**: Elasticsearch, **L**ogstash, Kibana. Logstash is Elastic's own data processing pipeline. It's powerful but runs on the JVM and is heavy, so it was never a good fit for running one copy on every Kubernetes node. The Kubernetes community settled on Fluentd, and later Fluent Bit, as the collector instead, and "EFK" stuck.

The two letters aren't exclusive. Plenty of setups run Fluent Bit on the nodes and send everything to a central Logstash or Fluentd for heavy processing. The storage and UI stay the same.

## How logs work in Kubernetes

Before you can collect logs, you need to know where they are.

### Write to stdout and stderr

The Kubernetes convention, inherited from the Twelve-Factor App, is that a container writes its logs to **standard output and standard error**, not to files. The application doesn't decide where logs go. It just writes them, and the platform takes care of the rest.

### Where the logs end up

The container runtime (containerd or CRI-O) captures each container's stdout and stderr and writes them to files on the node[^1]:

```text
/var/log/pods/<namespace>_<pod-name>_<pod-uid>/<container-name>/0.log
```

For convenience there's also a flat directory of symlinks, with the Pod, namespace and container encoded in the file name. That's the directory log collectors usually read:

```text
/var/log/containers/<pod-name>_<namespace>_<container-name>-<container-id>.log
```

Each line is written in the CRI log format: a timestamp, the stream, a flag, and the original message.

```text
2026-11-20T10:15:30.123456789Z stdout F {"level":"info","msg":"order created","order_id":4711}
2026-11-20T10:15:30.124001234Z stderr F panic: runtime error: index out of range
```

The `F` means "full line". Very long lines are split into several entries marked `P` (partial), which a collector has to stitch back together.

`kubectl logs` simply asks the kubelet to read these files for you.

### Rotation, and why logs disappear

The kubelet rotates these files so they can't fill the disk. By default, a container keeps at most 5 files of up to 10 MiB each, controlled by `containerLogMaxSize` and `containerLogMaxFiles` in the kubelet configuration[^1]. For a chatty service that's minutes of history, not days.

Worse, when a Pod is deleted or evicted, its log files are deleted too. `kubectl logs --previous` shows the last crashed container, but nothing older. And if the node itself dies, everything on it is gone.

Kubernetes deliberately doesn't solve this. It provides no storage for logs; it leaves cluster-level logging to you[^1].

### Three ways to collect logs

The Kubernetes docs describe three patterns for cluster-level logging[^1]:

1. **A node agent.** A DaemonSet runs one collector on every node, reads the files under `/var/log/containers`, and ships them. Applications don't need to change at all. This is by far the most common pattern, and it's what EFK uses.
2. **A sidecar.** For applications that can only log to files, a sidecar container tails the file and either writes it to its own stdout (so the node agent picks it up) or ships it directly. (I covered sidecars in my post on multi-container Pods.)
3. **Push from the application.** The application sends logs straight to the backend over the network. This couples your code to your logging backend and loses logs whenever the backend is slow or down, so use it only if you have a good reason.

## The components

### Fluent Bit and Fluentd: the collectors

Fluentd and Fluent Bit come from the same project family, and both are graduated CNCF projects. They solve the same problem with different trade-offs:

| | Fluent Bit | Fluentd |
| --- | --- | --- |
| Written in | C | Ruby (with C parts) |
| Memory | A few MB | Tens of MB, often more |
| Plugins | About 100 built in | Over 1,000, mostly community gems |
| Typical role | Agent on every node | Central aggregator |
| Configuration | YAML (classic `.conf` still works) | Its own tag-based format |

For a new setup, start with **Fluent Bit alone** on every node, sending straight to Elasticsearch. It reads the container files, parses them, adds Kubernetes metadata, and handles retries and buffering. Its small footprint matters when there's a copy on every node.

Add **Fluentd as an aggregator** in the middle only when you need it: complex routing to many destinations, a plugin that only exists for Fluentd, or a central place to buffer and throttle before Elasticsearch. Fluent Bit forwards to it with the `forward` output. One gotcha if you do this: Fluent Bit v5.0 changed the default behaviour of the `forward` output, and the upgrade notes tell you to set `retain_metadata_in_forward_mode: false` explicitly when sending to a Fluentd aggregator[^5].

Fluent Bit processes every record through a pipeline:

```text
input → parser → filter → buffer → router → output
```

- **Inputs** read data: `tail` follows files, `systemd` reads the journal, and there are many more.
- **Parsers** turn raw text into structured records.
- **Filters** modify records: add Kubernetes metadata, drop noisy lines, redact fields.
- **Outputs** send them somewhere: Elasticsearch, OpenSearch, Loki, S3, Kafka.

Every record has a **tag**, and `match` patterns decide which filters and outputs see which records.

Since v3.2, Fluent Bit's YAML configuration format supports everything the classic format does plus features the classic format lacks, such as processors[^2]. Use YAML for anything new; all examples below use it.

### Elasticsearch: the store

Elasticsearch is a distributed search engine built on Apache Lucene. A few terms you need:

- A **document** is a JSON object. Each log line becomes one document.
- An **index** is a collection of documents, roughly like a table. For logs, you typically create one index per day, like `k8s-logs-2026.11.20`, so old days can be deleted as a whole.
- A **mapping** defines the type of each field: `keyword` for exact values like a namespace, `text` for full-text search, `date`, `long` and so on. Elasticsearch guesses mappings from the first document it sees with a field (dynamic mapping), which is convenient and also the source of a classic logging problem, covered below.
- An index is split into **shards** spread across nodes, and each shard can have **replicas** on other nodes, so you can lose a node without losing data.

Because every field is indexed, searches like "all errors in namespace `shop` in the last hour" return in milliseconds across billions of lines. The price is storage and memory: Elasticsearch is the most resource-hungry part of the stack by far.

A note on licensing, since it confused people for years: in 2021 Elastic moved Elasticsearch and Kibana from Apache 2.0 to the SSPL and its own Elastic License, which are not open source licenses. In 2024 it added the OSI-approved **AGPL** as a third option, so they're open source again[^8]. The ECK operator used below is under the Elastic License v2, which is free to use; the free "Basic" tier covers everything in this post.

### Kibana: the UI

Kibana is a web application that talks to Elasticsearch. For logs, the parts you'll use most are:

- **Discover**, for searching and browsing log lines, with a histogram of matches over time.
- **Dashboards**, for charts like "errors per service over time".
- **Dev Tools**, a console for sending requests straight to the Elasticsearch API.

You query with **KQL** (Kibana Query Language), a simple filter syntax, or **ES|QL**, a piped query language that can also aggregate.

## Building the pipeline

Here's the plan: run Elasticsearch and Kibana with the ECK operator, then deploy Fluent Bit as a DaemonSet with Helm. Everything goes in a `logging` namespace.

```text
┌──────────── every node ────────────┐
│ containers → /var/log/containers/  │
│                  │                 │
│            Fluent Bit (DaemonSet)  │
└──────────────────┼─────────────────┘
                   │ HTTPS
                   ▼
           Elasticsearch (3 nodes)
                   │
                   ▼
                Kibana  ←── you
```

### Step 1: Install the ECK operator

**Elastic Cloud on Kubernetes (ECK)** is Elastic's official operator. It extends Kubernetes to set up and manage Elasticsearch, Kibana and other Elastic applications[^7]. You describe a cluster in a custom resource; the operator creates the StatefulSets, Services, TLS certificates and the initial admin user, and handles rolling upgrades and scaling safely.

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
helm install elastic-operator elastic/eck-operator \
  --namespace elastic-system --create-namespace
```

(See my post on Helm if Helm is new to you.)

### Step 2: Create an Elasticsearch cluster

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: logs
  namespace: logging
spec:
  version: 9.5.5
  nodeSets:
    - name: default
      count: 3
      config:
        # Avoids having to raise vm.max_map_count on the nodes.
        # Fine for getting started; for production, set the sysctl instead.
        node.store.allow_mmap: false
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources:
                requests:
                  memory: 4Gi
                  cpu: "1"
                limits:
                  memory: 4Gi
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 50Gi
```

```bash
kubectl create namespace logging
kubectl apply -f elasticsearch.yaml
kubectl get elasticsearch -n logging -w
```

Wait until `HEALTH` shows `green`. Three nodes means every shard can have a replica on a different node. Elasticsearch sizes its JVM heap automatically from the container's memory limit, so setting requests equal to limits is all you need.

ECK creates a few things you'll need in the next steps:

- A Service, `logs-es-http`, on port 9200, with TLS enabled.
- A Secret, `logs-es-elastic-user`, holding the password of the built-in `elastic` superuser.
- A Secret, `logs-es-http-certs-public`, with the CA certificate clients need to verify TLS.

### Step 3: Add Kibana

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: logs
  namespace: logging
spec:
  version: 9.5.5
  count: 1
  elasticsearchRef:
    name: logs
```

`elasticsearchRef` is all the wiring Kibana needs: ECK configures the connection, credentials and TLS for you. To open it:

```bash
kubectl apply -f kibana.yaml

# Get the password for the "elastic" user
kubectl get secret logs-es-elastic-user -n logging \
  -o jsonpath='{.data.elastic}' | base64 -d; echo

kubectl port-forward -n logging svc/logs-kb-http 5601
```

Browse to `https://localhost:5601` (the certificate is self-signed) and log in as `elastic`.

### Step 4: Deploy Fluent Bit

Fluent Bit's official Helm chart runs it as a DaemonSet with the right host mounts and RBAC (the Kubernetes filter needs to read Pod metadata from the API server)[^12]. The chart's built-in config values use the classic format, so the values below add a YAML config file and point Fluent Bit at it. Check this against the `values.yaml` of your chart version, as defaults change between releases.

```yaml
# fluent-bit-values.yaml
args:
  - --workdir=/fluent-bit/etc
  - --config=/fluent-bit/etc/conf/fluent-bit.yaml

env:
  - name: ELASTIC_PASSWORD
    valueFrom:
      secretKeyRef:
        name: logs-es-elastic-user
        key: elastic

extraVolumes:
  - name: es-ca
    secret:
      secretName: logs-es-http-certs-public
extraVolumeMounts:
  - name: es-ca
    mountPath: /certs
    readOnly: true

config:
  extraFiles:
    fluent-bit.yaml: |
      service:
        flush: 1
        log_level: info
        http_server: on
        http_listen: 0.0.0.0
        http_port: 2020
        health_check: on

      pipeline:
        inputs:
          - name: tail
            tag: kube.*
            path: /var/log/containers/*.log
            exclude_path: /var/log/containers/fluent-bit*
            multiline.parser: cri, docker
            db: /var/fluent-bit/state/flb_kube.db
            mem_buf_limit: 50MB
            skip_long_lines: on
            refresh_interval: 10

        filters:
          - name: kubernetes
            match: kube.*
            merge_log: on
            keep_log: off
            k8s-logging.parser: on
            k8s-logging.exclude: on
            labels: on
            annotations: off

        outputs:
          - name: es
            match: kube.*
            host: logs-es-http.logging.svc
            port: 9200
            http_user: elastic
            http_passwd: ${ELASTIC_PASSWORD}
            tls: on
            tls.verify: on
            tls.ca_file: /certs/ca.crt
            logstash_format: on
            logstash_prefix: k8s-logs
            suppress_type_name: on
            replace_dots: on
            trace_error: on
            retry_limit: false
```

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit -n logging -f fluent-bit-values.yaml
```

What each section does:

**The `tail` input** follows every file in `/var/log/containers`. The tag `kube.*` expands to include the file name, which is how the Kubernetes filter later learns the Pod, namespace and container. The `cri, docker` multiline parsers strip the CRI prefix and reassemble partial `P` lines into one record. The `db` file records how far Fluent Bit has read in each file, so a restart neither loses nor duplicates logs. `mem_buf_limit` caps how much it buffers in memory if Elasticsearch can't keep up; when the limit is hit, it pauses reading instead of running out of memory. Excluding its own logs avoids a feedback loop if Fluent Bit ever starts logging its own output errors.

**The `kubernetes` filter** looks up each Pod (from a local cache, going to the API server only on a miss) and adds a `kubernetes` object with the namespace, Pod name, container name, image, node and labels[^3]. Two settings matter a lot:

- `merge_log: on` checks whether the log message is JSON. If it is, its fields are lifted into the record as real, searchable fields. More on this in the next section.
- `k8s-logging.parser` and `k8s-logging.exclude` let teams control parsing from their own Pods with the annotations `fluentbit.io/parser` and `fluentbit.io/exclude: "true"`, without touching the central config.

**The `es` output** writes to Elasticsearch over HTTPS, verifying the certificate against ECK's CA[^4]:

- `logstash_format: on` with `logstash_prefix: k8s-logs` writes to one index per day, such as `k8s-logs-2026.11.20`, and adds an `@timestamp` field.
- `suppress_type_name: on` is required for Elasticsearch 8 and later, which removed mapping types.
- `replace_dots: on` turns dots in field names into underscores. Without it, Kubernetes labels like `app` and `app.kubernetes.io/name` cause mapping conflicts, because Elasticsearch reads dots as object nesting. `app` can't be both a string and an object, so Elasticsearch rejects the document.
- `trace_error: on` logs Elasticsearch's response when a document is rejected. Without it, you only see that something failed, not why.
- `retry_limit: false` retries forever instead of dropping data after a few failures. Pair it with a sensible `mem_buf_limit`, or filesystem buffering for production.

Check that it's working:

```bash
kubectl logs -n logging daemonset/fluent-bit | tail
kubectl port-forward -n logging svc/logs-es-http 9200 &
curl -sk -u "elastic:$PASSWORD" "https://localhost:9200/_cat/indices/k8s-logs-*?v"
```

You should see today's index with a growing `docs.count`.

### Step 5: Search in Kibana

In Kibana, go to **Discover** and create a **data view** for the index pattern `k8s-logs-*`, with `@timestamp` as the time field. You'll see every log line from every container, newest first.

A few KQL queries to try:

```text
kubernetes.namespace_name : "shop"
kubernetes.namespace_name : "shop" and level : "error"
kubernetes.labels.app : "checkout" and not msg : *healthz*
kubernetes.pod_name : checkout-* and order_id : 4711
```

Add fields like `kubernetes.pod_name` and `msg` as columns, and save useful searches. For anything that needs counting, switch to ES|QL:

```text
FROM k8s-logs-*
| WHERE kubernetes.namespace_name == "shop" AND level == "error"
| STATS errors = COUNT(*) BY kubernetes.labels.app
| SORT errors DESC
```

That's a "which service is throwing the most errors" table in four lines, and a good first panel for a dashboard.

### Step 6: Delete old logs automatically

Logs pile up fast. Without a retention policy, Elasticsearch's disks fill up, and once a disk crosses its high watermark Elasticsearch stops writing to it. **Index Lifecycle Management (ILM)** deletes old indices on a schedule[^10]. In Kibana's **Dev Tools**:

```text
PUT _ilm/policy/k8s-logs
{
  "policy": {
    "phases": {
      "hot":    { "actions": {} },
      "delete": { "min_age": "14d", "actions": { "delete": {} } }
    }
  }
}

PUT _index_template/k8s-logs
{
  "index_patterns": ["k8s-logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "k8s-logs",
      "number_of_shards": 1,
      "number_of_replicas": 1
    }
  }
}
```

Every new `k8s-logs-*` index now picks up the policy and is deleted 14 days after it was created. One shard per daily index is plenty for small and medium clusters; too many small shards is one of the most common ways to make Elasticsearch slow.

## Structured logging: help your future self

Compare these two log lines:

```text
2026-11-20 10:15:30 INFO Order 4711 created for customer 93 in 182ms
```

```json
{"ts":"2026-11-20T10:15:30Z","level":"info","msg":"order created","order_id":4711,"customer_id":93,"duration_ms":182}
```

To Elasticsearch, the first is one blob of text. You can full-text search it, but you can't ask "show me orders that took over 500 ms" or "count errors per customer". The second, thanks to `merge_log`, becomes separate fields: `duration_ms` is a number you can filter and chart, and `order_id` is an exact value you can match.

So, if you control the application:

- **Log JSON, one object per line.** Most languages have a structured logger built in or a library away: `log/slog` in Go, `structlog` or `python-json-logger` in Python, Logback with a JSON encoder in Java.
- **Use the same field names everywhere.** If one service writes `level` and another `severity`, every query needs both. Agree on a small set (`level`, `msg`, `trace_id`, `user_id`) or adopt the Elastic Common Schema or OpenTelemetry's semantic conventions.
- **Keep field types stable.** If `order_id` is a number in one service and a string in another, whichever arrives first in a day's index sets the mapping, and the other service's documents are rejected.
- **Include a trace or request ID.** It's the one field that lets you follow a single request across every service.
- **Never log secrets or personal data you don't need.** Once a password is in Elasticsearch, it's in every snapshot and every query result until the index is deleted.

## Common pitfalls

**Multi-line stack traces.** A Java or Python stack trace is many lines, and by default each line becomes a separate document, which is nearly useless. Fluent Bit's `multiline` filter has built-in parsers for `java`, `python` and `go` that merge them back into one record. Better still, have the application log exceptions as a single JSON field.

**Mapping explosions and conflicts.** Every unique field name becomes part of the index mapping, and Elasticsearch caps an index at 1,000 fields by default. One service that logs a map with user IDs as keys can hit the limit in minutes, after which new fields are rejected. Keep dynamic keys out of field names, and consider turning label collection off or limiting it if you have many unique labels.

**Backpressure and lost logs.** If Elasticsearch is down or slow, Fluent Bit buffers. When the memory buffer is full, it stops reading files, and if the kubelet rotates a file away before Fluent Bit gets back to it, those lines are lost. For production, enable filesystem buffering (`storage.type: filesystem`) so Fluent Bit can buffer to disk, and watch its metrics on port 2020.

**Running as superuser.** The example uses the `elastic` user to keep things short. In production, create a dedicated user or API key whose role can only create documents in `k8s-logs-*`, and give Fluent Bit that. If a node is compromised, the attacker gets write access to logs, not your whole cluster. (See my post on Kubernetes Secrets for how to keep that credential safe.)

**Cost.** Logs are often the largest data volume in a cluster. Drop what you never read (health checks, debug logs in production) at the collector with a `grep` filter, rather than paying to store and index it.

## Alternatives worth knowing

EFK isn't the only option, and it's worth knowing what else exists:

- **OpenSearch** is the AWS-led fork of Elasticsearch from 2021, under Apache 2.0, with OpenSearch Dashboards instead of Kibana. Fluent Bit has an `opensearch` output, so the pipeline above works with minor changes.
- **Grafana Loki** indexes only labels (namespace, app, Pod), not the log content. It's much cheaper to run but slower for full-text search across large volumes. Fluent Bit or Grafana Alloy ship the logs, and Grafana is the UI.
- **The OpenTelemetry Collector** can collect logs alongside metrics and traces, and is becoming the default choice for teams standardizing on OpenTelemetry.
- **Vector** is a fast collector and router written in Rust, an alternative to Fluent Bit or Fluentd.

EFK's strength is search: when you need to find a needle across terabytes of logs with arbitrary queries, Elasticsearch is hard to beat. If you mostly filter by service and time and care about cost, look at Loki.

## Bottom line

- **Kubernetes doesn't keep your logs.** They live in rotated files on the node and disappear with the Pod or the node.
- **Applications write to stdout and stderr**; a node agent DaemonSet collects everything without touching application code.
- **Fluent Bit collects and enriches**, adding Kubernetes metadata to every line. Add Fluentd as an aggregator only when you need its routing or plugins.
- **Elasticsearch stores and indexes**, and **Kibana searches and visualizes**. The ECK operator makes running both on Kubernetes manageable.
- **Set retention from day one** with ILM, or your disks will fill up.
- **Log structured JSON** with consistent field names, and your logs become data you can query instead of text you grep.

## References

[^1]: [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/), Kubernetes documentation: where container logs are stored, log rotation settings, and cluster-level logging patterns.
[^2]: [YAML configuration files](https://docs.fluentbit.io/manual/5.0/administration/configuring-fluent-bit/yaml), Fluent Bit documentation: sections and features of the YAML format.
[^3]: [Kubernetes filter](https://docs.fluentbit.io/manual/5.0/data-pipeline/filters/kubernetes), Fluent Bit documentation: metadata enrichment, `merge_log`, and Pod annotations.
[^4]: [Elasticsearch output](https://docs.fluentbit.io/manual/data-pipeline/outputs/elasticsearch), Fluent Bit documentation: connection, TLS, index naming and compatibility options.
[^5]: [Upgrade notes](https://docs.fluentbit.io/manual/5.0/installation/upgrade-notes), Fluent Bit documentation: v5.0 change to `forward` output metadata.
[^6]: [Fluent Bit release announcements](https://fluentbit.io/announcements/), Fluent Bit: release history of the v5.x series.
[^7]: [Elastic Cloud on Kubernetes](https://elastic.co/guide/en/cloud-on-k8s/current/index.html), Elastic documentation: the ECK operator and the applications it manages.
[^8]: [Elasticsearch Is Open Source. Again!](https://www.elastic.co/blog/elasticsearch-is-open-source-again), Elastic blog, August 2024: addition of the AGPL license option.
[^9]: [Elastic 9.5](https://www.elastic.co/blog/whats-new-elastic-9-5-0), Elastic blog: release announcement for the 9.5 series.
[^10]: [Index lifecycle management](https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management), Elastic documentation: ILM policies and phases.
[^11]: [Fluentd](https://www.fluentd.org/), project website.
[^12]: [Fluent Helm charts](https://github.com/fluent/helm-charts), GitHub: Helm charts for Fluent Bit and Fluentd.
