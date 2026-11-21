---
title: "Message Brokers Explained: RabbitMQ, ActiveMQ Artemis, and How They Differ from Kafka"
date: 2026-11-16
draft: true
description: "What a message broker is, why you would put one between your services, the core concepts (queues, topics, acknowledgements, dead-lettering, protocols), and how RabbitMQ and Apache ActiveMQ Artemis approach the job, compared with Kafka's log-based model."
summary: "Instead of calling each other directly, services can hand messages to a broker that stores, routes and delivers them. Here is what a message broker does, the concepts you need, and how RabbitMQ and ActiveMQ Artemis compare, with Kafka as the counterpoint."
tags: ["messaging", "message broker", "rabbitmq", "activemq artemis", "kafka", "amqp", "distributed systems"]
categories: ["Programming"]
---

Sooner or later, every system with more than one service runs into the same question: how should these things talk to each other?

The obvious answer is a direct call. Service A sends an HTTP or gRPC request to service B and waits for the answer. That works well until B is slow, B is down, B gets ten times more traffic than usual, or five other services also need to know what A just did.

A **message broker** is the other answer. This post covers what a broker is, why you would use one, the concepts that show up in every broker, and how two popular ones, **RabbitMQ** and **Apache ActiveMQ Artemis**, approach the job. Kafka comes in at the end, because it's often lumped in with them but works quite differently.

## What a message broker is

A message broker is a server that sits between the programs that **produce** messages and the programs that **consume** them. The producer doesn't call the consumer. It hands a message to the broker, and the broker takes responsibility for getting it to the right place, now or later.

```text
 Direct call                         Through a broker

 ┌─────────┐   HTTP    ┌─────────┐   ┌──────────┐      ┌────────┐      ┌──────────┐
 │Service A│──────────►│Service B│   │ Producer │─────►│ Broker │─────►│ Consumer │
 └─────────┘  (waits)  └─────────┘   └──────────┘      │ ┌────┐ │      └──────────┘
                                       (moves on)       │ │queue│ │
                                                        │ └────┘ │
                                                        └────────┘
```

A good analogy is a post office. You drop off a letter and go home. The post office stores it, sorts it, and delivers it when the recipient is there. You don't need to know whether the recipient is home, or even exactly where they live.

## Why put a broker in the middle

**Decoupling in time.** Producer and consumer don't have to be running at the same moment. If the consumer is down for a deploy, messages wait in a queue and get processed when it comes back.

**Decoupling in space.** The producer only knows the broker and a destination name. It doesn't know how many consumers there are, where they run, or what language they're written in.

**Load levelling.** A burst of 10,000 orders lands in a queue, and the workers process them at whatever rate they can sustain. Without a broker, that burst lands directly on the workers and probably knocks them over.

**Scaling out.** Add more consumers to the same queue and the broker spreads the messages between them. This is the *competing consumers* pattern, and it's the simplest way to scale background work.

**Reliability.** Messages can be persisted to disk and replicated. A message is only removed once a consumer confirms it handled it, so a crash mid-task doesn't lose work.

**Fan-out.** One event, such as "user signed up", can be delivered to every service that cares about it (email, billing, analytics) without the producer knowing any of them exist.

The cost is one more piece of infrastructure to run, monitor and secure, and a system that is now **asynchronous**: the producer no longer gets an immediate answer, and you have to think about ordering, duplicates and failures more explicitly.

## Core concepts

These ideas show up in every broker, under slightly different names.

### Queues: point-to-point

A **queue** holds messages until a consumer takes them. Each message goes to **exactly one** consumer. If several consumers read the same queue, they share the work.

Use queues for *commands* and *work items*: "resize this image", "send this email", "process this payment".

### Topics: publish/subscribe

With **publish/subscribe**, every subscriber gets **its own copy** of each message. The producer publishes to a topic, and the broker delivers a copy to each subscription.

Use pub/sub for *events*: "order placed", "user deleted". The producer is announcing that something happened, and doesn't care who listens.

Most real systems combine the two: an event is fanned out to one queue per interested service, and within each service several instances compete for messages on that service's queue.

### Acknowledgements and delivery guarantees

