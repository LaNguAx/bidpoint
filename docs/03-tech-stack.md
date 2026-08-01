# Tech stack

Around thirty tools. The stack optimizes for **production-realistic** — running things the way real teams run them on Kubernetes, so nothing has to be unlearned later.

## The selection rule

Two filters, applied in order:

1. **Does this teach something on the list in [01 Thesis](01-thesis.md)?** If not, it's out. No exceptions for "industry standard."
2. **Is this how it's actually done?** Where a real team would reach for an operator, a managed service, or a specific pattern, do that — even when a simpler hack exists. The simpler hack teaches a habit you'd have to break.

There is a real cost to this: more Kubernetes surface to learn before writing domain code. The tradeoff was taken deliberately.

## Application

| Tool | Why |
| --- | --- |
| **Java (Temurin LTS)** | The job. |
| **Spring Boot** | The framework those jobs use. |
| **Spring Modulith** | Enforces module boundaries inside the monolith; provides durable event publication. |
| **Spring Data JPA** | Persistence. |
| **Flyway** | Versioned schema migration. |
| **Spring Security (resource server)** | JWT validation, independently, in every service. |
| **Spring for Apache Kafka** | Kafka integration. |
| **Spring AMQP** | RabbitMQ integration. |
| **Actuator + Micrometer** | Health, readiness, metrics. |
| **Micrometer Tracing / OpenTelemetry** | Trace context propagation across REST, Kafka, and RabbitMQ boundaries. |
| **AWS SDK for Java v2** | Used directly, so the cloud API stays visible. |
| **Maven (+ wrapper)** | The only build authority. No Nx, no pnpm, no Node tooling. |
| **Jib** | Container images from Maven, no Dockerfile. |

## Data and messaging — via operators

Everything stateful runs under a Kubernetes operator, because that is how it is actually done and because the alternative got worse.

**Why not "just use a community Helm chart":** Bitnami — whose charts were the de facto community charts for Postgres, Kafka, RabbitMQ, and Redis — deprecated its free catalog on 28 August 2025. Images moved to a `bitnamilegacy` repository that receives no updates and is being wound down through August 2026. The charts remain published but will not deploy without overriding every image reference by hand. That path is a dead end.

| Component | Operator | Why this one |
| --- | --- | --- |
| **Kafka** | **Strimzi** | CNCF-hosted, actively maintained, free, and the standard for Kafka on Kubernetes. Also *easier* than the alternative — a ~20-line `Kafka` custom resource replaces hand-rolled StatefulSets, storage, KRaft config, listeners, and rolling upgrades. |
| **PostgreSQL** | **CloudNativePG** | CNCF-hosted. The modern standard for Postgres on Kubernetes: failover, backups, point-in-time recovery, rolling minor upgrades. |
| **RabbitMQ** | **RabbitMQ Cluster Operator** | Official. Makes clustering and quorum queues tractable. |
| **Keycloak** | **Keycloak Operator** | Official. Declarative realm and client configuration. |
| **Redis** | *none* — plain Deployment, official image | Redis is non-authoritative here: cache and ephemeral coordination only. Losing it must never lose data, so HA machinery buys nothing. |

### The operator pattern is itself a lesson

A custom resource declares **desired state**. A controller watches actual state and continuously reconciles toward desired, converging after failure without human intervention.

That is the Kubernetes control loop, and it is the same reconciliation thinking behind the outbox relay and projection rebuilds in this system. It's a distributed-systems concept, not administrative trivia.

## Testing

| Tool | Role |
| --- | --- |
| **JUnit, Mockito, AssertJ** | Unit and behavior tests. |
| **Testcontainers** | Real PostgreSQL, Kafka, RabbitMQ, Redis in tests. No H2, no mocked infrastructure. |
| **Awaitility** | Asserting on asynchronous outcomes without `Thread.sleep`. |
| **WireMock** | Simulating external providers. |
| **Toxiproxy** | Injecting latency, partitions, and resets. |
| **ArchUnit** | Failing the build when a module boundary is violated. |
| **k6** | Load, concurrency, SSE. |
| **Spotless / JaCoCo** | Formatting gate; coverage as signal, never a target. |

Testing against real infrastructure via Testcontainers is a genuine interview differentiator — most candidates have only mocked.

**Tests do not need a cluster.** Testcontainers starts what they need and throws it away.

## Observability — the full LGTM stack

| Tool | Signal |
| --- | --- |
| **OpenTelemetry Collector** | Vendor-neutral collection for all three signals. The industry standard, and the seam that makes backends swappable. |
| **Prometheus** | Metrics — request rate, errors, latency, outbox age, consumer lag, queue depth. |
| **Loki** | Log aggregation, queryable across services. |
| **Tempo** | Distributed tracing. |
| **Grafana** | One pane over all three. |

Tracing earns its place *specifically* in this project. Following a single bid through REST → transaction → outbox → Kafka → `notification-service` → RabbitMQ → worker, as one connected trace, is the clearest possible demonstration of what's being built — and being able to show that trace is a strong interview artifact.

