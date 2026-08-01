# Decision delta: 1.0 to 2.0

Status: Canonical
Last validated: 2026-08-01
Amends: [1.0/21 Decision register](../1.0/21-decision-register.md)

This is the complete record of what 2.0 changes. Every 1.0 decision not listed here is **retained unchanged and remains Canonical**, including all of ADR-001 through ADR-018, ADR-020, ADR-021, ADR-023, ADR-026, ADR-027, ADR-029, and ADR-030.

Per 1.0's change discipline, each amendment identifies the affected owner, the boundary touched, the migration and rollback path, and what it turns a staged or excluded item into. Revisions use the `-R1` suffix against the original identifier so 1.0 remains traceable.

## Amendments to existing decisions

### ADR-019-R1 — Nx removed; Maven is the sole build authority

| Field | Record |
| --- | --- |
| **1.0 position** | ADR-019: BidPoint is a future Nx + Maven monorepo; `@nx/maven` guarded experimental; pnpm owns the Nx/Node toolchain. Canonical. |
| **2.0 decision** | Maven Wrapper and the Maven reactor are the only build authority. Nx, `@nx/maven`, pnpm, and Node as build tooling are **Excluded** until a frontend is selected. |
| **Status** | Canonical |
| **Rationale** | It fails the enforcement rule in [01](01-project-thesis.md): it maps to no learning outcome. It is an experimental plugin wrapping a build system that already works, in a repository with zero Node code, while the frontend remains Open. 1.0's own guard required every Nx target to document and prove a direct Maven equivalent — meaning both were maintained permanently. That guard is the evidence the tool does not pay for itself. |
| **Consequences** | Loses cross-language affected-task orchestration and caching, which a single-language repository does not need. The `pnpm nx local-up core\|platform` command contract from 1.0 is void; replaced by ordinary Maven commands plus a Compose file in Phase A and Helm/Argo in Phase B. Nothing else in the design depends on Nx. |
| **Migration and rollback** | Nothing to migrate; no code exists. Reintroduce scoped to a frontend package if one is selected, with the 1.0 guard reinstated. Maven layout is unaffected either way. |
| **Staged/Excluded impact** | Turns a Canonical component into an Excluded one. Reconsideration trigger recorded in [05](05-exclusions-and-open-questions.md). |
| **Affected documents** | [1.0/16](../1.0/16-monorepo-and-repository-structure.md) (root contract otherwise retained), [02](02-technology-stack.md), [03](03-delivery-roadmap.md) |

### ADR-025-R1 — Telemetry consolidated to one stack

| Field | Record |
| --- | --- |
| **1.0 position** | ADR-025: local OTel/Prometheus/Loki/Tempo/Grafana maps to remote ADOT/AMP/CloudWatch Logs/X-Ray/Managed Grafana. Canonical. |
| **2.0 decision** | OpenTelemetry Collector, Prometheus, Loki, Tempo, and Grafana run in **both** environments. ADOT handles AWS-side collection; CloudWatch Logs is retained because EKS logs there regardless. **AWS X-Ray is Excluded.** AMP and Amazon Managed Grafana are demoted to **Optional comparison**. |
| **Status** | Canonical |
| **Rationale** | 1.0 required building and operating two complete telemetry pipelines. The genuine lesson is that OpenTelemetry is a vendor-neutral seam, and that is demonstrated more directly by repointing one exporter than by maintaining a parallel stack. X-Ray specifically offers nothing Tempo does not, at additional integration cost. |
| **Consequences** | Loses hands-on operation of AWS-managed telemetry services, which is a real if minor employability signal — hence Optional rather than Excluded for AMP and Managed Grafana. Self-hosting observability on EKS carries its own operational cost, which is accepted as the cheaper of the two. Metrics, logs, and traces remain distinct signals; telemetry remains redacted, bounded, and non-authoritative. |
| **Migration and rollback** | Repointing OTel exporters is a configuration change, not an architectural one. Reverting to the 1.0 mapping requires no application change. |
| **Staged/Excluded impact** | Demotes two Canonical components to Optional comparison; excludes one. Promtail and the legacy X-Ray SDK/daemon remain Excluded as in 1.0. |
| **Affected documents** | [1.0/14](../1.0/14-observability.md), [02](02-technology-stack.md) |

### ADR-017-R1 — Kubernetes is a Phase B lesson, not a Phase A prerequisite

