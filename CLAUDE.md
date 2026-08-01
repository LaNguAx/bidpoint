# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository state

**Documentation only. There is no code.** No Maven reactor, `pom.xml`, dependencies, manifests, container images, cluster, CI pipeline, or cloud infrastructure exists.

What physically exists: markdown under [project-discovery/](project-discovery/), and [project-discovery/structure-template/](project-discovery/structure-template/) — 634 directories and 788 files that are **all zero bytes**.

Nothing has been implemented. The next unstarted step is stage **A1, repository fitness** — a Maven reactor, the `com.bidpoint` namespace, module boundaries, and quality gates. See [2.0/03-delivery-roadmap.md](project-discovery/design-system/2.0/03-delivery-roadmap.md).

> **The structure template is a brainstorm, not a target.** It was sketched to think through what a layout *could* look like. It is not a commitment, not a specification, and the real project does not have to end up shaped like it. Do not treat it as a blueprint to satisfy, and do not preserve its structure for its own sake — when it disagrees with what the code actually needs, the code wins.
>
> Two further cautions: every file in it is empty, so names like `pom.xml`, `nx.json`, `package.json`, and `project.json` are placeholders rather than configuration; and the tree still reflects the Nx layout that ADR-019-R1 removed, so parts of it are known-stale. Never cite it as evidence anything is set up, and verify a file is non-empty before calling it initialized.

## Design baseline and precedence

| Location | Status |
| --- | --- |
| [design-system/2.0/](project-discovery/design-system/2.0/) | **Canonical.** 7 documents. |
| [design-system/1.0/](project-discovery/design-system/1.0/) | Superseded. 25 documents, kept as history. |

**Where they conflict, 2.0 wins.** Conflicts are concentrated in 1.0's documents 11–14 and 16–22 (platform, delivery, observability, monorepo, stacks, versions, roadmap, decisions) — 1.0 specifies Nx and a duplicated AWS telemetry stack, both removed.

**2.0 deliberately does not restate** 1.0's `02`–`10`, `15`, and `23`. Those remain the canonical contract for domain, ownership, messaging, security, testing, and evidence standards — read them directly rather than assuming 2.0 covers them.

Start at [2.0/99-handoff.md](project-discovery/design-system/2.0/99-handoff.md) for orientation, then [2.0/04-decision-delta.md](project-discovery/design-system/2.0/04-decision-delta.md) for what changed from 1.0 and why.

## Architecture essentials

Spring Modulith `core-platform` modular monolith — modules `profiles`, `catalog`, `listings`, `auctions`, `orders`, plus `payment` with a fake adapter — alongside three services: `bidding-service`, `notification-service`, `notification-worker`. **Four deployables total** (ADR-038). `api-gateway`, `payment-service`, `realtime-service`, and `search-service` were removed; SSE is served from multiple `core-platform` replicas. Background in [1.0/03](project-discovery/design-system/1.0/03-architecture-overview.md), [1.0/04](project-discovery/design-system/1.0/04-core-platform-modular-monolith.md), [1.0/05](project-discovery/design-system/1.0/05-microservice-boundaries.md).

These invariants are the point of the project. Generated code may never violate them:

- **`bidding-service` alone** owns bids, current price, and current winner. **`auctions` alone** owns lifecycle and the `AuctionClosed` order trigger. A bidding-owned outcome is audit input and **cannot create an order**.
- Close sequence: `CLOSING` → idempotent Bidding fence/finalize REST → atomic commit of `CLOSED` **plus** one `AuctionClosed`. The REST acknowledgement is *not* the order trigger.
- Every owner writes only its own data. No cross-service joins, no shared business-domain models.
- Producers use transactional outboxes; consumers deduplicate by stable ID. Delivery is at-least-once — **never claim exactly-once**.
- Kafka carries durable replayable facts. RabbitMQ carries targeted work. Redis is never authoritative. SSE is display-only, never command authority.
- Lag is acceptable in search, notifications, and SSE. Accepting an invalid bid or charging twice is not.

**The two-broker split is the point, not redundancy.** Kafka carries durable, replayable, per-key-ordered facts many consumers read. RabbitMQ carries one work obligation that exactly one of N competing workers executes. The canonical chain is `bidding-service` —Kafka fact→ `notification-service` —RabbitMQ job→ `notification-worker` at N replicas. Never propose collapsing them into one broker.

## Commands

**No build, lint, or test command exists.** Any command resembling one is fabricated — do not offer it.

Works today, from the repo root:

```bash
# Verify every relative markdown link resolves. Run from the ROOT —
# starting inside project-discovery/ misses the root README.
for f in $(find . -name '*.md' -not -path './.git/*' -not -path './project-discovery/structure-template/*'); do
  d=$(dirname "$f"); grep -oE '\]\([^)#]+\)' "$f" | sed -E 's/^..//; s/.$//' | while read -r l; do
    case "$l" in http*) continue;; esac; [ -e "$d/$l" ] || echo "BROKEN: $f -> $l"; done; done
```

