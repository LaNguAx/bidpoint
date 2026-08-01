# BidPoint standalone handoff (2.0)

Status: Canonical
Last validated: 2026-08-01
Supersedes: [1.0/99 New-chat handoff](../1.0/99-new-chat-handoff.md)

Self-contained entry point for a new developer or AI session. Requires no prior conversation history.

## Project identity and present state

**BidPoint — A Distributed Systems Reference Platform** is a deliberately bounded, real-time auction marketplace used to practise production-grade backend engineering and distributed-systems behavior. The domain is credible enough to create concurrency, ownership, ordering, idempotency, partial-failure, and recovery problems, and small enough that marketplace feature breadth does not hide them.

It is built for two stated reasons: **employability** as a backend engineer in Spring/Java ecosystems, and **genuine understanding** of how distributed systems behave and fail. Where those goals conflict, the conflict is recorded in [04 Decision delta](04-decision-delta.md) rather than resolved silently.

**Current repository reality:** documentation-only. No application code, Maven reactor, dependencies, manifests, container images, Kubernetes cluster, CI workflow, cloud infrastructure, commands, or running services exist. Every component below is target design.

## Where the design lives

`project-discovery/design-system/` holds two versioned baselines:

- **[1.0/](../1.0/README.md)** — the original discovery corpus, 24 documents. **Superseded**, kept intact as history and rationale. Its domain, ownership, and messaging documents remain the canonical contract and are inherited by 2.0.
- **[2.0/](README.md)** — the current baseline. A lean revision: six documents changing technology periphery, pin policy, and delivery sequence.

**Where 2.0 and 1.0 conflict, 2.0 wins.** 2.0 changes nothing about the architecture.

Inherited unchanged from 1.0 and still authoritative: [02 product domain](../1.0/02-product-domain-and-capabilities.md), [03 architecture overview](../1.0/03-architecture-overview.md), [04 core platform](../1.0/04-core-platform-modular-monolith.md), [05 microservice boundaries](../1.0/05-microservice-boundaries.md), [06 data ownership](../1.0/06-data-ownership-and-consistency.md), [07 REST/SSE](../1.0/07-rest-sse-and-service-communication.md), [08 Kafka/outbox](../1.0/08-kafka-events-and-transactional-outbox.md), [09 RabbitMQ](../1.0/09-rabbitmq-jobs-and-notifications.md), [10 security](../1.0/10-security-and-identity.md), [15 testing](../1.0/15-testing-and-quality.md), [23 primary sources](../1.0/23-primary-sources.md).

## Product scope and invariants

Identity and marketplace-profile onboarding, classified listing drafts and images, publication, auction scheduling/open/close/cancel/settle, concurrent bidding, live updates, post-auction orders, provider-backed payments, notification delivery, durable history. Initial discovery is ordinary browse and filter; full-text search is staged.

- Keycloak owns credentials. Core `profiles` owns marketplace profile activation keyed by the validated Keycloak subject. `UserRegistered` means idempotent profile activation, not credential creation.
- A listing moves `DRAFT` to `PUBLISHED`; only its owner edits the draft; publication requires valid content, classification, images, and auction configuration.
- An auction follows `SCHEDULED -> OPEN -> CLOSING -> CLOSED -> SETTLED`, with an authorized path to `CANCELLED`. Core `auctions` owns timing, `closeAt`, and monotonic `lifecycleVersion`.
- `bidding-service` is the only authority for bids, current price, and current winner, using trusted server/database time, minimum-increment and bidder rules, idempotency, and measured concurrency control.
- Reusing an idempotency key for the same logical bid returns the recorded result and cannot create another accepted bid. After every committed acceptance there is one authoritative price and at most one current winner.
- Close/cancel moves Core to `CLOSING`, then issues an idempotent Bidding REST fence/finalize command. Its acknowledgement is **not** an order trigger; Core retries the same identity after ambiguity.
- After acknowledgement, `auctions` atomically commits `CLOSED` and one producer-owned `AuctionClosed` via durable Spring Modulith event publication. `orders` consumes only after commit and deduplicates by event/auction identity. A bidding-owned outcome is audit and projection input only.
- `payment-service` owns payment state, provider calls, verified webhooks, reconciliation, and audit. Card data is outside BidPoint.
- Search, notification views, and SSE may lag and must expose recovery behavior. Invalid bid acceptance and duplicate charging may not be excused by eventual consistency.

