# .github/

Repository-level automation and governance.

## What belongs here

- CI workflows
- Build and test automation
- Release workflows
- Frontend delivery automation — its mechanism depends on the web deployment model, which is undecided ([`frontend/web/`](../frontend/web/README.md))
- Infrastructure validation and apply workflows
- GitOps manifest validation
- Dependency and security automation
- Issue and pull request templates
- `CODEOWNERS` and ownership policy

## What does not belong here

- Application code
- Infrastructure definitions — [`infra/`](../infra/README.md) defines infrastructure; workflows here only validate and apply it
- Kubernetes manifests
- Secrets. Workflows read credentials from repository or environment secrets and from federated identity; nothing sensitive is committed.

## Path-aware workflows

Workflows should be scoped by path so a change in one domain does not run the entire repository pipeline. A mobile commit should not trigger infrastructure validation, and an infrastructure change should not rebuild both frontends.

This is also the mechanism that makes module boundaries observable: if a workflow needs paths from two domains to do one job, the boundary between them is probably wrong.

## Ownership

`CODEOWNERS` is what actually enforces module ownership in a single repository. Directory structure alone enforces nothing.

## Interacts with

Every module. CI builds [`backend/`](../backend/README.md) and [`frontend/`](../frontend/README.md), publishes artifacts to external registries, validates [`gitops/`](../gitops/README.md) and [`infra/`](../infra/README.md), and commits the image versions that GitOps then reconciles.

## Delivery boundary

CI builds and publishes artifacts. It never deploys to clusters — cluster workloads are deployed by changing declared state in [`gitops/`](../gitops/README.md) and letting the GitOps controller reconcile. Artifacts that do not run on the cluster (a static web bundle, a mobile store release) are the one case where CI delivers directly. If the web application is later deployed as a cluster workload, it follows the gitops path like any other workload. Keeping build separate from cluster deployment is the point.

## Current state

`CODEOWNERS` and the pull request template provide baseline governance. Concrete workflow definitions and per-module ownership rules are not implemented yet.
