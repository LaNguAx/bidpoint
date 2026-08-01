# RabbitMQ jobs and notifications

Status: Superseded by [design-system 2.0](../2.0/README.md)
Last validated: 2026-08-01

RabbitMQ carries explicit, targeted work requests for a competing worker pool. Kafka carries business facts; RabbitMQ does not replace it for durable fan-out or become an unstructured queue of domain events. This is a target delivery design, not an implemented integration.

## The notification boundary

`notification-service` owns notification policy: it consumes Kafka business facts, decides whether a recipient/channel requires a notification, records the durable intent, and writes an explicit job request to its PostgreSQL job outbox in the same local transaction. It never calls a delivery provider directly.

`notification-worker` owns execution: it consumes RabbitMQ jobs as a competing consumer, invokes the selected delivery provider, records each attempt, applies idempotency and retry classification, reconciles ambiguous attempts, and routes terminal or unknown failures to DLQ/quarantine. It does not decide whether a business fact deserves a notification. The delivery provider is **Open**; selection requires stable-key idempotency plus a reconciliation/status lookup by that key, and credentials are remote secrets and never committed.

| Layer | Owns | Does not own |
| --- | --- | --- |
| Kafka producer | Committed business fact | Delivery policy or provider attempt |
| `notification-service` | Policy, recipient/channel intent, stable job ID, PostgreSQL job outbox | Provider invocation |
| Outbox relay | Recoverable publication of a committed work request | Policy decision or provider result |
| RabbitMQ | Targeted competing-consumer delivery | Business authority or durable user history |
| `notification-worker` | Attempt/audit records, provider idempotency, retry/DLQ outcome | Fact interpretation and notification policy |

## Canonical learning flow

Core `profiles` publishes producer-owned `UserRegistered` only after verified Keycloak identity evidence has activated the marketplace profile idempotently. It is the application fact for marketplace profile activation, not the Keycloak credential-creation event. `notification-service` consumes the fact under stable-ID deduplication, decides a welcome email is required, records the notification intent and a `SendWelcomeEmail` work request in its PostgreSQL job outbox atomically. A relay publishes the explicit work request to RabbitMQ. `notification-worker` receives the job, invokes the selected provider with a stable delivery/idempotency key, and persists attempt/result evidence.

```mermaid
sequenceDiagram
    participant K as "Kafka"
    participant N as "notification-service"
    participant DB as "Intent + job outbox"
    participant R as "Relay"
    participant Q as "RabbitMQ"
    participant W as "notification-worker"
    participant P as "Delivery provider"
    K->>N: UserRegistered (at least once)
    N->>DB: Deduplicate; commit intent + SendWelcomeEmail job
    R->>Q: Publish explicit work request
    Q->>W: Deliver job (at least once)
    W->>P: Idempotent delivery request
    W->>W: Record terminal, retryable, or UNKNOWN result
```

The job has a stable job ID, notification/intent ID, correlation and causation IDs, recipient/channel reference, template/data version as applicable, and an idempotency key. It carries no password, token, card data, or secret. A worker redelivery checks durable evidence before another provider call. The selected provider must accept the stable idempotency key and support reconciliation/status lookup by it; a provider without both capabilities is not selectable.

## Failure, retry, and operator contract

Notification processing is at least once, never an end-to-end exactly-once promise. Duplicates are anticipated in the Kafka fact, outbox relay, RabbitMQ delivery, and worker/provider boundary. Stable IDs plus durable inbox/deduplication make repeated local processing idempotent, while the provider's stable-key contract resolves repeated calls to its recorded result. BidPoint does not claim that its local attempt record alone can prove duplicate-free external delivery.

Only classified transient failures retry. Retries use bounded exponential backoff with jitter and retain attempt count, last error category, correlation data, and next-attempt time. If a provider may have accepted the request but a timeout or worker crash occurs before a durable result is recorded, the attempt becomes `UNKNOWN` and is quarantined without automatic replay. Reconciliation queries the provider by the same stable key; it records the returned delivery outcome, or permits a controlled retry under that same key only after the provider establishes that no delivery was accepted. If the provider cannot establish an outcome, the job stays quarantined for explicit operator resolution. Invalid job data, policy/configuration faults, an unsupported template, or exhausted retry budget become terminal. Terminal jobs reach a DLQ/quarantine route instead of cycling indefinitely. Operators can inspect the immutable intent, attempts, underlying fact/job IDs, provider reconciliation evidence, and error classification.

An unavailable provider does not invalidate the originating registration, bid, payment, or order. The intent remains observable as pending or failed where the product exposes it. This preserves the distinction between authoritative marketplace state and an eventually consistent delivery experience.

## Relationship to other asynchronous work

RabbitMQ's competing consumer model is appropriate when one delivery obligation should be completed by one worker from a scalable pool. It is not the default mechanism for bid facts, auction outcomes, payment facts, realtime state, or search indexing; those are durable/replayable Kafka facts with independent consumers. The `notification-service` job outbox repeats the transactional-outbox discipline: it records the policy result and work request atomically, then lets a relay publish later.

**Excluded:** Spring Cloud Stream as a baseline abstraction and direct provider calls from `notification-service`. Notification views and delivery states may lag; they must never cause a second payment, invalid bid acceptance, or a loss of the authoritative marketplace record.
