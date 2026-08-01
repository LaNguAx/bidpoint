# BidPoint standalone new-chat handoff

Status: Superseded by [design-system 2.0](../2.0/README.md)
Last validated: 2026-08-01

This document is the self-contained entry point for a new developer or AI agent. It records the accepted target, the repository's actual state, the decisions that must be preserved, the facts that remain open, and the safe next-step protocol. It requires no prior conversation history.

## Project identity and present state

**BidPoint — A Distributed Systems Reference Platform** is a deliberately bounded, real-time auction marketplace used to practise production-grade backend engineering and distributed-systems behavior. Its business domain is credible enough to create concurrency, ownership, ordering, idempotency, partial-failure, and recovery problems, but intentionally small enough that marketplace feature breadth does not hide those lessons.

The intended audience is engineers implementing, testing, operating, and recovering a distributed backend, plus reviewers who need an explicit contract against which future implementation choices can be challenged. The principal learning objectives are:

- authoritative module/service ownership rather than shared tables;
- local ACID transactions combined with transactional outboxes and at-least-once delivery;
- safe concurrent bidding, per-auction ordering, idempotency, and hot-key/partition behavior;
- deliberate use of REST, SSE, Kafka, and RabbitMQ according to communication purpose;
- derived projections, visible lag, retries, DLQ/quarantine, replay, and failure recovery;
- OIDC/OAuth2 identity, independent resource-server validation, object authorization, and secret isolation;
- production delivery, scaling, telemetry, load, and failure-injection evidence.

**Current repository reality:** this is a documentation-only discovery repository. No application code, initialized Nx workspace, Maven reactor, dependencies, manifests, container images, Kubernetes cluster, CI workflow, cloud infrastructure, commands, or running services exist yet. Every component and command below describes target design unless explicitly described as an existing document or Git commit.

## Product scope and invariants

The bounded product supports identity and marketplace-profile onboarding, classified listing drafts and images, publication, auction scheduling/open/close/cancel/settle behavior, concurrent bidding, live updates, post-auction orders, provider-backed payments, notification delivery, and durable history. Initial discovery is ordinary browse/filter; full-text search is staged.

Core lifecycle and business invariants:

- Keycloak owns credentials and identity. Core `profiles` owns marketplace profile activation keyed by the validated Keycloak subject. `UserRegistered` means idempotent marketplace-profile activation, not Keycloak credential creation.
- A listing moves from `DRAFT` to `PUBLISHED`; only its owner edits the draft, and publication requires valid content, classification, images, and auction configuration.
- An auction follows `SCHEDULED -> OPEN -> CLOSING -> CLOSED -> SETTLED`, with an explicitly authorized cancellation path to `CANCELLED`. Core `auctions` owns timing, `closeAt`, and monotonic `lifecycleVersion`.
- `bidding-service` is the only authority for bids, current price, and current winner. It accepts only eligible bids using trusted server/database time, minimum-increment and bidder rules, idempotency, and measured concurrency control.
- Reusing an idempotency key for the same logical bid returns the recorded result and cannot create another accepted bid. After every committed acceptance there is one authoritative current price and at most one current winner.
- Close/cancel first moves Core to `CLOSING`, then uses an idempotent Bidding REST fence/finalize command. Its stable acknowledgement/final outcome is not an order trigger; Core retries the same identity after ambiguity.
- After close acknowledgement, `auctions` atomically commits `CLOSED` and one producer-owned `AuctionClosed` through durable Spring Modulith event publication. It carries winner/no-winner, final price, lifecycle version, and stable identity. `orders` consumes only after commit and deduplicates by event/auction identity; a bidding-owned outcome is audit/projection input only.
- `payment-service` owns payment state, provider calls, verified webhooks, reconciliation, and audit. Stable provider/payment identities prevent double charging; card data is outside BidPoint.
- Search, notification views, and SSE may lag and must expose recovery behavior. Invalid bid acceptance and duplicate charging may not be excused by eventual consistency.

See [project thesis](01-project-thesis.md) and [product domain and capabilities](02-product-domain-and-capabilities.md) for the complete product contract.

## Canonical topology and precise ownership

