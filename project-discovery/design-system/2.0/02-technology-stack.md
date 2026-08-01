# Technology stack (2.0)

Status: Canonical
Last validated: 2026-08-01
Supersedes: [1.0/17 Local stack](../1.0/17-local-technology-stack.md), [1.0/18 Remote stack](../1.0/18-remote-technology-stack.md), [1.0/19 Versions and compatibility](../1.0/19-versions-and-compatibility.md)

Local and remote inventories are merged here because 1.0's split duplicated two thirds of its rows and made the remote stack look larger than it needs to be. Phase labels come from [01 Project thesis](01-project-thesis.md).

## Pin policy — changed in 2.0

1.0 pinned every component to an exact latest release. That is correct as reproducibility policy and wrong as a version *selection* policy for a solo learning project. It produced one documented contradiction already (Boot 4.1.0 against Spring Cloud AWS 4.0.0) and would produce more.

2.0 keeps exact pinning and changes what gets pinned to:

| Tier | Policy | Rationale |
| --- | --- | --- |
| **Behavioral** — PostgreSQL, Kafka, RabbitMQ, Redis, Keycloak, Kubernetes | Exact pin, current stable minor. Their behavior *is* the lesson; reproducibility matters most here. | You are learning what these do under failure. Version drift changes the answer. |
| **Framework** — Spring Boot, Spring Cloud, Spring Modulith | Exact pin, **one minor behind current** unless a needed feature requires the newest. | Settled releases have documentation, examples, and answered questions. Debugging a brand-new major teaches nothing on the outcome list. |
| **Tooling** — Maven, Jib, Spotless, JaCoCo, ArchUnit, Helm, k3d | Pin within a stable family; float patches. | Low behavioral risk; not worth upgrade ceremony. |
| **Managed AWS families** | Unchanged from 1.0: verify region/account availability, add-on compatibility, quotas, and rollback at implementation. Never infer from a general project page; never use `latest`. | Provider owns the patch level. |

**This resolves 1.0's Open compatibility gate.** Aligning Spring Boot down to the version Spring Cloud AWS documents support for removes the unconfirmed pairing entirely, rather than carrying it as risk. The exact train and Boot minor are verified against upstream release documentation at implementation time — 2.0 deliberately does not restate version numbers it cannot validate today. See [04 Decision delta](04-decision-delta.md), ADR-028-R1.

Exact 1.0 pins remain a valid starting reference in [1.0/19](../1.0/19-versions-and-compatibility.md); they are a snapshot to re-validate, not a contract.

## Application stack — unchanged from 1.0

Every row below is retained without modification. This is the part of 1.0 that was right.

| Component | Status | Responsibility |
| --- | --- | --- |
| Eclipse Temurin (LTS) | Canonical | JVM. |
| Maven + Maven Wrapper | Canonical | **Sole** Java build, dependency, test, and reactor authority. |
| Spring Boot | Canonical | Backend framework. |
| Spring Cloud | Canonical | Cloud release train. |
| Spring Cloud Gateway | Canonical | Runtime application gateway; the only WebFlux component. |
| Spring Modulith | Canonical | Core module boundaries and durable event publication. |
| Spring Security resource server | Canonical | Independent JWT validation in every backend. |
| Spring Data JPA | Canonical | Owner-local relational persistence. |
| Flyway | Canonical | Schema migration. |
| Spring for Apache Kafka | Canonical | Durable fact handling. |
| Spring AMQP | Canonical | Targeted work handling. |
| Actuator | Canonical | Health and readiness. |
| Micrometer + Observation | Canonical | Metrics and trace-context propagation. |
| OpenTelemetry integration | Canonical | Trace export. |
| AWS SDK for Java v2 (BOM) | Canonical | Cloud protocol SDK, visible rather than wrapped. |
| Jib Maven Plugin | Canonical | JVM images; no routine Dockerfiles. |

**Removed:** Nx, `@nx/maven`, pnpm, Node.js as build tooling. See [04 Decision delta](04-decision-delta.md), ADR-019-R1. pnpm and Node return if and when a frontend is selected, scoped to that frontend only.

