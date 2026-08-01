# Delivery and learning roadmap

Status: Superseded by [design-system 2.0](../2.0/README.md)
Last validated: 2026-08-01

Every stage produces a vertical slice and lesson; target inclusion does not mean day-one activation.

| Stage | Outcome/lesson | Acceptance evidence | Staged afterwards |
|---|---|---|---|
| 1. Repository fitness | Maven + guarded Nx; authority/boundaries. | Direct Maven/matching Nx; Modulith/ArchUnit/Spotless/JaCoCo. | Runtime/features/frontend. |
| 2. Local core/identity | k3d/k3s, Traefik/Gateway API, data/brokers/Keycloak/S3Mock/HPA; readiness. | Migrations/seed/JWT/readiness/status-down. | CI/GitOps/full telemetry/KEDA/Istio/Rollouts. |
| 3. Core listing/auction | Profiles/catalog/listings/lifecycle; local transactions. | Module/Flyway/ownership/authorization tests. | Search/bids/payment/notifications. |
| 4. Bidding/idempotency | Safe bids/fence; hot-record authority. | Concurrency proves one price/winner, same key, fence retry. | Kafka/realtime/payment. |
| 5. Kafka/outbox/SSE | Facts/replay; at-least-once/lag. | Atomic outbox, dedupe/order, reconnect/refetch. | Registry/search/notification. |
| 6. RabbitMQ notification | Intent/job/worker; facts vs work. | Retry/DLQ/UNKNOWN/stable keys. | Real provider. |
| 7. Order/payment | One order/provider adapter; recovery/webhooks. | Redelivery once, simulated timeout/reconcile. | Provider Open, no cards. |
| 8. Observability/load | OTel signals/Toxiproxy/k6; RED/lag/capacity. | Prometheus/Loki/Tempo/Grafana, SLI/runbooks/results. | SLO/chaos/Istio faults. |
| 9. Local delivery | Jenkins/Jib/Argo/blue-green; provenance/rollback. | JCasC/digest/sync/rollback evidence. | Canary/signing. |
| 10. EKS parity | Terraform/EKS/RDS/Redis/MSK-MQ/S3/secrets/AWS telemetry. | Ownership/add-on check, scoped AWS smoke cleanup/no Jenkins apply. | MSK/OpenSearch/Karpenter/DR. |
| 11. Resilience | OpenSearch/KEDA/Istio/canary/chaos; trade-offs. | Search/scaling/policy/fault/canary evidence. | Optional expansion. |

Preserve Maven authority, digest review, producer contracts, owner data, no secrets. Ordinary tests avoid external calls; Testcontainers is disposable, AWS smoke explicit/cost-controlled. HPA is HTTP; KEDA queue/lag; node groups capacity. **Open:** frontend, payment provider, notification delivery provider, MSK, registry, license. **Staged:** OpenSearch/Istio/KEDA/canary/load-chaos/Debezium/gRPC/GitHub Actions/ElastiCache/Karpenter/DR. **Excluded:** Ansible, API Gateway, Eureka, initial Cloud Stream, MinIO/AIStor, LocalStack, Promtail, ingress-nginx, H2, legacy X-Ray, Jenkins deploy, Compose full environment, RabbitMQ Streams, physical database per Modulith module, environment branches, cosmetic microservices.