When a broker delivers a message, it doesn't know whether the consumer actually processed it. So the consumer sends an **acknowledgement** (ack) once it's done. Until then, the broker keeps the message. If the consumer disconnects or crashes without acking, the broker redelivers it, possibly to another consumer.

That gives you the three classic guarantees:

| Guarantee | What it means | How you get it |
| --- | --- | --- |
| At-most-once | A message may be lost, but is never delivered twice | Ack on receipt, before processing |
| At-least-once | A message is never lost, but may be delivered more than once | Ack after processing |
| Exactly-once | Each message has its effect exactly once | Very hard end to end. Usually at-least-once plus idempotent consumers |

In practice, almost everyone runs **at-least-once** and makes consumers **idempotent**: processing the same message twice has the same effect as processing it once. A common trick is to give every message a unique ID and have the consumer record which IDs it has already handled.

### Prefetch and flow control

Brokers push messages to consumers in batches for efficiency. The **prefetch** (or credit) limit controls how many unacknowledged messages a consumer may hold at once. Too high, and one slow consumer hoards messages while others sit idle. Too low, and throughput suffers from round trips. A small number, such as 10 to 50, is a sensible starting point for work queues.

### Dead-letter queues

Some messages will never succeed: malformed payloads, a bug that throws on a specific input, a reference to a record that no longer exists. If the broker keeps redelivering them forever, they block the queue (a *poison message*). A **dead-letter queue** (DLQ) is where messages go after they are rejected or exceed a delivery limit, so the rest of the queue keeps moving and a human can inspect the failures later.

### Durability and persistence

A **durable** queue survives a broker restart. A **persistent** message is written to disk before the broker confirms it. You usually want both for anything that matters, plus **publisher confirms** so the producer knows the broker actually stored the message.

### Protocols

Brokers speak standard wire protocols, which is what lets a Python producer talk to a Java consumer through the same broker:

- **AMQP 0-9-1**: the protocol RabbitMQ was built around, with exchanges and bindings baked in.
- **AMQP 1.0**: an OASIS and ISO standard. Despite the name, it's a different protocol from 0-9-1, and it doesn't define broker topology.
- **MQTT**: lightweight publish/subscribe, designed for IoT and unreliable networks.
- **STOMP**: a simple text protocol, easy to use from scripting languages and browsers.
- **JMS / Jakarta Messaging**: a *Java API*, not a wire protocol. Java code is written against JMS, and the client library maps it onto whatever protocol the broker speaks.

## RabbitMQ

RabbitMQ is an open source broker written in Erlang, and probably the most widely deployed broker outside the Java world. Its defining idea is that **producers never publish directly to queues**.

### Exchanges, bindings and routing keys

In RabbitMQ, producers send messages to **exchanges**, which route them to zero or more queues according to **bindings** and **routing keys**.[^1] Consumers read from queues. The exchange is the routing layer in between:

```text
                         binding: "order.*"
              ┌──────────┐ ───────────────► [ queue: billing ] ──► billing service
 producer ──► │ exchange │
 (key:        │ (topic)  │ ───────────────► [ queue: audit ]   ──► audit service
  order.paid) └──────────┘ binding: "#"
```

There are four main exchange types:

| Exchange type | Routes a message to... | Typical use |
| --- | --- | --- |
| `direct` | Queues whose binding key equals the routing key exactly | Work queues, routing by task type |
| `topic` | Queues whose binding pattern matches the routing key (`*` = one word, `#` = zero or more) | Event routing by category, e.g. `order.*`, `*.eu.#` |
| `fanout` | Every bound queue, ignoring the routing key | Broadcasting events |
| `headers` | Queues whose binding matches message header values | Routing on several attributes at once |

There is also a nameless **default exchange**, a direct exchange that every queue is automatically bound to by its own name. That's why "publish to queue X" seems to work in simple examples. One subtlety: if no binding matches, the message is dropped unless the publisher asked for it to be returned (the `mandatory` flag) or an alternate exchange is configured.

RabbitMQ also supports exchange-to-exchange bindings, an extension to AMQP 0-9-1, so routing can be layered.[^2]