The architecture is a Spring Modulith `core-platform` modular monolith surrounded by focused services. It is deliberately not an all-microservices design.

### Core modules

`core-platform` contains exactly five modules:

| Module | Authority |
| --- | --- |
| `profiles` | Marketplace profile activation and profile data, separate from Keycloak identity. |
| `catalog` | Categories and classification rules. |
| `listings` | Listing drafts/publication and S3 object metadata/lifecycle. |
| `auctions` | Auction schedule, lifecycle, `closeAt`, lifecycle version, close/cancel coordination, and authoritative `AuctionClosed` order trigger; never authoritative bids. |
| `orders` | Idempotent post-auction trade records and buyer/seller history; never payment charging. |

### Surrounding deployables

| Deployable | Exact ownership | Explicit non-ownership |
| --- | --- | --- |
| `api-gateway` | Spring Cloud Gateway Server WebFlux; public application routing, CORS, request/correlation IDs, trace propagation, early JWT rejection, Redis-backed rate limits. | Identity storage, final business authorization, or domain data. |
| `bidding-service` | Bid acceptance/state, current price/winner, per-auction concurrency and idempotency, auction-state projection, fence/finalize serialization, bid outbox. | Auction scheduling/lifecycle authority. |
| `payment-service` | Payment state, provider integration, verified webhooks, reconciliation, retries, audit, payment outbox. | Orders or card-data storage. |
| `realtime-service` | Kafka-derived live view, Redis fan-out/bounded replay, SSE from every replica, reconnect/refetch signaling. | Auction or bid authority. |
| `notification-service` | Kafka-fact consumption, notification policy, durable intent, PostgreSQL job outbox, explicit RabbitMQ work publication. | Provider calls. |
| `notification-worker` | RabbitMQ job execution, provider calls, attempt audit, stable-key idempotency, retry classification, reconciliation, DLQ/quarantine. | Deciding which facts merit notifications. |
| `search-service` | **Staged:** OpenSearch indexing and full-text REST query. | Authoritative listing data or initial browse/filter. |

The name is always `search-service`, never `search-indexer`. PostgreSQL is authoritative storage; Redis is acceleration/ephemeral coordination; S3 stores listing objects while `listings` owns their marketplace metadata; Kafka carries durable replayable facts; RabbitMQ carries targeted work; staged OpenSearch is derived only.

The full boundary maps are in [architecture overview](03-architecture-overview.md), [Core platform](04-core-platform-modular-monolith.md), and [microservice boundaries](05-microservice-boundaries.md).

## End-to-end flows

### Front door and request flow

Local public traffic is `browser -> pinned external Traefik -> api-gateway (Spring Cloud Gateway) -> Kubernetes Service -> owning backend`. AWS traffic is `Route 53 -> AWS WAF -> ALB via AWS Load Balancer Controller -> api-gateway -> Kubernetes Service -> owning backend`. Keycloak uses a distinct authentication host/path and is not routed through Spring Cloud Gateway. Kubernetes Gateway API is configuration, not a runtime gateway product; AWS API Gateway is excluded.

The gateway rejects invalid tokens early, but every backend independently validates JWTs as a resource server. The target service/module makes business and object-level authorization decisions.

### Bid flow

`client -> api-gateway -> bidding-service -> authoritative bid transaction + outbox -> Kafka`. A client supplies a stable idempotency key. Bidding validates its owner-local auction projection, trusted time, amount, and bidder rules, serializes or conflict-detects concurrent updates, commits the result plus outbox row, and later publishes. A timeout after commit is resolved with the same key. Relay failure leaves a durable row; duplicate Kafka publication is expected and deduplicated.

### Auction close and order flow

Core `auctions` commits `CLOSING`, calls the idempotent Bidding fence/finalize REST command, and retries after timeout. Bidding's acknowledgement/final outcome is not the order trigger. After close acknowledgement, `auctions` atomically commits `CLOSED` and one producer-owned `AuctionClosed` through the durable Spring Modulith event-publication/outbox mechanism. `orders` consumes only after commit and creates at most one trade using event/auction deduplication. The same `AuctionClosed` may be externalized to Kafka with the same identity. A bidding-owned outcome is audit/projection input only and cannot create an order. Cancellation commits `CANCELLED` without `AuctionClosed`.

