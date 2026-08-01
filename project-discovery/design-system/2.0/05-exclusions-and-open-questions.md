# Exclusions and open questions (2.0)

Status: Canonical
Last validated: 2026-08-01
Supersedes: [1.0/22 Exclusions and open questions](../1.0/22-exclusions-and-open-questions.md)

An **Open** item needs evidence before selection; it is not a request to decide now. An **Excluded** item is deliberately outside the baseline and may be reconsidered only when its stated trigger is met. **Optional comparison** exercises are later learning work and cannot mutate the canonical target merely by being evaluated.

Only the rows that changed in 2.0 are annotated. Everything else is carried forward from 1.0 verbatim in substance.

## Genuine open choices

One 1.0 item is closed; the rest stand.

| Open choice | Why it remains open | Decide by | Decision criteria | Work blocked |
| --- | --- | --- | --- | --- |
| Frontend companion stack: framework, state, data fetching, testing, design system | CSR is selected, but a frontend ecosystem would add dependencies and conventions before the browser product slice is implemented. | Before the first authenticated browser vertical slice needs client state, API caching, component testing, or visual conventions. | Compatible with CSR/SSE and token handling; maintainability; accessibility; testing story; bundle and developer ergonomics. **2.0:** the Nx/pnpm criterion from 1.0 is dropped — a frontend selection now also decides whether Node tooling returns at all, per ADR-019-R1. | Frontend implementation, client test harness, component conventions. Backend REST/SSE contracts are not blocked. |
| Payment provider and its sandbox/webhook contract | The provider must support stable payment-intent/idempotency identifiers, webhook signature verification, replay handling, and reconciliation after ambiguous requests. | Before the `payment-service` provider adapter, real webhook contract tests, or any real payment smoke test. | Stable-key idempotency; lookup and reconciliation by provider key; webhook signature and replay guarantees; geography and currency; sandbox quality; operational and audit suitability; cost and secret-management fit. | Provider adapter, real payment lifecycle, webhook fixtures, payment smoke evidence. **A7 is not blocked** — a fake adapter satisfies its acceptance evidence. No card data enters BidPoint while open. |
| Notification delivery provider and reconciliation contract | The worker needs externally verifiable idempotency after ambiguous delivery attempts; a local attempt record cannot prove whether the provider accepted delivery. | Before a real provider adapter, credentials, contract tests, reconciliation runbook, or delivery smoke test. | Stable provider idempotency key; status lookup by that key; unambiguous accepted/not-accepted outcomes; channels and regions; sandbox quality; delivery and audit evidence; cost, limits, secret-management fit. | Provider-specific adapter and configuration, real fixtures, reconciliation automation. **A6 is not blocked** — durable intent and job processing with a fake adapter satisfies its acceptance evidence. |
| Amazon MSK mode: Express 4.2.x versus Provisioned 4.1.x, and local Kafka alignment | Managed operating model, broker family, capability, cost, and local parity implications need current AWS evidence. | Before Terraform creates an MSK cluster or the remote Kafka acceptance set is finalized — that is, not before B3. | Regional availability; supported client and protocol behavior; partition/throughput and operational controls; **cost under a create-and-destroy cycle per ADR-033**; quotas; encryption and networking; observability; local Kafka contract alignment. | Remote Kafka infrastructure, capacity plan, remote broker smoke tests. All of Phase A is unblocked. |
| Schema registry: Apicurio versus AWS Glue Schema Registry | Event envelope and versioning are decided; registry ownership and managed-service tradeoffs are not. | Before a registry enters local or remote event publication or compatibility gates. | Compatibility mode; schema formats; local/remote parity; AWS integration; access model; operational overhead; migration and export path; CI compatibility checks. | Registry-backed serialization and registry-specific compatibility automation. Events must carry explicit versions and defined consumer unknown-version behavior regardless. |
| Public license | Reuse, contribution, and distribution intent has not been selected. | Before public distribution or accepting external contribution under asserted terms. | Intended reuse and commercial constraints; dependency-license compatibility; contributor policy; clarity. | Repository license, contributor terms, public release packaging. **Relevant sooner in 2.0:** a portfolio project shown to employers is a form of publication. |
| Timing and success criteria for comparison exercises | Comparisons are useful only after the canonical path produces measurable behavior. | At the corresponding stage, after baseline acceptance evidence exists. | Hypothesis; bounded scope; baseline metric; operational cost; reversibility; learning value; proof the trial will not become an undeclared dependency. | The experiments themselves, not the canonical stages. |