### Queue types

Modern RabbitMQ has three kinds of queue:

- **Classic queues**: the original, non-replicated queue type. Good for transient or low-importance data.
- **Quorum queues**: replicated across nodes using the Raft consensus algorithm. This is the default choice for anything you can't afford to lose.
- **Streams**: an append-only, replayable log with non-destructive consumption, much closer to Kafka's model.

Classic queue *mirroring*, the old way to replicate queues, was removed in RabbitMQ 4.0 after three years of deprecation, leaving quorum queues and streams as the replicated data types.[^3]

### Protocols

RabbitMQ 4.0 made **AMQP 1.0 a core protocol** that is always enabled, alongside AMQP 0-9-1, with no plugin needed.[^4] MQTT and STOMP are available as plugins. Recent releases have kept adding to streams, including AMQP filter expressions in 4.1 and SQL filter expressions in 4.2 for broker-side filtering.[^5]

### A quick look

Run a broker with the management UI on port 15672 (login `guest` / `guest`):

```bash
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:4-management
```

A producer and a consumer in Python, using the `pika` client:

```python
# producer.py
import json
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
ch = conn.channel()

ch.exchange_declare(exchange="orders", exchange_type="topic", durable=True)
ch.confirm_delivery()  # publisher confirms

order = {"id": 42, "total": 19.99}
ch.basic_publish(
    exchange="orders",
    routing_key="order.paid",
    body=json.dumps(order),
    properties=pika.BasicProperties(delivery_mode=2, message_id="order-42-paid"),
)
conn.close()
```

```python
# consumer.py
import json
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
ch = conn.channel()

ch.exchange_declare(exchange="orders", exchange_type="topic", durable=True)
ch.queue_declare(queue="billing", durable=True, arguments={"x-queue-type": "quorum"})
ch.queue_bind(queue="billing", exchange="orders", routing_key="order.*")
ch.basic_qos(prefetch_count=10)

def handle(channel, method, properties, body):
    order = json.loads(body)
    print(f"billing order {order['id']}")
    channel.basic_ack(delivery_tag=method.delivery_tag)  # ack after processing

ch.basic_consume(queue="billing", on_message_callback=handle)
ch.start_consuming()
```

The producer only knows about the `orders` exchange and the routing key. Adding an `audit` service later is just another queue bound to the same exchange. The producer doesn't change.

## Apache ActiveMQ Artemis

ActiveMQ Artemis is the next-generation broker of the Apache ActiveMQ project. It grew out of JBoss HornetQ, which was donated to Apache, and is written in Java. The older broker is now called **ActiveMQ Classic**. Artemis is the base of Red Hat AMQ Broker and is widely used in the enterprise Java world.

### Multi-protocol by design

Artemis describes itself as a multi-protocol, embeddable, high-performance, clustered, asynchronous messaging system.[^6] The broker core is protocol-agnostic: it doesn't actually know anything about JMS, and each protocol is a module that translates onto the core.[^7] It supports:

- **Core**, its own native protocol (also compatible with HornetQ clients)
- **AMQP 1.0**
- **OpenWire**, the ActiveMQ Classic wire protocol, so Classic clients can move over without code changes
- **MQTT**
- **STOMP**

The default configuration ships an acceptor per protocol, plus a generic one that accepts all of them on port 61616.[^8]

### Addresses, routing types and queues

Artemis has its own model, which maps neatly onto both point-to-point and pub/sub:

- An **address** is where producers send messages.
- Each address has one or more **queues** attached.
- The address's **routing type** decides how messages are distributed:
  - **anycast**: each message goes to one queue on the address. This is point-to-point, what JMS calls a *queue*.
  - **multicast**: each message is copied to every queue on the address. This is pub/sub, what JMS calls a *topic*, with each subscription getting its own queue.

```text
 address "orders"  (anycast)            address "events"  (multicast)
 ┌──────────────────────────┐           ┌──────────────────────────┐
 │  queue "orders"          │           │  queue "billing-sub"     │ ◄── copy
 │   ▲ one consumer per msg │           │  queue "audit-sub"       │ ◄── copy
 └──────────────────────────┘           └──────────────────────────┘
```

