---
title: "Apache Artemis: An Introduction, and How to Run It on Kubernetes"
date: 2026-11-19
draft: true
description: "What Apache Artemis is now that it's an independent Apache project, how its addresses, queues, journal and HA work, and two ways to run it on Kubernetes: a plain StatefulSet with the official image, and the ArkMQ broker operator."
summary: "Apache Artemis, formerly ActiveMQ Artemis, became its own top-level Apache project in late 2025. Here is a quick tour of how the broker works, followed by two ways to deploy it on Kubernetes: a hand-written StatefulSet for a single broker, and the ArkMQ operator for clusters."
tags: ["kubernetes", "apache artemis", "activemq artemis", "messaging", "message broker", "operator", "statefulset"]
categories: ["Programming"]
---

In my earlier post on message brokers I compared RabbitMQ and ActiveMQ Artemis at a high level. This post goes deeper on Artemis: what has changed with the project, how the broker is put together, and how to actually run it on Kubernetes.

There are two realistic ways to deploy it:

1. **A plain StatefulSet** with the official container image. Simple, transparent, good for a single broker.
2. **The ArkMQ broker operator**, the community operator for Artemis. Better for clusters, scale-down with message migration, and managing addresses as Kubernetes resources.

We'll do both.

## First, the name change

If you last looked at Artemis a year ago, it was "Apache ActiveMQ Artemis", a sub-project of ActiveMQ. That's no longer the case.

In November 2025 the ActiveMQ PMC decided to split into two separate top-level projects, **Apache ActiveMQ** and **Apache Artemis**, each with its own website, at activemq.apache.org and artemis.apache.org.[^1] The ASF formally announced Artemis as a top-level project on December 3, 2025.[^2]

What that means in practice:

- **Version numbers continue.** There was no reset to 1.0. Artemis 2.50.0, released in January 2026, was the first release under the new name.[^3] The current release line is 2.5x (2.57.0 came out in September 2026).[^3]
- **The Maven groupId changed** from `org.apache.activemq` to `org.apache.artemis`, so update your dependency declarations when you upgrade.[^4]
- **The container image moved** to `apache/artemis` on Docker Hub.[^5] Older tutorials use `apache/activemq-artemis`.
- **Java packages didn't change.** Class names still start with `org.apache.activemq.artemis`, so your `broker.xml`, JAAS config and code keep working.
- Artemis requires **Java 17 or later**.[^4]

"ActiveMQ" on its own now means the original broker, previously called ActiveMQ Classic. It's still maintained and still releasing.

## What Artemis is

Artemis is a multi-protocol message broker written in Java. It started life as JBoss HornetQ, which Red Hat donated to Apache, and it's the upstream for Red Hat AMQ Broker.

The broker supports AMQP 1.0, MQTT 3.1.1, MQTT 5 and STOMP,[^6] along with its own **Core** protocol and **OpenWire**, the ActiveMQ Classic wire protocol. Java applications typically use the JMS / Jakarta Messaging API on top of the Core client. Everything else uses AMQP 1.0, MQTT or STOMP clients. All of these can share the same queues: a message sent over MQTT can be consumed over AMQP.

### Addresses, routing types and queues

The Artemis data model is small:

- An **address** is what producers send to.
- One or more **queues** are bound to an address.
- The **routing type** decides how messages are distributed:
  - **anycast**: each message goes to one queue on the address. This is point-to-point, a JMS *queue*.
  - **multicast**: each message is copied to every queue on the address. This is pub/sub, a JMS *topic*. Each subscriber gets its own queue.

```text
 address "orders"   routing: ANYCAST         address "events"   routing: MULTICAST
 ┌─────────────────────────────┐             ┌─────────────────────────────┐
 │ queue "orders"              │             │ queue "billing"   ◄── copy  │
 │  consumers compete for msgs │             │ queue "audit"     ◄── copy  │
 └─────────────────────────────┘             └─────────────────────────────┘
```

By default Artemis **auto-creates** addresses and queues when a client first uses them. That's convenient in development and something you usually want to restrict in production.

### Acceptors

An **acceptor** is a network listener. It defines a port and which protocols it accepts. The default configuration has one acceptor per protocol (5672 for AMQP, 1883 for MQTT, 61613 for STOMP), plus an "all protocols" acceptor on **61616**. The web console runs on **8161**.