### Closed in 2.0

| Former open choice | Resolution |
| --- | --- |
| Spring Boot 4.1.0 plus Spring Cloud AWS 4.0.0 compatibility | **Closed.** Resolved by aligning Boot to the version Spring Cloud AWS documents support for, under the tiered pin policy in ADR-028-R1. The gate is removed rather than carried as risk. Exact versions verified against upstream documentation at implementation. |

## Explicit exclusions from the baseline

New and reclassified rows are marked. Unmarked rows carry forward from 1.0 unchanged.

| Excluded item | Rationale | Reconsider only when |
| --- | --- | --- |
| **Nx, `@nx/maven`, pnpm, and Node as build tooling** *(new in 2.0)* | Maps to no learning outcome; experimental plugin wrapping a working build system in a repository with no Node code. 1.0's own guard required maintaining a Maven equivalent for every Nx target. | A frontend package is selected and genuinely needs Node workspace management. Scope it to that package; Maven authority over Java is not negotiable. |
| **AWS X-Ray** *(new in 2.0; the legacy SDK/daemon was already excluded)* | Offers nothing Tempo does not, at additional integration cost, in a design where OTel is already the seam. | A required AWS-native trace integration cannot be served by ADOT into Tempo, and does not create dual trace ownership. |
| Docker Compose as the **canonical full environment** *(narrowed in 2.0)* | The learning target requires Kubernetes controllers, routing, readiness, scaling, delivery, and telemetry boundaries. **Still true, and still excluded in this form.** A narrow Phase A developer aid is now Canonical per ADR-017-R1 and claims no parity. | Unchanged: Compose may never replace `core`/`platform` or be presented as platform parity. The Phase A aid is deleted or ignored once B1 passes. |
| AWS API Gateway | Spring Cloud Gateway is the runtime application gateway; ALB and WAF are the AWS front door. | A documented public-edge requirement cannot be met by ALB plus Spring Cloud Gateway with clear ownership, cost, or security benefit. |
| Eureka | Kubernetes Services and DNS already provide internal discovery. | A non-Kubernetes runtime or measured discovery need makes Kubernetes-native discovery insufficient. |
| Spring Cloud Kubernetes discovery server | Kubernetes-native DNS avoids a second discovery authority. | A supported, evidence-backed requirement needs behavior unavailable from Services and DNS. |
| Spring Cloud Stream initially | Explicit Kafka and RabbitMQ client contracts teach delivery semantics and avoid premature abstraction. | Baseline outbox, ordering, retry, and recovery evidence is complete and a bounded comparison shows net benefit without hiding those semantics. |
| Valkey as canonical managed Redis | Redis Cloud is the current choice; Redis remains non-authoritative. | The staged ElastiCache/Valkey comparison produces better evidence on compatibility, cost, operations, and migration risk. |
| Karpenter initially | Managed node groups are the remote capacity baseline. | Measured capacity constraints persist after node-group policy is tuned. |
| Istio sidecars | The staged mesh direction is ambient. | A supported requirement cannot be met by ambient mode, justified with security or performance evidence. |
| MinIO and AIStor | S3Mock is the deliberately narrow local S3 double. | A specific S3-contract gap blocks learning and a comparison justifies a second storage product. |
| LocalStack OSS | The target uses real protocol-focused components plus scoped AWS smoke tests, not broad cloud emulation. | A bounded AWS integration need cannot be safely or economically covered by existing doubles plus scoped smoke tests. |
| Promtail | Logs flow through the OpenTelemetry Collector to Loki. | The collector pipeline cannot meet a demonstrated requirement without creating dual ingestion ownership. |
| ingress-nginx | Traefik is the local edge; ALB is the AWS edge. | A documented routing capability cannot be met by the selected front-door stack. |
| H2 | Testcontainers PostgreSQL is needed to prove migrations, transactions, locking, and SQL behavior. | A narrowly isolated test makes no persistence-semantic claim and cannot be mistaken for integration evidence. |
| Direct Jenkins production deployment | Jenkins builds and proposes; Argo CD reconciles. | The GitOps control plane is unavailable in a bounded, separately authorized, audited, reversible emergency procedure. |
| OpenSearch on day one | Browse and filter remain Core capabilities; full-text search is a staged derived view. | Core behavior, Kafka facts, and projection/recovery evidence are stable and search need is demonstrated. |
| RabbitMQ Streams | RabbitMQ carries targeted work to competing workers; Kafka owns durable replayable facts. | A one-work-obligation stream requirement cannot be satisfied by the Kafka/RabbitMQ split. |
| Physical database per Modulith module on day one | Core modules need independent logical write boundaries, not premature operational splits. | Evidence proves independent deployment, recovery, or scaling requires a physical split with no shared-writer transition. |
| Shared cross-service business models | Producer-owned contracts prevent cross-service implementation coupling. | A cross-cutting type is proven mechanical rather than domain behavior and is accepted as a narrowly scoped shared library. |
| Environment branches | Same-repository GitOps uses environment directories on `main` and promotes an immutable digest. | A governed promotion or audit requirement cannot be met by reviewed path-scoped changes on `main`. |
| Ansible | Terraform, Helm, Argo CD, and JCasC have distinct ownership. | A gap remains after assigning an asset to those owners, without creating dual mutable ownership. |
| Microservices solely for appearance | A service needs an independent consistency, scaling, failure, or integration responsibility. | Evidence demonstrates one, with data, contract, operational, and migration consequences recorded. |
| Frontend application-stack selection during backend work | CSR establishes the boundary without selecting a UI ecosystem prematurely. | The first browser vertical slice reaches the frontend gate above. |

