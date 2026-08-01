# Technology stack (2.0)

Status: Canonical
Last validated: 2026-08-01
Supersedes: [1.0/17 Local stack](../1.0/17-local-technology-stack.md), [1.0/18 Remote stack](../1.0/18-remote-technology-stack.md), [1.0/19 Versions and compatibility](../1.0/19-versions-and-compatibility.md)
Revised: stack reduction, ECS-first AWS target, k3d from stage 1 (ADR-035 through ADR-039)

## Selection rule

Every tool must pass one test: **does it appear in backend engineering job postings, or is it platform/SRE tooling that is industry-standard in large organizations but rarely asked of a backend developer?**

The second category is where stacks go to die. It is legitimate technology, widely deployed, and largely irrelevant to the stated goal. 1.0 and the first draft of 2.0 were full of it — roughly forty tools, of which a third were mesh, progressive-delivery, operator, and autoscaler tooling that a backend candidate is essentially never asked about.

The stack below is about twenty tools. Nothing in it is obscure.

## Pin policy

Exact pinning is reproducibility policy. Version *selection* is tiered:

| Tier | Policy | Rationale |
| --- | --- | --- |
| **Behavioral** — PostgreSQL, Kafka, RabbitMQ, Redis, Keycloak, Kubernetes | Exact pin, current stable minor. | Their behavior is the lesson; drift changes the answer. |
| **Framework** — Spring Boot, Spring Modulith | Exact pin, **one minor behind current**. | Settled releases have documentation, examples, and answered questions. |
| **Tooling** — Maven, Jib, Spotless, ArchUnit, Testcontainers | Pin within a stable family; float patches. | Low behavioral risk. |
| **AWS managed families** | Verify region availability, compatibility, and quotas at implementation. | Provider owns the patch level. |

Never use `latest`. Never infer a compatible pair from a general project page. Verify against publisher documentation at implementation — Context7 first, publisher docs authoritative. The pins in [1.0/19](../1.0/19-versions-and-compatibility.md) are a snapshot to re-validate, not a contract.

Aligning Spring Boot one minor behind also closes 1.0's unconfirmed Boot / Spring Cloud AWS pairing.

## Application stack

| Component | Status | Responsibility |
| --- | --- | --- |
| Eclipse Temurin (LTS) | Canonical | JVM. |
| Maven + Maven Wrapper | Canonical | **Sole** build, dependency, test, and reactor authority. |
| Spring Boot | Canonical | Backend framework. |
| Spring Modulith | Canonical | Core module boundaries and durable event publication. |
| Spring Security resource server | Canonical | Independent JWT validation in every backend. |
| Spring Data JPA | Canonical | Owner-local relational persistence. |
| Flyway | Canonical | Schema migration. |
| Spring for Apache Kafka | Canonical | Durable fact handling. |
| Spring AMQP | Canonical | Targeted work handling. |
| Actuator | Canonical | Health, readiness, metrics endpoint. |
| Micrometer | Canonical | Metrics and trace-context propagation. |
| AWS SDK for Java v2 (BOM) | Canonical | Cloud protocol SDK, used directly rather than wrapped. |
| Jib Maven Plugin | Canonical | JVM images; no routine Dockerfiles. |

**Removed:** Nx, `@nx/maven`, pnpm, Node build tooling (ADR-019-R1). **Spring Cloud Gateway removed** (ADR-038) — an eighth deployable for a pattern better learned later.

## Data, messaging, identity

| Component | Status | Responsibility |
| --- | --- | --- |
| PostgreSQL | Canonical | Authoritative data and outboxes. |
| Kafka | Canonical | Durable replayable facts. Highest-value distributed item in the stack. |
| RabbitMQ | Canonical | Targeted work to competing workers. Plain broker — **no cluster operator**. |
| Redis | Canonical | Acceleration and ephemeral coordination; never authoritative. |
| Keycloak | Canonical | OIDC/OAuth2 identity. One realm, one client — kept deliberately simple. |
| LocalStack or S3 direct | Canonical | Object storage for listing images. |

