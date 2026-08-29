# infra/

Infrastructure as Code. Defines the cloud resources and foundational platform that BidPoint runs on.

## What belongs here

- AWS account and environment topology
- VPCs, subnets, and networking
- EKS clusters and node capacity
- IAM roles and policies
- ECR repositories
- Databases and caches
- DNS and load balancers
- KMS keys and secret-manager resources
- Storage and connectivity
- Cloud-level observability resources

Terraform or OpenTofu is the likely mechanism. The infrastructure itself is not implemented yet.

## What does not belong here

- Kubernetes workload definitions — see [`gitops/`](../gitops/README.md)
- Application code or application configuration
- Plaintext secrets. Secret-manager **resources** are created here; secret **values** are placed into them out of band and never committed.

## Ownership boundary

```text
infra/   → creates the Kubernetes cluster and AWS resources
gitops/  → deploys software onto the cluster
```

This split is deliberate. Infrastructure changes are slow, stateful, and expensive to reverse; workload changes are fast and routine. Merging the two would force every deployment through the risk profile of the slower one.

## Environments

`dev`, `stage`, and `prod` each run on a remote Kubernetes cluster provisioned from here. `local` is local development on a developer's machine and has no cloud infrastructure.

## Interacts with

- [`gitops/`](../gitops/README.md) — consumes the clusters created here
- [`.github/`](../.github/AUTOMATION.md) — validates and applies infrastructure changes

## Likely later

Environment topology and a separation between long-lived foundational resources and resources that are created and destroyed per environment.
