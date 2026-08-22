# gitops/

The desired runtime state of Kubernetes workloads. Git is the source of truth; a GitOps controller such as ArgoCD reconciles the cluster toward what is declared here.

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
- Environment-specific workload configuration
- Platform workloads deployed to the cluster

## What does not belong here

- Cloud resource provisioning — the cluster itself, networking, and managed services are [`infra/`](../infra/README.md)
- Application source code
- Plaintext secrets. Secret **references** are declared here; secret **values** are resolved at runtime from a secret-management system.

## Environments

```text
dev
staging
prod
```

Environment isolation is a hard boundary. A change intended for one environment must not implicitly alter another.

## Intended separation

```text
gitops/apps/      BidPoint application workloads
gitops/platform/  cluster-level software — observability and shared platform components
```

Neither is created yet. Application workloads and platform components are separated because they have different owners, different upgrade cadences, and different blast radii.

## Interacts with

- [`infra/`](../infra/README.md) — provides the cluster this state is applied to
- [`backend/`](../backend/README.md) and [`frontend/`](../frontend/README.md) — the workloads being deployed
- [`.github/`](../.github/AUTOMATION.md) — validates manifests and publishes the image versions referenced here