### Kafka fact flow

Each owner commits state and a producer-owned event envelope to its local transactional outbox. A relay publishes to Kafka at least once. Facts carry stable `eventId`, type/version, aggregate ID, time, producer, correlation/causation, trace context where appropriate, and minimal payload. Consumers use durable inbox/deduplication, classify retryable failures, quarantine poison events, preserve identity through replay, and never edit producer-owned data. Auction facts partition by `auctionId` for per-auction order; there is no global order.

### RabbitMQ notification flow

`Kafka business fact -> notification-service decision -> durable intent + local job outbox -> relay -> RabbitMQ job -> notification-worker -> provider`. Policy and delivery execution remain separate owners. The worker uses a stable provider key, records attempts, bounds retries, sends terminal failures to DLQ/quarantine, and marks ambiguous provider outcomes `UNKNOWN`. It reconciles before any controlled retry and never assumes a local record alone proves duplicate-free external delivery.

### Payment and webhook flow

An order/client action reaches `payment-service`, which owns the provider intent and audit. Provider requests use stable idempotency identities. Webhooks require signature verification and replay protection, then apply state transitions idempotently and commit payment facts through an outbox. An ambiguous provider timeout is reconciled before retry; the same logical payment cannot charge twice.

### Realtime and SSE flow

Owner outboxes publish Kafka facts; `realtime-service` consumes them, maintains an ephemeral Redis-backed view/fan-out, and serves SSE from all replicas. Clients reconnect with the last event ID for bounded replay. When the replay window has a gap, the service signals the client to refetch authoritative REST state. SSE is a display channel, never command authority.

### Jenkins-to-GitOps deployment flow

`GitHub webhook -> Jenkins controller -> ephemeral Kubernetes agent -> Maven/Nx validation -> Jib image -> ECR immutable digest -> narrow GitOps environment change -> review/merge on main -> Argo CD reconciliation -> Kubernetes/EKS`, with Argo Rollouts staged after ordinary deployment/blue-green evidence. Jenkins builds and publishes; it never directly deploys production. One digest is promoted through `deploy/dev`, `deploy/stage`, and `deploy/prod`; there are no environment branches or per-environment rebuilds.

Communication and reliability details are in [data ownership](06-data-ownership-and-consistency.md), [REST/SSE communication](07-rest-sse-and-service-communication.md), [Kafka/outbox](08-kafka-events-and-transactional-outbox.md), and [RabbitMQ/notifications](09-rabbitmq-jobs-and-notifications.md).

## Local and AWS platform targets

### Local environment

- **Canonical:** k3d 5.9.0 running pinned k3s `v1.35.6+k3s1` / Kubernetes 1.35.6, separately installed Traefik 3.7.9/chart 41.0.2, Gateway API 1.5.1, PostgreSQL, Redis, Kafka/Strimzi, RabbitMQ/operator, Keycloak, S3Mock, metrics-server/HPA, and basic telemetry.
- **Canonical target profile `core`:** base cluster, edge, all baseline deployables, data/brokers/identity/S3 double, HPA, and basic telemetry.
- **Canonical target profile `platform`:** everything in `core`, plus Jenkins, Argo CD, Argo Rollouts, KEDA, Istio ambient, and the full Prometheus/Loki/Tempo/Grafana stack.
- **Staged within the broader learning path:** KEDA, Istio ambient, progressive canary, advanced chaos, and full-text search/OpenSearch.
- **Excluded:** Docker Compose as the canonical complete environment. A narrowly scoped Compose aid is only an **Optional comparison/reconsideration trigger** after measured developer friction and cannot claim platform parity.

Future commands such as `pnpm nx local-up core|platform`, `local-down`, `local-status`, and confirmed-destructive `local-reset` are contracts only and do not exist.

### AWS target