Compared with RabbitMQ, routing is simpler (there is no general exchange/binding layer), but features like *diverts* let you copy or move messages between addresses when you need it.

### Persistence: the journal

Each Artemis server has its own high-performance persistent journal, an append-only file store optimised for message persistence.[^9] On Linux it can use asynchronous I/O (libaio) to write directly to disk. This journal is a big part of why Artemis has a reputation for throughput.

For queues that grow larger than memory, Artemis uses **paging**: messages beyond a configured size are written to page files and read back as consumers catch up, instead of the broker running out of memory.

### High availability

Artemis offers both **shared-store** HA, where a primary and backup use the same storage, and **replication**, where the primary streams its journal to a backup over the network. It also supports clusters with message load balancing, and **federation** and **broker connections** for linking brokers across sites.

### A quick look

Run a broker. The web console is on port 8161:

```bash
docker run -d --name artemis \
  -p 61616:61616 -p 8161:8161 \
  -e ARTEMIS_USER=admin -e ARTEMIS_PASSWORD=admin \
  apache/activemq-artemis:latest
```

The broker ships with a CLI that can produce and consume test messages:

```bash
# send 10 messages to an anycast queue called "orders"
docker exec artemis /var/lib/artemis-instance/bin/artemis producer \
  --user admin --password admin \
  --destination queue://orders --message-count 10

# consume them
docker exec artemis /var/lib/artemis-instance/bin/artemis consumer \
  --user admin --password admin \
  --destination queue://orders --message-count 10
```

From Java, you'd typically use the JMS (Jakarta Messaging) API with the Artemis client, or Spring's `JmsTemplate`. From other languages, use any AMQP 1.0 client (such as Qpid Proton) or STOMP.

## RabbitMQ vs ActiveMQ Artemis

Both are mature, capable, general-purpose brokers. The differences are more about ecosystem and model than about raw capability.

| | RabbitMQ | ActiveMQ Artemis |
| --- | --- | --- |
| Language / runtime | Erlang | Java |
| Origin | Rabbit Technologies, now Broadcom (VMware Tanzu) | JBoss HornetQ, donated to Apache |
| Native model | Exchanges → bindings → queues | Addresses with anycast / multicast queues |
| Routing flexibility | Very high (topic patterns, headers, E2E bindings) | Simpler, plus diverts and filters |
| Protocols | AMQP 0-9-1, AMQP 1.0 (core); MQTT, STOMP (plugins) | Core, AMQP 1.0, OpenWire, MQTT, STOMP |
| Java / JMS fit | Via a JMS client | First-class |
| Replicated queues | Quorum queues (Raft), streams | Journal replication or shared store |
| Replayable log | Streams | Not the primary model |
| Typical home | Polyglot microservices, task queues | Enterprise Java, Jakarta EE, ActiveMQ Classic migrations |

A rough rule of thumb:

- Choose **RabbitMQ** if you have a polyglot stack, want rich routing, or want a huge ecosystem of client libraries and tutorials.
- Choose **Artemis** if you're in the Java/JMS world, are migrating from ActiveMQ Classic, or need many protocols on one broker.

## Where Kafka fits

Kafka is usually mentioned in the same breath, but it's built on a different idea.

RabbitMQ and Artemis are **"smart broker, simple consumer"** systems. The broker tracks the delivery state of every message, and once a message is acknowledged, it's gone.

Kafka is a **distributed, partitioned commit log**. Each partition is an ordered, immutable sequence of records that is continually appended to.[^10] Records are **not deleted when consumed**: the cluster keeps all published records for a configurable retention period, whether they have been read or not.[^11] Each consumer just tracks its position, the *offset*, and can rewind to reprocess old data or skip to the latest.[^12] Load balancing and fan-out are both handled by **consumer groups**: consumers in the same group share partitions, and different groups each get every record.

