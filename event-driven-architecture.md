# Event-Driven Architecture (EDA) in Microservices

Summary based on [Event-driven Architecture (EDA) em uma Arquitetura de Microsserviços](https://medium.com/@marcelomg21/event-driven-architecture-eda-em-uma-arquitetura-de-micro-servi%C3%A7os-1981614cdd45) by Marcelo M. Gonçalves.

---

## Introduction

In microservices architectures, there are two main communication styles: **synchronous** and **asynchronous**. Event-Driven Architecture (EDA) implements the **asynchronous** communication model. Ideally, a microservice should have **low coupling** and communicate using the **Message-passing pattern** or **Event-Driven Architecture**.

---

## Synchronous Communication

- Exposes APIs based on REST/gRPC over HTTP
- Uses blocking (or non-blocking I/O) calls — the client must wait for the response
- Creates coupling through the contract between client and server
- API design is tied to HTTP: methods (GET, POST, PUT, DELETE, PATCH), headers (Authorization, Accept, Content-Type), status codes (200, 201, 400, 403, 500, 503)
- The client must mirror the API endpoints and know the orchestrated operations
- **Request-Response** flow: client sends request → server processes → server returns response

**Limitation:** When multiple services need to be updated, the calling application must centralize the flow, call each endpoint, and orchestrate the entire process. REST remains a solid starting point for many solutions.

---

## Asynchronous Communication (EDA)

EDA is an architectural design pattern where components communicate via **event streams**, notifying about **state changes** and promoting **low coupling**.

### How it works

1. **Publishers** send events to an **Event Bus** (Broker)
2. The Broker **broadcasts** events to all **Subscribers** interested in those events
3. Publishers and Subscribers **do not know each other** — the Broker handles routing
4. Asynchrony optimizes resource usage: no blocking while waiting for responses

### Key components

| Component      | Role                                                                                                      |
| -------------- | --------------------------------------------------------------------------------------------------------- |
| **Event Bus**  | Infrastructure backbone that delivers events to subscribers                                               |
| **Broker**     | Message/Stream broker (e.g., Apache Kafka, RabbitMQ, ActiveMQ, IBM MQ) — must be reliable, fast, scalable |
| **Publisher**  | Produces and publishes events                                                                             |
| **Subscriber** | Subscribes to events and reacts to them                                                                   |
| **Event**      | Represents a notable occurrence in the system                                                             |

### Adoption context

- Suited for microservices that need **autonomy** and **independence**
- Works well with **containers** (e.g., Docker) and **serverless deployment**
- One event can be delivered to **multiple subscribers** in a single broadcast

---

## Adopting Event-Driven Architecture

EDA assumes a **distributed architecture** with independent components (asynchronous microservices), enabling greater agility and smarter interactions with an event-oriented ecosystem.

### Main components

- **Events**
- **Producers** (Publishers)
- **Consumers** (Subscribers)
- **Event Bus**
- **Brokers**

### Business value

- **Easy to extend** the ecosystem with new components
- Add new subscribers that react to existing events **without changing** current implementations
- Modular and low-risk evolution

### What are events?

- Capture **facts** and **behaviors** (similar to real life)
- Can come from multiple sources in **real time**
- A **notable occurrence** inside or outside the business domain
- Announce **state changes** of objects
- Act as a mechanism for **distributing application state**
- A related, ordered sequence of events forms a **Stream** over time

---

## Event Types in EDA

| Type                             | Description                                                     |
| -------------------------------- | --------------------------------------------------------------- |
| **Event Notification**           | Only notifies that something relevant happened; minimal payload |
| **Event-Carried State Transfer** | Carries a **payload** with relevant data about the occurrence   |

Both can include **headers**, **metadata**, and **timestamps** for context. For structured payloads, consider **Apache Avro** (JSON schema + compact binary serialization).

---

## Orchestration vs Choreography

| Style             | Responsibility                                                              | Typical use                   |
| ----------------- | --------------------------------------------------------------------------- | ----------------------------- |
| **Orchestration** | Central component controls and coordinates the flow (e.g., REST API caller) | Synchronous, request-response |
| **Choreography**  | Each service reacts to events independently; no central coordinator         | EDA, event-driven flows       |

In EDA, the default is **choreography**: each microservice is responsible for the events it cares about. **Orchestration** can still be used when a central component coordinates the flow.

### Split-Join pattern

- Activities run **in parallel**
- Flow continues only after the **last** event notification arrives
- Enables parallel processing with synchronization points

---

## The Role of the Broker

- **Intermediates** communication between microservices (no direct contact)
- **Routes messages** to the right subscribers
- Message Broker pattern does not inherently include transactional models — that depends on the implementation (e.g., Kafka, RabbitMQ)

### Eventual consistency

- EDA promotes **eventual consistency**
- No **atomic distributed transactions** across microservices
- Each component manages its **own internal transaction** and notifies others via events
- When errors occur, **compensation events** can be generated to roll back or correct state

### Message Broker vs Event Streaming

| Aspect             | Message Broker                          | Event Streaming                                                  |
| ------------------ | --------------------------------------- | ---------------------------------------------------------------- |
| **History**        | Messages consumed and typically removed | Can revisit past messages in order (stream history)              |
| **Replay**         | Not supported                           | Replay sequences to reconstruct behavior                         |
| **Event Sourcing** | Limited                                 | Aggregate state from events; rebuild from snapshots + change log |
| **Consumers**      | Typically one consumer per message      | Multiple consumers can read the same stream in parallel          |

Event streaming extends beyond Request-Response and supports **Publisher-Subscriber** (0 to N subscribers) with **anonymity** and **asynchrony**.

---

## Combining EDA with Synchronous HTTP APIs

- A **REST API** can sit in front of an EDA ecosystem
- The API receives requests and **triggers events** into the event stream
- Asynchronous microservices subscribed to those events remain **anonymous** from the API’s perspective
- **Hybrid approach:** Use REST for entry points; use events for internal, async processing

### Event-first design

- Events carry **past occurrences** as a consequence of executed actions
- In event-driven design, **events should be imagined first** during modeling
- The rest of the system is built **around** those events
- Think: behaviors → stimuli → derived actions → implementation

### When to Choose EDA vs Synchronous Architecture

There is no universal rule — the choice depends on the use case. Below is a decision guide.

#### Prefer EDA (Event-Driven) when:

| Scenario                                    | Why EDA fits                                                                                                                                                                     |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Multiple consumers of the same event**    | One event triggers several independent actions (e.g., `OrderCreated` → inventory, notifications, analytics). Synchronous would require the caller to know and call each service. |
| **Fire-and-forget / background processing** | The caller does not need an immediate response (e.g., send email, update analytics, sync to warehouse). EDA avoids blocking and timeouts.                                        |
| **High throughput / burst traffic**         | Events can be buffered; consumers process at their own pace. Synchronous calls under load lead to cascading failures and timeouts.                                               |
| **Loose coupling**                          | Publishers and subscribers are independent. Adding a new consumer does not change the producer. Synchronous creates direct dependencies.                                         |
| **Resilience to downstream failures**       | If a consumer is down, events wait in the broker. With sync, a failing service can block or fail the entire flow.                                                                |
| **Audit trail / replay**                    | Event streams keep history; you can replay, debug, or rebuild state. REST calls are not stored by default.                                                                       |
| **Event sourcing / CQRS**                   | The system is built around an event log. Synchronous request-response does not match this model.                                                                                 |
| **Cross-domain / cross-team**               | Different teams own different services; events define clear boundaries. Sync often leads to tight coupling and shared contracts.                                                 |

#### Prefer Synchronous (REST/gRPC) when:

| Scenario                        | Why sync fits                                                                                        |
| ------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **Immediate response required** | User or client needs the result now (e.g., "create account and return session", "validate payment"). |
| **Simple request-response**     | Single call, single response, no fan-out. Sync is simpler and easier to debug.                       |
| **Strong consistency needed**   | Operation must be atomic or read-after-write consistent. EDA implies eventual consistency.           |
| **Query / read operations**     | Fetching data (GET) is typically sync: "give me this resource."                                      |
| **Low complexity**              | Small system, few services. EDA adds broker, retries, idempotency, etc.                              |
| **Strict SLAs / latency**       | Need predictable, low latency. Async adds variable delay.                                            |

#### Hybrid approach

Many systems use both:

- **Sync at the edge:** REST API for user-facing requests; returns quickly after minimal processing.
- **Async internally:** Publishes events for everything else (notifications, analytics, other services).

**Rule of thumb:** If the caller must wait for the result to continue, use sync. If the action can be acknowledged and processed later, use EDA.

---

## Event-Driven Design (~1.5h)

Core patterns for building reliable event-driven systems.

### Idempotency with Natural Key

**Problem:** In at-least-once delivery, the same event can be processed multiple times (e.g., after retries or consumer restarts). Processing it twice can cause duplicates (e.g., double charge, duplicate order).

**Solution:** Make consumers **idempotent** — processing the same event multiple times must produce the same result as processing it once.

**Natural key approach:**

- Use a **business identifier** (natural key) that uniquely identifies the operation: `orderId`, `transactionId`, `userId + eventType + timestamp`, etc.
- Before applying the side effect, check if that key was already processed (e.g., in DB or cache)
- If already processed → skip or return success (idempotent response)
- If not → process and store the key as "processed"

**Example:** For `OrderCreated`, use `orderId` as the natural key. On processing: `INSERT INTO orders ... ON CONFLICT (order_id) DO NOTHING` or `SELECT ... WHERE order_id = ?` before insert.

---

### Retry with Exponential Backoff

**Problem:** Transient failures (network glitch, temporary unavailability) often succeed on retry. Immediate retries can overload the system and worsen the situation.

**Solution:** **Exponential backoff** — wait longer between each retry attempt.

| Attempt | Wait time (example) |
| ------- | ------------------- |
| 1       | 1 s                 |
| 2       | 2 s                 |
| 3       | 4 s                 |
| 4       | 8 s                 |
| 5       | 16 s (or max cap)   |

**Common formula:** `delay = baseDelay * 2^attempt` (with jitter and max cap)

**Best practices:**

- Add **jitter** (random variation) to avoid thundering herd
- Define **max retries** and **max delay**
- After max retries → send to DLQ or alert
- Distinguish **retriable** (5xx, timeout) from **non-retriable** (4xx validation) errors

---

### Outbox Pattern

**Problem:** You need to **update the database** and **publish an event** atomically. If you publish after the DB commit and the process crashes in between, the event is lost. If you publish before the commit and the commit fails, you have an inconsistent event.

**Solution:** **Transactional Outbox** — write the event to an **outbox table** in the **same transaction** as the business data.

```
1. BEGIN TRANSACTION
2.   UPDATE orders SET status = 'shipped' WHERE id = ?
3.   INSERT INTO outbox (aggregate_id, event_type, payload, created_at) VALUES (?, 'OrderShipped', ?, NOW())
4. COMMIT
```

A separate process (e.g., poller or CDC) reads from the outbox and publishes to the message broker, then marks the row as published or deletes it.

**Benefits:**

- Atomicity: DB and event are consistent
- No lost events if the process crashes before publish
- Events are published only after successful commit

---

### Saga Pattern

**Problem:** A business process spans **multiple services** (e.g., Order → Inventory → Payment → Shipping). Each has its own DB. You cannot use a single distributed transaction.

**Solution:** **Saga** — break the process into **local transactions** per service. If one fails, run **compensating transactions** to undo previous steps.

**Choreography vs Orchestration:**

| Style             | Saga coordination                                                              |
| ----------------- | ------------------------------------------------------------------------------ |
| **Choreography**  | Each service reacts to events and emits its own events; no central coordinator |
| **Orchestration** | Orchestrator tells each service what to do and handles failures                |

**Example (Order Saga):**

1. OrderService: create order → `OrderCreated`
2. InventoryService: reserve stock → `StockReserved` or `StockReservationFailed`
3. PaymentService: charge → `PaymentCompleted` or `PaymentFailed`
4. On failure: compensating events (e.g., `ReleaseStock`, `RefundPayment`)

**Trade-offs:** Sagas can leave the system in an intermediate state during failures. Compensation logic must be designed and tested carefully.

---

### Eventual Consistency

**Definition:** Data across services is not guaranteed to be consistent at every moment. After a finite time, if no new updates occur, all replicas converge to the same state.

**In EDA:** Each service updates its own data and publishes events. Other services update asynchronously when they receive those events. There is a **time window** where data is inconsistent.

**When to choose eventual consistency?**

| Choose eventual consistency when                                                                              | Prefer strong consistency when                                                                                        |
| ------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **High availability** is required — system must stay up even if some nodes are unavailable                    | **Correctness** is critical and must be immediate (e.g., financial balance, seat reservation)                         |
| **Throughput** and **scale** matter — distributed transactions (2PC) are expensive and limit scalability      | The operation is **local** and single-service (e.g., single DB transaction)                                           |
| **Loose coupling** is a goal — services can evolve independently                                              | The business requires **instant** read-after-write consistency (e.g., "show my updated balance" right after transfer) |
| **Temporary inconsistency** is acceptable — e.g., "order confirmed" vs "inventory updated" can lag by seconds | **Regulatory** or **audit** rules require immediate consistency                                                       |
| **Event-driven** or **async** flows are natural — e.g., notifications, analytics, inventory sync              | **User-facing** flows need immediate feedback (e.g., "payment succeeded" must be confirmed before redirect)           |

**Example:** E-commerce order flow — eventual consistency is acceptable: order is created, inventory is reserved, payment is processed. A short delay (seconds) before all systems show the same state is usually fine. The alternative (2PC across Order, Inventory, Payment) would be slower and more complex.

**Example:** Bank transfer — strong consistency is often required: the balance must be correct immediately; you cannot show "transfer in progress" for a critical debit.

---

## Challenges and Considerations

| Challenge                | Description                                                                         |
| ------------------------ | ----------------------------------------------------------------------------------- |
| **Eventual consistency** | No atomic distributed transactions; requires extra design and implementation effort |
| **Tracing & logging**    | Harder in distributed systems; need correlation IDs, distributed tracing            |
| **Error handling**       | Must define retries, DLQs, compensation, and failure recovery                       |
| **Infrastructure**       | Need a dedicated platform for the Broker (e.g., Apache Kafka, RabbitMQ)             |
| **Domain maturity**      | Requires clear understanding of events, behaviors, and domain boundaries            |

---

## Conclusion

Adopting EDA is a **journey**: start with simple use cases and deepen your understanding of the business process. Maturity grows from understanding how events flow, how to ensure observability, reliability, and dependencies. The transition tends to move from basic implementations toward a richer, more modular event-driven ecosystem.

---

## References

- [Original article (Portuguese)](https://medium.com/@marcelomg21/event-driven-architecture-eda-em-uma-arquitetura-de-micro-servi%C3%A7os-1981614cdd45)
- [Apache Kafka](https://kafka.apache.org/)
