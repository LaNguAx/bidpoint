# Delivery roadmap (2.0)

Status: Canonical
Last validated: 2026-08-01
Supersedes: [1.0/20 Delivery and learning roadmap](../1.0/20-delivery-and-learning-roadmap.md)
Revised: k3d from stage 1, AWS tracer bullet at A4 (ADR-035, ADR-037)

Every stage produces a vertical slice and a lesson. Acceptance evidence is the deliverable, per [01](01-project-thesis.md) and ADR-032 — a stage is not complete when the feature runs, but when the proof exists.

## Sequencing principles

Three ordering rules, each fixing a specific failure in earlier drafts:

1. **Bidding concurrency arrives early (A3).** It is the thesis. 1.0 buried it behind build tooling and a platform build.
2. **AWS arrives early and small (A4).** Earlier drafts put AWS at stage eleven of twelve, making the primary employability item hostage to finishing everything else. A4 is a *tracer bullet* — deploy one thing end-to-end while the system is small enough to understand, so integration risk surfaces at the start rather than the end.
3. **Kubernetes is present from A1.** No Compose phase, no migration. Learning k8s against a two-service system is easier than retrofitting it onto nine.

## Phase A — Correctness

*Answers: can I build a distributed system that stays correct under concurrency and partial failure, and prove it?*

Runs on local k3d. Ordinary tests use Testcontainers and need no cluster.

| Stage | Outcome and lesson | Acceptance evidence |
| --- | --- | --- |
| **A1. Foundation** | Maven reactor, `com.bidpoint` namespace, module boundaries, quality gates. k3d cluster with PostgreSQL. One service deployed and reachable. | Direct Maven build; ArchUnit and Spring Modulith boundary tests failing on a deliberate violation; Spotless and JaCoCo wired; `kubectl get pods` showing a healthy service; Flyway migration applied in-cluster. |
| **A2. Core domain and identity** | Profiles, catalog, listings, auction scheduling and lifecycle. Keycloak, JWT validation, object-level authorization. | Module ownership tests; resource-server rejecting an invalid token; owner-only edit enforced and tested; no cross-module writes. |
| **A3. Bidding, concurrency, idempotency** | **The thesis.** Safe concurrent bids, fence/finalize, hot-record authority. | Concurrency test proving exactly one price and one winner under contention; idempotency-key reuse returning the recorded result rather than a second bid; fence retry after simulated ambiguity; concurrency mechanism chosen **by measurement**, with the measurement recorded. |
| **A4. AWS tracer bullet** | Containerize one service and run it on real AWS. Terraform, IAM, ECR, ECS Fargate, RDS, Secrets Manager, CloudWatch. | `terraform apply` from zero to a reachable service; image promoted by digest to ECR; secrets injected from Secrets Manager, never in the image; logs in CloudWatch; **`terraform destroy` returning to zero**; billing alarm set. Cost recorded. |
| **A5. Kafka facts, outbox, replay** | Durable facts, at-least-once delivery, observable lag. | Atomic state-plus-outbox commit; crash-window test between commit and publish; consumer deduplication by stable ID; per-`auctionId` ordering; replay rebuilding a projection without repeating an external effect. |
| **A6. Realtime and SSE** | Derived live view, bounded replay, reconnect. Multiple `core-platform` replicas make fan-out real. | SSE served from more than one replica; reconnect with last event ID; replay-gap detection signalling authoritative refetch; SSE proven non-authoritative. |
| **A7. Kafka to RabbitMQ fan-out** | Facts versus work. Policy separate from delivery, competing consumers. | The full chain observed end to end: Kafka fact → `notification-service` decision → durable intent and job outbox → RabbitMQ → **N `notification-worker` replicas competing**, each job executed once; retry classification; DLQ and quarantine; `UNKNOWN` provider outcome reconciled before controlled retry with a stable key. |
| **A8. Orders and payment** | One order per close; provider adapter; webhook recovery. | Redelivered `AuctionClosed` creating exactly one order; simulated provider timeout reconciled without double charge; verified webhook signature and replay protection. Fake adapter is sufficient — provider remains **Open**. |
| **A9. Observability and load** | Metrics that make partial failure visible; failure injection; capacity. | Prometheus and Grafana showing RED, outbox age, consumer lag, DLQ depth; Toxiproxy recovery evidence; k6 results with stated conditions; hot-partition behavior measured. |

**Phase A is complete and demonstrable on its own.** If nothing further is built, the goal in [01](01-project-thesis.md) is met — and A4 means real AWS experience is already banked.

## Phase B — Production delivery

*Answers: can I ship and operate that system the way a team does?*

Adds no domain behavior.

| Stage | Outcome and lesson | Acceptance evidence |
| --- | --- | --- |
| **B1. CI/CD** | GitHub Actions: build, test, image, deploy. Build separated from deploy. | Pipeline running the full Maven verify and Testcontainers suite; image tagged by digest; deployment to ECS on merge; rollback to a prior digest demonstrated. |
| **B2. Full AWS deployment** | All four deployables on ECS Fargate against RDS, S3, and self-hosted brokers. | Every service healthy behind the ALB; secrets from Secrets Manager; logs in CloudWatch; complete create-and-destroy cycle; **recorded cost per session**. |
| **B3. EKS exercise** | Bounded: stand up EKS, deploy the same charts used locally, tear down. | Same Helm charts running on EKS as on k3d; documented differences from ECS; cluster destroyed; spend recorded. Budget ~$20 of credits. |

Per ADR-033 the AWS environment is created and destroyed per session and never left running. Teardown is acceptance evidence, not cleanup.

## Preserved constraints

Binding at every stage:

- Maven is the sole build authority. Producer-owned contracts; owner-local data; no shared business models.
- Producers use transactional outboxes; consumers deduplicate by stable ID. At-least-once, never exactly-once.
- No secrets in Git, Terraform state, Helm values, events, jobs, logs, or diagnostics.
- Ordinary test suites make no external calls. Testcontainers is disposable; AWS smoke tests are explicit and cost-controlled.
- Coverage is diagnostic evidence, never a target to game.
- Every component maps to a learning outcome in [01](01-project-thesis.md), or it is removed rather than staged.

**Open:** frontend, payment provider, notification delivery provider, schema registry, license.
**Excluded:** Nx, Istio, KEDA, Argo Rollouts, Argo CD, Jenkins, Strimzi and RabbitMQ operators, Spring Cloud Gateway, Traefik as a study item, Loki, Tempo, MSK, Amazon MQ, Redis Cloud, AMP, Managed Grafana, X-Ray, OpenSearch, CloudFront, Karpenter, multi-region DR, shared business models, environment branches, cosmetic microservices.

Registers are in [05](05-exclusions-and-open-questions.md); rationale for each removal is in [04](04-decision-delta.md).