| Field | Record |
| --- | --- |
| **1.0 position** | ADR-017: the canonical local full environment is k3d/k3s with `core` and `platform` profiles; Docker Compose cannot be represented as canonical full-platform parity. Canonical. |
| **2.0 decision** | k3d/k3s with `core` and `platform` profiles **remains the canonical full local environment** and is unchanged as a target. It is required from **Phase B onward**. For Phase A, Testcontainers plus a narrow Docker Compose developer aid is Canonical. The Compose aid explicitly claims **no** platform parity and never replaces `core` or `platform`. |
| **Status** | Canonical |
| **Rationale** | 1.0's exclusion rationale — that the learning target requires Kubernetes controllers, routing, readiness, scaling, delivery, and telemetry boundaries — is entirely true for Phase B and irrelevant to writing a correct bid transaction. Requiring a full cluster before the first bid is placed inverts the thesis. Kubernetes is also better learned with real services to deploy into it. |
| **Consequences** | Two local environments exist during Phase A, with a genuine risk of Compose-only assumptions leaking into application code. Mitigated by B1 acceptance evidence requiring every service to run in-cluster and the Compose aid to be demonstrably unnecessary. The prohibition on claiming Compose as parity is retained verbatim from 1.0. |
| **Migration and rollback** | Compose is deleted or ignored once B1 passes. No application code should differ between environments; if it does, that is a defect surfaced by B1. |
| **Staged/Excluded impact** | Converts an Excluded item into a Canonical, phase-scoped one. 1.0's own reconsideration trigger — "a narrowly scoped developer aid is separately accepted without claiming parity or replacing `core`/`platform`" — is the trigger being invoked. |
| **Affected documents** | [1.0/11](../1.0/11-local-development-platform.md), [02](02-technology-stack.md), [03](03-delivery-roadmap.md), [05](05-exclusions-and-open-questions.md) |

### ADR-028-R1 — Framework pins move one minor behind; Boot compatibility gate closed

| Field | Record |
| --- | --- |
| **1.0 position** | ADR-028: exact direct pins at latest; Spring Boot 4.1.0 + Spring Cloud AWS 4.0.0 is an unconfirmed pairing held as an explicit Open gate. |
| **2.0 decision** | Exact pinning is retained. Selection policy is tiered per [02](02-technology-stack.md): behavioral components pin to current stable; **framework components pin one minor behind current** unless a required feature forces otherwise; tooling floats patches within a stable family; AWS managed families follow 1.0's verification policy unchanged. The Boot/Cloud AWS gate is **closed by aligning Boot down** to the version Spring Cloud AWS documents support for. |
| **Status** | Canonical |
| **Rationale** | Pinning is reproducibility policy; pinning everything to the newest release is a *selection* policy that costs debugging time in a currency unrelated to the learning outcomes. Settled releases have documentation, examples, and answered questions. 1.0 had already surfaced one incompatibility from this policy before any code existed. |
| **Consequences** | Forgoes newest framework features, none of which the design requires. Requires the exact Boot minor and Spring Cloud train to be verified against upstream release documentation at implementation time — 2.0 deliberately does not restate version numbers it cannot validate today, per 1.0's "never infer, never use `latest`" rule. |
| **Migration and rollback** | Moving forward a minor later is an ordinary reviewed upgrade under 1.0's existing compatibility/rollback policy. |
| **Staged/Excluded impact** | Closes one Open item. Spring Cloud AWS remains conditionally adoptable rather than gated on an unconfirmed pairing. |
| **Affected documents** | [1.0/19](../1.0/19-versions-and-compatibility.md), [1.0/23](../1.0/23-primary-sources.md), [02](02-technology-stack.md), [05](05-exclusions-and-open-questions.md) |

### ADR-022-R1 — Jenkins retained, with the reason recorded

| Field | Record |
| --- | --- |
| **1.0 position** | ADR-022: Jenkins with JCasC; builds and proposes, never deploys production; Argo CD reconciles. Ansible excluded. Canonical. |
| **2.0 decision** | **Retained without modification**, moved to Phase B, timeboxed, and never permitted to block Phase A. GitHub Actions remains an Optional comparison. |
| **Status** | Canonical |
| **Rationale** | Recorded explicitly because it is the one decision where the two goals in [01](01-project-thesis.md) diverge. On understanding alone Jenkins would be cut: the lesson it teaches — CI builds and proposes, GitOps reconciles — is available from GitHub Actions far more cheaply. It survives on employability: Jenkins is what large Java, banking, and insurance shops run, and it differentiates precisely because most candidates have only used hosted CI. |
| **Consequences** | A JCasC controller with ephemeral Kubernetes agents is a substantial Phase B workstream. Accepted deliberately. If Phase B stalls on Jenkins specifically, the recorded fallback is to take the GitHub Actions comparison as the baseline and note the substitution — this does not affect Argo CD's reconciliation authority either way. |
| **Migration and rollback** | Jenkins and GitHub Actions both produce an immutable digest and a narrow GitOps change; substituting one for the other does not touch ADR-021 or ADR-023. |
| **Staged/Excluded impact** | None. Status unchanged; only the rationale and phase placement are new. |
| **Affected documents** | [1.0/13](../1.0/13-ci-cd-and-gitops.md), [02](02-technology-stack.md), [03](03-delivery-roadmap.md) |