Both brokers are retained deliberately. The canonical flow that justifies them:

```
bidding-service  --Kafka fact-->  notification-service  --RabbitMQ job-->  notification-worker (N replicas)
   (producer)                       (decides policy)                         (competing consumers)
```

Kafka carries a durable, replayable, ordered-per-key fact that many independent consumers read. RabbitMQ carries a single work obligation that exactly one of N competing workers executes. Collapsing them into one broker loses the distinction, which is one of the more valuable lessons here.

**Removed:** Strimzi and RabbitMQ Cluster Operator (ADR-036). The goal is to learn Kafka and RabbitMQ, not their Kubernetes operators — operator lifecycle management is a k8s-admin skill.

## Testing and evidence

| Component | Status | Responsibility |
| --- | --- | --- |
| JUnit, Mockito, AssertJ | Canonical | Unit and behavior evidence. |
| Testcontainers | Canonical | Real PostgreSQL/Kafka/RabbitMQ/Redis integration behavior. |
| Awaitility | Canonical | Bounded convergence assertions. |
| WireMock | Canonical | Provider contract simulation. |
| Toxiproxy | Canonical | Network failure injection. |
| ArchUnit | Canonical | Module boundary enforcement. |
| Spotless | Canonical | Deterministic formatting gate. |
| JaCoCo | Canonical | Coverage as diagnostic evidence, never a target. |
| k6 | Canonical | Load, SSE, and capacity evidence. |

Per ADR-032 these produce the project's primary deliverable. Integration testing against real infrastructure via Testcontainers is also a genuine interview differentiator — most candidates have only mocked.

## Local environment

Kubernetes from stage 1 (ADR-037). There is no Compose phase and no migration between environments.

| Component | Status | Responsibility |
| --- | --- | --- |
| Docker | Canonical | Container runtime. |
| k3d / k3s | Canonical | Local Kubernetes. Free, disposable, unlimited iteration — this is where Kubernetes depth is built. |
| kubectl | Canonical | Cluster interaction. |
| Helm | Canonical | Packaging for own services and third-party charts. |
| Prometheus + Grafana | Canonical | Metrics and dashboards. RED, outbox age, consumer lag, queue depth. |

Postgres, Kafka, RabbitMQ, Redis, and Keycloak run **in-cluster from community Helm charts** — no operators. Ordinary tests use Testcontainers and need no cluster at all.

**Removed:** Traefik and Gateway API as study items (ADR-036) — use whatever k3d ships with; the edge is not the lesson. Loki and Tempo removed (ADR-039) — Prometheus and Grafana cover the metrics lesson; add tracing later if wanted.

## AWS target

**ECS Fargate is the primary compute target. EKS is one bounded exercise at the end** (ADR-035).

The reasoning is budget. An EKS control plane costs roughly $73/month before any compute, which consumes a $100 credit allowance in about five weeks with nothing else running. ECS Fargate has no control-plane fee and still teaches the AWS surface that interviews actually probe: IAM roles, task definitions, ECR, VPC and networking, RDS connectivity, secret injection, and log shipping.

Kubernetes depth comes from k3d locally, where iteration is free. EKS is then a short, deliberate exercise to join the two — not the vehicle for learning either.

| Component | Status | Responsibility |
| --- | --- | --- |
| Terraform | Canonical | All AWS infrastructure. |
| Amazon ECR | Canonical | Image registry. |
| **Amazon ECS Fargate** | Canonical | Primary compute. No control-plane fee. |
| Amazon RDS PostgreSQL | Canonical | Managed authoritative store. |
| Amazon S3 | Canonical | Listing objects. |
| AWS IAM | Canonical | Roles, task roles, least privilege. |
| AWS Secrets Manager | Canonical | Secret storage and injection. |
| CloudWatch Logs | Canonical | Log destination. |
| Application Load Balancer | Canonical | Public entry. |
| VPC | Canonical | Networking. **Avoid NAT Gateway** — see below. |
| GitHub Actions | Canonical | CI and deployment. |
| Amazon EKS | Canonical, **bounded** | One exercise: stand up, deploy, tear down, document. Budget ~$20 of credits. |