**Open:** Spring Cloud AWS remains gated, but the gate is now trivially satisfiable under the aligned pin policy above.

## Data, messaging, identity — unchanged from 1.0

| Component | Status | Responsibility |
| --- | --- | --- |
| PostgreSQL | Canonical | Authoritative data and outboxes. |
| Redis | Canonical | Acceleration and realtime coordination; explicitly non-authoritative. |
| Kafka | Canonical | Durable replayable facts. |
| RabbitMQ | Canonical | Targeted work to competing workers. |
| Keycloak | Canonical | OIDC/OAuth2 identity and credential boundary. |
| Adobe S3Mock | Canonical | Local S3 double; real S3 smoke tests still required. |

## Testing and evidence — unchanged, and promoted in importance

| Component | Status | Responsibility |
| --- | --- | --- |
| JUnit, Mockito, AssertJ | Canonical | Unit and behavior evidence. |
| Testcontainers | Canonical | Real PostgreSQL/Kafka/RabbitMQ/Redis integration behavior. |
| WireMock | Canonical | Provider contract simulation. |
| Awaitility | Canonical | Bounded convergence assertions. |
| Toxiproxy | Canonical | Network failure injection. |
| ArchUnit | Canonical | Module boundary enforcement. |
| JaCoCo | Canonical | Coverage as diagnostic evidence, never a target. |
| Spotless | Canonical | Deterministic formatting gate. |
| k6 | Canonical | Load, SSE, and capacity evidence. |

Per [01](01-project-thesis.md), output from these tools is the project's primary deliverable. Testcontainers additionally carries more weight in 2.0: it is what makes Phase A runnable without a cluster.

## Local environment — resequenced in 2.0

| Component | Status | Phase | Note |
| --- | --- | --- | --- |
| Testcontainers-backed integration | Canonical | A | Primary dependency environment for correctness work. |
| Docker Compose (narrow developer aid) | **Canonical for Phase A** | A | Reclassified from Excluded. Runs Postgres, Kafka, RabbitMQ, Redis, Keycloak, S3Mock for local iteration. Explicitly claims **no** platform parity. |
| k3d / k3s | Canonical | B | Local Kubernetes; the lesson, not the prerequisite. |
| Traefik + Helm chart | Canonical | B | Local cluster edge. |
| Kubernetes Gateway API | Canonical | B | Configuration API, not a runtime gateway product. |
| metrics-server | Canonical | B | HPA input. |
| Helm | Canonical | B | Kubernetes packaging. |
| Strimzi | Canonical | B | Kafka operator; Compose/Testcontainers cover Phase A. |
| RabbitMQ Cluster Operator | Canonical | B | Broker lifecycle in cluster. |
| Jenkins + Helm chart | Canonical | B | CI controller, JCasC, ephemeral agents. **Retained** — see below. |
| Argo CD | Canonical | B | GitOps reconciliation. |
| Argo Rollouts | Staged | B | Blue-green before canary. |
| KEDA | Staged | B | Queue/lag worker scaling. |
| Istio ambient | Staged | B | Identity, policy, fault injection. |
| OpenSearch | Staged | B | After browse/filter baseline. |

The Compose reclassification is the single change most likely to determine whether this project is finished. 1.0's exclusion rationale — that the learning target requires Kubernetes controllers, routing, readiness, scaling, delivery, and telemetry — is true for Phase B and irrelevant to writing a correct bid transaction. See [05](05-exclusions-and-open-questions.md).

## Observability — consolidated in 2.0

| Component | Status | Phase | Note |
| --- | --- | --- | --- |
| OpenTelemetry Collector | Canonical | A (local) / B (cluster) | The vendor-neutral seam. This is the actual lesson. |
| Prometheus | Canonical | A/B | Metrics store, both environments. |
| Loki | Canonical | A/B | Redacted log store, both environments. |
| Tempo | Canonical | A/B | Trace store, both environments. |
| Grafana | Canonical | A/B | Dashboards, both environments. |
| ADOT Collector | Canonical | B | AWS-side collection. |
| CloudWatch Logs | Canonical | B | Retained; it is where EKS logs land regardless. |
| Amazon Managed Prometheus | **Optional comparison** | B | Demoted from Canonical. |
| AWS X-Ray | **Excluded** | — | Demoted from Canonical. Strictly less capable than Tempo here and teaches nothing additional. |
| Amazon Managed Grafana | **Optional comparison** | B | Demoted from Canonical. |

