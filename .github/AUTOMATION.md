# .github/

Repository-level automation and governance.

## What belongs here

- CI workflows
- Build and test automation
- Release workflows
- Frontend deployment automation
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

CI builds and publishes artifacts. It does not deploy to clusters directly — deployment happens by changing declared state in [`gitops/`](../gitops/README.md) and letting the GitOps controller reconcile. Keeping build separate from deploy is the point.

## Likely later

Concrete workflow definitions, `CODEOWNERS`, and repository templates. None are implemented yet.
