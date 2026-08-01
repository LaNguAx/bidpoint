# Local technology stack

Status: Canonical
Last validated: 2026-08-01

Future local inventory; exact pins are reproducibility policy, not installed software.

| Component | Status | Version/family policy | Responsibility | Why needed |
|---|---|---|---|---|
| Eclipse Temurin | Canonical | Java 25.0.4 LTS | JVM | Reproducible backend runtime. |
| Node.js | Canonical | 24.18.0 LTS | Nx tooling runtime | Cross-language UX. |
| pnpm | Canonical | 11.4.0 | Node workspace manager | Manages Nx/future frontend packages only. |
| Nx | Canonical | 23.1.0 | Graph, affected tasks, cache | Consistent orchestration UX. |
| `@nx/maven` | Canonical | 23.1.0; experimental, direct Maven remains valid | Maven graph integration | Use only while Nx targets reproduce Maven. |
| Maven | Canonical | 3.9.16 | Java build/reactor authority | Independent escape hatch. |
| Spring Boot | Canonical | 4.1.0 | Backend framework | Runtime integration. |
| Spring Cloud | Canonical | 2025.1.2 | Cloud release train | Boot 4.1.x support. |
| Spring Cloud Gateway | Canonical | 5.0.2 | Runtime application gateway | Routes public traffic behind Traefik. |
| Spring Modulith | Canonical | 2.1.0 | Core module boundaries | Bounded modular monolith. |
| Spring Cloud AWS | Open | 4.0.0; Boot 4.1 gate | Spring AWS integration | Requires supported pair/proof. |
| AWS SDK for Java | Canonical | v2 via BOM | AWS protocol SDK | Visible underlying cloud SDK. |
| Spring Security resource server | Canonical | Boot-managed | JWT validation | Every backend validates tokens. |
| Spring Data JPA | Canonical | Boot-managed | Relational persistence | Owner-local data. |
| Flyway | Canonical | Boot-managed | Schema migration | Controlled database changes. |
| Spring for Apache Kafka | Canonical | Boot-managed | Kafka integration | Durable fact handling. |
| Spring AMQP | Canonical | Boot-managed | RabbitMQ integration | Targeted work handling. |
| Actuator | Canonical | Boot-managed | Health endpoints | Readiness/liveness. |
| Micrometer | Canonical | Boot-managed | Metrics | Operational signals. |
| Micrometer Observation | Canonical | Boot-managed | Trace/correlation instrumentation | Context propagation. |
| OpenTelemetry integration | Canonical | Boot-managed compatible | Trace export | Distributed diagnosis. |
| Jib Maven Plugin | Canonical | 3.5.2 | JVM image build | No routine Dockerfiles. |
| ArchUnit | Canonical | 1.4.2 | Architecture tests | Boundary enforcement. |
| JaCoCo | Canonical | 0.8.15 | Coverage report | Evidence, not target gaming. |
| Spotless Maven Plugin | Canonical | 3.6.0 | Formatting | Deterministic CI gate. |
| JUnit | Canonical | compatible BOM/pin policy | Unit tests | Fast behavior evidence. |
| Mockito | Canonical | compatible BOM/pin policy | Test doubles | Isolated unit behavior. |
| AssertJ | Canonical | compatible BOM/pin policy | Assertions | Readable test evidence. |
| Testcontainers | Canonical | compatible BOM/pin policy | Disposable dependencies | Real integration behavior. |
| WireMock | Canonical | compatible BOM/pin policy | HTTP simulation | Provider contract tests. |
| Awaitility | Canonical | compatible BOM/pin policy | Async assertions | Bounded convergence tests. |
| Toxiproxy | Canonical | compatible BOM/pin policy | Failure injection | Network recovery tests. |
| k6 | Canonical | 1.7.1 | Load/SSE testing | Capacity evidence. |
| k3d | Canonical | 5.9.0 | Local cluster lifecycle | Kubernetes-native environment. |
| k3s | Canonical | `v1.35.6+k3s1` | Local Kubernetes distribution | Pinned cluster runtime. |
| Kubernetes | Canonical | 1.35.6 | Orchestration API | Canonical full environment, not Compose. |
| Traefik | Canonical | 3.7.9 | Local cluster edge | Replaces disabled bundled Traefik. |
| Traefik Helm chart | Canonical | 41.0.2 | Traefik package | Repeatable edge install. |
| Kubernetes Gateway API | Canonical | 1.5.1 | Gateway configuration API | Not the application gateway. |
| metrics-server | Canonical | Kubernetes-compatible | Resource metrics API | HPA input. |
| Helm | Canonical | 4.2.3 | Kubernetes packaging | Deploy apps/add-ons. |
| PostgreSQL | Canonical | 18.4 | Authoritative data/outboxes | Transaction/concurrency practice. |
| Redis | Canonical | 8.2.6 | Acceleration/realtime coordination | Explicitly non-authoritative. |
| Kafka | Canonical | 4.2.1 | Durable facts | Replay/lag/partition practice. |
| Strimzi | Canonical | 1.1.0 | Kafka operator | Local Kafka lifecycle. |
| RabbitMQ | Canonical | 4.2.9 | Targeted work broker | Retries/DLQ. |
| RabbitMQ Cluster Operator | Canonical | 2.22.3 | RabbitMQ operator | Local broker lifecycle. |
| Keycloak | Canonical | 26.7.0 | OIDC/OAuth2 identity | Credential boundary. |
| Adobe S3Mock | Canonical | 5.1.0 | S3 test/development double | S3 practice; AWS smoke remains required. |
| OpenTelemetry Collector Contrib | Canonical | 0.157.0 | Telemetry collection | Local signal routing. |
| Prometheus | Canonical | 3.13.2 | Metrics store | Metrics analysis. |
| Loki | Canonical | 3.7.4 | Log store | Redacted logs. |
| Tempo | Canonical | 3.0.2 | Trace store | Trace analysis. |
| Grafana | Canonical | 13.1.1 | Signal exploration | Dashboards. |
| Jenkins | Canonical | 2.568.1 LTS | CI controller | Ephemeral-agent CI. |
| Jenkins Helm chart | Canonical | 5.9.45 | Jenkins package | Versioned JCasC install. |
| Argo CD | Canonical | 3.4.6 | GitOps reconciliation | No Jenkins production apply. |
| Argo Rollouts | Staged | 1.9.1 | Progressive delivery | Blue-green before canary. |
| KEDA | Staged | 2.20.1 | Queue/lag worker scaling | Complements HPA. |
| Istio ambient | Staged | 1.30.3 full profile | Identity/policy/fault learning | Add when operable. |
| OpenSearch | Staged | 3.5 | Search index | After browse/filter baseline. |

**Open:** frontend libraries, payment provider, MSK mode, schema registry, license. **Excluded:** Docker Compose full environment, MinIO/AIStor, LocalStack OSS, ingress-nginx, H2, Promtail, Eureka, Spring Cloud Stream initially, AWS API Gateway, routine Dockerfiles.
