# Kafka Concepts

Summary of Apache Kafka core concepts based on [Apache Kafka — Parte 1: Conceitos](https://medium.com/@guilhermelodi/apache-kafka-parte-1-conceitos-74adbdb1327) by Guilherme Lodi.

---

## What is Apache Kafka?

Apache Kafka is a **distributed streaming platform** with three main capabilities:

1. **Process** data streams
2. **Store** streams in a fault-tolerant system
3. **Publish and subscribe** to data streams (similar to a message queue)

### Basic Flow

1. **Producer** sends a message to a **topic**
2. The **topic** stores the message
3. A **consumer** reads the message

---

## Core Concepts

### Broker

- A **Broker** (or Kafka Node) is a server instance running inside a Kafka Cluster
- A Kafka Cluster can have one or more brokers
- Brokers store topic partitions
- High throughput: hundreds of megabytes per second for read/write operations
- Originally developed at LinkedIn; now used by companies like Uber, Netflix, Spotify

### Message

The message is the element that flows through the architecture. It contains:

| Component  | Description                     |
| ---------- | ------------------------------- |
| **Value**  | Byte array (e.g., String, JSON) |
| **Key**    | Optional, usually a String      |
| **Header** | Metadata                        |

### Topic

Topics organize messages by category or purpose. When creating a topic you define:

- **Name**
- **Number of partitions**
- **Replication factor** (number of synchronized partition replicas)

Important properties:

- Messages within each partition are **ordered**
- Each message in a partition has a unique **offset** (sequential identifier)
- Messages are **immutable** once published
- Messages are retained for **7 days by default** (whether consumed or not)
- Unlike RabbitMQ, messages are not removed when read

### Producer

- Any application or command that sends messages to a Kafka topic
- **Partitioning logic:**
  - **No key**: messages are distributed in round-robin across partitions
  - **With key**: all messages with the same key go to the same partition (as long as partition count stays the same)

### Consumer

- Reads messages from one or more partitions of a topic
- Requests the next message when ready to consume

**Ordering guarantees:**

- Messages sent to the same partition are stored in order
- Consumers read messages from a partition in the same order they were stored

**Consumer Groups:**

- When a consumer belongs to a **Consumer Group**, Kafka stores the **consumer offset** (last read offset per partition)
- If a consumer goes offline, it can resume from where it left off
- Multiple partitions allow different consumers to read in parallel
- Each consumer in a group is assigned exclusive partitions
- Example: a topic with 3 partitions can be read by up to 3 consumers
- If a consumer fails, another consumer in the group takes over its partitions until it recovers

---

## Streaming Platform vs Message Queue

| Aspect                | Traditional Message Queue (RabbitMQ, ActiveMQ, SQS)             | Streaming Platform (Kafka)                                                                        |
| --------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Consumption model** | Message is removed after consumption (or acknowledgment)        | Messages persist for a retention period (e.g., 7 days); multiple consumers can read the same data |
| **Storage**           | Queue in memory or disk; focus on delivery and removal          | Persistent log; data is retained and can be reprocessed                                           |
| **Replay**            | Cannot reprocess old messages                                   | Can reprocess by reading from a specific offset                                                   |
| **Throughput**        | Generally lower; designed for task queues                       | High throughput; designed for large event volumes                                                 |
| **Ordering**          | Order per queue; can be affected by retries and multiple queues | Order guaranteed within each partition                                                            |
| **Use cases**         | Async tasks, point-to-point decoupling                          | Event sourcing, data pipelines, real-time analytics, large-scale system integration               |

**Summary:**

- **Traditional MQ**: "Send the message, deliver it once, remove it." Focus on **decoupling** and **delivery**.
- **Streaming (Kafka)**: "Store the event stream and allow multiple consumers and reprocessing." Focus on **event history** and **high volume**.

Kafka behaves like a **distributed event log** that can be read multiple times; a traditional MQ behaves like a **message queue** that empties as messages are consumed.

---

## Advanced Concepts

### Partitions

- Partitions divide a topic into ordered, immutable sequences of messages
- Each partition is an independent log; ordering is guaranteed **only within** a partition
- More partitions = higher parallelism (more consumers can read in parallel)
- Partition count can be increased but not decreased
- Messages with the same key always go to the same partition (when partition count is unchanged)

### Consumer Groups

- A **Consumer Group** is a set of consumers that work together to consume a topic
- Each partition is assigned to **exactly one** consumer in the group
- Consumers in the same group do not receive duplicate messages (each partition has one owner)
- Different consumer groups can read the same topic independently (each maintains its own offset)
- Group membership is managed by the Kafka Group Coordinator

### Offset: Manual vs Auto

| Mode              | Description                                                                     | When to use                                                          |
| ----------------- | ------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| **Auto commit**   | Kafka automatically commits offsets at a configurable interval (e.g., every 5s) | Simple use cases; acceptable to reprocess some messages on failure   |
| **Manual commit** | Application explicitly calls `commitSync()` or `commitAsync()` after processing | When you need at-least-once: commit only after successful processing |

**Risk of auto-commit:** If the consumer crashes after commit but before processing finishes, the message is lost (from consumer perspective). If it crashes before commit, the message may be processed twice.

### Rebalancing

- **Rebalancing** occurs when partition assignments change within a consumer group
- Triggers: consumer joins, consumer leaves (crash or graceful shutdown), new partitions added
- During rebalance, all consumers stop processing until new assignments are distributed
- **Eager rebalance:** All consumers give up partitions; short pause for everyone
- **Cooperative rebalance (Incremental):** Consumers give up only affected partitions; less disruptive
- Minimize rebalances by keeping consumer sessions stable and using `session.timeout.ms` appropriately

### At-Least-Once

- Message is delivered **at least once**; duplicates are possible
- Achieved by: process message → commit offset only after success
- If consumer crashes before commit, it will reprocess the same message(s) on restart
- **Trade-off:** Simplicity vs risk of duplicate processing (handle with idempotency)

### Exactly-Once (Idempotent Producer)

- Message is delivered **exactly once**; no duplicates
- **Producer side:** `enable.idempotence=true` — producer retries with the same sequence number; broker deduplicates
- **Consumer side:** Use transactional reads + idempotent processing, or store offset and output in the same transaction (e.g., Kafka → DB in one transaction)
- **Kafka Transactions:** Producer can write to multiple partitions atomically; consumer can read committed messages only

### DLQ (Dead Letter Queue)

- A **Dead Letter Queue** is a topic where failed messages are sent after N retries
- Use when a message cannot be processed (e.g., invalid format, business rule violation)
- Prevents poison messages from blocking the main consumer
- DLQ messages should be monitored, analyzed, and optionally reprocessed after fix

### Ordering Guarantees

- **Per partition:** Messages in the same partition are strictly ordered
- **Across partitions:** No global order; partitions are independent
- **Strategy:** Use a consistent key so related messages go to the same partition
- **Trade-off:** Same key = same partition = ordering, but limits parallelism for that key

### Backpressure

- **Backpressure** = slowing down producers when consumers cannot keep up
- Kafka naturally provides backpressure: consumers poll at their own pace; brokers buffer messages
- If producers write faster than consumers read, lag grows (monitor `consumer lag`)
- **Mitigation:** Scale consumers (add partitions + consumers), optimize processing, use batch processing, or temporarily throttle producers

---

## Practice Questions & Answers

### How to avoid duplicate processing?

1. **Idempotent consumers:** Design processing so that executing the same message twice has the same effect as once (e.g., upsert by ID, check-before-insert)
2. **Exactly-once semantics:** Use idempotent producer + transactional consumer (read-process-write in one transaction)
3. **Deduplication:** Store processed message IDs (e.g., in DB or cache) and skip if already seen
4. **Manual commit:** Commit offset only after successful, idempotent processing

### How to guarantee ordering?

1. **Same partition:** Use a consistent message key so related messages go to the same partition
2. **Single partition:** One partition = total order (but no parallelism)
3. **Partition by entity:** e.g., `userId` as key — all events for a user are ordered
4. **Accept scope:** Ordering is per partition; across partitions there is no guarantee

### How to scale?

1. **Add partitions:** Increase partitions on the topic (cannot decrease)
2. **Add consumers:** Up to one consumer per partition per consumer group
3. **Add consumer instances:** Scale horizontally within the same group
4. **Add consumer groups:** Different groups can consume the same topic for different purposes
5. **Note:** More partitions = more parallelism but also more overhead (broker, replication, rebalancing)

### What happens if a consumer dies?

1. **Detection:** Broker detects via `session.timeout.ms` (heartbeat) or `max.poll.interval.ms` (no poll)
2. **Rebalancing:** Group coordinator triggers a rebalance
3. **Reassignment:** Partitions previously owned by the dead consumer are assigned to remaining consumers
4. **Resume:** New owners start from the last committed offset for each partition
5. **Risk:** If offset was not committed, some messages may be reprocessed (at-least-once)

### How to do graceful shutdown?

1. **Stop polling:** Stop requesting new messages
2. **Finish in-flight:** Complete processing of messages already fetched
3. **Commit offsets:** Call `commitSync()` to persist the last processed offset
4. **Leave group:** Call `consumer.close()` — triggers rebalance so other consumers take over
5. **Config:** Use `consumer.close()` which handles leave + commit; avoid `kill -9`, use `SIGTERM` and handle it to run shutdown logic

---

## References

- [Confluent: What is Apache Kafka?](https://www.confluent.io/what-is-apache-kafka/)
- [Apache Kafka](https://kafka.apache.org/)
