# Notification System Design with Kafka

Step-by-step design and evolution for a Kafka-based notification system.

---

## Initial Design: Notification System with Kafka

### Requirements

- Send notifications (email, push, SMS) to users
- Events trigger notifications (e.g., order shipped, password reset, marketing campaigns)
- Decouple producers (order service, auth service) from consumers (notification workers)
- Reliable delivery, at-least-once semantics

---

### Step 1: Core Architecture

```
[Order Service]     ──►  [Kafka: notifications]  ──►  [Notification Worker]
[Auth Service]      ──►       (topic)            ──►  [Notification Worker]
[Marketing Service] ──►                            ──►  [Notification Worker]
```

**Components:**

| Component     | Role                                                                                                                 |
| ------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Producers** | Order, Auth, Marketing services publish events (e.g., `OrderShipped`, `PasswordResetRequested`, `CampaignTriggered`) |
| **Topic**     | `notifications` — receives all notification events                                                                   |
| **Consumer**  | Notification Worker consumes events, calls email/SMS/push providers                                                  |

---

### Step 2: Message Schema

```json
{
  "eventId": "evt-123-uuid",
  "eventType": "OrderShipped",
  "userId": "user-456",
  "timestamp": "2025-03-01T10:00:00Z",
  "payload": {
    "orderId": "ord-789",
    "trackingNumber": "TRK-001",
    "channel": "email"
  }
}
```

**Key:** `userId` — ensures all events for a user go to the same partition (ordering per user).

**Headers:** `trace-id`, `source-service` for observability.

---

### Step 3: Topic Configuration

| Config             | Value  | Reason                                    |
| ------------------ | ------ | ----------------------------------------- |
| Partitions         | 12–24  | Parallelism for consumers; start moderate |
| Replication factor | 2–3    | Fault tolerance                           |
| Retention          | 7 days | Replay, debugging                         |
| Compression        | lz4    | Lower storage, faster I/O                 |

---

### Step 4: Producer Configuration

| Config      | Value  | Reason                          |
| ----------- | ------ | ------------------------------- |
| acks        | all    | Durability; wait for replicas   |
| retries     | 3      | Transient failures              |
| idempotence | true   | No duplicates on retry          |
| key         | userId | Partitioning, ordering per user |

---

### Step 5: Consumer Configuration

| Config             | Value                | Reason                              |
| ------------------ | -------------------- | ----------------------------------- |
| group.id           | notification-workers | Single consumer group               |
| enable.auto.commit | false                | Manual commit after successful send |
| max.poll.records   | 100                  | Batch size per poll                 |

**Processing flow:**

1. Poll batch
2. For each event: call provider (email/SMS/push)
3. On success: `commitSync()`
4. On failure: retry with exponential backoff; after max retries → DLQ

---

### Step 6: Idempotency

- Use `eventId` as natural key
- Before sending: check if `eventId` already processed (DB or cache)
- Avoid duplicate notifications (e.g., same email twice)

---

### Step 7: DLQ (Dead Letter Queue)

- Topic: `notifications-dlq`
- Send failed events after N retries
- Monitor, alert, manual reprocess or fix

---

### Step 8: Observability

- **Metrics:** consumer lag, throughput, error rate
- **Tracing:** `trace-id` in headers, correlate across services
- **Logging:** structured logs with `eventId`, `userId`

---

## Evolution 1: 10 Million Users

### Challenges

- High volume of events
- Many concurrent consumers
- Storage and throughput

### Solutions

| Aspect               | Change                                                                                                                |
| -------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **Partitions**       | Increase to 50–100+ (more users → more parallelism)                                                                   |
| **Consumers**        | Scale horizontally; up to 1 consumer per partition per group                                                          |
| **Topic sharding**   | Split by channel: `notifications-email`, `notifications-push`, `notifications-sms` — different throughput per channel |
| **Batch processing** | Consumer processes in batches; reduce per-message overhead                                                            |
| **Compression**      | lz4 or zstd to reduce storage and network                                                                             |
| **Retention**        | Tune (e.g., 3–7 days) to balance storage vs replay needs                                                              |

### Capacity Estimate

- 10M users, assume 1 event/user/day → ~120 events/s average
- Peak (e.g., 10x) → ~1,200 events/s
- 100 partitions, 1 KB/event → ~1.2 MB/s — well within Kafka capacity

---

## Evolution 2: Multi-Region

### Challenges

- Users in different regions (latency, compliance)
- Data residency (GDPR, etc.)
- Disaster recovery

### Solutions

| Aspect                         | Approach                                                                              |
| ------------------------------ | ------------------------------------------------------------------------------------- |
| **Cluster per region**         | `us-east`, `eu-west`, `ap-south` — each with its own Kafka cluster                    |
| **Routing**                    | Route users to the cluster in their region (e.g., by `userId` hash or geo)            |
| **Replication across regions** | Use MirrorMaker 2 or Confluent Replicator to replicate critical topics for DR         |
| **Producer routing**           | Services in each region produce to the local cluster                                  |
| **Consumer locality**          | Notification workers run in the same region as the cluster (low latency to providers) |

### Architecture

```
[US Services]  ──►  [Kafka US]  ──►  [Workers US]  ──►  [Email/Push US]
[EU Services]  ──►  [Kafka EU]  ──►  [Workers EU]  ──►  [Email/Push EU]
[APAC Services] ──► [Kafka APAC] ──► [Workers APAC] ──► [Email/Push APAC]
```

### Cross-Region Replication (DR)

- Replicate `notifications` to a standby region
- Failover: switch consumers to the standby cluster if primary fails
- Use **active-passive** or **active-active** depending on requirements

---

## Evolution 3: Broker Failure

### Challenges

- Single broker down
- Entire cluster down
- No message loss, minimal downtime

### Solutions

| Layer            | Mitigation                                                                                 |
| ---------------- | ------------------------------------------------------------------------------------------ |
| **Replication**  | `replication.factor=3`, `min.insync.replicas=2` — tolerate 1 broker down without data loss |
| **Broker count** | At least 3 brokers; spread partitions across them                                          |
| **Producer**     | `acks=all` — only ack when replicas confirm                                                |
| **Consumer**     | Offsets in Kafka; on broker failure, rebalance assigns partitions to live brokers          |
| **Monitoring**   | Alert on under-replicated partitions, broker down                                          |
| **Automation**   | Auto-restart brokers (K8s, systemd); replace failed nodes                                  |

### Failure Scenarios

| Scenario                  | Behavior                                                                                         |
| ------------------------- | ------------------------------------------------------------------------------------------------ |
| **1 broker down**         | Replication keeps data; rebalance moves leadership; brief pause, then resume                     |
| **2 brokers down (RF=3)** | Some partitions lose in-sync replicas; producers may block (`acks=all`); need to restore brokers |
| **Entire cluster down**   | No consumption; use multi-region replication for failover to standby                             |

### Configuration Summary

```
# Broker / Topic
replication.factor=3
min.insync.replicas=2

# Producer
acks=all
retries=10
```

---

## Summary: Design Checklist

| Phase              | Key decisions                                                        |
| ------------------ | -------------------------------------------------------------------- |
| **Initial**        | Single topic, partitions by userId, manual commit, idempotency, DLQ  |
| **10M users**      | More partitions, topic sharding by channel, horizontal scaling       |
| **Multi-region**   | Cluster per region, local routing, optional cross-region replication |
| **Broker failure** | RF=3, min.insync.replicas=2, acks=all, monitoring, automation        |
