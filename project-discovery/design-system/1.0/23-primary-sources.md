# BidPoint primary sources and validation record

Status: Superseded by [design-system 2.0](../2.0/README.md)
Last validated: 2026-08-01

This document records the evidence policy for the BidPoint target design. It is not a lockfile and does not prove that a dependency, AWS resource, or runtime is installed. The exact baseline and compatibility gates are in [versions and compatibility](19-versions-and-compatibility.md); accepted choices are indexed by the [decision register](21-decision-register.md).

## Source policy

1. Use the publisher, upstream project, or cloud-provider documentation/release page as the source of truth. Search-result snippets, unofficial blogs, and memory may help locate a source but are never authoritative evidence.
2. Distinguish **exact-release evidence** (a named version exists), **design/usage evidence** (how a product should be used), **compatibility evidence** (a supported version pairing), and **managed-service availability** (an AWS family/version is available subject to region/account checks). A general project page does not prove a particular release or pairing.
3. Recheck volatile release, compatibility, and AWS managed-service facts before implementation, dependency upgrades, infrastructure creation, or any claim newer than this validation date. Record the checked date, source, scope, result, and unresolved condition in the affected decision.
4. Preserve evidence with the decision it supports. A source can establish an available release without selecting it, and it can document a design option without making that option canonical.

## Context7 validation record

Context7 was queried on 2026-08-01 as a discovery and validation aid for the following questions. The linked official publisher material remains authoritative.