## Topology and ownership

Spring Modulith `core-platform` modular monolith surrounded by focused services. Deliberately not all-microservices.

| Core module | Authority |
| --- | --- |
| `profiles` | Marketplace profile activation and data, separate from Keycloak identity. |
| `catalog` | Categories and classification rules. |
| `listings` | Listing drafts/publication and S3 object metadata/lifecycle. |
| `auctions` | Schedule, lifecycle, `closeAt`, lifecycle version, close/cancel coordination, authoritative `AuctionClosed`; never authoritative bids. |
| `orders` | Idempotent post-auction trade records and history; never payment charging. |

| Deployable | Owns | Does not own |
| --- | --- | --- |
| `api-gateway` | Spring Cloud Gateway WebFlux; public routing, CORS, correlation IDs, trace propagation, early JWT rejection, Redis-backed rate limits. | Identity storage, final business authorization, domain data. |
| `bidding-service` | Bid acceptance/state, current price/winner, per-auction concurrency and idempotency, auction projection, fence/finalize, bid outbox. | Auction scheduling or lifecycle authority. |
| `payment-service` | Payment state, provider integration, verified webhooks, reconciliation, retries, audit, payment outbox. | Orders or card data. |
| `realtime-service` | Kafka-derived live view, Redis fan-out and bounded replay, SSE from every replica, reconnect signalling. | Auction or bid authority. |
| `notification-service` | Kafka-fact consumption, notification policy, durable intent, job outbox, RabbitMQ work publication. | Provider calls. |
| `notification-worker` | RabbitMQ job execution, provider calls, attempt audit, stable-key idempotency, retry classification, reconciliation, DLQ/quarantine. | Deciding which facts merit notifications. |
| `search-service` | **Staged:** OpenSearch indexing and full-text REST query. | Authoritative listing data or initial browse/filter. |

The name is always `search-service`, never `search-indexer`. PostgreSQL is authoritative; Redis is acceleration and ephemeral coordination; S3 stores listing objects while `listings` owns their metadata; Kafka carries durable replayable facts; RabbitMQ carries targeted work; staged OpenSearch is derived only.

## Key flows

**Front door.** Local: `browser -> Traefik -> api-gateway -> Kubernetes Service -> backend`. AWS: `Route 53 -> WAF -> ALB -> api-gateway -> Kubernetes Service -> backend`. Keycloak uses a distinct authentication host and is not routed through Spring Cloud Gateway. Kubernetes Gateway API is configuration, not a runtime gateway product; AWS API Gateway is excluded. The gateway rejects invalid tokens early, but every backend independently validates JWTs and the owner makes business and object-level authorization decisions.

**Bid.** `client -> api-gateway -> bidding-service -> authoritative transaction + outbox -> Kafka`. Client supplies a stable idempotency key. Bidding validates its owner-local auction projection, trusted time, amount, and bidder rules; serializes or conflict-detects; commits result plus outbox row; publishes later. A timeout after commit resolves with the same key. Duplicate publication is expected and deduplicated.

**Close and order.** `auctions` commits `CLOSING`, calls the idempotent fence/finalize command, retries after timeout, then atomically commits `CLOSED` plus one `AuctionClosed`. `orders` creates at most one trade using event/auction deduplication. Cancellation commits `CANCELLED` without `AuctionClosed`.

