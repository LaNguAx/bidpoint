# Architecture

## Shape

A **Spring Modulith monolith** (`core-platform`) plus three services. Not all-microservices — a service exists only when it has an independent consistency, scaling, or failure responsibility.

```
                        ┌──────────────────┐
   browser ──REST/SSE──▶│  core-platform   │  profiles · catalog · listings
                        │  (N replicas)    │  auctions · orders · payment
                        └────────┬─────────┘
                                 │ REST (fence/finalize)
                                 ▼
                        ┌──────────────────┐
                        │ bidding-service  │  bids · price · winner
                        └────────┬─────────┘
                                 │ Kafka facts
                                 ▼
                     ┌───────────────────────┐
                     │ notification-service  │  decides policy
                     └───────────┬───────────┘
                                 │ RabbitMQ jobs
                                 ▼
                     ┌───────────────────────┐
                     │ notification-worker   │  × N competing consumers
                     └───────────────────────┘
```

## The four deployables

| Deployable | Owns | Must never own |
| --- | --- | --- |
| **`core-platform`** | Five Modulith modules — `profiles`, `catalog`, `listings`, `auctions`, `orders` — plus `payment` behind a fake provider adapter. Serves REST and SSE from multiple replicas. | Bids, current price, current winner. |
| **`bidding-service`** | Bid acceptance, current price, current winner, per-auction concurrency control, idempotency, its own auction-state projection, the bid outbox. | Auction scheduling or lifecycle authority. |
| **`notification-service`** | Consumes Kafka facts, decides what deserves a notification, writes durable intent and a job outbox, publishes RabbitMQ work. | Calling providers. |
| **`notification-worker`** | Executes RabbitMQ jobs, calls providers, records attempts, classifies retries, handles DLQ and quarantine. **Runs at N replicas.** | Deciding which facts deserve notifications. |

`bidding-service` is separate because it's the concurrency hot spot with its own scaling and failure profile. The notification pair is split because *deciding* and *delivering* fail differently and need separate durable records.

## Core modules

| Module | Authority |
| --- | --- |
| `profiles` | Marketplace profile activation, keyed by the validated Keycloak subject. Keycloak owns credentials; this owns the marketplace identity. |
| `catalog` | Categories and classification rules. |
| `listings` | Listing drafts, publication, S3 object metadata. |
| `auctions` | Schedule, lifecycle, `closeAt`, `lifecycleVersion`, close/cancel coordination, and the authoritative `AuctionClosed` event. |
| `orders` | Post-auction trade records and buyer/seller history. |
| `payment` | Payment state, provider calls, webhooks, reconciliation, audit. Fake adapter until a provider is chosen. |

Modules share one PostgreSQL deployment but **not** write authority. Each owns its own tables. No cross-module writes, no cross-module joins.

## Lifecycles

**Listing:** `DRAFT → PUBLISHED`. Only the owner edits a draft. Publication requires valid content, classification, images, and auction configuration.

**Auction:** `SCHEDULED → OPEN → CLOSING → CLOSED → SETTLED`, with an authorized path to `CANCELLED`.

## Invariants

These are the point of the project. Code that violates one is wrong even if it passes.

**1. Bidding owns bids. `auctions` owns lifecycle.**
`bidding-service` is the only authority for bids, current price, and current winner. `auctions` is the only authority for when an auction opens and closes.

**2. Only `auctions` can trigger an order.**
The close sequence is:

```
auctions commits CLOSING
  → idempotent REST fence/finalize call to bidding-service
  → on acknowledgement, auctions atomically commits CLOSED + one AuctionClosed event
  → orders consumes AuctionClosed and creates at most one trade
```

The REST acknowledgement is **not** the order trigger. A bidding-owned outcome is audit and projection input only — it can never create an order. Cancellation commits `CANCELLED` with no `AuctionClosed`.

**3. Idempotency is mandatory on bids.**
Every bid carries a client-supplied stable key. Reusing the key for the same logical bid returns the recorded result — it never creates a second accepted bid. After every committed acceptance there is exactly one current price and at most one current winner.