- **Canonical:** Terraform-owned VPC/EKS managed-node-group foundation, Route 53/WAF/ALB, ECR, RDS PostgreSQL, S3, Amazon MQ RabbitMQ family, Redis Cloud, Secrets Manager/Pod Identity/CSI, and AWS telemetry destinations.
- **Canonical workload ownership:** Helm packages workloads/add-ons; Argo CD reconciles same-repository desired state; Jenkins proposes immutable-digest changes; none shares resource ownership with Terraform.
- **Canonical observability mapping:** local OTel/Prometheus/Loki/Tempo/Grafana maps to remote ADOT/AMP/CloudWatch Logs/X-Ray/Managed Grafana. Metrics, logs, and traces remain distinct.
- **Staged:** Amazon OpenSearch Service, KEDA, Istio ambient, Argo Rollouts canary, Karpenter comparison/adoption, and multi-region disaster recovery.
- **Open:** MSK Express 4.2.x versus Provisioned 4.1.x, registry choice, Cloud AWS compatibility, payment provider, notification delivery provider, frontend stack, and license.

See [local development platform](11-local-development-platform.md), [AWS production platform](12-aws-production-platform.md), and [observability](14-observability.md).

## Repository, package, and contract rules

The target is a polyglot Nx + Maven monorepo. Nx supplies graph, affected execution, caching, and task UX; Maven Wrapper/reactor remains the independent Java build, dependency, test, and reactor authority. pnpm manages Nx/Node tooling and future frontend packages. `@nx/maven` is experimental and guarded: each Nx Maven target must document and prove a direct Maven equivalent; upgrades require compatibility evidence; thin custom wrappers are the fallback.

Future top-level ownership is `apps/backend/` for deployables, intentionally nested `core-platform/` for its one deployable/local-transaction boundary, `libs/java/` for cross-cutting mechanics only, `platform/` for reusable platform definitions, `deploy/` for GitOps environment composition, plus `tests/`, `tools/`, and `docs/`. The Java root package is `com.bidpoint`.

Each Core module is one Maven JAR with public `api/`, separately named internal Spring Modulith `api/events/`, implementation `internal/`, and producer-owned external Kafka `contracts/events/`. There are no separate API/implementation artifacts. Ordinary services and modules use familiar Spring MVC-style organization and split further only when measured complexity justifies it; Spring Cloud Gateway uses WebFlux.

External REST/event contracts are producer-owned. Shared Java libraries may contain cross-cutting mechanics such as security, observability, web, messaging, reliable messaging, and test support. They must never contain business-domain models, entities, repositories, tables, or decision services. Jib is the routine JVM image mechanism; routine service Dockerfiles are excluded.

### Exact structure-template status

`project-discovery/structure-template/` is a proposed layout reference only. It contains exactly **634 directories and 788 zero-byte placeholder files**; all 788 files are empty, and **634 `.gitkeep` files** provide directory coverage. Empty names such as `package.json`, `nx.json`, `pnpm-workspace.yaml`, `pom.xml`, `project.json`, and manifest-looking paths are placeholders, not configuration. The template is not an initialized Nx, Maven, or Kubernetes workspace and must not be treated as executable or altered during discovery.

See [monorepo and repository structure](16-monorepo-and-repository-structure.md).

## Data, transactions, ordering, and recovery

- A module or service is the sole writer of its owned data. There are no cross-service joins or shared business-domain tables. Logical Core schemas may initially share one PostgreSQL deployment without sharing authority.
- Synchronous hard invariants commit in the owning local transaction. Cross-owner current data comes through APIs; lag-tolerant needs use owner-local projections.
- Producer state and publication intent commit atomically through a local outbox. Consumers use stable-ID inbox/deduplication. Delivery is at least once; BidPoint never claims end-to-end exactly once.
- Kafka ordering is per partition and auction facts partition by `auctionId`. Hot auctions create hot records and hot partitions; load tests must measure conflict, skew, latency, and lag before choosing mitigation that preserves per-auction order.
- Exact bid concurrency technique (optimistic, pessimistic, or atomic database operation) is a staged implementation choice selected from measurement and proven with invariant tests, not a discovery-level Open architectural question.
- Retries are bounded, exponential, jittered, and limited to classified transient failures. Poison messages and ambiguous external outcomes go to DLQ/quarantine with stable IDs, attempts, correlation, and operator evidence.
- Replay begins from an explicit scope/offset, preserves logical identity, rebuilds derived state, and never repeats a charge, email, or other externally visible effect.
- Recovery evidence distinguishes a rejected command, committed state awaiting publication, consumer lag, retryable outage, poison message, and unknown provider outcome.