## Delivery

| Tool | Role |
| --- | --- |
| **GitHub Actions** | CI: build, test, image, push. |
| **Argo CD** | CD: GitOps reconciliation. Git holds desired state; Argo converges the cluster toward it. |
| **Jib** | Image build inside Maven. |
| **Helm** | Packaging services and dependencies. |

**Not Jenkins.** Actions + Argo CD is the modern production combination; Jenkins is the legacy one. Adding it would mean building a JCasC controller with ephemeral agents to learn a separation of build from deploy that Actions already teaches.

The flow: **push → Actions builds and tests → image to registry by digest → commit the digest to the GitOps repo → Argo CD reconciles the cluster.** CI never deploys directly. That separation is the point.

## Local environment

k3d runs a real Kubernetes cluster on the machine, free and disposable. This is where Kubernetes depth is built — unlimited iteration, no cost, delete and recreate freely.

| Tool | Role |
| --- | --- |
| **Docker** | Container runtime. |
| **k3d / k3s** | Local Kubernetes. |
| **kubectl** | Cluster interaction. |
| **Helm** | Installing operators and charts. |
| **Argo CD** | GitOps, learned here first. |

Operators install from their own official charts or manifests — Strimzi, CloudNativePG, RabbitMQ, Keycloak — none of which depend on the Bitnami catalog.

## Remote environment — AWS on EKS

The same manifests, operators, and Helm charts run locally and on AWS. One deployment model, not two.

| Tool | Role |
| --- | --- |
| **Terraform** | All AWS infrastructure. Written to be destroyed and recreated. |
| **Amazon EKS** | Kubernetes runtime. |
| **EKS managed node groups** | Node capacity. |
| **Amazon ECR** | Image registry. |
| **Amazon RDS PostgreSQL** | Production database. CloudNativePG runs locally; real teams use RDS in production, and so does this. |
| **Amazon S3** | Listing images. |
| **IAM + EKS Pod Identity** | Workload identity, least privilege. The AWS skill that transfers everywhere. |
| **AWS Secrets Manager** | Secret storage and injection. |
| **AWS Load Balancer Controller + ALB** | Public entry. |
| **CloudWatch Logs** | Where EKS logs land regardless. |

Kafka and RabbitMQ run **via the same operators on EKS** — not MSK and Amazon MQ. Managed brokers are the largest line items available, and the operators are already known from local work.

### Budget

**$100 in credits, and that is plenty — as long as nothing is left running.**

| Item | Hourly |
| --- | --- |
| EKS control plane | $0.10 |
| 2 × t3.medium nodes | ~$0.08 |
| ALB | ~$0.02 |
| **Total** | **~$0.21/hr** |

That's roughly **475 cluster-hours**, or about sixty 8-hour sessions. The always-on figure (~$150/month) is what makes EKS look unaffordable, and it's irrelevant if the cluster is destroyed at the end of every session.

Rules:

- **Create and destroy per session.** Terraform layout, seeding, and teardown designed for repeated cycles from the first commit, not retrofitted.
- **Avoid the NAT Gateway** — ~$0.045/hr plus data processing, more than the nodes. Use public subnets or VPC endpoints for a learning environment.
- **Set a billing alarm before the first `terraform apply`.** Not after.
- Smallest viable instance classes. They teach the same lessons.

*Verify these prices against the AWS pricing pages before relying on them — they are estimates and they change.*

## Versions

- **Behavioral tools** (PostgreSQL, Kafka, RabbitMQ, Redis, Keycloak, Kubernetes) — pin to current stable. Their behavior *is* the lesson.
- **Operators** (Strimzi, CloudNativePG, RabbitMQ, Keycloak) — pin exactly, and check the compatibility matrix; each supports a specific range of the thing it operates.
- **Frameworks** (Spring Boot, Spring Modulith) — pin **one minor behind current**. Settled releases have documentation and answered questions.
- **Tooling** (Maven, Jib, Spotless, ArchUnit, Testcontainers) — pin loosely, float patches.
- **AWS** — verify availability, add-on compatibility, and quotas at the time of use.

**Never use `latest`. Never infer a compatible version pair from a general project page.** Check current releases before naming a version — this changes faster than any model's training data, and the Bitnami situation above is exactly what stale assumptions cost.

## Still excluded

| Not included | Why |
| --- | --- |
| Istio / service mesh | Large lift, and its lessons are proven at the application layer here. The most likely candidate to add later. |
| KEDA | HPA teaches the autoscaling concept. |
| Argo Rollouts | Progressive delivery on top of GitOps; add only after ordinary deployment works. |
| Jenkins | Actions + Argo CD is the modern combination. |
| MSK, Amazon MQ, ElastiCache | Budget, and the operators already teach the operational side. |
| OpenSearch / full-text search | Browse and filter covers discovery. |
| CloudFront, Karpenter, multi-region DR | Out of scope at this size. |
| Nx, pnpm, Node tooling | No frontend exists. Maven alone is the build. |
