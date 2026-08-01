# Local development platform

Status: Superseded by [design-system 2.0](../2.0/README.md)
Last validated: 2026-08-01

BidPoint's complete local target environment is Kubernetes, not Docker Compose. It is a reproducible learning platform for the same boundaries that matter remotely: deployables, brokers, data stores, identity, edge routing, autoscaling, delivery, and telemetry. This repository contains the design only; no cluster, scripts, images, manifests, or Nx targets exist yet.

## Canonical cluster and edge

`k3d` 5.9.0 creates a pinned k3s `v1.35.6+k3s1` cluster (Kubernetes 1.35.6). Its bundled Traefik is disabled. A separately installed, pinned Traefik 3.7.9 using Helm chart 41.0.2 is the external local edge. Kubernetes Gateway API 1.5.1 expresses edge configuration; it is not BidPoint's application gateway. Public application traffic is:

```text
developer/browser -> pinned Traefik -> Spring Cloud Gateway (`api-gateway`) -> Kubernetes Service -> owning backend
```

Keycloak has a distinct authentication host or path and is not routed through Spring Cloud Gateway. Spring Cloud Gateway owns application routes, CORS, trace/correlation propagation, early JWT rejection, and Redis-backed rate limits; each owner backend still validates JWTs and performs business/object authorization. There is no Eureka, ingress-nginx, or AWS API Gateway in this local design.

## Two intentional profiles

The future developer UX is a contract, not a currently available command:

| Profile | Intended command | Includes | Purpose |
| --- | --- | --- | --- |
| `core` | `pnpm nx local-up core` | The base cluster, edge, application deployables, PostgreSQL, Redis, Kafka/Strimzi, RabbitMQ/operator, Keycloak, Adobe S3Mock, metrics-server/HPA, and basic telemetry. | Fastest full-system path for feature and integration work. |
| `platform` | `pnpm nx local-up platform` | Everything in `core`, plus Jenkins, Argo CD, Argo Rollouts, KEDA, Istio ambient, and the complete local observability stack. | Delivery, rollout, scaling, mesh, and operations exercises. |

Matching future contracts are `pnpm nx local-down <profile>`, `pnpm nx local-status <profile>`, and `pnpm nx local-reset <profile>`. `down` stops the selected environment; `status` reports component readiness and useful endpoints without exposing credentials; `reset` is an explicitly destructive local-only recreation that must require confirmation and document what persistent local data it removes. Their exact task implementation, arguments, and scripts remain unimplemented.

## Component ownership and readiness

| Component | Target local responsibility | Ready only when |
| --- | --- | --- |
| k3d/k3s, Helm, Gateway API, Traefik | Local cluster, package installation, and external edge. | API is reachable, nodes are Ready, installed charts/controllers are Available, and configured routes resolve. |
| Application Helm charts | Package each deployable: `api-gateway`, `core-platform`, `bidding-service`, `payment-service`, `realtime-service`, `notification-service`, `notification-worker`, plus staged `search-service`. | Required ConfigMaps/secrets references resolve, migrations complete, and application readiness—not merely a running container—passes. |
| PostgreSQL 18.4 | Owner-local authoritative state, outboxes, inbox/deduplication, and audit records. | Accepts authenticated connections and each owner migration is at the expected version. |
| Redis 8.2.6 | Gateway rate limits and realtime fan-out/bounded replay; never authoritative business state. | Reachable and the dependent application can perform its constrained health check. |
| Kafka 4.2.1 via Strimzi 1.1.0 | Durable replayable business facts. | Operator and broker are Ready; required topics and access policy are present. |
| RabbitMQ 4.2.9 via RabbitMQ Cluster Operator 2.22.3 | Targeted notification work and DLQ/quarantine routes. | Operator, cluster, exchanges, queues, bindings, and consumers' required access are ready. |
| Keycloak 26.7.0 | OIDC/OAuth2 identity, users, credentials, MFA, sessions, and tokens. | Realm/client/role configuration and a non-secret bootstrap path are available; applications can retrieve issuer metadata. |
| Adobe S3Mock 5.1.0 | Development/test double for the S3 consumer contract. | Bucket/bootstrap configuration is available and application object operations pass a smoke check. |
| metrics-server aligned to Kubernetes 1.35 | Resource metrics for HPA. | Metrics API serves pods so HPA decisions can be observed. |
| Basic telemetry | Minimum Collector/export path and correlation-preserving logs in `core`. | Application metrics, traces, and redacted structured logs are accepted by the selected basic endpoints. |

S3Mock is canonical locally because it exercises an S3 consumer API without introducing a second storage product and administration model. It is a development/test double, not production-compatible proof; small explicit AWS S3 smoke tests remain required. **Excluded baseline:** MinIO/AIStor and LocalStack OSS. Testcontainers, rather than the shared cluster, is canonical for isolated integration tests.

## Lifecycle contract

Startup is dependency-aware: create the cluster and platform controllers first; establish data, identity, object-storage, and broker prerequisites next; then apply owner migrations/seeds; then make application deployments ready. A future status command must distinguish a resource existing, its controller reporting readiness, migration/seed completion, and an owner API actually being ready. A failing migration blocks the dependent owner rather than serving a partially initialized process.

Seeds are minimal, deterministic, non-secret fixtures for local discovery (for example development realm configuration and sample category data), never a substitute for migrations. Each data owner owns its schema migration and seed lifecycle. No service may seed or mutate another owner's schema simply because the local cluster is shared. Test data for isolated suites is created by the suite through Testcontainers; it must not depend on a developer's retained cluster state.

HPA is part of `core` to teach HTTP workload scaling from resource metrics. It does not make queue or lag-driven workers correctly scalable; **Staged:** KEDA in `platform` supplies that responsibility. Managed node capacity and Karpenter are remote/deferral concerns, not a claim that local HPA can scale nodes.

## Platform profile additions and boundaries

`platform` adds Jenkins 2.568.1 LTS (chart 5.9.45), Argo CD 3.4.6, Argo Rollouts 1.9.1, KEDA 2.20.1, Istio 1.30.3 ambient, OpenTelemetry Collector Contrib 0.157.0, Prometheus 3.13.2, Loki 3.7.4, Tempo 3.0.2, and Grafana 13.1.1. Jenkins uses ephemeral Kubernetes agents and configuration-as-code; it does not become an application deployer. Argo CD reconciles Git desired state; it, not Terraform, owns Kubernetes workload desired state. Argo Rollouts is **Staged** for progression after ordinary Deployment/blue-green evidence. Istio ambient, service identity, mTLS, authorization policy, and fault injection are **Staged** defense in depth; they never replace application-layer JWT validation or authorization.

Docker Compose is **Excluded as the canonical complete environment** because it omits the Kubernetes/controller/rollout learning boundary. A narrowly scoped future Compose developer aid is only an **Optional comparison/reconsideration trigger** after measured developer friction; it cannot redefine `core`, replace `platform`, or be presented as platform parity.
