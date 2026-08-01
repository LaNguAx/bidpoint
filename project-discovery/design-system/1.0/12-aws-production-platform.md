# AWS production platform

Status: Canonical
Last validated: 2026-08-01

This is the target AWS production design, not provisioned infrastructure. It preserves the owner boundaries of the local platform while using managed services where that exposes production operations rather than creating undifferentiated self-managed work. Separate AWS accounts per environment are preferred when practical; canonical repository layout uses environment directories even if the learning start uses one account.

## Declarative ownership boundary

| Owner | Owns | Must not own |
| --- | --- | --- |
| Terraform 1.15.8 | Account/environment foundations, VPC/subnets, EKS managed node groups, IAM and EKS Pod Identity, Route 53, WAF, ALB prerequisites, ECR, RDS, S3, MSK, Amazon MQ, Redis Cloud integration, staged OpenSearch, Secrets Manager, observability destinations, and their cloud policies. | Helm-rendered application/platform workloads or duplicate Kubernetes resources reconciled by Argo CD. |
| Helm | Versioned package definitions for each application and platform add-on. Application deployables own their charts and external contracts; reusable platform definitions belong under `platform/`. | AWS account resources, mutable environment policy, or direct production imperative deployment. |
| Argo CD 3.4.6 | Kubernetes desired state reconciled from Git. Initially it reads same-repository desired state on `main`, composed under `deploy/` environment directories. | AWS infrastructure and arbitrary Jenkins commands. |
| Jenkins 2.568.1 LTS | CI validation, immutable image build/publish, and automated GitOps change proposal. | Direct production `kubectl apply`, secret storage, or ownership of production desired state. |

Terraform uses an encrypted, versioned S3 backend with native S3 lockfile. DynamoDB locking is **Excluded** for a new configuration. State inputs, Git, Helm values, and logs never carry application secrets.

## Network and runtime

The target runtime is Amazon EKS with managed node groups. The EKS control-plane version must be currently supported and selected as a compatibility set with the AWS Load Balancer Controller 3.4.3, Secrets Store CSI Driver 1.6.0/AWS provider 3.1.0, metrics/add-on needs, Istio when staged, and workload requirements; it is not pinned merely to mirror local Kubernetes 1.35.6. Kubernetes Services provide internal discovery. Karpenter is **Staged/deferred**: managed node groups provide node capacity but are not a pod-aware autoscaler.

HPA scales HTTP workloads from application/resource metrics. KEDA is **Staged** and, when introduced, owns scale signals for queue/lag-driven workers; neither HPA nor a node group is described as a substitute for the other. Resource limits, requests, disruption controls, zones/subnets, and backup/recovery settings are environment-policy decisions applied through GitOps and Terraform's separate scopes.

## Public front door and identity

```text
Route 53 -> AWS WAF -> ALB (AWS Load Balancer Controller) -> Spring Cloud Gateway -> Kubernetes Service -> owning backend
```

ACM provides edge certificates. The ALB is an AWS load-balancing edge; Spring Cloud Gateway is BidPoint's runtime application gateway, responsible for routing, CORS, correlation/trace propagation, early JWT rejection, and Redis-backed rate limits. It does not own final backend authorization. Keycloak uses a distinct authentication host or path and is not routed through Spring Cloud Gateway. Kubernetes Gateway API is configuration, not the runtime gateway. **Excluded:** AWS API Gateway, Eureka, and direct Jenkins production deployment.

## Managed data and messaging responsibilities

| Service | Canonical responsibility and constraint |
| --- | --- |
| RDS PostgreSQL 18.4 | Authoritative owner-local state, outboxes, inbox/deduplication, and audit. Version support is confirmed, but region, instance class, backups, restore drills, extensions, encryption, and per-owner access must be checked. |
| Redis Cloud managed service | Current canonical managed Redis choice for rate limits and realtime fan-out/bounded replay; never source of truth. **Staged comparison:** ElastiCache/Valkey. |
| Amazon MSK | Durable Kafka facts. Broker choice is **Open** between Express 4.2.x and Provisioned 4.1.x; sizing, regions, private connectivity, and schema-registry strategy remain explicit gates. |
| Amazon MQ for RabbitMQ | Targeted notification work, retry/DLQ/quarantine. Use the RabbitMQ 4.2 family, with worker idempotency and provider reconciliation still enforced. |
| S3 | Listing image objects; `listings` owns marketplace metadata/lifecycle. Explicit AWS S3 smoke tests validate the real boundary. |
| Amazon OpenSearch Service | **Staged** derived full-text store owned by `search-service`; never authoritative listing data. |

Each application accesses only its owner-local database/schema and only cloud resources necessary for its responsibility. Kafka facts remain producer-owned/replayable; RabbitMQ remains targeted work. No shared cloud credential or network reachability grants cross-owner write authority.

## Secrets and workload identity

Secrets Manager, EKS Pod Identity, Secrets Store CSI Driver, and the AWS provider are the target secret delivery chain. Terraform creates policies and references but does not put secret material into state inputs. Workloads receive least-privilege identities and mount or retrieve only the required secret. No secret, bearer token, database password, provider credential, payment data, or private key may enter Git, Helm values, Terraform state inputs, event/job payloads, diagnostic output, or logs. Rotation and revocation must have a tested workload recovery path.

## Availability, recovery, and status

Production environments must expose health/readiness separately from liveness. A process is ready only after its owned migrations/dependencies needed for safe work are satisfied; it must not accept traffic merely because it started. Backups, point-in-time recovery where offered, restore evidence, multi-AZ/availability choices, capacity alarms, and account/region availability are environment-specific acceptance evidence rather than claims made by this document.

**Staged:** Amazon OpenSearch Service, KEDA, Istio ambient/auth policies/fault injection, Argo Rollouts progression, Karpenter, multi-region DR, and the Redis Cloud versus ElastiCache/Valkey comparison. **Open:** MSK mode, schema registry, payment provider, notification delivery provider, frontend libraries, public license. All production choices are target design until a compatible, cost-controlled environment and smoke evidence exist.
