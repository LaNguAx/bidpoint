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

**Local — where you build.** k3d runs a real Kubernetes cluster on the machine, free and disposable. PostgreSQL, Kafka, RabbitMQ, Redis, and Keycloak install from community Helm charts. Iteration is unlimited, so this is where Kubernetes depth is built. Ordinary tests use Testcontainers and need no cluster at all.

**Remote — AWS, where you prove it ships.** ECS Fargate, not EKS: an EKS control plane is ~$73/month against **$100 in total credits**. Terraform, ECR, RDS, S3, IAM, Secrets Manager, CloudWatch, ALB. Kafka and RabbitMQ run as containers, not MSK/Amazon MQ.

**AWS is created and destroyed every session — never left running.** Watch the NAT Gateway (~$32/month, more than the database). Set a billing alarm before the first `terraform apply`. Details in [tech stack](docs/03-tech-stack.md).

## Delegating to subagents

When work genuinely parallelizes, **delegate it**. Good candidates: independent research threads, investigating several files at once, exploring options that don't depend on each other, anything where a cold agent doesn't need this conversation's context.

- **Default subagents to Sonnet 5 at high effort** unless something specifically needs more.
- Ask first if the scope is unclear or the work is expensive — otherwise just go.
- Don't delegate work that's faster done inline. A cold agent re-deriving context you already have is slower than doing it yourself.
- Relay what matters from their results; he doesn't see them.

## Invariants generated code must never violate

Full detail in [architecture](docs/02-architecture.md). The short version:

- **`bidding-service` alone** owns bids, current price, and current winner. **`auctions` alone** owns lifecycle and the `AuctionClosed` event that triggers an order. A bidding-owned outcome **cannot create an order**.
- State changes and outgoing events **commit in one local transaction** (transactional outbox), then publish separately.
- Delivery is **at-least-once**. Every consumer deduplicates by stable ID. Never claim exactly-once.
- **Kafka carries facts** many consumers read; **RabbitMQ carries work** one of N competing workers executes. They are not redundant — never propose collapsing them.
- Owners write only their own data. No cross-service joins, no shared business models.
- Lag is acceptable in derived state. Accepting an invalid bid or double-charging never is.

## Repository state

**Documentation only — there is no code yet.** No Maven reactor, no `pom.xml`, no dependencies, no cluster, no infrastructure.

**There is no build, lint, or test command.** Anything resembling one is fabricated — don't offer it.

After stage A1, the build is plain Maven via wrapper: `./mvnw verify`, `./mvnw -pl <module> test`, single test `./mvnw test -Dtest=ClassName#method`. **No Nx, no pnpm, no Node tooling.**

**Next step is A1** — Maven reactor, `com.bidpoint` namespace, module boundaries, quality gates, k3d cluster, one service deployed. See [roadmap](docs/04-roadmap.md).

[`old/`](old/) holds superseded design material. It describes decisions that were later reversed — seven deployables, Nx, Jenkins, Istio, EKS-as-primary. **Don't work from it**, and don't maintain it.