## Security, testing, delivery, and observability principles

**Security:** Keycloak owns identity; each backend validates JWTs; owner code performs business/object authorization. Secrets use least privilege and are never in Git, Terraform inputs/state, Helm values, events/jobs, logs, or diagnostics. Payment webhooks require signatures and replay protection. Logs/events minimize and redact data. Staged Istio ambient identity/mTLS/policy adds defense in depth and never replaces application validation.

**Testing and quality:** prove domain invariants with unit/module tests, Spring Modulith/ArchUnit boundaries, Testcontainers PostgreSQL/Kafka/RabbitMQ/Redis behavior, WireMock provider contracts, Awaitility bounded convergence, Toxiproxy failures, and k6 concurrency/SSE/load evidence. Test transaction boundaries, outbox crash windows, duplicates, replay, fence retries, hot records/partitions, retry/DLQ, `UNKNOWN` provider outcomes, and authorization. Ordinary suites make no external calls; scoped AWS smoke tests are explicit, cost-controlled, and cleaned up. Coverage is diagnostic evidence, not a target to game.

**Delivery:** Jenkins uses JCasC and ephemeral agents; Maven remains build authority; Jib produces immutable images; ECR digests are promoted through reviewed Git changes; Argo CD alone reconciles workload desired state. Terraform, Helm, and Argo CD retain distinct scopes. Rollbacks use a previously recorded compatible digest/configuration; upgrades review compatibility, security, migration, licensing, and recovery.

**Observability:** instrumentation preserves correlation/causation across REST and asynchronous boundaries. RED, transaction/concurrency, outbox age, Kafka lag/skew, RabbitMQ depth/retries/DLQ, provider reconciliation, SSE connections/replay gaps, rollout, and infrastructure signals must make partial failure visible. Metrics, structured redacted logs, and traces are complementary, distinct signals. Dashboards are paired with alert/runbook and recovery evidence.

See [security and identity](10-security-and-identity.md), [CI/CD and GitOps](13-ci-cd-and-gitops.md), [testing and quality](15-testing-and-quality.md), and [delivery roadmap](20-delivery-and-learning-roadmap.md).

## Version baseline and compatibility policy

Exact direct pins at validation:

- build/runtime: Temurin 25.0.4 LTS; Node 24.18.0 LTS; pnpm 11.4.0; Maven 3.9.16; Nx and experimental `@nx/maven` 23.1.0;
- Spring: Boot 4.1.0; Cloud 2025.1.2; Gateway 5.0.2; Modulith 2.1.0; Cloud AWS 4.0.0 behind an Open gate; Jib 3.5.2;
- quality: ArchUnit 1.4.2; JaCoCo 0.8.15; Spotless 3.6.0; k6 1.7.1; Boot-managed/compatible policies for Spring integrations and test libraries;
- data/messaging: PostgreSQL 18.4; Redis 8.2.6; Kafka 4.2.1; Strimzi 1.1.0; RabbitMQ 4.2.9; RabbitMQ Cluster Operator 2.22.3; S3Mock 5.1.0; Keycloak 26.7.0;
- local platform: k3d 5.9.0; k3s `v1.35.6+k3s1`; Kubernetes 1.35.6; Traefik 3.7.9/chart 41.0.2; Gateway API 1.5.1; Helm 4.2.3;
- delivery/staged platform: Jenkins 2.568.1 LTS/chart 5.9.45; Argo CD 3.4.6; Rollouts 1.9.1; KEDA 2.20.1; Istio 1.30.3; Terraform 1.15.8;
- telemetry: OTel Collector Contrib 0.157.0; Prometheus 3.13.2; Loki 3.7.4; Tempo 3.0.2; Grafana 13.1.1; ADOT 0.48.0;
- AWS add-ons: AWS Load Balancer Controller 3.4.3; Secrets Store CSI Driver 1.6.0; AWS provider 3.1.0.