On Kubernetes, acceptors map directly onto container ports and Services.

### The journal and paging

Artemis persists messages to its own append-only **journal**, not to a database. On Linux it can use asynchronous I/O (libaio) to write straight to disk, which is a big part of its throughput. The journal lives in the broker instance's `data/` directory, which matters for Kubernetes because that directory **must** be on a persistent volume.

When an address holds more messages than fit in memory, Artemis **pages** them to disk and reads them back as consumers catch up. By default the broker starts paging once all addresses together use half of the JVM's maximum heap (`global-max-size`). That's why container memory limits and JVM heap settings need to agree.

### High availability

Artemis has three main HA options:

- **Shared store:** a primary and a backup point at the same storage. Only one is active at a time.
- **Replication:** the primary streams its journal to a backup over the network.
- **Mirroring** (broker connections over AMQP): asynchronous copy of messages to another broker, typically in another site, for disaster recovery.

On Kubernetes, the simplest form of HA is often just the platform itself: a single broker in a StatefulSet with a persistent volume. If the pod dies, Kubernetes restarts it, possibly on another node, and it reattaches the same journal. You'll have a short outage (tens of seconds), but no message loss. For many internal workloads that's enough. If it isn't, use the operator.

### Configuration: broker.xml and broker properties

The main config file is `etc/broker.xml`. Artemis also supports **broker properties**: simple `key=value` lines that are applied after the XML is parsed, and can change any attribute of the internal configuration.[^7] For example:

```properties
addressSettings.#.maxDeliveryAttempts=5
addressSettings.#.deadLetterAddress=DLQ
addressConfigurations.orders.routingTypes=ANYCAST
addressConfigurations.orders.queueConfigs.orders.routingType=ANYCAST
```

Broker properties fit Kubernetes nicely, because you can keep them in a ConfigMap or put them straight into an operator custom resource instead of templating a large XML file. One catch: a property whose name doesn't match anything is silently ignored.[^7] Check the broker log after changes.

## Option 1: a plain StatefulSet

This is the "no magic" approach: the official image, one broker, one persistent volume.

### How the official image works

The `apache/artemis` image creates a broker instance at `/var/lib/artemis-instance` on first start, then runs it. It's controlled by a few environment variables:[^5]

| Variable | Default | Purpose |
| --- | --- | --- |
| `ARTEMIS_USER` | `artemis` | Admin username |
| `ARTEMIS_PASSWORD` | `artemis` | Admin password |
| `ANONYMOUS_LOGIN` | `false` | Allow unauthenticated clients |
| `EXTRA_ARGS` | `--http-host 0.0.0.0 --relax-jolokia` | Extra arguments for `artemis create` (replaces the default) |

To customise configuration, mount files such as `broker.xml` or `artemis.profile` into `/var/lib/artemis-instance/etc-override`. They're copied into `etc/` after the instance is created.[^5]

Because the instance, including the journal, lives under `/var/lib/artemis-instance`, that's where the persistent volume goes.

### The manifests

A namespace and a Secret for the admin credentials:

```bash
kubectl create namespace messaging

kubectl -n messaging create secret generic artemis-admin \
  --from-literal=username=admin \
  --from-literal=password="$(openssl rand -base64 24)"
```

A ConfigMap with broker properties, mounted as an override. The image picks up a `broker.properties` file from `etc/`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: artemis-config
  namespace: messaging
data:
  broker.properties: |
    # Dead-lettering and redelivery for every address
    addressSettings.#.deadLetterAddress=DLQ
    addressSettings.#.maxDeliveryAttempts=5
    addressSettings.#.redeliveryDelay=1000
    addressSettings.#.redeliveryMultiplier=2
    # Pre-create the queues we use
    addressConfigurations.orders.routingTypes=ANYCAST
    addressConfigurations.orders.queueConfigs.orders.routingType=ANYCAST
    addressConfigurations.DLQ.routingTypes=ANYCAST
    addressConfigurations.DLQ.queueConfigs.DLQ.routingType=ANYCAST
