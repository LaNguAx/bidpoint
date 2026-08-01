# Delivery roadmap (2.0)

Status: Canonical
Last validated: 2026-08-01
Supersedes: [1.0/20 Delivery and learning roadmap](../1.0/20-delivery-and-learning-roadmap.md)

Every stage produces a vertical slice and a lesson. Target inclusion does not mean day-one activation. Acceptance evidence is the deliverable, per [01 Project thesis](01-project-thesis.md).

## What changed from 1.0

1.0's eleven stages were correctly *scoped* and badly *ordered* for a solo build. Bidding concurrency — the reason the auction domain was chosen — was stage 4, sitting behind a build-orchestration stage and a full k3d/Strimzi/operator platform stage. Weeks of yak-shaving stood between starting and the first interesting problem, which is the most common way projects like this die.

2.0 changes three things and nothing else:

1. **Stage 1 shrinks.** Maven reactor, module boundaries, and quality gates. Nx is gone, so most of the original stage goes with it.
2. **The local Kubernetes platform moves to Phase B.** Testcontainers and a narrow Compose file carry Phase A. Kubernetes is built later, when Kubernetes itself is the lesson and there are real services to deploy into it.
3. **The stages are grouped into two phases** that each end in something demonstrable.

Stage content, ownership rules, and acceptance evidence are otherwise inherited from 1.0.

## Phase A — Correctness

*Answers: can I build a distributed system that stays correct under concurrency and partial failure, and prove it?*

Runs on Testcontainers plus a narrow Docker Compose aid. No cluster.

| Stage | Outcome and lesson | Acceptance evidence |
| --- | --- | --- |
| **A1. Repository fitness** | Maven reactor, `com.bidpoint` namespace, module boundaries, quality gates. | Direct Maven build; Spring Modulith and ArchUnit boundary tests failing on a deliberate violation; Spotless and JaCoCo wired. |
| **A2. Core domain and identity** | Profiles, catalog, listings, auction scheduling and lifecycle; local ACID transactions; Keycloak identity. | Flyway migrations; module ownership tests; JWT validation in a resource server; object-level authorization tests; no cross-module writes. |
| **A3. Bidding, concurrency, idempotency** | **The thesis.** Safe concurrent bids, fence/finalize, hot-record authority. | Concurrency test proving exactly one price and one winner under contention; idempotency-key reuse returning the recorded result; fence retry after simulated ambiguity; the chosen concurrency mechanism selected by measurement, with the measurement recorded. |
| **A4. Kafka facts, outbox, replay** | Durable facts, at-least-once delivery, observable lag. | Atomic state-plus-outbox commit; crash-window test between commit and publish; consumer deduplication; per-`auctionId` ordering; replay rebuilding a projection without repeating an external effect. |
| **A5. Realtime and SSE** | Derived live view, bounded replay, reconnect. | SSE from multiple replicas; reconnect with last event ID; replay-gap detection signalling authoritative refetch; SSE proven non-authoritative. |
| **A6. RabbitMQ notifications** | Facts versus work; policy separate from delivery. | Durable intent and job outbox; worker retry classification; DLQ and quarantine; `UNKNOWN` provider outcome reconciled before controlled retry with a stable key. |
| **A7. Orders and payment** | One order per close; provider adapter; webhook recovery. | Redelivered `AuctionClosed` creating exactly one order; simulated provider timeout reconciled without double charge; verified webhook signature and replay protection. Provider remains **Open**; a fake adapter is sufficient. |
| **A8. Observability and load** | OTel signals, failure injection, capacity. | Correlation preserved across REST and async boundaries; Prometheus/Loki/Tempo/Grafana showing RED, outbox age, consumer lag, DLQ depth; Toxiproxy recovery evidence; k6 results with stated conditions; hot-partition behavior measured. |

**Phase A is complete and demonstrable on its own.** If nothing further is built, the stated goal in [01](01-project-thesis.md) is met.

## Phase B — Platform

*Answers: can I run and deliver that system the way production teams do?*

Adds no domain behavior. Deploys the Phase A artifact.

| Stage | Outcome and lesson | Acceptance evidence |
| --- | --- | --- |
| **B1. Local Kubernetes** | k3d/k3s, Traefik, Gateway API, Strimzi and RabbitMQ operators, Keycloak, S3Mock, HPA, readiness. | Every Phase A service running in-cluster; readiness and liveness proven by a deliberate dependency outage; HPA scaling on load; Compose aid demonstrably not required. |
| **B2. Delivery** | Jenkins JCasC, ephemeral agents, Jib, ECR digest promotion, Argo CD, blue-green. | Immutable digest promoted through `dev`/`stage`/`prod` by reviewed Git change; Argo sync history; rollback to a prior digest; no direct Jenkins apply. |
| **B3. AWS parity** | Terraform, EKS, RDS, S3, ECR, Amazon MQ, Redis Cloud, secrets, ALB, AWS telemetry. | Ownership separation between Terraform/Helm/Argo; add-on compatibility check; **full create-and-destroy cycle demonstrated**; scoped cost-controlled smoke tests cleaned up. |
| **B4. Resilience** | OpenSearch, KEDA, Istio ambient, canary, chaos. | Search projection and recovery; queue-depth scaling distinct from HPA; mesh identity and policy; canary with automated rollback; chaos results with trade-offs stated. |

Per [02 Technology stack](02-technology-stack.md), the AWS environment is created and destroyed per session and never left running. Teardown is acceptance evidence, not cleanup.

## Preserved constraints

Unchanged from 1.0 and binding at every stage:

- Maven remains build authority. Producer-owned contracts; owner-local data; no shared business models.
- Digest promotion is reviewed; Argo CD alone reconciles workload state.
- No secrets in Git, Terraform state, Helm values, events, jobs, logs, or diagnostics.
- Ordinary test suites make no external calls. Testcontainers is disposable; AWS smoke tests are explicit and cost-controlled.
- HPA scales on HTTP and resources; KEDA scales on queue and lag; node groups provide capacity. These are three different control loops.
- Coverage is diagnostic evidence, never a target to game.

**Open:** frontend, payment provider, notification delivery provider, MSK mode, schema registry, license.
**Staged:** OpenSearch, Istio, KEDA, canary, chaos, Debezium, gRPC, GitHub Actions, ElastiCache/Valkey, Karpenter, multi-region DR.
**Excluded:** Nx while no frontend exists, X-Ray, Ansible, AWS API Gateway, Eureka, initial Spring Cloud Stream, MinIO/AIStor, LocalStack, Promtail, ingress-nginx, H2, direct Jenkins deployment, RabbitMQ Streams, physical database per Modulith module, environment branches, cosmetic microservices.

Full registers are in [05 Exclusions and open questions](05-exclusions-and-open-questions.md).