Spring Cloud 2025.1.2 explicitly supports Spring Boot 4.1.x. In contrast, Spring Cloud AWS 4.0.0 currently documents Boot 4.0.x/Framework 7.0.x compatibility, not Boot 4.1.x. Therefore **Boot 4.1.0 + Cloud AWS 4.0.0 is not a confirmed pairing**. Resolve it through upstream support/a release, or focused dependency-resolution and S3/BOM/SDK-v2 proof plus an explicit risk and rollback decision, or align Boot. Do not declare support by inference.

RDS PostgreSQL 18.4 and Amazon MQ RabbitMQ 4.2 are managed engine families. EKS, Redis Cloud, S3, ECR, Route 53, WAF, ALB, ACM, Secrets Manager, Pod Identity, AMP, CloudWatch Logs, X-Ray, Managed Grafana, CloudFront, and OpenSearch follow provider-managed-family policy: verify current region/account availability, compatible add-ons, quotas, security/release notes, and rollback evidence at the point of implementation. Never infer an exact compatible release from a general project page or use `latest`.

See [local stack](17-local-technology-stack.md), [remote stack](18-remote-technology-stack.md), [versions and compatibility](19-versions-and-compatibility.md), and [primary sources](23-primary-sources.md).

## Staged, excluded, comparison, and open work

### Staged capabilities

Full-text `search-service`/OpenSearch, Istio ambient identity/mTLS/policy/fault work, KEDA queue/lag scaling, Argo progressive canary, advanced chaos, fuller supply-chain controls, Karpenter evaluation, CloudFront/frontend delivery, and multi-region DR enter only after their baseline ownership and recovery evidence. Staged does not mean implemented or day one.

### Explicit baseline exclusions

AWS API Gateway; Eureka; Spring Cloud Kubernetes discovery server; initial Spring Cloud Stream; Valkey as canonical; initial Karpenter; Istio sidecars; MinIO/AIStor; LocalStack OSS; Promtail; ingress-nginx; H2 as integration evidence; legacy X-Ray SDK/daemon; direct Jenkins production deployment; OpenSearch day one; Docker Compose as the canonical full environment; RabbitMQ Streams; physical database per Modulith module on day one; shared cross-service business models; environment branches; Ansible; microservices solely for appearance; and frontend application-stack selection in current backend discovery. Routine per-service Dockerfiles, gateway-only authorization, unreviewed floating tags, and end-to-end exactly-once claims are also outside the baseline.

An exclusion is not an unresolved question. Its evidence-based reconsideration trigger is recorded in [exclusions and open questions](22-exclusions-and-open-questions.md).

### Optional later comparisons

Debezium CDC outbox versus application relay; gRPC versus REST for one internal path; GitHub Actions versus Jenkins; ElastiCache/Valkey versus Redis Cloud; Karpenter versus managed-node-group scaling; Apicurio versus AWS Glue schema-registry options; SSE versus WebSockets only if requirements change; and multi-region DR. A comparison preserves the canonical baseline until an explicit decision records consequences and migration/recovery evidence.

### Every unresolved choice

1. Frontend framework and companion state/data-fetching/testing/design-system libraries; CSR is already decided.
2. Payment provider and its stable-idempotency, webhook-signature, sandbox, and reconciliation contract.
3. Notification delivery provider and its stable-key idempotency plus reconciliation/status lookup contract.
4. Amazon MSK Express 4.2.x versus Provisioned 4.1.x and local Kafka alignment.
5. Schema registry: Apicurio locally/possibly remotely for parity versus AWS Glue Schema Registry.
6. Spring Boot 4.1.0/Spring Cloud AWS 4.0.0 compatibility resolution.
7. Public license; no license exists until selected.
8. Timing and measurable success criteria for optional comparison exercises.

These choices are not requests for an immediate answer. Decide them only at the gates and criteria in document 22.

## Repository paths and discovery history

Current paths:

- repository root: `C:\Users\Itay\Desktop\BidPoint`;
- discovery entry: `project-discovery/README.md`;
- complete knowledge-base index: `project-discovery/design-system/README.md`;
- standalone handoff: `project-discovery/design-system/99-new-chat-handoff.md`;
- immutable layout reference: `project-discovery/structure-template/`.

