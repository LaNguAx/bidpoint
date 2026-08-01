# Tech stack

Roughly twenty tools. Nothing here is obscure.

## The selection rule

Every tool has to pass one test:

> **Does this appear in backend engineering job postings, or is it platform/SRE tooling that is industry-standard in large organizations but rarely asked of a backend developer?**

The second category is where side-project stacks go to die. Service meshes, GitOps reconcilers, operators, and progressive-delivery controllers are legitimate, widely deployed, and almost never asked of a backend candidate. An earlier version of this stack had about forty tools and a third of them were in that category.

The other rule, from [01 Thesis](01-thesis.md): every tool must map to a lesson. If it can't, it's removed — not deferred.

## Application

| Tool | Why |
| --- | --- |
| **Java (Temurin LTS)** | The job. |
| **Spring Boot** | The framework those jobs use. |
| **Spring Modulith** | Enforces module boundaries inside the monolith, and provides durable event publication. |
| **Spring Data JPA** | Persistence. |
| **Flyway** | Versioned schema migration — how real teams change databases. |
| **Spring Security (resource server)** | JWT validation in every service independently. |
| **Spring for Apache Kafka** | Kafka integration. |
| **Spring AMQP** | RabbitMQ integration. |
| **Actuator + Micrometer** | Health, readiness, metrics. |
| **AWS SDK for Java v2** | Used directly rather than wrapped, so the cloud API stays visible. |
| **Maven (+ wrapper)** | The only build authority. No Nx, no pnpm, no Node tooling. |
| **Jib** | Builds container images from Maven with no Dockerfile. |

## Data and messaging

| Tool | Role |
| --- | --- |
| **PostgreSQL** | Authoritative storage and outboxes. Transactions and locking are core lessons. |
| **Kafka** | Durable replayable facts. The highest-value distributed item on a CV. |
| **RabbitMQ** | Targeted work to competing workers. Plain broker, no cluster. |
| **Redis** | Caching and ephemeral coordination. Never authoritative. |
| **Keycloak** | OIDC/OAuth2 identity. One realm, one client — deliberately simple. |

Both brokers are kept on purpose. See [02 Architecture](02-architecture.md) for why they are not redundant.

## Testing

| Tool | Role |
| --- | --- |
| **JUnit, Mockito, AssertJ** | Unit and behavior tests. |
| **Testcontainers** | Real PostgreSQL, Kafka, RabbitMQ, and Redis in tests. No H2, no mocks for infrastructure. |
| **Awaitility** | Asserting on asynchronous outcomes without `Thread.sleep`. |
| **WireMock** | Simulating external providers. |
| **Toxiproxy** | Injecting network failure — latency, partitions, resets. |
| **ArchUnit** | Failing the build when a module boundary is violated. |
| **k6** | Load testing, concurrency, SSE. |
| **Spotless / JaCoCo** | Formatting gate; coverage as diagnostic signal, never a target. |

Testing against real infrastructure via Testcontainers is a genuine interview differentiator — most candidates have only mocked.

## Local environment

**Kubernetes from day one.** k3d runs a real cluster on your machine for free, with unlimited iteration. This is where Kubernetes depth gets built.

| Tool | Role |
| --- | --- |
| **Docker** | Container runtime. |
| **k3d / k3s** | Local Kubernetes cluster. Disposable — delete and recreate freely. |
| **kubectl** | Cluster interaction. |
| **Helm** | Packaging your services, and installing dependencies. |
| **Prometheus + Grafana** | Metrics and dashboards: request rates, error rates, latency, outbox age, consumer lag, queue depth. |

PostgreSQL, Kafka, RabbitMQ, Redis, and Keycloak install **from community Helm charts** — no operators. The goal is to learn the brokers, not their Kubernetes operators.

**Ordinary tests don't need the cluster at all** — Testcontainers spins up what they need and throws it away.

## Remote environment (AWS)

**ECS Fargate is the target, not EKS.**