**Kafka facts.** Each owner commits state and a producer-owned envelope to its local outbox; a relay publishes at least once. Facts carry stable `eventId`, type/version, aggregate ID, time, producer, correlation/causation, and minimal payload. Consumers deduplicate, classify retryable failures, quarantine poison events, and never edit producer-owned data. Auction facts partition by `auctionId`; there is no global order.

**Notifications.** `Kafka fact -> notification-service decision -> durable intent + job outbox -> relay -> RabbitMQ -> notification-worker -> provider`. The worker uses a stable provider key, records attempts, bounds retries, sends terminal failures to DLQ, marks ambiguous outcomes `UNKNOWN`, and reconciles before any controlled retry.

**Payments.** `payment-service` owns provider intent and audit. Requests use stable idempotency identities. Webhooks require signature verification and replay protection, apply transitions idempotently, and commit facts through an outbox. Ambiguous timeouts reconcile before retry.

**Realtime.** Owner outboxes publish Kafka facts; `realtime-service` maintains an ephemeral Redis-backed view and serves SSE from all replicas. Clients reconnect with the last event ID for bounded replay; on a gap the service signals a REST refetch. SSE is a display channel, never command authority.

## Delivery plan — two phases

**Phase A (correctness)** runs on Testcontainers plus a narrow Docker Compose aid. No cluster.

A1 repository fitness (Maven reactor, boundaries, quality gates) → A2 core domain and identity → **A3 bidding concurrency and idempotency, the thesis** → A4 Kafka facts, outbox, replay → A5 realtime and SSE → A6 RabbitMQ notifications → A7 orders and payment → A8 observability and load.

**Phase B (platform)** adds no domain behavior.

B1 local Kubernetes (k3d/k3s, Traefik, operators, HPA) → B2 delivery (Jenkins, Jib, ECR digests, Argo CD) → B3 AWS parity (Terraform, EKS, RDS, managed services) → B4 resilience (OpenSearch, KEDA, Istio ambient, canary, chaos).

Phase A is independently complete and demonstrable. Full acceptance evidence per stage is in [03 Delivery roadmap](03-delivery-roadmap.md).

## Stack summary

Maven is the sole build authority — **there is no Nx**. Temurin LTS, Spring Boot/Cloud/Modulith pinned one minor behind current, Spring Cloud Gateway (the only WebFlux component), Spring Security resource server, Spring Data JPA, Flyway, Spring for Kafka, Spring AMQP, Actuator, Micrometer, OpenTelemetry, AWS SDK v2, Jib. PostgreSQL, Redis, Kafka, RabbitMQ, Keycloak, S3Mock. JUnit, Mockito, AssertJ, Testcontainers, WireMock, Awaitility, Toxiproxy, ArchUnit, JaCoCo, Spotless, k6.

Phase B adds k3d/k3s, Traefik, Gateway API, Helm, Strimzi, RabbitMQ Cluster Operator, metrics-server, Jenkins, Argo CD, Terraform, EKS, RDS, ECR, S3, Amazon MQ, Redis Cloud, Secrets Manager, ALB. Telemetry is one OTel → Prometheus/Loki/Tempo/Grafana stack in **both** environments; ADOT collects on AWS and CloudWatch Logs is retained. **X-Ray is excluded**; AMP and Managed Grafana are optional comparisons.

Pin policy: behavioral components at current stable, frameworks one minor behind, tooling floating within a family, AWS managed families verified at implementation. Never infer a version from a general project page; never use `latest`. Exact 1.0 pins in [1.0/19](../1.0/19-versions-and-compatibility.md) are a snapshot to re-validate, not a contract.

AWS infrastructure is created and destroyed per session and never left running.

## Repository and package rules

Maven reactor; Java root package `com.bidpoint`. Top-level: `apps/backend/` for deployables with `core-platform/` intentionally nested as one deployable and local-transaction boundary, `libs/java/` for cross-cutting mechanics only, `platform/`, `deploy/` for GitOps composition, plus `tests/`, `tools/`, `docs/`.

