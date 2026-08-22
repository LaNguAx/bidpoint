# Tech stack

Two sections: **[Local](#local)** and **[Remote](#remote)**. Each lists its complete stack line by line. They overlap heavily — that duplication is deliberate, so neither section has to be read in terms of the other.

The stack optimizes for **production-realistic** — running things the way real teams run them on Kubernetes, so nothing has to be unlearned later.

## The selection rule

Two filters, applied in order:

1. **Does this teach something on the list in [01 Thesis](01-thesis.md)?** If not, it's out. No exceptions for "industry standard."
2. **Is this how it's actually done?** Where a real team would reach for an operator, a managed service, or a specific pattern, do that — even when a simpler hack exists. The simpler hack teaches a habit you'd have to break.

There is a real cost to this: more Kubernetes surface to learn before writing domain code. The tradeoff was taken deliberately.

---

# Local

k3d runs a real Kubernetes cluster on the machine, free and disposable. This is where Kubernetes depth is built — unlimited iteration, no cost, delete and recreate freely.

## Application

Everything here ships inside the four deployables plus the gateway.

| Tool | Why |
| --- | --- |
| **Amazon Corretto 25 (LTS)** | AWS's OpenJDK build, so the local JDK matches what runs on EKS. Virtual threads no longer pin on `synchronized` at this level. |
| **Spring Boot 4.0.x** | Not 4.1. The Spring Cloud release train (**2025.1.x**) is what Gateway, Spring Cloud AWS, and Spring Cloud Contract are version-locked to, and it targets Boot 4.0.x / Framework 7.0.x. Taking 4.1 means going off-train and overriding managed versions by hand. |
| **Spring Modulith** | Enforces module boundaries inside `core-platform`. `ApplicationModules.verify()` fails the build on a violation — this replaces ArchUnit entirely. |
| **Spring MVC + virtual threads** | Blocking code, non-blocking scale. SSE connections stop costing a platform thread each. Not WebFlux — reactive would make JPA, stack traces, and debugging harder for throughput this system doesn't need. |
| **Spring Cloud Gateway (WebFlux, 5.0.x)** | Edge routing, token relay, per-route Java filters, Resilience4j at the edge. Requires Netty — the `-server-webflux` starter will not run in a servlet container. **Every service still validates its own JWT**; the gateway routes, it never authorizes on anyone's behalf. |
| **Spring Data JPA / Hibernate** | Persistence. No `JdbcClient` yet — revisit if A3's measurement says the atomic conditional update is the winning concurrency mechanism. |
| **Flyway** | Versioned schema migration. Numbered SQL files applied once, in order, tracked in `flyway_schema_history`. Prevents schema drift between local, k3d, and RDS, and makes schema changes reviewable in git. Hibernate's `ddl-auto` is never used outside throwaway experiments. |
| **Spring Data Redis (Lettuce)** | SSE fan-out view and ephemeral coordination. |
| **Spring Security (resource server)** | JWT validation, independently, in every service. No gateway doing it once on everyone's behalf. |
| **Spring for Apache Kafka** | Facts. |
| **Spring AMQP** | Work. |
| **Jackson + a versioned event envelope** | Event serialization. The schema is a versioned artifact in the repo; a test proves a v2 consumer still reads v1. No schema registry — see [Excluded](#deliberately-excluded). |
| **Hand-rolled transactional outbox** | In `bidding-service`. Modulith's `@Externalized` would do it for you and you would learn nothing — the outbox *is* the A5 lesson. Modulith's event registry stays available for the cases where the lesson isn't. |
| **Conditional-claim scheduling** | `core-platform` runs N replicas, so `@Scheduled` fires auction close on every one of them. Fixed in SQL — `UPDATE auction SET state='CLOSING' WHERE id=? AND state='OPEN'` — losers update zero rows and no-op. Same optimistic-concurrency mechanism as the A3 bidding path. No ShedLock. |
| **Resilience4j** | Timeout, exponential backoff with jitter, circuit breaker, bulkhead. Covers the fence/finalize call, payment provider calls, and notification provider calls. Spring Retry only retries; the lesson here is what happens when retrying makes an outage worse. |
| **`RestClient` + `@HttpExchange`** | Declarative service-to-service HTTP. `RestTemplate` is in maintenance mode. |
| **Bean Validation** | Request validation at the edge. |
| **Problem Details (RFC 9457)** | Standardized error bodies. Built into Spring Framework, no dependency. |
| **Structured JSON logging** | Native to Spring Boot, one property. Loki is not queryable without structured fields. |
| **Actuator + Micrometer** | Health, readiness, metrics. |
| **Micrometer Tracing + OpenTelemetry** | Trace context propagation across REST, Kafka, and RabbitMQ boundaries. |
| **Spring Cloud AWS 4.x** | S3 and Secrets Manager integration. It is **built on AWS SDK v2**, not a replacement for it — you get autoconfigured clients and a Secrets Manager property source, and can still inject the raw SDK client when you want the API visible. |
| **Gradle (+ wrapper)** | The only build authority. No Nx, no pnpm, no Node tooling. |
| **Jib (Gradle plugin)** | Container images from the build. No Dockerfile, reproducible layers. |
| **springdoc-openapi** | Generated REST contract. Weak learning value, good artifact. |
| **Lombok** | Boilerplate. Note: with Gradle, `annotationProcessor` ordering against MapStruct matters or the mappers generate empty. |
| **MapStruct** | DTO ↔ entity mapping. |

## Cluster

| Tool | Why |
| --- | --- |
| **Docker Desktop (WSL2)** | Container runtime on Windows 11. |
| **k3d (k3s in Docker)** | Real multi-node Kubernetes in seconds, disposable, with a built-in registry and load balancer. |
| **kubectl** | Cluster interaction. |
| **Helm** | Installing operators and vendor charts. |
| **Helm for your own services too** | One chart per service, values per environment. Matches "same manifests, only values differ" without a second templating tool. Argo CD would support Kustomize equally well; the point is picking one. |
| **Gateway API (v1.5) + Traefik v3** | Gateway API is the *specification*; Traefik is the controller implementing it. You write `Gateway` and `HTTPRoute`, never `Ingress`. Reasons below. |
| **k3d built-in registry** | Jib pushes here, pods pull locally. No internet in the inner loop. |

### Why Gateway API and not Ingress

Four reasons, in order of weight:

1. **Parity.** The AWS Load Balancer Controller has GA Gateway API support (GA in controller v3.0.0; ALB serving L7 `HTTPRoute`/`GRPCRoute` since v2.14.0). The *same `HTTPRoute` manifests* run on k3d behind Traefik and on EKS behind an ALB — only the `GatewayClass` differs. With `Ingress` you would write Traefik annotations locally and ALB annotations remotely: two sets of files.
2. **`Ingress` is feature-frozen.** It still works, but all development moved to Gateway API, which reached v1.5 in February 2026.
3. **`ingress-nginx` retired in March 2026.** The most widely deployed ingress controller in the world stopped being maintained. Learning the Ingress resource now means learning what the ecosystem is migrating off.
4. **It teaches a role split `Ingress` cannot express.** Platform owns `GatewayClass` and `Gateway`; app teams own `HTTPRoute` and attach to it. That separation is a real multi-tenant-cluster concept.

**Cost:** k3s installs the Gateway API CRDs but leaves the provider off. Enable it with a `HelmChartConfig` in `kube-system` setting `providers.kubernetesGateway.enabled: true` — about ten lines, once.

`HTTPRoute` overlaps Spring Cloud Gateway on path routing, header matching, and traffic splitting. What Spring Cloud Gateway still uniquely buys: token relay, Java-written filters, request aggregation, and Resilience4j at the edge. Be able to say why both layers exist.

## Stateful dependencies — via operators

Everything stateful runs under a Kubernetes operator, because that is how it is actually done and because the alternative got worse.

**Why not "just use a community Helm chart":** Bitnami — whose charts were the de facto community charts for Postgres, Kafka, RabbitMQ, and Redis — deprecated its free catalog on 28 August 2025. Images moved to a `bitnamilegacy` repository that receives no updates and is being wound down through August 2026. The charts remain published but will not deploy without overriding every image reference by hand. That path is a dead end.

| Component | How it runs | Why |
| --- | --- | --- |
| **PostgreSQL** | **CloudNativePG** | CNCF-hosted. Failover, backups, point-in-time recovery, rolling minor upgrades. Local Postgres runs here *specifically to learn how to set a database up correctly* — remote uses RDS, and the divergence is intentional. |
| **Kafka** | **Strimzi** | CNCF-hosted, actively maintained, free, and the standard for Kafka on Kubernetes. A ~20-line `Kafka` custom resource replaces hand-rolled StatefulSets, storage, KRaft config, listeners, and rolling upgrades. Redpanda would be lighter on a laptop but it isn't Kafka, and it breaks parity with EKS. |
| **RabbitMQ** | **RabbitMQ Cluster Operator** | Official. Makes clustering and quorum queues tractable. |
| **Keycloak** | **Keycloak Operator** | Official. `KeycloakRealmImport` makes realm and client configuration declarative. Needs a Postgres — CloudNativePG provides it. |
| **Redis** | plain Deployment, official image | Non-authoritative: cache and ephemeral coordination only. Losing it must never lose data, so HA machinery buys nothing. Redis 8 added AGPLv3 in May 2025, so current Redis is OSI open source again — Valkey is not needed. Verify the license on the exact version at pin time. |

### The operator pattern is itself a lesson

A custom resource declares **desired state**. A controller watches actual state and continuously reconciles toward desired, converging after failure without human intervention.

That is the Kubernetes control loop, and it is the same reconciliation thinking behind the outbox relay and projection rebuilds in this system. It's a distributed-systems concept, not administrative trivia.

## Secrets

A Kubernetes `Secret` holds key→value pairs **base64-encoded — that is encoding, not encryption**. Pods consume them as env vars or files, and Spring Boot picks them up with zero code. Consumption is the easy half.

Delivery is the hard half. Argo CD applies whatever is in git, so a `Secret` manifest in git means the password is in git. The value has to reach the cluster by a path that isn't the repo.

| Tool | Why |
| --- | --- |
| **External Secrets Operator** | You commit an `ExternalSecret` CR — *"fetch `db-password` from store X, create Secret Y"* — and the operator pulls the value at runtime. The value never touches git. A separate `SecretStore` CR says what X is. |
| **HashiCorp Vault (dev mode)** | The local backing store. One pod, in-cluster, throwaway. |

The `ExternalSecret` files are byte-identical across environments; only the `SecretStore` changes — Vault locally, AWS Secrets Manager remotely. That is exactly "same manifests, only values differ."

Sealed Secrets was rejected because its ciphertext is per-cluster, so local and remote would need different sealed files. Spring Cloud Config Server solves config distribution for non-Kubernetes deployments; ConfigMaps and Secrets already do that here.

## Build and delivery

| Tool | Why |
| --- | --- |
| **Gradle (+ wrapper)** | The build. |
| **Jib (Gradle plugin)** | Image build inside the build. |
| **GitHub Actions** | CI: build, test, image, digest commit. Runs on GitHub but gates both environments. |
| **Argo CD** | CD: GitOps reconciliation. Git holds desired state; Argo converges the cluster toward it. Learned locally first. Flux is lighter and more purist, but Argo's UI is a materially better learning and demo artifact. |
| **Single repository** | Application code and deployment manifests live together. |

**Not Jenkins.** Actions + Argo CD is the modern production combination; Jenkins is the legacy one. Adding it would mean building a JCasC controller with ephemeral agents to learn a separation of build from deploy that Actions already teaches.

The flow: **push → Actions builds and tests → image to registry by digest → commit the digest → Argo CD reconciles the cluster.** CI never deploys directly. That separation is the point.

**One repo means the CI loop must be broken explicitly.** Actions commits the image digest back to the same repo, which would retrigger Actions. Fix it with `paths-ignore` on the deployment directory plus `[skip ci]` in the digest commit message. This has to exist from B1 or the pipeline loops.

## Observability — the full LGTM stack

| Tool | Why |
| --- | --- |
| **OpenTelemetry Operator** | Manages the Collector; can auto-instrument. |
| **OpenTelemetry Collector** | Vendor-neutral collection for all three signals. The seam that makes backends swappable. |
| **OTLP for everything** | Applications push traces, metrics, **and** logs as OTLP to the Collector, which fans out. One path to reason about instead of one for traces and a Prometheus scrape for metrics. |
| **kube-prometheus-stack** | Prometheus Operator, Grafana, node-exporter, kube-state-metrics, and default dashboards in one chart. Metrics: request rate, errors, latency, outbox age, consumer lag, queue depth. |
| **Loki** (`SingleBinary` locally) | Log aggregation, queryable across services. |
| **Tempo** (monolithic locally) | Distributed tracing. |
| **Grafana** | One pane over all three. Ships with kube-prometheus-stack. |

Tracing earns its place *specifically* in this project. Following a single bid through REST → transaction → outbox → Kafka → `notification-service` → RabbitMQ → worker, as one connected trace, is the clearest possible demonstration of what's being built — and being able to show that trace is a strong interview artifact.

## Testing

**Tests do not need a cluster.** Testcontainers starts what they need and throws it away.

| Tool | Why |
| --- | --- |
| **JUnit 5** | Test framework. |
| **AssertJ** | Fluent assertions. |
| **Mockito** | Mocking collaborators — never infrastructure. |
| **Testcontainers + `@ServiceConnection`** | Real PostgreSQL, Kafka, RabbitMQ, Redis, and Keycloak in tests. Boot wires the connection automatically. No H2, no embedded brokers, no mocked infrastructure. |
| **Awaitility** | Asserting on asynchronous outcomes without `Thread.sleep`. |
| **WireMock** | Simulating external providers. |
| **Toxiproxy** | Injecting latency, partitions, and resets — the failure-injection evidence. |
| **Spring Modulith `ApplicationModules.verify()`** | Fails the build when a module boundary is violated. Replaces ArchUnit. |
| **Spring Cloud Contract** | Producer-driven contract tests across the fence/finalize REST boundary and Kafka event shapes. This is what catches producer/consumer drift in the absence of a schema registry. JVM-native and on the same 2025.1.x train. |
| **k6** | Load, concurrency, SSE. `k6-operator` runs it in-cluster. |
| **Pitest** | Mutation testing — proves the tests actually fail when the code breaks. Fits "evidence, not coverage" better than JaCoCo does. |
| **Spotless** | Formatting gate. |
| **JaCoCo** | Coverage as signal, never a target. |

Testing against real infrastructure via Testcontainers is a genuine interview differentiator — most candidates have only mocked.

## Developer ergonomics

| Tool | Why |
| --- | --- |
| **k9s** | Terminal UI over the cluster. The largest quality-of-life win on this list. |
| **stern** | Multi-pod log tailing. |
| **Tilt** | Rebuild and redeploy on save. Without it the inner loop is `gradle jib` and then waiting for an Argo sync. |

---

# Remote

AWS on EKS. Three environments — `dev`, `stage`, `prod` — as three namespaces on **one** cluster, each with its own Kafka, RabbitMQ, Keycloak, and RDS instance. All three run simultaneously. Roughly 55–60 pods in total.

The same manifests, operators, and Helm charts run here and locally. One deployment model, not two.

Local's **Testing** and **Developer ergonomics** sections have no counterpart here — tests run against Testcontainers and never touch a cluster, and the inner loop is a local concern by definition. Everything else appears in both.

## Application

Identical to Local, reproduced rather than cross-referenced.

| Tool | Why |
| --- | --- |
| **Amazon Corretto 25 (LTS)** | AWS's OpenJDK build, so the local JDK matches what runs on EKS. Virtual threads no longer pin on `synchronized` at this level. |
| **Spring Boot 4.0.x** | Not 4.1. The Spring Cloud release train (**2025.1.x**) is what Gateway, Spring Cloud AWS, and Spring Cloud Contract are version-locked to, and it targets Boot 4.0.x / Framework 7.0.x. |
| **Spring Modulith** | Enforces module boundaries inside `core-platform`. `ApplicationModules.verify()` fails the build on a violation. |
| **Spring MVC + virtual threads** | Blocking code, non-blocking scale. SSE connections stop costing a platform thread each. |
| **Spring Cloud Gateway (WebFlux, 5.0.x)** | Edge routing, token relay, per-route Java filters, Resilience4j at the edge. **Every service still validates its own JWT.** |
| **Spring Data JPA / Hibernate** | Persistence. |
| **Flyway** | Versioned schema migration. Also the mechanism that rebuilds each RDS instance from empty on every session — see [Stateful dependencies](#stateful-dependencies) below. |
| **Spring Data Redis (Lettuce)** | SSE fan-out view and ephemeral coordination. |
| **Spring Security (resource server)** | JWT validation, independently, in every service. |
| **Spring for Apache Kafka** | Facts. |
| **Spring AMQP** | Work. |
| **Jackson + a versioned event envelope** | Event serialization. No schema registry. |
| **Hand-rolled transactional outbox** | In `bidding-service`. The outbox *is* the A5 lesson. |
| **Conditional-claim scheduling** | `UPDATE auction SET state='CLOSING' WHERE id=? AND state='OPEN'`. N replicas, one winner, no distributed lock. |
| **Resilience4j** | Timeout, exponential backoff with jitter, circuit breaker, bulkhead. |
| **`RestClient` + `@HttpExchange`** | Declarative service-to-service HTTP. |
| **Bean Validation** | Request validation at the edge. |
| **Problem Details (RFC 9457)** | Standardized error bodies. |
| **Structured JSON logging** | Native to Spring Boot. Loki is not queryable without structured fields. |
| **Actuator + Micrometer** | Health, readiness, metrics. |
| **Micrometer Tracing + OpenTelemetry** | Trace context propagation across REST, Kafka, and RabbitMQ boundaries. |
| **Spring Cloud AWS 4.x** | S3 and Secrets Manager integration, built on AWS SDK v2. |
| **Gradle (+ wrapper)** | The only build authority. |
| **Jib (Gradle plugin)** | Container images from the build. |
| **springdoc-openapi** | Generated REST contract. |
| **Lombok** | Boilerplate. |
| **MapStruct** | DTO ↔ entity mapping. |

**Three deltas from Local:**

- **Spring Cloud AWS points at real S3, Secrets Manager, and Parameter Store** instead of local stubs. Credentials come from Pod Identity, never from configuration.
- **Images are `arm64`.** See [Cluster](#cluster-1).
- **JVM heap is set with `-XX:MaxRAMPercentage`, never a fixed `-Xmx`.** Node memory is the binding constraint here in a way it never is locally — ~60 JVM pods against ~35 GiB of schedulable memory. A container-unaware JVM will claim a quarter of the *node* and the scheduler will not stop it.

## Cluster

| Tool | Why |
| --- | --- |
| **Amazon EKS** | Kubernetes runtime. |
| **EKS managed node groups** | Node capacity, explicitly *not* Auto Mode — see below. |
| **Graviton (`t4g.large`), ~5 nodes** | ~20% cheaper than the x86 equivalent, which matters when three environments run at once. |
| **Spot for dev/stage, On-Demand for prod** | The single biggest cost lever available: ~$0.24/hr saved. Interruption handling is a genuine lesson, and confining On-Demand to prod is what a real team does anyway. |
| **EKS add-ons** | VPC CNI, kube-proxy, CoreDNS, EBS CSI driver (PVCs for Kafka, RabbitMQ, Prometheus), **EKS Pod Identity Agent**. |
| **gp3 storage class** | Cheaper and faster than gp2, and the default worth knowing. |

**Graviton has a consequence worth stating plainly:** local is `amd64` (Windows/WSL2), remote is `arm64`. **Jib must produce multi-arch or arm64-targeted images**, and an image that only exists for `amd64` will schedule and then crash-loop with an exec format error. Real friction, real lesson, and reversible to `t3.large` at roughly 24% more cost if it proves more trouble than it's worth.

**Why not EKS Auto Mode.** It adds a management fee of almost exactly **12% on top of the EC2 price** — the published rate for `t4g.medium` is $0.00403/hr against an instance price of $0.0336 — and takes over node provisioning, scaling, and patching, which is precisely the surface being learned here. Correct choice for a team that wants to stop thinking about nodes; wrong choice for someone whose goal is to understand them.

**Watch the Kubernetes version.** EKS control plane is **$0.10/hr during standard support** (the first 14 months a version is available). A cluster left on an old version drops into **extended support at $0.60/hr — 6×**. On an ephemeral cluster this is unlikely to bite, but the Terraform default must be pinned to a currently-supported version rather than left to drift.

**Pod density** is not expected to be the constraint — `t4g.large` allows 35 pods and memory runs out first. If that inverts, VPC CNI **prefix delegation** is the lever.

## Networking and entry

| Tool | Why |
| --- | --- |
| **VPC, public subnets, no NAT Gateway** | See the arithmetic below. |
| **S3 gateway endpoint** | Free, and it keeps ECR layer pulls and Loki/Tempo traffic off the public path. |
| **AWS Load Balancer Controller + ALB** | Serves the **same `HTTPRoute` resources written locally**. Only the `GatewayClass` differs. Requires controller v2.14+ for L7 Gateway API. |
| **One shared `Gateway`, three namespaces** | A single `Gateway` in a platform namespace; `dev`, `stage`, and `prod` each attach their own `HTTPRoute` with their own hostname, authorized by the listener's `allowedRoutes.namespaces` selector. Saves ~$0.045/hr against one ALB per environment, and cross-namespace route attachment is a Gateway API capability `Ingress` has no equivalent for. |
| **external-dns** | Required, not optional. The ALB's DNS name changes on every recreate, so a static Route53 record breaks at the start of every session. |
| **Route53 hosted zone** | `dev.` / `stage.` / `prod.` subdomains under a domain you already own. |
| **ACM wildcard certificate** | `*.<subdomain>`, DNS-validated, free. One cert covers all three environments. Lives in the persistent stack. |

**Why no NAT Gateway, and why VPC endpoints aren't the answer either.** A NAT Gateway is **$0.045/hr plus $0.045/GB processed** — roughly two-thirds the cost of an entire `t4g.large` node, for a component whose only job is letting private subnets reach the internet.

The instinctive fix is private subnets plus interface endpoints, and it is *worse*. Interface endpoints are **$0.01/hr each, billed per availability zone**, plus $0.01/GB processed. ECR api, ECR dkr, STS, Secrets Manager, and CloudWatch Logs across two AZs is ten endpoint-hours — **$0.10/hr, more than double the NAT Gateway it would replace.** The S3 *gateway* endpoint is the exception that makes this workable: it is free, and it covers ECR layer pulls (which are S3-backed) along with all Loki and Tempo traffic.

So: **nodes in public subnets, guarded by restrictive security groups, plus the free S3 gateway endpoint.** A production system would run private subnets with a NAT Gateway and eat the cost. This is a deliberate learning-environment tradeoff, and it is the one place in this document where the cheap answer wins over the realistic one.

## Stateful dependencies

| Component | How it runs | Why |
| --- | --- | --- |
| **PostgreSQL** | **RDS, 3 × db.t4g.micro, Single-AZ** | One instance per namespace. No Multi-AZ — it doubles both the instance and the storage rate ($0.115 → $0.23/GB-mo), and failover was already taught by CloudNativePG locally. Paying twice to learn it once is not a tradeoff. **CloudNativePG runs locally specifically to learn how to set a database up; RDS is what production uses.** The divergence is the point. |
| **Kafka** | **Strimzi**, one cluster per namespace | Same operator, same custom resources as local. |
| **RabbitMQ** | **RabbitMQ Cluster Operator**, per namespace | Same. |
| **Keycloak** | **Keycloak Operator**, per namespace | Same. The domain is what gives it a stable issuer URL across recreates — without one, every teardown invalidates every token and redirect. |
| **Redis** | plain Deployment, per namespace | Same. Still non-authoritative. |
| **Amazon S3** | Listing image objects | `listings` owns the metadata. |

**Kafka and RabbitMQ run via the same operators on EKS — not MSK and Amazon MQ.** Managed brokers are the largest line items available, and the operators are already known from local work.

**`db.t4g.micro` has 1 GiB of RAM, and that is the number to watch.** Each instance carries three logical databases (`core-platform`, `bidding`, `notification`) and takes connections from every replica of every service in its namespace. Postgres sizes `max_connections` from instance memory, so at 1 GiB the ceiling is around 110 — reachable with five services holding default HikariCP pools of ten across multiple replicas. **Set pool sizes explicitly rather than accepting defaults**, and expect to move to `db.t4g.small` (2 GiB, $0.032/hr, +$0.048/hr across three) if connection exhaustion shows up. Sizing a connection pool against a real `max_connections` is a worthwhile lesson in its own right; discovering it through a production outage is not.

**RDS is now the slowest part of a session.** Instance creation and deletion run 10–15 minutes each; three of them set the floor on how fast a cluster can be stood up and torn down. Budget for it rather than being surprised by it.

**Data does not survive a session, by design.** Flyway migrations plus a seed job rebuild schema and test data on every create. Final-snapshot-and-restore would preserve data and was passed over deliberately: recreating from empty keeps the seeding and migration path exercised on every single cycle, which is the entire point of infrastructure "written to be destroyed and recreated." A path run twice a week does not rot; a path run twice a year does.

## Secrets

Same operator, same `ExternalSecret` resources as local. Only the `SecretStore` changes — Vault locally, AWS here.

| Tool | Why |
| --- | --- |
| **External Secrets Operator** | Identical `ExternalSecret` files to local. The backing store is the only difference. |
| **SSM Parameter Store (standard tier)** — dev and stage | Free. SecureString values encrypted with KMS. |
| **AWS Secrets Manager** — prod only | $0.40/secret/month. Across three environments that would be ~$9.60/month — about 10% of the entire budget — for something Parameter Store does for nothing. |
| **IAM + EKS Pod Identity** | Workload identity, least privilege. No static keys anywhere. |

**Splitting the tiers is deliberate, not just thrift.** ESO speaks both providers, so it costs nothing in complexity, and running both is the only way to actually learn where the line is: Secrets Manager buys rotation, cross-account sharing, and versioning; Parameter Store buys free. Knowing which one a given secret deserves is the skill.

**Pod Identity, not IRSA.** No per-role OIDC trust policies, no service-account annotation juggling. Consumers: ESO, AWS Load Balancer Controller, external-dns, the EBS CSI driver, Loki and Tempo (S3), and the application itself (listing images).

## Build and delivery

| Tool | Why |
| --- | --- |
| **GitHub Actions + IAM OIDC** | Actions assumes a repo-scoped IAM role to push to ECR. **No static AWS keys in GitHub.** |
| **Amazon ECR** | One repository per deployable, with a lifecycle policy expiring untagged images so storage doesn't creep. |
| **Argo CD + ApplicationSet** | A list generator produces three Applications from one Helm chart and three values files. |
| **Helm** | Same charts as local, different values. |

**Promotion is a pull request.** CI commits the image digest to `deploy/dev/values.yaml`. Promoting to stage is a PR copying that digest to `deploy/stage/values.yaml`; prod likewise. Nothing is rebuilt between environments — the *same digest* moves forward, which is what makes the promotion meaningful. Rollback is `git revert`.

The flow is unchanged from local: **push → Actions builds and tests → image to ECR by digest → commit the digest → Argo CD reconciles.** CI never holds cluster credentials.

## Observability

Same LGTM stack, three changes.

| Tool | Why |
| --- | --- |
| **OpenTelemetry Operator + Collector** | Unchanged. All signals over OTLP. |
| **kube-prometheus-stack** | Unchanged. Local EBS TSDB, short retention — Prometheus data does not need to outlive the cluster. |
| **Loki → S3** | Object-store backed instead of PVCs. |
| **Tempo → S3** | Same. |
| **Grafana** | Unchanged. |
| **CloudWatch Logs** | EKS **control-plane** logs only — API server, audit, authenticator. Billed on ingestion, so enable selectively rather than all five streams by default. |

**Loki and Tempo writing to S3 is the one genuinely better-than-local arrangement.** It is cheaper than EBS, it is how both actually run in production, and — the real win — **logs and traces survive cluster teardown.** The end-to-end bid trace captured in session N is still queryable in session N+1, which turns the project's best artifact into something durable rather than something you have to recreate to show anyone.

**Amazon Managed Prometheus and Grafana were considered and rejected.** They cost money, and self-hosting the stack is what was already learned locally. Managed observability is a reasonable production choice and a wasteful learning one.

## Terraform layout

Two stacks. This split is what makes "destroy every session" survivable.

| Stack | Lifetime | Contents |
| --- | --- | --- |
| **`bootstrap/`** | **Never destroyed.** ~$4.70/month | S3 state bucket, ECR repositories, Route53 hosted zone, ACM certificate, IAM OIDC provider and the GitHub Actions role, AWS Budgets alarm, and the Secrets Manager / Parameter Store entries themselves. |
| **`environment/`** | **Destroyed every session** | VPC, subnets, EKS cluster, node groups, RDS instances, EKS add-ons. |

Without the split, teardown would also destroy every image, the domain records, the CI trust relationship, and every secret — and each session would begin by rebuilding all of them. The persistent stack costs under $5/month and removes that entirely.

**State locking uses S3 natively** (`use_lockfile = true`, Terraform 1.10+). The DynamoDB lock table that every older tutorial specifies is no longer required, and leaving it out is one fewer resource to explain.

## Budget

**$100 in credits. Three isolated environments running at once is the expensive shape, and the numbers should be read with that in mind.**

Per hour the cluster exists:

| Item | Unit rate | On-Demand | Spot for dev/stage |
| --- | --- | --- | --- |
| EKS control plane | $0.10/hr per cluster | $0.100 | $0.100 |
| 5 × t4g.large | $0.0672/hr | $0.336 | ~$0.120 |
| ALB (one, shared) | $0.0225/hr + $0.008/LCU-hr | $0.023 | $0.023 |
| 3 × db.t4g.micro | $0.016/hr | $0.048 | $0.048 |
| RDS storage, 60 GiB gp3 | $0.115/GB-mo | ~$0.010 | ~$0.010 |
| EBS, ~250 GiB gp3 | $0.08/GB-mo | ~$0.027 | ~$0.027 |
| **Total** | | **~$0.54/hr** | **~$0.33/hr** |

Persistent, per month:

| Item | Rate | Monthly |
| --- | --- | --- |
| Route53 hosted zone | $0.50/zone (first 25) | $0.50 |
| ECR storage | $0.10/GB-mo | ~$0.50 |
| S3 — state, Loki, Tempo | — | ~$0.50 |
| Secrets Manager, prod only | $0.40/secret-mo | ~$3.20 |
| **Total** | | **~$4.70/mo** |

**What $100 buys.** Spread over three months the persistent stack takes ~$14, leaving ~$86. On-Demand that is roughly **160 cluster-hours — about 20 eight-hour sessions.** On Spot, roughly **260 hours, about 32 sessions.**

That is **about a third of the 475 hours a single environment would have bought**, and the difference is precisely the price of three isolated environments running simultaneously. It is affordable, but it is not the comfortable margin a one-environment setup would have had. **Use Spot for dev and stage.**

*Every unit rate above was verified against the AWS Price List API for `us-east-1` on 2026-08-01. Rates change and regions differ — re-verify before relying on them. The totals are estimates built on assumed sizing; the rates are not.*

## Session discipline

- **Only `environment/` is destroyed.** `bootstrap/` persists by design, and destroying it means re-pushing every image and recreating every secret.
- **The Budgets alarm exists before the first `terraform apply`.** Not after.
- **Scale node groups to zero during long breaks.** The control plane keeps billing $0.10/hr; nothing else does.
- **Verify teardown reached zero.** Check Cost Explorer the next day — an orphaned ALB, EBS volume, or Elastic IP is the classic leak, and it bills silently.
- **Record the cost of every session** and compare it against the table above. The estimate is only useful if it's checked.
- **Never leave the cluster running overnight.** At ~$0.54/hr, one forgotten weekend is $26 — a quarter of the budget.

---

## Versions

- **Behavioral tools** (PostgreSQL, Kafka, RabbitMQ, Redis, Keycloak, Kubernetes) — pin to current stable. Their behavior *is* the lesson.
- **Operators** (Strimzi, CloudNativePG, RabbitMQ, Keycloak, External Secrets, OpenTelemetry) — pin exactly, and check the compatibility matrix; each supports a specific range of the thing it operates.
- **Spring** — Boot 4.0.x, and everything on the **Spring Cloud 2025.1.x** train (Gateway 5.0.x, Spring Cloud AWS 4.x, Spring Cloud Contract) moves with it as one unit. Never mix trains.
- **EKS Kubernetes version** — pin exactly, to a version inside standard support. Drifting out costs 6× ($0.60/hr against $0.10/hr).
- **AWS Load Balancer Controller** — pin exactly; **v2.14+** is required for L7 Gateway API (`HTTPRoute`, `GRPCRoute`).
- **Gateway API CRDs** — pin exactly and match against what Traefik and the AWS controller each support. Both sides must agree or the same `HTTPRoute` will not behave identically.
- **Tooling** (Gradle, Jib, Spotless, Testcontainers, Pitest) — pin loosely, float patches.
- **AWS** — verify availability, add-on compatibility, and quotas at the time of use.

**Never use `latest`. Never infer a compatible version pair from a general project page.** Check current releases before naming a version — this changes faster than any model's training data, and the Bitnami situation above is exactly what stale assumptions cost.

## Deliberately excluded

| Not included | Why |
| --- | --- |
| **Spring Cloud Stream** | Abstracts Kafka and RabbitMQ behind one binder API, erasing the fact-versus-work distinction that [02 Architecture](02-architecture.md) calls one of the most valuable things in the project. Actively harmful here. |
| **WebFlux for services** | Would make JPA, stack traces, and debugging harder to buy throughput this system doesn't need. Virtual threads instead. (Spring Cloud Gateway is the one exception — it requires Netty.) |
| **Schema registry** (Confluent, Apicurio) | Another deployable and operator. Schema evolution is taught instead by a versioned envelope in the repo plus Spring Cloud Contract. Revisit if consumer count grows. |
| **ShedLock** | Its lease can expire while the holder is still alive, so the task must be idempotent anyway. Once it is, the conditional-claim UPDATE is sufficient and free. |
| **ArchUnit** | Spring Modulith's `verify()` covers boundary enforcement. |
| **`JdbcClient`** | Not yet. Revisit if A3's measurement picks the atomic conditional update. |
| **Spring Cloud Config Server** | ConfigMaps and Secrets are the Kubernetes-native answer. |
| **Sealed Secrets, SOPS** | Per-cluster ciphertext breaks local/remote manifest parity. |
| **Redpanda** | Lighter locally, but it isn't Kafka and it breaks parity with EKS. |
| **Gatling, Pact** | k6 and Spring Cloud Contract cover the same ground. |
| **`Ingress`, ingress-nginx** | Feature-frozen and retired respectively. Gateway API instead. |
| **Bucket4j / rate limiting** | Real concern for bid flooding, no lesson not already covered. Candidate for later. |
| **EKS Auto Mode** | Adds ~12% on top of EC2 and takes over node provisioning, scaling, and patching — precisely the surface being learned. Right answer for a team that wants to stop thinking about nodes; wrong one here. |
| **EKS-managed Argo CD** | AWS now bills a managed Argo CD at $0.0015 per Application-hour. Cheap, but self-hosting Argo is free beyond pod resources and is what was already learned locally — and running the controller yourself is part of understanding reconciliation. |
| **IRSA** | Superseded by EKS Pod Identity, which drops the per-role OIDC trust policies and service-account annotations. |
| **NAT Gateway** | $0.045/hr plus $0.045/GB — two-thirds of a whole node, to do nothing but reach the internet. Public subnets instead. Interface endpoints are *not* the cheaper alternative: $0.01/hr each **per AZ**, about $0.10/hr for the five that would be needed. |
| **Amazon Managed Prometheus / Grafana** | Costs money, and self-hosting the LGTM stack is what was already learned locally. |
| **Istio / service mesh** | Large lift, and its lessons are proven at the application layer here. The most likely candidate to add later. |
| **KEDA** | HPA teaches the autoscaling concept. |
| **Argo Rollouts** | Progressive delivery on top of GitOps; add only after ordinary deployment works. |
| **Jenkins** | Actions + Argo CD is the modern combination. |
| **MSK, Amazon MQ, ElastiCache** | Budget, and the operators already teach the operational side. |
| **OpenSearch / full-text search** | Browse and filter covers discovery. |
| **CloudFront, Karpenter, multi-region DR** | Out of scope at this size. |
| **Nx, pnpm, Node tooling** | No frontend exists. Gradle alone is the build. |