**Self-host the brokers on AWS.** Kafka and RabbitMQ run as containers, not MSK and Amazon MQ. Managed brokers are the largest line items available and you will already know both operationally from local work. One short managed-service comparison, if wanted, is enough to speak to what managed buys.

**Removed from the AWS target:** MSK, Amazon MQ, Redis Cloud, AMP, Managed Grafana, X-Ray, CloudFront, OpenSearch Service, Karpenter, KEDA, Istio (ADR-035, ADR-036).

### Budget discipline

$100 in credits is the real constraint, and it is enough if infrastructure is treated as disposable.

- **Create and destroy per session** (ADR-033). Never leave the environment running. Terraform layout, seeding, and teardown must be designed for repeated cycles from the start.
- **Watch the NAT Gateway.** Roughly $32/month plus data processing, and it is the classic silent credit-killer — it costs more than the database. Use public subnets or VPC endpoints for a learning environment.
- Prefer the smallest viable instance classes; they teach the same lessons.
- Set a billing alarm before the first `terraform apply`.

## What was cut, and why

| Removed | Category | Reason |
| --- | --- | --- |
| Istio ambient | Service mesh | Platform/SRE domain. Enormous lift; duplicates application-layer lessons. |
| KEDA | Autoscaling | Niche. HPA teaches the concept. |
| Argo Rollouts | Progressive delivery | Platform-team concern. |
| Argo CD | GitOps | Genuinely useful, genuinely platform engineering. CD is learned adequately without a reconciler. |
| Jenkins | CI | **Reversal of ADR-022-R1.** A JCasC controller with ephemeral agents is one of the largest lifts in the plan, for a tool declining in postings. GitHub Actions teaches the identical separation of build from deploy. |
| Strimzi, RabbitMQ Operator | Operators | Learn the brokers, not their operators. |
| Traefik, Gateway API | Edge | Not the lesson. |
| Spring Cloud Gateway | Application gateway | An extra deployable for a pattern better learned later. |
| Loki, Tempo | Telemetry stores | Two more things to run; Prometheus and Grafana suffice. |
| MSK, Amazon MQ, Redis Cloud | Managed data | Budget. Self-host what you already know. |
| AMP, Managed Grafana, X-Ray | Managed telemetry | Budget and duplication. |
| OpenSearch / `search-service` | Search | Was already staged; removed entirely. |
| CloudFront, Karpenter, multi-region DR | AWS extras | Out of scope at this size. |

An exclusion is not an unresolved question. Reconsideration triggers are in [05](05-exclusions-and-open-questions.md).

## Deployables: seven to four

Volume is not only tools. Every service is an image, a chart, configuration, secrets, and a deployment.

| Deployable | Why it must be separate |
| --- | --- |
| `core-platform` | The Spring Modulith monolith: `profiles`, `catalog`, `listings`, `auctions`, `orders`. One local-transaction boundary. |
| `bidding-service` | The concurrency lesson. Independent write authority over bids, price, and winner; separate scaling and failure behavior. |
| `notification-service` | Consumes Kafka facts, decides policy, publishes RabbitMQ work. The bridge between the two brokers. |
| `notification-worker` | Competing consumers. **Runs at N replicas** — this is where work distribution becomes visible. |

`payment-service` becomes a module inside `core-platform` with a fake provider adapter; extract it later if provider isolation justifies it. `realtime-service` folds into `core-platform`, which runs multiple replicas so SSE fan-out remains a real problem. `api-gateway` and `search-service` are removed.

Every deployable retains an independent consistency, scaling, or failure responsibility. None exists for appearance.