Each Core module is one Maven JAR with public `api/`, separately named internal `api/events/`, implementation `internal/`, and producer-owned external `contracts/events/`. No separate API/implementation artifacts. Services use familiar Spring MVC-style organization and split further only when measured complexity justifies it.

External REST and event contracts are producer-owned. Shared libraries may hold cross-cutting mechanics only — never business-domain models, entities, repositories, tables, or decision services. Jib is the routine image mechanism; per-service Dockerfiles are excluded.

`project-discovery/structure-template/` is a **brainstorm, not a target**. It sketches what a layout could look like; it is not a commitment, and the implementation does not have to match it. Where it disagrees with what the code actually needs, the code wins.

It contains **634 directories and 788 zero-byte placeholder files**, all empty, with 634 `.gitkeep` files. Empty names like `package.json`, `nx.json`, `pom.xml`, and `project.json` are placeholders, not configuration. It reflects 1.0's Nx assumptions and predates ADR-019-R1, so parts of it are known-stale. Do not treat it as executable and do not alter it during discovery.

## Open questions

Not requests for an immediate answer; decide at the gates in [05](05-exclusions-and-open-questions.md).

1. Frontend framework and companion libraries; CSR is decided. A frontend selection also decides whether Node tooling returns at all.
2. Payment provider and its idempotency, webhook-signature, sandbox, and reconciliation contract.
3. Notification delivery provider and its stable-key idempotency and status-lookup contract.
4. Amazon MSK Express 4.2.x versus Provisioned 4.1.x, and local Kafka alignment.
5. Schema registry: Apicurio versus AWS Glue.
6. Public license — relevant sooner than 1.0 assumed, since a portfolio project shown to employers is a form of publication.
7. Timing and success criteria for optional comparison exercises.

*(1.0's Spring Boot / Spring Cloud AWS compatibility gate is closed — see ADR-028-R1.)*

## Repository paths and history

- repository root: `c:\Users\itaya\Desktop\bidpoint`
- discovery entry: `project-discovery/README.md`
- superseded baseline: `project-discovery/design-system/1.0/`
- current baseline: `project-discovery/design-system/2.0/`
- layout reference: `project-discovery/structure-template/`

Discovery history is four commits: workspace skeleton; product and architecture; platform stack and practices; decisions, sources, and handoff. The 2.0 revision follows them. Confirm the current Git log rather than assuming a SHA. No commit implements the target.

## Instructions for the next developer or AI session

- **Do not implement automatically.** Read this handoff, [04 Decision delta](04-decision-delta.md), and the inherited 1.0 documents for the stage in question; agree a bounded plan and acceptance evidence first.
- **Distinguish target design from existing files.** Only documentation and empty placeholders exist.
- **Acceptance evidence is the deliverable.** A stage is not complete when the feature runs — it is complete when the concurrency proof, crash-window test, measurement, or reconciliation trace exists.
- **Every component must map to a learning outcome in [01](01-project-thesis.md).** One that cannot is removed, not staged. This rule removed Nx and X-Ray.
- **Preserve accepted decisions unless evidence justifies change**, and state consequences first: affected owners, contracts and data, migration, compatibility, operations, rollback.
- **Verify volatile facts against official upstream and provider documentation.** Context7 is a discovery aid; publisher documentation is authoritative.
- **Name exact components rather than "the gateway".** Distinguish Spring Cloud Gateway `api-gateway`, Traefik, ALB, AWS Load Balancer Controller, Kubernetes Gateway API, and excluded AWS API Gateway.
- **Do not add design documents.** The corpus is closed per ADR-034; record changes as amendments to [04](04-decision-delta.md). Runbooks and notes written *from* evidence are outputs, not design.
- **Do not claim commands, manifests, dependencies, or infrastructure exist** because the structure template names them. Verify non-empty content first.