| Topic queried | Validation result used in this discovery | Authoritative follow-up |
| --- | --- | --- |
| Spring Modulith module arrangement and verification | The Core design uses explicit module boundaries, `api/` surfaces, separately named internal events, and later verification/testing rather than package convention alone. | [Spring Modulith project](https://spring.io/projects/spring-modulith/) and the Core contract in [04](04-core-platform-modular-monolith.md). |
| Spring Cloud AWS S3, BOM, and AWS SDK v2 integration; compatibility matrix | Cloud AWS 4.0.0 supports the Spring/AWS SDK integration design, but its documented Boot compatibility is a gate for Boot 4.1.0 rather than confirmation. | [Cloud AWS 4.0.0 reference](https://docs.awspring.io/spring-cloud-aws/docs/4.0.0/reference/html/index.html) and [upstream compatibility matrix](https://github.com/awspring/spring-cloud-aws#compatibility). |
| Nx official Maven support and experimental status | Nx has official Maven documentation; BidPoint keeps Maven Wrapper/reactor authoritative and treats `@nx/maven` as guarded experimental. | [Nx Maven introduction](https://nx.dev/docs/technologies/java/maven/introduction) and [16](16-monorepo-and-repository-structure.md). |

## Corrected fact record

The following facts were explicitly corrected and checked at the stated validation date. They are date-bound evidence, not a promise that a remote managed service is deployable in every account or Region.

| Fact at 2026-08-01 | Evidence kind | Authoritative source | Design consequence |
| --- | --- | --- | --- |
| Amazon RDS supports PostgreSQL 18.4. | Managed-service availability | [RDS PostgreSQL supported versions](https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-versions.html) | RDS PostgreSQL 18.4 is the remote target engine, still subject to Region, instance, extension, backup, and client checks. |
| Keycloak 26.7.0 exists/current at validation. | Exact-release evidence | [Keycloak 26.7.0 release notes](https://www.keycloak.org/docs/26.7.0/release_notes/) | The identity baseline is 26.7.0; Keycloak remains the identity owner, not marketplace-profile authority. |
| Istio 1.30.3 exists. | Exact-release evidence | [Istio 1.30.3 announcement](https://istio.io/latest/news/releases/1.30.x/announcing-1.30.3/) | Istio ambient is staged and requires EKS/local compatibility evidence before adoption. |
| Argo CD 3.4.6 was the latest release on 2026-08-01. | Exact-release evidence | [Argo CD v3.4.6 release](https://github.com/argoproj/argo-cd/releases/tag/v3.4.6) | Argo CD reconciles Git desired state; the statement is date-bound and must be refreshed before upgrade. |

## Exact releases and build/runtime inventory

| Technology or decision | Evidence kind | Official primary source | Notes |
| --- | --- | --- | --- |
| Temurin 25.0.4 LTS | Exact-release | [Temurin 25.0.4 release](https://github.com/adoptium/temurin25-binaries/releases/tag/jdk-25.0.4-ga) | Direct JVM pin. |
| Node.js 24.18.0 LTS | Exact-release | [Node 24.18.0 release](https://nodejs.org/en/blog/release/v24.18.0) | Nx/Node runtime pin. |
| pnpm 11.4.0 | Exact-release | [pnpm v11.4.0 release](https://github.com/pnpm/pnpm/releases/tag/v11.4.0) | Workspace manager for Nx/Node tooling. |
| Nx 23.1.0 | Exact-release | [Nx 23.1.0 package record](https://www.npmjs.com/package/nx/v/23.1.0) | Direct pin; Maven integration is separately guarded. |
| Nx Maven integration | Design/usage | [Nx Maven introduction](https://nx.dev/docs/technologies/java/maven/introduction) | Official support material does not remove the experimental-plugin guard or Maven authority. |
| Maven 3.9.16 policy | Release/history evidence | [Apache Maven history](https://maven.apache.org/docs/history) | Maven Wrapper/reactor is the independent Java authority. |
| Spring Boot 4.1.0 and Spring Cloud 2025.1.2 | Product and compatibility evidence | [Spring Boot](https://spring.io/projects/spring-boot/), [Spring Cloud](https://spring.io/projects/spring-cloud/) | Discovery records Spring Cloud 2025.1.2 support for Boot 4.1.x; refresh official compatibility evidence before change. |
| Spring Cloud Gateway 5.0.2 | Design/usage | [Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway) | `api-gateway` is the runtime application gateway. |
| Spring Modulith 2.1.0 | Design/usage | [Spring Modulith](https://spring.io/projects/spring-modulith/) | Supports the Core module/verification approach; use a release-specific source before changing its pin. |
| Spring Cloud AWS 4.0.0, S3/BOM/AWS SDK v2 | Compatibility and design/usage | [Cloud AWS 4.0.0 reference](https://docs.awspring.io/spring-cloud-aws/docs/4.0.0/reference/html/index.html), [compatibility matrix](https://github.com/awspring/spring-cloud-aws#compatibility) | Cloud AWS 4.0.0 documents Boot 4.0.x compatibility. It does **not** confirm Boot 4.1.0; the pairing remains Open. |
| Jib 3.5.2 routine JVM images | Release evidence | [Jib releases](https://github.com/GoogleContainerTools/jib/releases) | Jib, rather than routine Dockerfiles, is the chosen image path. |
| PostgreSQL 18.4 | Exact-release | [PostgreSQL 18.4 release notes](https://www.postgresql.org/docs/release/18.4/) | Local PostgreSQL pin; remote RDS availability is separately evidenced above. |
| Redis 8.2.6 | Release evidence | [Redis releases](https://github.com/redis/redis/releases) | Redis is acceleration/coordination, never business authority. |
| Kafka 4.2.1 and Strimzi 1.1.0 | Release evidence | [Kafka downloads](https://kafka.apache.org/downloads), [Strimzi releases](https://github.com/strimzi/strimzi-kafka-operator/releases) | Local facts platform; remote MSK family/mode remains Open. |
| RabbitMQ 4.2.9 and Cluster Operator 2.22.3 | Release evidence | [RabbitMQ server releases](https://github.com/rabbitmq/rabbitmq-server/releases), [Cluster Operator releases](https://github.com/rabbitmq/cluster-operator/releases) | Targeted work and DLQ/quarantine platform. |
| Adobe S3Mock 5.1.0 | Release evidence | [S3Mock releases](https://github.com/adobe/S3Mock/releases) | Local S3 double; real S3 smoke evidence remains required. |
| OpenSearch 3.5 | Exact-release | [OpenSearch 3.5.0](https://opensearch.org/versions/opensearch-3-5-0.html) | Staged only; `search-service` is never listing authority. |
| k3d 5.9.0, k3s/Kubernetes 1.35.6 | Release and support evidence | [k3d releases](https://github.com/k3d-io/k3d/releases), [k3s 1.35 release notes](https://docs.k3s.io/release-notes/v1.35.X), [Kubernetes 1.35 patches](https://kubernetes.io/releases/patch-releases/#1-35) | Canonical local Kubernetes set. |
| Traefik 3.7.9/chart 41.0.2 and Gateway API 1.5.1 | Release evidence | [Traefik 3.7.9](https://github.com/traefik/traefik/releases/tag/v3.7.9), [Traefik chart releases](https://github.com/traefik/traefik-helm-chart/releases), [Gateway API releases](https://github.com/kubernetes-sigs/gateway-api/releases) | Traefik is local front door; Gateway API is configuration, not runtime application gateway. |
| Helm 4.2.3 and KEDA 2.20.1 | Exact-release | [Helm 4.2.3](https://github.com/helm/helm/releases/tag/v4.2.3), [KEDA 2.20.1](https://github.com/kedacore/keda/releases/tag/v2.20.1) | Helm packages workloads; KEDA is staged for lag/depth scaling. |
| Jenkins 2.568.1 LTS/chart 5.9.45 | Release evidence | [Jenkins stable changelog](https://www.jenkins.io/changelog-stable/), [Jenkins Helm chart releases](https://github.com/jenkinsci/helm-charts/releases) | CI only, with JCasC; no direct production deployment. |
| Argo Rollouts 1.9.1 | Exact-release | [Argo Rollouts v1.9.1](https://github.com/argoproj/argo-rollouts/releases/tag/v1.9.1) | Staged progression after ordinary deployment/blue-green evidence. |
| Terraform 1.15.8 | Release evidence | [Terraform releases](https://github.com/hashicorp/terraform/releases) | AWS infrastructure owner, separate from Argo CD workload ownership. |
| OTel Collector Contrib 0.157.0 and ADOT 0.48.0 | Release evidence | [OpenTelemetry Collector releases](https://github.com/open-telemetry/opentelemetry-collector-releases/releases), [AWS OTel Collector releases](https://github.com/aws-observability/aws-otel-collector/releases) | Local and remote telemetry collectors; signals have distinct destinations. |
| Prometheus, Loki, Tempo, Grafana, k6 | Release evidence | [Prometheus releases](https://github.com/prometheus/prometheus/releases), [Loki releases](https://github.com/grafana/loki/releases), [Tempo releases](https://github.com/grafana/tempo/releases), [Grafana releases](https://github.com/grafana/grafana/releases), [k6 releases](https://github.com/grafana/k6/releases) | Refresh exact pins before a change; testing/observability roles are defined in [14](14-observability.md) and [15](15-testing-and-quality.md). |

## AWS and platform design sources

| Decision area | Evidence kind | Official primary source | Boundary supported |
| --- | --- | --- | --- |
| EKS supported versions and managed node groups | Managed-service availability | [EKS Kubernetes versions](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html) | EKS version is a compatibility set, not a blind mirror of local k3s. |
| ALB and controller | Design/usage | [AWS Load Balancer Controller on EKS](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html), [controller releases](https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases) | Route 53/WAF/ALB are front-door components before Spring Cloud Gateway. |
| Workload identity and Secrets Manager | Design/usage | [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html), [Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver/releases), [AWS provider releases](https://github.com/aws/secrets-store-csi-driver-provider-aws/releases), [Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) | Least-privilege secrets delivery; never put secret material in Git/state/logs. |
| MSK and Amazon MQ for RabbitMQ | Managed-service availability | [MSK supported versions](https://docs.aws.amazon.com/msk/latest/developerguide/supported-kafka-versions.html), [Amazon MQ engine versions](https://docs.aws.amazon.com/amazon-mq/latest/developer-guide/broker-engine.html) | MSK mode/registry are Open; Amazon MQ uses the RabbitMQ 4.2 family subject to availability. |
| S3, ECR, RDS | Managed-service availability | [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html), [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html), [Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html) | Images use immutable ECR digests; S3 holds objects; RDS holds owner-local authoritative data. |
| AMP, CloudWatch Logs, X-Ray, Managed Grafana | Managed-service/design | [Amazon Managed Service for Prometheus](https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html), [CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html), [AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html), [Amazon Managed Grafana](https://docs.aws.amazon.com/grafana/latest/userguide/what-is-Amazon-Managed-Service-Grafana.html) | Metrics, logs, traces, and exploration remain distinct. |

## Protocol, reliability, and delivery sources

| Design subject | Evidence kind | Official primary source | BidPoint application |
| --- | --- | --- | --- |
| Kafka ordering and delivery semantics | Design/usage | [Kafka design](https://kafka.apache.org/documentation/#design), [Kafka consumer configuration](https://kafka.apache.org/documentation/#consumerconfigs) | Partition auction facts by `auctionId`, retain stable IDs, deduplicate, monitor lag, and do not claim end-to-end exactly once. |
| RabbitMQ reliability and DLQ | Design/usage | [RabbitMQ reliability guide](https://www.rabbitmq.com/docs/reliability), [dead-letter exchanges](https://www.rabbitmq.com/docs/dlx) | Targeted work uses retries, durable attempts, DLQ/quarantine, and provider reconciliation. |
| Kubernetes HPA and KEDA Kafka/Rabbit scalers | Design/usage | [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/), [KEDA Kafka scaler](https://keda.sh/docs/latest/scalers/apache-kafka/), [KEDA RabbitMQ scaler](https://keda.sh/docs/latest/scalers/rabbitmq-queue/) | HPA is baseline HTTP/resource scaling; KEDA is staged for queue/lag workloads. |
| Argo CD GitOps and Argo Rollouts | Design/usage | [Argo CD GitOps](https://argo-cd.readthedocs.io/en/stable/user-guide/), [Argo Rollouts](https://argo-rollouts.readthedocs.io/en/stable/) | Jenkins proposes digest changes; Argo CD reconciles desired state; progressive rollout is staged. |
| Terraform S3 backend lockfile | Design/usage | [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3) | Terraform owns cloud foundation and uses the S3 lockfile policy, not a DynamoDB lock baseline. |
| Keycloak OIDC and Spring Security resource servers | Design/usage | [Keycloak OIDC](https://www.keycloak.org/securing-apps/oidc-layers), [Spring Security resource server](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/index.html) | Keycloak owns identity; each backend validates tokens and makes owner-level authorization decisions. |
| Spring Kafka, Spring AMQP, Spring Modulith events, Spring Cloud Gateway | Design/usage | [Spring Kafka](https://docs.spring.io/spring-kafka/reference/), [Spring AMQP](https://docs.spring.io/spring-amqp/reference/), [Spring Modulith events](https://docs.spring.io/spring-modulith/reference/events.html), [Spring Cloud Gateway reference](https://docs.spring.io/spring-cloud-gateway/reference/) | Supports explicit client integrations, internal module events, and the named application-gateway role. |

## Validation cadence and limits

At discovery time, links support the target-design baseline and identify gates; they do not validate a local installation or remote account. Before implementation, validate exact artifact versions and dependency resolution. Before AWS provisioning, validate Region/account availability, quotas, add-on compatibility, pricing, and permission model. Before a production claim, record smoke/operational evidence and refresh release/security notes. For every change, update [19](19-versions-and-compatibility.md), the relevant detailed design, and [21](21-decision-register.md) if the canonical decision changes.
