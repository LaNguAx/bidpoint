# Testing and quality

Status: Canonical
Last validated: 2026-08-01

BidPoint uses a practical testing portfolio: the smallest fast test that can prove a behavior, supplemented by integration, contract, system, load, recovery, and AWS evidence where the boundary requires it. It is not a pursuit of a single coverage percentage, and this repository contains no test implementation yet.

## Portfolio and ownership

| Layer/tooling | What it proves | Ownership and limits |
| --- | --- | --- |
| JUnit, Mockito, AssertJ | Pure domain rules, command validation, idempotency decisions, error classification, and small adapters in isolation. | Developers own fast deterministic tests; mocks do not prove driver, broker, SQL, or serialization behavior. |
| Spring Boot Test | Spring wiring, configuration properties, security slices, transactions, and owner APIs where a Spring context is material. | Owner service/module retains responsibility; keep scope focused. |
| Spring Modulith verification and module integration tests | `core-platform` module boundaries, permitted dependencies, internal event behavior, and module-level integration. | Core owns its modules; passing tests do not make shared tables or cross-service writes valid. |
| ArchUnit 1.4.2 | Package/architecture boundaries and forbidden dependencies. | Enforced in CI; it complements, not replaces, code review and integration evidence. |
| Testcontainers | Real PostgreSQL, Kafka, RabbitMQ, Redis, and Adobe S3Mock integration behavior. | Canonical isolated integration platform; use real service APIs, migrations, and ephemeral data. No H2 substitute. |
| WireMock | HTTP dependency contracts and future payment-provider simulation. | Default for non-owned HTTP dependencies; real provider calls are not routine suite behavior. |
| Toxiproxy | Latency, disconnect, partial-failure, and recovery behavior at a network boundary. | Inject only controlled failures and assert safe retry/UNKNOWN/quarantine behavior. |
| Awaitility | Bounded assertions for asynchronous convergence. | Tests must expose a specific deadline and diagnostic state; never use unbounded sleeps. |
| k6 1.7.1 | Load, concurrency, SSE reconnect/replay, and hot-auction behavior. | Run against explicitly designated environments/data; its output informs capacity and SLO thresholds. |
| JaCoCo 0.8.15 and Spotless Maven Plugin 3.6.0 | Coverage visibility and deterministic formatting. | Coverage informs gaps; it is not a quality target by itself. Formatting is a CI gate. |

JUnit/Mockito/AssertJ/Testcontainers/WireMock/Awaitility/Toxiproxy use a compatible BOM or intentional pin policy rather than invented independent versions. Maven remains the Java test/build authority; Nx may orchestrate affected cross-language work when its experimental `@nx/maven` integration proves reproducible.

## Required behavior evidence

| Concern | Evidence required before treating it as implemented |
| --- | --- |
| Authoritative owner behavior | Unit and owner integration tests prove lifecycle/authorization/validation rules; database tests prove migrations and transaction semantics. |
| Concurrent bidding | Deterministic concurrent test harness drives competing requests for one auction and proves one authoritative price/winner, legal increments, safe conflicts, and stable same-key retry results. It measures the selected database concurrency mechanism rather than assuming one. |
| Close/fence/order path | Tests prove no order from Bidding acknowledgement/outcome, atomic `CLOSED` plus one auctions-owned `AuctionClosed`, after-commit handling, and at most one order under event/auction redelivery. |
| Outbox and Kafka | Database + Testcontainers tests prove atomic state/outbox commit, relay retry/duplicate publication tolerance, consumer stable-ID deduplication, ordering by `auctionId`, schema/contract compatibility, lag/replay, and no repeated local/external effect. |
| RabbitMQ notification work | Tests prove intent plus job-outbox atomically, competing-worker redelivery, bounded retry classification, DLQ/quarantine, stable provider key, and `UNKNOWN` reconciliation before any controlled retry. |
| REST/SSE | Contract/system tests prove JWT/owner authorization, idempotency/error semantics, bounded SSE replay, reconnect, and authoritative refetch after a gap. SSE updates never prove bid acceptance. |
| Persistence/recovery | Restore exercises prove owner data can be restored and migrations replayed safely; replay drills retain stable event/job identity and do not duplicate payment/notification outcomes. |
| Delivery | GitOps/rollout tests prove readiness, image digest promotion, Argo CD sync, blue-green/rollback behavior, and later canary only with compatibility and meaningful analysis signals. |
| AWS | Small explicit, cost-controlled smoke tests prove real S3 and selected AWS integration paths. They are separate from ordinary suites and clean up scoped resources. |

No external calls occur in ordinary unit or integration suites. Testcontainers provides disposable real dependencies; WireMock simulates HTTP dependencies; a real AWS smoke suite uses dedicated non-production accounts/roles and a strict scope/cost budget. It must not require a developer's local cluster or create unbounded resources.

## Contracts, compatibility, and quality gates

Producers own external event contracts. Contract/schema compatibility checks cover event type/version, required envelope fields, consumers' unknown-version behavior, and compatible rollout order. The schema registry remains **Open**; tests must not imply that a registry has been selected. REST and provider contracts similarly require version-aware fixtures and negative cases for signatures, replay, timeout, and invalid payloads.

CI gates compile, test, enforce Spring Modulith/ArchUnit boundaries, run formatting, publish JaCoCo reporting, and run the relevant integration/compatibility suites. Supply-chain, image scanning/SBOM/signing, performance, and rollout gates are introduced at their stated delivery milestones with actionable policy. A green percentage without invariant, integration, or recovery evidence is insufficient.

## Failure, performance, and staged practice

Toxiproxy experiments cover database/broker/provider-like latency, connection loss, and partial failure; assertions check bounded retry, preserved outbox/inbox evidence, delayed projection visibility, and quarantine rather than data loss. k6 exercises bid contention/hot records and Kafka partition skew, request error/latency, SSE fan-out/replay gaps, and worker backlogs. Tests use deterministic data, explicit time windows, and safe cleanup; production-like destructive experiments require a dedicated approved environment.

**Staged:** comprehensive load and chaos/failure testing, full rollout/canary analysis, Debezium-outbox comparison, gRPC comparison, GitHub Actions comparison, and multi-region recovery exercises. **Excluded:** H2, external provider/AWS calls in ordinary suites, coverage-percentage gaming, fixed sleep-based asynchronous assertions, and claiming exactly-once behavior from a single test layer.