| | Traditional broker (RabbitMQ, Artemis) | Log (Kafka) |
| --- | --- | --- |
| After consumption | Message is removed | Record stays until retention expires |
| Delivery state | Tracked per message by the broker | Each consumer tracks its own offset |
| Replay | Not really (except RabbitMQ streams) | Built in |
| Ordering | Per queue, weakened by competing consumers and redelivery | Strict per partition |
| Routing | Rich (exchanges, filters, selectors) | By topic and partition key only |
| Sweet spot | Task queues, request/reply, complex routing, per-message retries | Event streaming, high-volume pipelines, event sourcing, analytics |

The lines have blurred: RabbitMQ streams give you a replayable log, and Kafka has added share groups for queue-like consumption. But the default mental model still holds: **reach for a broker to distribute work, and a log to record and replay events.**

## Common pitfalls

- **Assuming exactly-once.** You will see duplicates: after a consumer crash, a network blip, or a redelivery. Make consumers idempotent.
- **No dead-letter queue.** One poison message can stall a queue forever. Configure a DLQ and a delivery limit from day one, and alert on it.
- **Unbounded queues.** A queue nobody consumes from will grow until the broker runs out of memory or disk. Set length limits or TTLs, and monitor queue depth.
- **Treating the broker as a database.** Brokers are built for messages in flight, not long-term storage or queries. Long-lived queues of millions of messages are a smell.
- **Huge messages.** Send a reference (an object storage key or database ID), not a 50 MB payload.
- **Ordering assumptions.** Competing consumers and redeliveries reorder messages. If order matters, design for it explicitly (single active consumer, partition keys, or sequence numbers).
- **Forgetting publisher confirms.** Without them, a producer can believe a message was sent when the broker never stored it.

## Bottom line

- A **message broker** decouples producers and consumers in time and space, levels load, and makes it easy to scale workers and fan out events.
- The core concepts are the same everywhere: **queues** for work, **pub/sub** for events, **acks** for reliability, **DLQs** for failures, and standard **protocols** for interoperability.
- **RabbitMQ** stands out for its flexible exchange-based routing, quorum queues and streams, and broad language support.
- **ActiveMQ Artemis** stands out for its multi-protocol core, fast journal, and first-class place in the Java/JMS ecosystem.
- **Kafka** is a log, not a queue: records are retained and replayed, which makes it the better fit for event streaming rather than task distribution.

[^1]: [How to Use RabbitMQ Exchanges Effectively](https://oneuptime.com/blog/post/2026-01-26-rabbitmq-exchanges-guide/view), OneUptime blog, January 2026.
[^2]: [Exchanges](https://www.rabbitmq.com/docs/exchanges), RabbitMQ documentation: exchange types, bindings and exchange-to-exchange bindings.
[^3]: [What's New in RabbitMQ 4.0](https://www.rabbitmq.com/docs/4.0/whats-new), RabbitMQ documentation: removal of classic queue mirroring.
[^4]: [RabbitMQ 4.0.1 release notes](https://github.com/rabbitmq/rabbitmq-server/releases/tag/v4.0.1), GitHub: AMQP 1.0 becomes a core protocol.
[^5]: [RabbitMQ blog: New Features](https://www.rabbitmq.com/blog/tags/new-features): AMQP filter expressions (4.1) and SQL filter expressions (4.2) for streams.
[^6]: [Apache ActiveMQ Artemis: Preface](https://activemq.apache.org/artemis/docs/1.2.0/preface.html), ActiveMQ Artemis documentation.
[^7]: [Core Architecture](https://activemq.apache.org/artemis/docs/latest/architecture.html), ActiveMQ Artemis documentation.
[^8]: [Protocols and Interoperability](https://activemq.apache.org/artemis/docs/latest/protocols-interoperability.html), ActiveMQ Artemis documentation.
[^9]: [Core Architecture](https://activemq.apache.org/artemis/docs/latest/architecture.html), ActiveMQ Artemis documentation: the persistent journal.
[^10]: [Kafka introduction](https://kafka.apache.org/0100/api.html), Apache Kafka documentation: topics and partitions.
[^11]: [Kafka introduction](https://kafka.apache.org/0102/js), Apache Kafka documentation: retention of published records.
[^12]: [Kafka 0.8 introduction](https://kafka.apache.org/08/getting-started/introduction/), Apache Kafka documentation: consumer-controlled offsets.