An EKS control plane costs roughly **$73/month before any compute**, against **$100 in total credits**. That's five weeks of burn for nothing else. ECS Fargate has no control-plane fee and still teaches everything an interview probes: IAM roles, task definitions, ECR, VPC and networking, RDS connectivity, secret injection, and log shipping.

The deeper point: **learning Kubernetes and learning AWS are different goals.** Kubernetes is learned locally on k3d where iteration is free. AWS is learned on managed services that are cheap to create and destroy. EKS is where they meet — once, briefly, at the end.

| Tool | Role |
| --- | --- |
| **Terraform** | All AWS infrastructure. Written to be destroyed and recreated. |
| **ECR** | Container image registry. |
| **ECS Fargate** | Primary compute. No control-plane fee. |
| **RDS PostgreSQL** | Managed database. |
| **S3** | Listing images. |
| **IAM** | Roles, task roles, least privilege. The AWS skill that transfers everywhere. |
| **Secrets Manager** | Secret storage and injection. |
| **CloudWatch Logs** | Where logs land. |
| **ALB + VPC** | Public entry and networking. |
| **GitHub Actions** | CI and deployment. |
| **EKS** | **One bounded exercise:** stand it up, deploy the same Helm charts used locally, document the differences, tear it down. Budget ~$20 of credits. |

Kafka and RabbitMQ run as **containers on AWS**, not MSK and Amazon MQ. Managed brokers are the largest line items available, and you'll already know both operationally from local work.

## Budget discipline

$100 in credits is enough — if infrastructure is treated as disposable.

- **Create and destroy per session.** Never leave it running. Terraform layout, data seeding, and teardown must be designed for repeated cycles from the very first commit, not retrofitted.
- **Watch the NAT Gateway.** Roughly **$32/month plus data processing** — more than the database, and the classic silent credit-killer. Use public subnets or VPC endpoints for a learning environment.
- **Set a billing alarm before the first `terraform apply`.** Not after.
- Smallest viable instance classes. They teach the same lessons.

## Versions

- **Behavioral tools** (PostgreSQL, Kafka, RabbitMQ, Redis, Keycloak, Kubernetes) — pin to current stable. Their behavior *is* the lesson, so drift changes the answer.
- **Frameworks** (Spring Boot, Spring Modulith) — pin **one minor behind current**. Settled releases have documentation, examples, and answered questions. Debugging a brand-new major teaches nothing about distributed systems.
- **Tooling** (Maven, Jib, Spotless, ArchUnit, Testcontainers) — pin loosely, float patches.
- **AWS managed services** — verify availability, compatibility, and quotas at the time you use them.

**Never use `latest`. Never infer a compatible version pair from a general project page.** Check current releases before naming a version — the answer changes faster than any model's training data.

## What was cut, and why

| Removed | Why |
| --- | --- |
| Istio / service mesh | Platform and SRE domain. Enormous lift; duplicates lessons proven at the application layer. |
| KEDA | Niche autoscaler. HPA teaches the concept. |
| Argo CD, Argo Rollouts | GitOps reconciliation and progressive delivery are platform-team concerns. |
| Jenkins | A JCasC controller with ephemeral agents is one of the biggest lifts available, for a declining tool, teaching a build-vs-deploy separation GitHub Actions teaches in an afternoon. |
| Strimzi, RabbitMQ Operator | Learn the brokers, not their operators. Operator lifecycle is a Kubernetes-admin skill. |
| Spring Cloud Gateway / `api-gateway` | A whole extra deployable for a pattern better learned once the system exists. |
| Traefik / Gateway API as topics | Use whatever k3d ships with. The edge is plumbing, not the lesson. |
| Loki, Tempo | Two more stores to run. Prometheus and Grafana are enough; tracing is the first thing to add back if there's room. |
| MSK, Amazon MQ, Redis Cloud | Budget. Self-host what you already know. |
| AMP, Managed Grafana, X-Ray | Budget and duplication. |
| OpenSearch / full-text search | Browse and filter covers discovery. |
| CloudFront, Karpenter, multi-region DR | Out of scope at this size. |
| Nx, pnpm, Node build tooling | No frontend exists. Maven alone is the build. |

None of these are bad technology. They're just not what this project is for.