```

The StatefulSet and Services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: artemis-hdls
  namespace: messaging
spec:
  clusterIP: None              # headless, gives the pod a stable DNS name
  selector:
    app: artemis
  ports:
    - name: all
      port: 61616
---
apiVersion: v1
kind: Service
metadata:
  name: artemis
  namespace: messaging
spec:
  selector:
    app: artemis
  ports:
    - name: all                # Core, AMQP, OpenWire, MQTT, STOMP
      port: 61616
    - name: amqp
      port: 5672
    - name: console
      port: 8161
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: artemis
  namespace: messaging
spec:
  serviceName: artemis-hdls
  replicas: 1                  # one broker; replicas here would be independent brokers
  selector:
    matchLabels:
      app: artemis
  template:
    metadata:
      labels:
        app: artemis
    spec:
      securityContext:
        fsGroup: 1001          # the image runs as the non-root "artemis" user
      terminationGracePeriodSeconds: 60
      containers:
        - name: artemis
          image: apache/artemis:2.57.0   # pin a version; check Docker Hub for the latest
          env:
            - name: ARTEMIS_USER
              valueFrom:
                secretKeyRef: { name: artemis-admin, key: username }
            - name: ARTEMIS_PASSWORD
              valueFrom:
                secretKeyRef: { name: artemis-admin, key: password }
          ports:
            - { name: all, containerPort: 61616 }
            - { name: amqp, containerPort: 5672 }
            - { name: console, containerPort: 8161 }
          readinessProbe:
            tcpSocket: { port: 61616 }
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            tcpSocket: { port: 61616 }
            initialDelaySeconds: 60
            periodSeconds: 20
          resources:
            requests: { cpu: 500m, memory: 2Gi }
            limits:   { memory: 2Gi }
          volumeMounts:
            - name: data
              mountPath: /var/lib/artemis-instance
            - name: config
              mountPath: /var/lib/artemis-instance/etc-override
      volumes:
        - name: config
          configMap:
            name: artemis-config
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

A few things worth calling out:

- **`replicas: 1` is deliberate.** Scaling this StatefulSet to 3 gives you three *unrelated* brokers, each with its own messages, and clients randomly load-balanced between them. Clustering Artemis needs cluster connections and discovery configured. That's what the operator does for you.
- **Use block storage** (a normal `ReadWriteOnce` volume from your cloud's disk CSI driver). Avoid putting the journal on NFS or other network file systems unless you know what you're doing; journal performance and file locking both suffer.
- **Size memory deliberately.** The JVM heap is set in `artemis.profile`. Make sure it's comfortably below the container memory limit, and remember that paging starts at half the heap by default. If you override `artemis.profile`, ship it through the same `etc-override` ConfigMap.
- **The grace period matters.** A clean shutdown lets the broker close the journal properly and makes restart faster.

Apply it and check:

```bash
kubectl apply -f artemis.yaml
kubectl -n messaging rollout status statefulset/artemis
kubectl -n messaging logs artemis-0 | grep -i "server is now active"
```

### Try it

The broker image includes the `artemis` CLI. Send and receive some test messages from inside the pod:

```bash
PASS=$(kubectl -n messaging get secret artemis-admin -o jsonpath='{.data.password}' | base64 -d)

kubectl -n messaging exec artemis-0 -- /var/lib/artemis-instance/bin/artemis producer \
  --url tcp://artemis:61616 --user admin --password "$PASS" \
  --destination queue://orders --message-count 100

kubectl -n messaging exec artemis-0 -- /var/lib/artemis-instance/bin/artemis queue stat \
  --url tcp://artemis:61616 --user admin --password "$PASS"
```

And open the console:

```bash
kubectl -n messaging port-forward svc/artemis 8161:8161
# then browse to http://localhost:8161/console
```

Applications inside the cluster connect to `tcp://artemis.messaging.svc.cluster.local:61616` (Core/JMS) or `amqp://artemis.messaging.svc.cluster.local:5672`.

## Option 2: the ArkMQ broker operator

For anything beyond a single broker, use an operator. The community operator for Artemis has a slightly confusing history, so a quick orientation:

