# CI/CD and GitOps

Status: Superseded by [design-system 2.0](../2.0/README.md)
Last validated: 2026-08-01

BidPoint separates build authority, image provenance, environment approval, and runtime reconciliation. This is a target pipeline, not automation present in the repository.

## Canonical flow

```text
GitHub webhook -> Jenkins controller -> ephemeral Kubernetes build agent
-> Maven/Nx compile, tests, architecture and quality checks
-> Jib immutable JVM image -> ECR
-> automated GitOps environment change with image digest
-> environment-policy review/merge -> Argo CD sync -> Argo Rollouts -> EKS
```

Jenkins receives GitHub webhooks and uses ephemeral Kubernetes agents rather than long-lived mutable workers. Maven remains the independent Java build authority; Nx orchestrates cross-language/affected work once its guarded Maven integration can reproduce direct Maven targets. Jib Maven Plugin 3.5.2 builds routine JVM images. Jenkins never performs direct production `kubectl apply`.

## Desired state and promotion

Initially, GitOps is in the same repository. Desired state lives on `main` under environment directories `dev`, `stage`, and `prod`, represented by `deploy/`; environment branches are **Excluded**. An automated pipeline change references the immutable ECR image digest—not a mutable tag—and enters the review/merge policy for that environment. The exact approval and separation-of-duties policy is environment-owned and auditable.

Build once, promote the same digest. A production promotion cannot rebuild the source separately, silently change a base image, or substitute a tag. Rollback selects a previously recorded compatible digest/configuration; roll-forward is preferred when that is the safer recovery. A Git change and Argo CD reconciliation history must establish what desired version was requested, approved, synchronized, and healthy.

## Responsibilities and controls

| Stage | Owner and evidence |
| --- | --- |
| Source/trigger | GitHub records source revision and webhook; protected review policy applies by environment change. |
| CI controller | Jenkins 2.568.1 LTS uses Jenkins Configuration as Code, a versioned controller image, and versioned plugins. **Excluded:** Ansible and manual controller drift. |
| Build/test | Ephemeral agent runs direct Maven and Nx-relevant tasks, unit/module/integration suites, architecture boundaries, formatting, coverage reporting, and required compatibility checks. Failure is attached to revision evidence. |
| Image/publish | Jib builds an immutable image from pinned toolchains/base-image digests and publishes with least-privilege ECR access. The digest, SBOM/scanning/signing status, and source revision are recorded. |
| GitOps update | Automation creates a narrow environment desired-state change that references the published digest. It does not alter unrelated infrastructure, inject secret values, or directly deploy. |
| Reconciliation | Argo CD 3.4.6 reconciles Git to Kubernetes desired state. Terraform does not manage those same Kubernetes workload resources. |
| Release | Argo Rollouts 1.9.1 provides staged rollout controls and records rollout state; EKS runs the reconciled workload. |

Jenkins configuration, plugins, build toolchains, and base images are versioned/pinned. Dependency lockfiles apply to the pnpm/Nx toolchain and any selected frontend packages; Maven dependency management follows the documented BOM and compatibility policy. A tool/image update requires release/security-note review, relevant tests, and an intentional rollback path—not an unreviewed `latest` tag.

## Rollout progression and gates

Start with ordinary Kubernetes Deployment behavior and blue-green learning where it proves readiness, service routing, and rollback. Argo Rollouts can progress to canary only after APIs, event contracts, migrations, and consumers are backward/forward compatible and analysis signals can distinguish a healthy release from noise. A failed rollout pauses/aborts according to policy and preserves diagnostic evidence; it never treats merely scheduled pods as success.

Release gates are proportional: compile; targeted unit/module tests; integration tests with Testcontainers; architecture/format checks; contract/schema compatibility; image vulnerability/SBOM/signing policy at its adopted milestone; and environment-specific smoke/rollout evidence. Ordinary unit and integration suites make no external calls. Small real AWS smoke tests are explicit, access-controlled, cost-controlled, and separately labelled.

## Supply-chain and access policy

Milestones introduce SBOM generation, dependency/image scanning, signing, provenance retention, and admission/promotion policy when the system can enforce and operate them. They are not decorative dashboard checks. Publishing identities can publish only their allowed repository; GitOps automation can change only scoped environment paths; runtime Pods obtain secrets through Pod Identity/Secrets Manager rather than Jenkins. Secrets never enter console logs, build artifacts, Git, Helm values, or Terraform state inputs.

**Staged:** canary analysis, advanced signing/admission policy, GitHub Actions comparison, and fuller supply-chain controls. **Excluded:** Ansible, direct Jenkins production deployment, mutable release tags, environment branches, and rebuilding per environment.