## Optional comparison exercises, later only

| Comparison | Baseline held constant | Entry condition and evidence |
| --- | --- | --- |
| **AMP and Amazon Managed Grafana versus self-hosted** *(new in 2.0)* | OTel Collector remains the seam; Prometheus/Loki/Tempo/Grafana remain canonical in both environments. | After B3 telemetry works end to end. Compare operational cost, retention, query experience, and whether managed services justify a second pipeline. Some employability value in having operated them. |
| **GitHub Actions versus Jenkins** *(elevated in 2.0)* | Jenkins remains CI baseline; Argo CD remains the reconciler. | Unchanged criteria, but explicitly the **recorded fallback** if Phase B stalls on Jenkins specifically, per ADR-022-R1. |
| Debezium CDC outbox versus application relay | Transactional outbox, stable event identity, at-least-once processing, producer ownership. | After the application relay has measurable backlog and recovery behavior; assess failure modes, operational burden, ordering, replay. |
| gRPC versus REST on one internal path | Public APIs stay REST; ownership and data rules unchanged. | One latency- or streaming-sensitive internal path with a measured REST baseline and a contract migration/rollback plan. |
| ElastiCache/Valkey versus Redis Cloud | Redis stays acceleration and ephemeral coordination, never authority. | Compare managed availability, compatibility, cost, operations, security, migration and recovery. |
| Karpenter versus managed node groups | HPA/KEDA behavior and managed node groups remain baseline. | Use actual scheduling and capacity evidence; compare cost, disruption, security, operational controls. |
| Schema registry options | Explicit envelopes, versioning, stable IDs, consumer compatibility remain baseline. | After compatibility test requirements and event schema scope are defined. |
| SSE versus WebSockets | SSE remains the live-update baseline; REST remains command authority. | A documented bidirectional need that bounded SSE cannot meet, plus auth, reconnect, scaling, recovery evidence. |
| Multi-region disaster recovery | Single-region design and owner-local recovery remain baseline. | After backup/restore, replay, immutable deployment, and regional objectives are demonstrated. Note ADR-033 cost constraints. |

## Interpretation rules

Unchanged from 1.0. Open and Optional records do not authorize new dependencies, manifests, services, cloud resources, or claims that a capability exists. An exclusion is not an unresolved question. Any reconsideration must update [04 Decision delta](04-decision-delta.md), identify the affected documents, and rest on official primary-source evidence per [1.0/23](../1.0/23-primary-sources.md) rather than search snippets or memory.