## New decisions

### ADR-031 — The project is two phases, each independently complete

| Field | Record |
| --- | --- |
| **Decision** | Delivery is split into Phase A (correctness) and Phase B (platform), per [03](03-delivery-roadmap.md). Each ends in a demonstrable artifact. Phase A is the thesis and succeeds on its own. |
| **Status** | Canonical |
| **Rationale** | 1.0 presented eleven stages as one continuous march with no defined point of sufficiency, which makes the project read as unfinishable and puts completion at risk. The two halves also answer genuinely different questions and fail differently. |
| **Consequences** | Phase B must not introduce domain behavior; if it does, that behavior belongs in Phase A. Phase A must not assume a cluster. Neither phase may be declared complete without its acceptance evidence. |
| **Affected documents** | [01](01-project-thesis.md), [03](03-delivery-roadmap.md) |

### ADR-032 — Acceptance evidence is the primary deliverable

| Field | Record |
| --- | --- |
| **Decision** | The artifacts named in the acceptance-evidence column of [03](03-delivery-roadmap.md) — concurrency proofs, crash-window tests, measured hot-partition results, reconciliation traces, k6 figures with stated conditions — are the project's output. Running code is necessary but not sufficient. |
| **Status** | Canonical |
| **Rationale** | Serves both stated goals with one mechanism. For understanding, measurement is what separates knowing the vocabulary from knowing the behavior. For employability, these artifacts are the interview answer; a system that works but cannot be shown to work under contention demonstrates neither. |
| **Consequences** | Stages are not complete when the feature runs. A component that produces no evidence has no defence when challenged under the [01](01-project-thesis.md) enforcement rule. Coverage remains diagnostic and is not evidence in this sense. |
| **Affected documents** | [01](01-project-thesis.md), [03](03-delivery-roadmap.md), [1.0/15](../1.0/15-testing-and-quality.md) |

### ADR-033 — AWS infrastructure is created and destroyed per session

| Field | Record |
| --- | --- |
| **Decision** | The AWS environment is never left running. Terraform layout, data seeding, and teardown are designed for repeated create/destroy cycles, and a demonstrated full cycle is B3 acceptance evidence. |
| **Status** | Canonical |
| **Rationale** | EKS, RDS, MSK, Amazon MQ, Redis Cloud, ALB, and managed telemetry running continuously is a several-hundred-USD monthly bill. Cost, not difficulty, is the most likely cause of abandonment. |
| **Consequences** | Constrains Terraform design from the start: no long-lived manual state, no undocumented console changes, seeding must be reproducible. Some managed services cannot be economically exercised at length; where that occurs, a recorded local substitute plus a scoped cost-controlled smoke test is the accepted evidence. |
| **Affected documents** | [02](02-technology-stack.md), [03](03-delivery-roadmap.md), [1.0/12](../1.0/12-aws-production-platform.md) |

### ADR-034 — The design corpus is closed

| Field | Record |
| --- | --- |
| **Decision** | 1.0 plus this revision is the complete design documentation. Further design documents are not produced during implementation; changes are recorded as amendments to this delta. |
| **Status** | Canonical |
| **Rationale** | Twenty-four canonical documents, a five-word status vocabulary, and a 634-directory placeholder tree already exist for a project with no code. Continued documentation of unwritten software is the most available form of procrastination here. |
| **Consequences** | Implementation notes, runbooks, and ADRs written *from* evidence remain welcome — they are outputs, not design. Documents 1.0/11–14 and 1.0/16–22 stay readable for background rationale where 2.0 does not contradict them. |
| **Affected documents** | This document, [2.0/README](README.md) |

## Change discipline

Unchanged from 1.0. Before amending a Canonical entry, a decision record must identify the affected owner, the data or contract boundary, the migration and rollback path, compatibility impact, operational evidence, and whether the change converts a staged or excluded item into a baseline component. An Open choice is a gate with criteria in [05](05-exclusions-and-open-questions.md), not permission to choose opportunistically. Evidence expectations remain those of [1.0/23](../1.0/23-primary-sources.md): official upstream and provider documentation, never search snippets or recollection.