- It used to live at `github.com/artemiscloud/activemq-artemis-operator`, and many tutorials still point there.
- It now lives in the **ArkMQ** organisation (`arkmq-org`), which describes itself as "Apache Artemis in the cloud", and has been renamed the **ArkMQ Broker Operator**.[^8][^9]
- Red Hat's AMQ Broker operator is built from the same code.
- The main custom resource is moving from **`ActiveMQArtemis`** (`broker.amq.io/v1beta1`) to **`Broker`** (`broker.arkmq.org/v1beta2`).[^10] Addresses are still managed with **`ActiveMQArtemisAddress`** in the old `broker.amq.io` group.[^10]

Because the operator is mid-rename, always check what your installed version actually serves:

```bash
kubectl api-resources | grep -Ei 'arkmq|amq.io'
kubectl explain broker.spec.deploymentPlan
```

### Install the operator

The operator is published as an OCI Helm chart, so there's no `helm repo add`. Installing it needs cluster-admin; after that, regular users can create brokers.[^10]

```bash
helm install arkmq-operator \
  oci://quay.io/arkmq-org/helm-charts/arkmq-org-broker-operator \
  --namespace arkmq-operator --create-namespace

kubectl -n arkmq-operator get pods
```

If you prefer plain manifests, every release also ships a single YAML you can `kubectl apply`, and there's an OperatorHub bundle for OLM.[^10] By default the operator watches its own namespace; there's a cluster-wide install mode if brokers live elsewhere.

### Deploy a clustered broker

```yaml
apiVersion: broker.arkmq.org/v1beta2
kind: Broker
metadata:
  name: orders
  namespace: arkmq-operator
spec:
  deploymentPlan:
    size: 2                       # two broker pods, clustered
    persistenceEnabled: true      # PVC per pod for the journal
    messageMigration: true        # drain messages on scale-down
    requireLogin: true
    storage:
      size: 10Gi
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { memory: 2Gi }
  console:
    expose: true
  acceptors:
    - name: amqp
      protocols: amqp
      port: 5672
    - name: all
      protocols: all
      port: 61617
  brokerProperties:
    - addressSettings.#.deadLetterAddress=DLQ
    - addressSettings.#.maxDeliveryAttempts=5
    - addressSettings.#.redeliveryDelay=1000
    - addressConfigurations.DLQ.routingTypes=ANYCAST
    - addressConfigurations.DLQ.queueConfigs.DLQ.routingType=ANYCAST
```

When `size` is greater than one, the brokers form a cluster and use the default `ON_DEMAND` message redistribution policy, so messages move to whichever broker has consumers.[^10] If you leave `spec.image` empty, the operator uses its default broker image.[^10]

Behind the scenes the operator creates:

- a **StatefulSet** named `<name>-ss` (pods `orders-ss-0`, `orders-ss-1`),
- a **headless Service** `<name>-hdls-svc` for client connections,
- a Service per pod for each acceptor and for the console,
- a **Secret** with generated admin credentials, unless you supply your own,
- the cluster connection and discovery configuration you would otherwise write by hand.

Check it:

```bash
kubectl -n arkmq-operator get broker orders
kubectl -n arkmq-operator get pods -l application=orders-app
```

Clients inside the cluster connect to `orders-hdls-svc.arkmq-operator.svc.cluster.local` on the acceptor port.

### Scale-down without losing messages

This is the operator's killer feature. If you shrink a plain StatefulSet from 2 to 1, the messages sitting in `orders-ss-1`'s journal are stranded on its volume. With `messageMigration: true` and persistence on, the operator moves those messages to a remaining broker before the pod goes away. The current recommended mechanism also needs `HAPolicyConfiguration.scaleDownConfiguration.enabled=false` in `brokerProperties` as the trigger for operator-managed scale-down, and broker version 2.43.0 or later; an older drainer-pod approach is deprecated.[^10]

### Addresses as Kubernetes resources

You can define queues with broker properties, as above, or declaratively:

```yaml
apiVersion: broker.amq.io/v1beta1
kind: ActiveMQArtemisAddress
metadata:
  name: orders-queue
  namespace: arkmq-operator
spec:
  addressName: orders
  queueName: orders
  routingType: anycast
  removeFromBrokerOnDelete: true
```

With `removeFromBrokerOnDelete: true`, deleting the resource deletes the address and queue on the broker. With `false`, they stay.[^10] This works well with GitOps: queues live in Git next to the application that uses them.

