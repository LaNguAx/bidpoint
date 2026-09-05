# gitops/

The desired runtime state of Kubernetes workloads in every environment. Git is the source of truth; a GitOps controller such as ArgoCD reconciles a cluster toward what is declared here.

Nothing in this directory provisions infrastructure, and nothing here contains application logic. It answers one question: given a cluster that already exists, what should be running on it?

## What belongs here

- Which application image version should run
- Replica counts
- Resource requests and limits
- Kubernetes Services
- Workload identities and ServiceAccounts
- Ingress and gateway exposure
- Health probes
- Autoscaling policy
- Deployment strategies
- Per-environment workload sizing — replica counts, resource limits, and scaling policy per environment
- Platform workloads deployed to the cluster

## What does not belong here

- Cloud resource provisioning — the cluster itself, networking, and managed services are [`infra/`](../infra/README.md)
- Application source code
- Backend application behavior settings — timeouts, retries, feature flags, log levels, and business thresholds are [`config/`](../config/README.md)
- Plaintext secrets. Secret **references** are declared here; secret **values** are resolved at runtime from a secret-management system.

## The boundary with config/

Both directories hold settings, and both can end up as environment variables inside a running backend container. The test is **which process reads the value**: Kubernetes reads what is declared here, at pod creation; the backend application reads what is in [`config/`](../config/README.md), at startup and on refresh.

A backend workload declared here may set **only** these environment variables:

```text
SPRING_PROFILES_ACTIVE     which environment the service is running as
SPRING_APPLICATION_NAME    which configuration set to request
SPRING_CLOUD_CONFIG_NAME   which dotted configuration source to request
SPRING_CONFIG_IMPORT       where the configuration server is
+ secret-backed variables sourced from Kubernetes Secrets
```

Those are bootstrap and identity — the minimum a service needs before it can fetch its own configuration. Any other backend application environment variable in a manifest here means a property is owned in two places, which in Spring Boot fails silently rather than loudly. [`config/`](../config/README.md) explains why.

## Environments

```text
local
dev
stage
prod
```

`local` is a Kind Kubernetes cluster on a developer's machine. `dev`, `stage`, and `prod` are remote Kubernetes clusters. This directory declares the desired workload state for all four; shared workload definitions must be separated from explicit environment overlays so local behavior remains as close to remote behavior as practical. [`infra/`](../infra/README.md) does not provision local.

Environment isolation is a hard boundary. A change intended for one environment must not implicitly alter another.

## Intended separation

```text
gitops/apps/      BidPoint application workloads
gitops/platform/  cluster-level software — observability and shared platform components
```

Neither is created yet. Application workloads and platform components are separated because they have different owners, different upgrade cadences, and different blast radii.

## Interacts with

- [`infra/`](../infra/README.md) — provides the cluster this state is applied to
- [`config/`](../config/README.md) — the configuration server deployed from here is what serves those settings to backend workloads
- [`backend/`](../backend/README.md) and [`frontend/`](../frontend/README.md) — the workloads being deployed
- [`.github/`](../.github/AUTOMATION.md) — validates manifests and publishes the image versions referenced here