1.0 required building and maintaining two complete telemetry pipelines. 2.0 runs one stack in both places and proves the portability claim by repointing a single exporter once — which demonstrates the OTel seam more directly than operating a parallel stack does. See [04 Decision delta](04-decision-delta.md), ADR-025-R1.

## AWS target — retained, cost-bounded

| Component | Status | Note |
| --- | --- | --- |
| Terraform | Canonical | AWS infrastructure; never Argo-owned workload state. |
| Amazon EKS + managed node groups | Canonical | Kubernetes runtime and node capacity. |
| Route 53, WAF, ALB, ACM, AWS Load Balancer Controller | Canonical | AWS front door. |
| RDS PostgreSQL | Canonical | Managed authoritative store. |
| Amazon S3 | Canonical | Image objects. |
| Amazon ECR | Canonical | Digest promotion. |
| Secrets Manager, EKS Pod Identity, Secrets Store CSI | Canonical | Least-privilege secret access. |
| Amazon MQ for RabbitMQ | Canonical | Managed work broker. |
| Redis Cloud | Canonical | Managed Redis. |
| Amazon MSK | Open | Express vs Provisioned mode unresolved. |
| Amazon OpenSearch Service, CloudFront, KEDA, Istio ambient | Staged | Later increments. |
| Karpenter, multi-region DR | Staged / Optional | Unchanged from 1.0. |

**New constraint in 2.0.** The AWS environment is built to be created and destroyed per working session, and is never left running. This is a design requirement on the Terraform layout, not an afterthought: state, data seeding, and teardown must be part of Phase B acceptance evidence.

The rationale is blunt. EKS, RDS, MSK, Amazon MQ, Redis Cloud, ALB, and managed telemetry left running continuously is a several-hundred-USD monthly bill, and cost — not difficulty — is the most likely cause of project abandonment. Where a managed service cannot be economically exercised, an explicitly recorded local substitute plus a scoped, cost-controlled smoke test is the accepted evidence, per [1.0/15](../1.0/15-testing-and-quality.md).

## On Jenkins

Jenkins is **retained as Canonical** in 2.0, against the general principle of cutting periphery, and the reason is recorded so it can be challenged later.

It costs real effort: a JCasC controller with ephemeral Kubernetes agents is a Phase B project of its own, and the lesson it teaches — CI builds and proposes, Argo CD reconciles — is available from GitHub Actions at a fraction of the effort. On the *understanding* goal alone it would be cut.

It survives on the **employability** goal. Jenkins is what large Java, banking, and insurance shops actually run, and it is a differentiator precisely because most candidates have only used hosted CI. This is the one place where the two stated goals in [01](01-project-thesis.md) genuinely diverge, and the tie is broken in favour of employability.

Conditions: Jenkins stays in Phase B, is timeboxed, and never blocks Phase A. GitHub Actions remains an **Optional comparison**, unchanged from 1.0.

## Summary of stack changes from 1.0

| Change | Component | 1.0 | 2.0 |
| --- | --- | --- | --- |
| Removed | Nx, `@nx/maven`, pnpm, Node build tooling | Canonical | Excluded until a frontend exists |
| Removed | AWS X-Ray | Canonical | Excluded |
| Demoted | AMP, Amazon Managed Grafana | Canonical | Optional comparison |
| Promoted | Docker Compose (narrow aid) | Excluded | Canonical, Phase A only |
| Relaxed | Framework version pins | Exact latest | Exact, one minor behind |
| Resolved | Boot / Cloud AWS pairing | Open gate | Closed by aligning Boot |
| Deferred | k3d, Strimzi, operators, Jenkins, Argo CD | Stage 1–2 prerequisites | Phase B |
| Retained | Everything in the application, data, messaging, and testing tables | Canonical | Canonical |