**4. State and event commit together, or not at all.**
Producers write their state change and the outgoing event to their **own database in one transaction** (the transactional outbox). A separate relay publishes afterward. This is the only way to survive a crash between "bid accepted" and "event published."

**5. Delivery is at-least-once. Never claim exactly-once.**
Duplicates will happen. Every consumer deduplicates by stable ID. End-to-end exactly-once is not achievable here and is never claimed.

**6. Owners write only their own data.**
No cross-service joins, no shared business-domain models, no reaching into another service's tables. Current data comes through an API; lag-tolerant needs use a local projection.

**7. Lag is fine in derived state. Wrong is never fine in authoritative state.**
See [01 Thesis](01-thesis.md).

## Kafka and RabbitMQ are different tools

This distinction is one of the more valuable things in the project, and collapsing the two brokers into one would destroy it.

| | Kafka | RabbitMQ |
| --- | --- | --- |
| Carries | A **fact** — something that happened | A **work obligation** — something to do |
| Consumers | Many, independent, each reading everything | One of N competing workers per job |
| Retention | Durable, replayable log | Consumed and gone |
| Ordering | Per partition (per `auctionId`) | Not guaranteed |
| Reprocessing | Rewind and replay | Redeliver on failure |

The canonical chain:

```
bidding-service ──Kafka fact──▶ notification-service ──RabbitMQ job──▶ notification-worker × N
   "a bid was placed"              "that deserves an email"              "send this one email"
```

A Kafka fact is broadcast — search, analytics, and notifications can all consume the same `BidPlaced`. A RabbitMQ job is a single obligation that exactly one worker executes. Scaling workers scales throughput without duplicating sends.

## Key flows

**Bid.** Client sends a bid with an idempotency key → `bidding-service` validates against its local auction projection, trusted server time, minimum increment, and bidder rules → serializes or conflict-detects concurrent updates → commits the bid plus an outbox row in one transaction → relay publishes to Kafka. A client timeout after commit resolves with the same key. Duplicate publication is expected and deduplicated downstream.

**Close and order.** As in invariant 2 above. `auctions` retries the fence call with the same identity after an ambiguous timeout.

**Kafka facts.** Each producer commits state and a producer-owned event envelope to its outbox. Facts carry a stable `eventId`, type and version, aggregate ID, timestamp, producer, and correlation ID. Consumers deduplicate, classify retryable failures, and quarantine poison events. Auction facts partition by `auctionId`, giving per-auction ordering — there is no global order.

**Notifications.** Kafka fact → `notification-service` decides → durable intent + job outbox → relay → RabbitMQ → one of N workers executes. The worker uses a stable provider key, records every attempt, bounds retries, sends terminal failures to a DLQ, and marks ambiguous provider outcomes `UNKNOWN` — reconciling before any retry rather than assuming a local record proves what the provider did.

**Payments.** The `payment` module owns provider intent and audit. Provider requests use stable idempotency identities. Webhooks require signature verification and replay protection, then apply transitions idempotently. An ambiguous timeout is reconciled before retry. Card data never enters BidPoint.

**Realtime.** Kafka facts feed an ephemeral Redis-backed view; `core-platform` serves SSE from all of its replicas. Clients reconnect with the last event ID for bounded replay; on a gap, the server tells the client to refetch authoritative state over REST. **SSE is a display channel, never command authority.**

## Storage roles

- **PostgreSQL** — the only authoritative store. All owned state and all outboxes.
- **Redis** — acceleration and ephemeral coordination. Never authoritative; losing it must never lose data.
- **S3** — listing image objects. `listings` owns the metadata about them.
- **Kafka** — durable replayable facts.
- **RabbitMQ** — targeted work.

## Security

Keycloak owns credentials and issues tokens. **Every backend independently validates JWTs** as a resource server — there is no gateway doing it once on everyone's behalf, and no service trusts another's say-so. The owning service makes business and object-level authorization decisions ("is this your listing to edit?"), because only the owner knows.

Secrets never appear in Git, Terraform state, Helm values, events, jobs, logs, or diagnostics.