### Security and custom config

For anything beyond the generated admin user, the operator uses **`extraMounts`**: you put JAAS `login.config`, user and role files, or a logging config into Secrets or ConfigMaps and reference them from the `Broker` spec. A Secret or ConfigMap whose name ends in `-logging-config` with a `logging.properties` key replaces the log4j2 configuration, for example to turn on audit logging.[^10]

## Which option should you pick?

| | Plain StatefulSet | ArkMQ operator |
| --- | --- | --- |
| Moving parts | Just your YAML | Operator + CRDs |
| Brokers | One | One or many, clustered |
| Scale-down | Messages stranded on the PVC | Messages migrated |
| Address management | broker.properties / broker.xml | Broker properties or `ActiveMQArtemisAddress` |
| Upgrades | Change the image tag | Operator handles rolling restarts |
| Good for | Dev, CI, small internal workloads | Production, multiple teams, HA |

My rule of thumb: start with the StatefulSet to learn how the broker behaves, and switch to the operator as soon as you need more than one broker.

## Production checklist

- **Pin versions** of both the broker image and the operator. Artemis only actively maintains the latest minor release,[^11] so plan to upgrade regularly rather than sitting on one version.
- **Persistent block storage** for the journal, with a StorageClass that supports volume expansion.
- **Dead-letter and expiry addresses** configured from day one, with alerts on DLQ depth.
- **Restrict auto-creation** of addresses and queues in production so typos don't silently create new queues.
- **Memory:** JVM heap well under the container limit, and `global-max-size` / paging set deliberately.
- **Metrics:** the image exposes management data through Jolokia on the console port. Enable the metrics plugin and scrape it with Prometheus, then alert on queue depth, consumer count and paging.
- **TLS** on acceptors that leave the cluster, and NetworkPolicies so only the namespaces that need the broker can reach it.
- **PodDisruptionBudget** so node drains don't take down every broker at once.
- **Don't expose the console publicly.** Port-forward or put it behind your SSO.

## Bottom line

- **Apache Artemis** is now its own top-level Apache project. Same broker, new name, new Maven groupId, new image (`apache/artemis`).
- Its model is **addresses + routing types + queues**, with a fast **journal** that needs a persistent volume.
- For a single broker, a **StatefulSet with the official image** is simple and perfectly adequate.
- For clusters and safe scale-down, use the **ArkMQ broker operator** (formerly ArtemisCloud), and check which CRD versions your install serves.

## References

[^1]: [Apache Artemis Project](https://artemis.apache.org/news/artemis-tlp), Apache Artemis news, November 2025.
[^2]: [The Apache Software Foundation Announces New Top-Level Projects](https://news.apache.org/foundation/entry/the-apache-software-foundation-announces-new-top-level-projects-3), ASF, December 3, 2025.
[^3]: [Apache Artemis News](https://artemis.apache.org/news), release announcements for 2.50.0 through 2.57.0.
[^4]: [Artemis 2.56.0 Release Notes](https://artemis.apache.org/components/artemis/download/release-notes-2.56.0): groupId change and Java 17 requirement.
[^5]: [Docker](https://artemis.apache.org/components/artemis/documentation/latest/docker.html), Apache Artemis documentation: official image, environment variables and `etc-override`.
[^6]: [Apache Artemis](https://artemis.apache.org/), project home page.
[^7]: [Configuration Reference](https://artemis.apache.org/components/artemis/documentation/latest/configuration-index.html), Apache Artemis documentation: broker properties.
[^8]: [arkmq-org on GitHub](https://github.com/arkmq-org), ArkMQ organisation.
[^9]: [How to Deploy ActiveMQ on Rancher](https://oneuptime.com/blog/post/2026-03-20-rancher-deploy-activemq/view), OneUptime blog, March 2026: ArkMQ as the current home of the ArtemisCloud operator.
[^10]: [arkmq-org Broker Operator Quick Start](https://github.com/arkmq-org/arkmq-org-broker-operator/blob/main/docs/getting-started/quick-start.md), ArkMQ operator documentation.
[^11]: [Apache Artemis](https://endoflife.date/apache-artemis), endoflife.date: release and support status.
