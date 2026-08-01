# Remote technology stack

Status: Canonical
Last validated: 2026-08-01

AWS target inventory; managed-family rows require region/account availability and compatibility checks.

| Component | Status | Version/family policy | Responsibility | Why needed |
|---|---|---|---|---|
| Eclipse Temurin | Canonical | Java 25.0.4 LTS | CI JVM | Reproducible builds. |
| Node.js | Canonical | 24.18.0 LTS | CI Nx runtime | Cross-language tooling. |
| pnpm | Canonical | 11.4.0 | Node workspace manager | Nx/future frontend packages only. |
| Nx | Canonical | 23.1.0 | CI graph/tasks/cache | Affected orchestration. |
| `@nx/maven` | Canonical | 23.1.0; experimental/direct Maven valid | Maven graph integration | Maven escape hatch. |
| Maven | Canonical | 3.9.16 | Java build authority | Independent build. |
| Spring Boot | Canonical | 4.1.0 | Backend framework | Application runtime. |
| Spring Cloud | Canonical | 2025.1.2 | Cloud release train | Supported Boot integration. |
| Spring Cloud Gateway | Canonical | 5.0.2 | Application gateway | Behind ALB. |
| Spring Modulith | Canonical | 2.1.0 | Core boundaries | Coherent core. |
| Spring Cloud AWS | Open | 4.0.0; Boot 4.1 gate | AWS integration | Needs proof/support. |
| AWS SDK for Java | Canonical | v2 via BOM | AWS protocol SDK | Visible cloud dependency layer. |
| Spring Security resource server | Canonical | Boot-managed | JWT validation | Service defense. |
| Spring Data JPA | Canonical | Boot-managed | Persistence | Owner data. |
| Flyway | Canonical | Boot-managed | Migration | Safe schema evolution. |
| Spring for Apache Kafka | Canonical | Boot-managed | Kafka | Facts. |
| Spring AMQP | Canonical | Boot-managed | RabbitMQ | Work. |
| Actuator | Canonical | Boot-managed | Health | Readiness/health. |
| Micrometer | Canonical | Boot-managed | Metrics | Production signals. |
| Micrometer Observation | Canonical | Boot-managed | Trace context | Correlation. |
| OpenTelemetry integration | Canonical | Boot-managed compatible | Trace export | Diagnosis. |
| Jib Maven Plugin | Canonical | 3.5.2 | JVM images | Immutable images. |
| ArchUnit | Canonical | 1.4.2 | Architecture checks | CI boundary evidence. |
| JaCoCo | Canonical | 0.8.15 | Coverage report | Gap visibility. |
| Spotless Maven Plugin | Canonical | 3.6.0 | Formatting | CI gate. |
| JUnit | Canonical | compatible BOM/pin policy | Unit tests | Fast behavior evidence. |
| Mockito | Canonical | compatible BOM/pin policy | Test doubles | Isolated behavior. |
| AssertJ | Canonical | compatible BOM/pin policy | Assertions | Readable evidence. |
| Testcontainers | Canonical | compatible BOM/pin policy | Disposable dependencies | Real integration evidence. |
| WireMock | Canonical | compatible BOM/pin policy | HTTP simulation | Provider contracts. |
| Awaitility | Canonical | compatible BOM/pin policy | Async assertions | Bounded convergence. |
| Toxiproxy | Canonical | compatible BOM/pin policy | Failure injection | Recovery evidence. |
| Terraform | Canonical | 1.15.8 | AWS infrastructure | Never Argo workload state. |
| Amazon EKS | Canonical | Supported control plane compatible with add-ons | Kubernetes runtime | Do not mirror local blindly. |
| EKS managed node groups | Canonical | EKS-compatible managed family | Node capacity | Not pod autoscaling. |
| Helm | Canonical | 4.2.3 | Packaging | Charts. |
| Argo CD | Canonical | 3.4.6 | GitOps reconciliation | Desired state from Git. |
| Argo Rollouts | Staged | 1.9.1 | Progressive delivery | Canary after evidence. |
| AWS Load Balancer Controller | Canonical | 3.4.3 | ALB controller | AWS edge integration. |
| Route 53 | Canonical | AWS-managed family | DNS | Public front door. |
| AWS WAF | Canonical | AWS-managed family | Web protection | Edge defense. |
| Application Load Balancer | Canonical | AWS-managed family | TLS ingress | Routes to Gateway. |
| AWS Certificate Manager | Canonical | AWS-managed family | Certificates | TLS. |
| RDS PostgreSQL | Canonical | 18.4, region/instance/extensions checked | Authoritative data | Managed relational store. |
| Redis Cloud | Canonical | Managed service patch policy | Coordination/acceleration | Managed Redis choice. |
| Amazon MSK | Open | Express 4.2.x or Provisioned 4.1.x | Kafka facts | Mode remains decision. |
| Amazon MQ for RabbitMQ | Canonical | RabbitMQ 4.2 family | Work broker | Managed retries/DLQ. |
| Amazon S3 | Canonical | AWS-managed family | Image objects | Production S3. |
| Amazon ECR | Canonical | AWS-managed family | OCI registry | Digest promotion. |
| AWS Secrets Manager | Canonical | AWS-managed family | Secret store | No secret Git/state/logs. |
| EKS Pod Identity | Canonical | AWS-managed family | Workload identity | Least privilege. |
| Secrets Store CSI Driver | Canonical | 1.6.0 | Secret materialization | Pod secret access. |
| AWS Secrets Store CSI provider | Canonical | 3.1.0 | AWS secret integration | Secret Manager provider. |
| ADOT Collector | Canonical | 0.48.0 | Telemetry collection | AWS signal routing. |
| Amazon Managed Service for Prometheus | Canonical | AWS-managed family | Metrics store | Production metrics. |
| CloudWatch Logs | Canonical | AWS-managed family | Log store | Redacted logs. |
| AWS X-Ray | Canonical | AWS-managed family | Trace store | Production traces. |
| Amazon Managed Grafana | Canonical | AWS-managed family | Exploration | Dashboards. |
| Amazon OpenSearch Service | Staged | OpenSearch 3.5 family | Search | Later full text. |
| CloudFront | Staged | AWS-managed family | Static delivery | Future frontend delivery. |
| KEDA | Staged | 2.20.1, EKS compatibility checked | Worker scaling | Queue/lag workloads. |
| Istio ambient | Staged | 1.30.3, EKS compatibility checked | Identity/policy/faults | Later learning. |
| Jenkins | Canonical | 2.568.1 LTS, JCasC/versioned plugins | CI | Never direct production apply. |
| k6 | Canonical | 1.7.1 | Load/smoke | Controlled runtime evidence. |

**Open:** payment, frontend, MSK, schema registry (Apicurio parity vs AWS Glue), license. **Excluded:** AWS API Gateway, Ansible, direct Jenkins production deployment, environment branches, legacy X-Ray SDK/daemon, ingress-nginx.