The discovery history is intentionally four commits, in order:

1. `chore: establish project discovery workspace` — repository and discovery skeleton established.
2. `docs: capture BidPoint product and system architecture` — product, ownership, and core distributed design captured.
3. `docs: document platform stack and engineering practices` — local/AWS platform, delivery, observability, testing, monorepo, version, and roadmap records captured.
4. `docs: add decisions sources and project handoff` — decision/exclusion/source registers, complete navigation, and this standalone handoff. At the time this file is authored, this is the required integration commit; confirm the current Git log rather than assuming a SHA.

No commit in this sequence implements the target. Do not push or mutate the structure template as part of discovery handoff work.

## Complete detailed-document map

1. [Project thesis](01-project-thesis.md)
2. [Product domain and capabilities](02-product-domain-and-capabilities.md)
3. [Architecture overview](03-architecture-overview.md)
4. [Core platform modular monolith](04-core-platform-modular-monolith.md)
5. [Microservice boundaries](05-microservice-boundaries.md)
6. [Data ownership and consistency](06-data-ownership-and-consistency.md)
7. [REST, SSE, and service communication](07-rest-sse-and-service-communication.md)
8. [Kafka events and transactional outbox](08-kafka-events-and-transactional-outbox.md)
9. [RabbitMQ jobs and notifications](09-rabbitmq-jobs-and-notifications.md)
10. [Security and identity](10-security-and-identity.md)
11. [Local development platform](11-local-development-platform.md)
12. [AWS production platform](12-aws-production-platform.md)
13. [CI/CD and GitOps](13-ci-cd-and-gitops.md)
14. [Observability](14-observability.md)
15. [Testing and quality](15-testing-and-quality.md)
16. [Monorepo and repository structure](16-monorepo-and-repository-structure.md)
17. [Local technology stack](17-local-technology-stack.md)
18. [Remote technology stack](18-remote-technology-stack.md)
19. [Versions and compatibility](19-versions-and-compatibility.md)
20. [Delivery and learning roadmap](20-delivery-and-learning-roadmap.md)
21. [Decision register](21-decision-register.md)
22. [Exclusions and open questions](22-exclusions-and-open-questions.md)
23. [Primary sources and validation record](23-primary-sources.md)

The [design-system index](README.md) supplies shorter reading paths and status interpretation.

## Instructions for the next developer or AI agent

- **Do not implement automatically.** First read this handoff, the relevant detailed documents, and the current repository/Git state; agree on a bounded implementation plan and acceptance evidence.
- **Distinguish target design from existing code/files.** The discovery describes future components; only documentation and empty placeholders currently exist.
- **Preserve accepted decisions unless evidence justifies change and state consequences before changing them.** Identify affected owners, contracts/data, migration, compatibility, operations, and rollback.
- **Verify volatile facts using Context7 and official primary sources.** Context7 is a discovery/validation aid; publisher/upstream/cloud-provider documentation is authoritative.
- **Prefer maintained Spring ecosystem tooling where compatible.** Do not hide critical delivery semantics behind an abstraction merely because a library exists.
- **Avoid unnecessary technology bloat.** Every service, library, datastore, controller, and tool needs an independent responsibility or learning outcome.
- **Name exact components rather than saying “the gateway”.** Distinguish Spring Cloud Gateway `api-gateway`, Traefik, ALB, AWS Load Balancer Controller, Kubernetes Gateway API, and excluded AWS API Gateway.
- **Ask only when a real architectural fork cannot be safely inferred.** Apply the canonical register and documented gates for routine details; surface genuine forks with criteria and consequences.
- **Do not claim commands, manifests, dependencies, or infrastructure exist merely because the template names them.** Verify non-empty content and executable configuration before describing anything as initialized or available.

Before implementation, refresh the volatile source/compatibility record, resolve only the Open gates required by the selected slice, inspect branch/worktree status, and turn the chosen roadmap stage into a contract-first plan with exact commands, ownership boundaries, failure evidence, acceptance criteria, and rollback. The [decision register](21-decision-register.md) is the baseline change-control index; the [primary-source record](23-primary-sources.md) defines acceptable evidence.
