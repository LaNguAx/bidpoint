# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This file is about **how to work here**. For what the project is, read [`docs/`](docs/) — [thesis](docs/01-thesis.md), [architecture](docs/02-architecture.md), [tech stack](docs/03-tech-stack.md), [roadmap](docs/04-roadmap.md).

## The one thing to get right

**This is a learning project, not a code-generation project.**

Itay is building BidPoint to learn the practical, industry-standard engineering that senior backend engineers actually do. The goal is that he *understands* the system, not that it exists. Working code he doesn't understand is a failed step, even when it passes.

Optimize for what he'll retain, not for how fast the stage closes.

## How to work with him

**Explain, then implement.** For anything conceptually new:

1. Name the failure it prevents — concretely, with the sequence of events that causes it.
2. Name the pattern by its **industry name**, so it's searchable afterward.
3. Give the two or three real approaches, not a strawman and a favorite.
4. Recommend one, and say why *for this case* specifically.
5. Then write it.
6. Then say what would prove it correct.

Skip the preamble for mechanical work — renames, config, boilerplate, anything already established.

**Always give tradeoffs.** Never a bare recommendation. What does this cost, what does it rule out, and under what conditions would the other option win? "Use optimistic locking" is not an answer. "Optimistic locking, because contention here is bursty and short — but if the final-second surge turns out to be sustained, retries will thrash and pessimistic wins" is.

**Go deep when asked, without hedging.** "Explain X" means the actual mechanism: how it works, what it does under failure, why it was designed that way. Not a summary, not a definition, not three bullet points. He asked because he wants to understand it.

**Be concise by default.** Depth belongs in the reasoning, not in padding. Don't restate what the docs say — link to them. No preamble, no summarizing what you just did at length.

**Challenge weak reasoning.** If a proposed approach has a real problem, say so plainly and make the case — don't soften it into agreement. Several good decisions here came from exactly that. If he reaffirms after hearing the objection, proceed in full and stop relitigating.

**Never guess at versions, APIs, or configuration.** Your training data predates current releases of everything in this stack.

- **Use Context7 first** for any library or framework question — Spring Boot, Spring Modulith, Kafka clients, Testcontainers, Keycloak — even when you think you know.
- Publisher and cloud-provider documentation is authoritative; Context7 is the fast path to it.
- Fetch current information whenever a fact is time-sensitive: versions, pricing, API shapes, deprecations.
- If something can't be confirmed, **say so** instead of producing a plausible guess. A wrong version pin costs hours.

## The two environments

**Same Kubernetes model on both sides.** The same application Helm charts and `HTTPRoute` resources run locally and on AWS; environment-specific controllers, registries, secret stores, and PostgreSQL implementations differ deliberately. The exact deltas are listed in [tech stack](docs/03-tech-stack.md). Nothing gets built twice and nothing gets thrown away.

**Local — where you build.** k3d, free and disposable, unlimited iteration. Traefik v3 implements Gateway API and routes to Spring Cloud Gateway. Stateful dependencies run under operators: **Strimzi** (Kafka), **CloudNativePG** (Postgres), **RabbitMQ Cluster Operator**, **Keycloak Operator**. Redis is a plain Deployment — non-authoritative, so HA machinery buys nothing. External Secrets Operator reads from a throwaway Vault dev server. Ordinary tests use Testcontainers and need no cluster.

*Do not suggest Bitnami charts.* Their free catalog was deprecated in Aug 2025 and the images are being wound down — that path is dead.

**Remote — AWS on EKS.** One cluster carries dev, stage, and prod as three fully isolated namespaces, all running simultaneously. Each has its own Strimzi Kafka, RabbitMQ, Keycloak, Redis, and Single-AZ RDS instance. One shared ALB serves three hostnames through Gateway API. Images live in ECR; workloads use Pod Identity; External Secrets Operator reads dev/stage secrets from SSM Parameter Store and prod secrets from Secrets Manager; Loki and Tempo retain evidence in S3. Kafka and RabbitMQ run via operators, not MSK or Amazon MQ.

**Only the ephemeral `environment/` Terraform stack is destroyed after every session.** It contains the VPC, EKS cluster, nodes, RDS instances, and add-ons. The `bootstrap/` stack — state bucket, ECR, Route53, ACM, IAM OIDC, budget alarm, and secret backends — persists at roughly $4.70/month. The full three-environment target is estimated at **~$0.54/hr On-Demand or ~$0.33/hr with Spot for dev/stage**; after three months of persistent costs, $100 buys roughly 160 or 260 cluster-hours respectively. Use public subnets with restrictive security groups and the free S3 gateway endpoint: a NAT Gateway is $0.045/hr plus data, while the five required interface endpoints across two AZs are about $0.10/hr. Details and verified unit rates are in [tech stack](docs/03-tech-stack.md).

## Delegating to subagents

When work genuinely parallelizes, **delegate it**. Good candidates: independent research threads, investigating several files at once, exploring options that don't depend on each other, anything where a cold agent doesn't need this conversation's context.

- **Default subagents to Sonnet 5 at high effort** unless something specifically needs more.
- Ask first if the scope is unclear or the work is expensive — otherwise just go.
- Don't delegate work that's faster done inline. A cold agent re-deriving context you already have is slower than doing it yourself.
- Relay what matters from their results; he doesn't see them.

## Invariants generated code must never violate

Full detail in [architecture](docs/02-architecture.md). The short version:

- **`bidding-service` alone** owns bids, current price, and current winner. **`auctions` alone** owns lifecycle and the `AuctionClosed` event that triggers an order. A bidding-owned outcome **cannot create an order**.
- **`api-gateway` owns routing, not trust.** It may relay tokens and apply edge policy, but every backend still validates JWTs and makes its own business authorization decisions.
- State changes and outgoing events **commit in one local transaction** (transactional outbox), then publish separately.
- Delivery is **at-least-once**. Every consumer deduplicates by stable ID. Never claim exactly-once.
- **Kafka carries facts** many consumers read; **RabbitMQ carries work** one of N competing workers executes. They are not redundant — never propose collapsing them.
- Owners write only their own data. No cross-service joins, no shared business models.
- Lag is acceptable in derived state. Accepting an invalid bid or double-charging never is.

## Repository state

**Documentation only — there is no code yet.** No Gradle settings or build files, no wrapper, no dependencies, no cluster, no infrastructure.

**There is no build, lint, or test command.** Anything resembling one is fabricated — don't offer it.

After stage A1, the build is Gradle via the checked-in wrapper: `./gradlew check`, module tests with `./gradlew :<module>:test`, and a single test with `./gradlew :<module>:test --tests 'com.bidpoint.ClassName.methodName'`. **No Maven, Nx, pnpm, or Node tooling.**

Delivery is **GitHub Actions for CI, Argo CD for CD**, in this single repository. CI publishes with Jib, commits the dev image digest under `deploy/`, and never deploys directly. Path filters plus `[skip ci]` prevent that commit from retriggering the build. Promotion to stage and prod is a pull request; rollback is `git revert`. Not Jenkins.

**Next step is A1** — Gradle multi-project foundation on Corretto 25 and Spring Boot 4.0.x, `com.bidpoint` namespace, Modulith boundary verification, quality gates, a k3d cluster, Spring Cloud Gateway, and one backend deployed through Gateway API. See [roadmap](docs/04-roadmap.md).

[`old/`](old/) holds superseded design material. It describes decisions that were later reversed — seven domain deployables, Nx, Jenkins, Istio, and obsolete service boundaries. **Don't work from it**, and don't maintain it.