After stage A1 exists, the build is **plain Maven via wrapper** — `./mvnw verify`, `./mvnw -pl <module> test`, single test `./mvnw test -Dtest=ClassName#method`. Not present yet. A1 also brings up a k3d cluster; ordinary tests use Testcontainers and need no cluster.

**No Nx, no pnpm, no `nx` targets.** ADR-019-R1 made Maven the sole build authority. 1.0's documents and the structure template both still reference Nx; both are stale on this point.

## Working style

This is a learning project. The stated goals are employability as a backend engineer and genuinely understanding how distributed systems fail. Code delivered without the reasoning behind it defeats the purpose.

**Explain, then implement.** Before writing code for a concept that is new here:

1. Name the failure it prevents, concretely.
2. Name the pattern by its industry name, so it is searchable afterward.
3. Give the two or three real approaches with trade-offs.
4. Recommend one and say why *for this case*.
5. Then write it, then say what would prove it correct.

Skip the preamble for mechanical work — renames, config, boilerplate, anything already established.

**Challenge reasoning rather than agreeing.** When a proposed approach is weak, say so plainly and make the case. Nx and the duplicated telemetry stack were removed from the baseline exactly this way. If a decision is reaffirmed after the objection, proceed with it in full and stop relitigating.

**Evidence is the deliverable** (ADR-032). A stage is not done when the feature runs. It is done when the concurrency proof, crash-window test, measurement, or reconciliation trace exists. Prefer teaching through a failing test over prose.

Keep answers tight. Depth belongs in the trade-off reasoning, not in restating what the design documents already say — link to them instead.

## Versions and currency

The training cutoff predates current releases of everything in this stack. **Use Context7 for any library or framework question** — Spring Boot, Spring Modulith, Spring for Kafka, Testcontainers, Keycloak — before answering from memory, even when the answer feels known.

Context7 is a discovery aid. **Publisher and cloud-provider documentation is authoritative** ([1.0/23](project-discovery/design-system/1.0/23-primary-sources.md)).

Pin policy (ADR-028-R1):

| Tier | Policy |
| --- | --- |
| Behavioral — PostgreSQL, Kafka, RabbitMQ, Redis, Keycloak, Kubernetes | Current stable. Their behavior is the lesson. |
| Framework — Spring Boot, Spring Cloud, Spring Modulith | **One minor behind current.** Settled releases have answered questions. |
| Tooling — Maven, Jib, Spotless, ArchUnit, Helm | Float patches within a stable family. |
| AWS managed families | Verify region/account availability, add-on compatibility, and quotas at implementation. |

**Never use `latest`. Never infer a compatible version pair from a general project page.** If a pairing cannot be confirmed, say so — that exact failure produced 1.0's unresolved Boot / Spring Cloud AWS gate. Version numbers in [1.0/19](project-discovery/design-system/1.0/19-versions-and-compatibility.md) are a snapshot to re-validate, not a contract.

## Constraints that bind proposals

- **Every component must map to a learning outcome** in [2.0/01](project-discovery/design-system/2.0/01-project-thesis.md). One that cannot is removed, not staged. This rule removed Nx and X-Ray.
- **Do not write new design documents** (ADR-034). The corpus is closed. Record changes as amendments in [2.0/04-decision-delta.md](project-discovery/design-system/2.0/04-decision-delta.md). Runbooks and notes written *from* evidence are outputs, not design.
- **Phase A before Phase B.** Phase A is A1–A9 (correctness), running on **local k3d from stage one** — there is no Compose phase (ADR-037), and proposing one is a regression. A4 is an early AWS tracer bullet: one service on ECS Fargate, then destroyed. Phase B is CI/CD, full AWS deployment, and one bounded EKS exercise. See [2.0/03](project-discovery/design-system/2.0/03-delivery-roadmap.md).
- **The stack was deliberately cut to ~20 tools** (ADR-035 through ADR-039) by asking of each one: *is a backend engineer ever actually asked about this, or is it platform/SRE tooling?* Removed, and **not to be reintroduced without naming the learning outcome first**: Istio, KEDA, Argo Rollouts, Argo CD, Jenkins, Strimzi and the RabbitMQ operator, Spring Cloud Gateway, Loki, Tempo, MSK, Amazon MQ, Redis Cloud, AMP, Managed Grafana, X-Ray, OpenSearch, CloudFront.
- **ECS Fargate is the AWS target, not EKS.** An EKS control plane is ~$73/month against **$100 total credits**. Kubernetes depth comes from free local k3d; EKS is one bounded exercise at B3. Watch the NAT Gateway (~$32/month) — it costs more than the database and is the classic silent credit-killer.
- **AWS is created and destroyed per session** (ADR-033), never left running. Cost is the main abandonment risk.
- Changing a Canonical decision requires stating consequences **first**: affected owner, contract and data boundary, migration, compatibility, rollback.
- Open questions ([2.0/05](project-discovery/design-system/2.0/05-exclusions-and-open-questions.md)) are gates with criteria, not invitations to decide opportunistically. A fake adapter unblocks the payment and notification stages.
