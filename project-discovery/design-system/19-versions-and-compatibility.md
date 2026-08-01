# Versions and compatibility policy

Status: Canonical
Last validated: 2026-08-01

Exact direct pins: Temurin 25.0.4 LTS; Node 24.18.0 LTS; pnpm 11.4.0; Maven 3.9.16; Nx and experimental `@nx/maven` 23.1.0; Boot 4.1.0; Cloud 2025.1.2; Gateway 5.0.2; Modulith 2.1.0; Jib 3.5.2; ArchUnit 1.4.2; JaCoCo 0.8.15; Spotless 3.6.0; k6 1.7.1; PostgreSQL 18.4; Redis 8.2.6; Kafka 4.2.1/Strimzi 1.1.0; RabbitMQ 4.2.9/operator 2.22.3; S3Mock 5.1.0; Keycloak 26.7.0; k3d 5.9.0; k3s `v1.35.6+k3s1`; Kubernetes 1.35.6; Traefik 3.7.9/chart 41.0.2; Gateway API 1.5.1; Helm 4.2.3; Jenkins 2.568.1/chart 5.9.45; Argo CD 3.4.6; Rollouts 1.9.1; KEDA 2.20.1; Istio 1.30.3; Terraform 1.15.8; OTel 0.157.0; Prometheus 3.13.2; Loki 3.7.4; Tempo 3.0.2; Grafana 13.1.1; ADOT 0.48.0; LBC 3.4.3; CSI 1.6.0/AWS provider 3.1.0.

Spring Boot manages Security resource server, JPA, Flyway, Kafka, AMQP, Actuator, Micrometer, Observation and compatible Otel. JUnit, Mockito, AssertJ, Testcontainers, WireMock, Awaitility, Toxiproxy use compatible BOM/intentional pin policy. AWS SDK v2 remains visible under Cloud AWS/BOM; do not invent pins.

| Gate | Requirement | State |
|---|---|---|
| Spring Cloud support | Spring Cloud 2025.1.2 explicitly supports Spring Boot 4.1.x starting with that service release. | Confirmed support assertion. |
| Boot + Cloud AWS | Cloud AWS 4.x documents Boot 4.0.x/Framework 7.0.x, not Boot 4.1.x. Obtain upstream support, prove focused suite with explicit risk decision, or align Boot. | **Open:** Boot 4.1.0 + Cloud AWS 4.0.0 unconfirmed. |
| Nx/Maven | Maven direct/reproducible; Nx equivalent; upgrades checked; wrappers fallback. | **Guarded:** plugin experimental. |
| K8s set | EKS/k3s, Istio, Strimzi, Traefik/Gateway API, metrics-server, KEDA, CSI/controllers/Helm tested together. | EKS need not mirror local. |
| RDS | 18.4 supported; region/instance/extensions/backup/client checked. | Confirmed engine; conditional deployment. |
| MSK/schema | Express 4.2.x vs Provisioned 4.1.x, registry, region/quota/client. | **Open.** |

RDS 18.4 and MQ RabbitMQ 4.2 are engine families. EKS and Redis Cloud/S3/ECR/Route53/WAF/ALB/ACM/Secrets/Pod Identity/AMP/CloudWatch/X-Ray/Managed Grafana/CloudFront/OpenSearch are provider-managed families; floating patches only when provider controls them and notices are reviewed. Updates require release/security/compatibility/migration/license review, patch-window tests, digest pinning, rollback/roll-forward, and never `latest`. **Excluded:** unreviewed floating tags.
